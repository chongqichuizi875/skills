---
name: autoresearch-loop
description: "长时程(数小时~数天)无人值守自动 research 的编排协议。当用户要让 Claude 自主跑一个跨多轮、需提交 GPU 训练/长实验、并持续迭代的 research 任务时使用。提供：状态全落盘 + 每轮 fresh session 防认知死循环、停滞检测与强制 pivot、方向多样性、实验编排轮询、轻量心跳看门狗，并在关键节点用 @codex-reviewer 跨厂商交叉验证。"
---

# AutoResearch Loop — 长时程无人值守编排协议

> **外部依赖与获取地址**（本 skill 与下列资源配套，若当前项目/机器没有，去这里取）：
> - `research-methodology` skill（判断层搭档）：同仓 https://github.com/chongqichuizi875/skills
> - `@codex-reviewer` 子 agent + Codex MCP 接入 + bootstrap 脚本 + 完整方法论框架（CLAUDE.md/PROJECT.md/cluster-scripts）：https://github.com/chongqichuizi875/projects
> - `/loop`、Agent tool、durable cron：Claude Code 内置能力（见 https://docs.claude.com/en/docs/claude-code ）。
> - superpowers 等方法论插件：Claude 官方插件市场 `claude-plugins-official`（`<claude-bin> plugin install <name>@claude-plugins-official`）。
> 缺 `@codex-reviewer` 时可临时退化为"本模型自评 + 人类 gate"，但会失去跨厂商外部验证的核心价值——优先配好（见 projects 仓 CLAUDE.md §5）。

> 本 skill 是**编排层**（怎么不死地跑数天），与 `research-methodology` skill（**判断层**：每一步对不对、防自欺）分工协作。
> 改编自 Deli_AutoResearch 开放协议（victorchen96.github.io/auto_research），**关键改造**：把原协议的"框架内五人格自评"替换为本项目已配的 **Codex(GPT-5.5 xhigh) 跨厂商外部评审**（`@codex-reviewer`）——外部交叉验证强于自评。看门狗按用户选择采用**轻量两层(L1+L2)**。

## 0. 这个协议解决什么（三大失败模式）

长时程 code agent 反复出现三类失败，根因是**工程脚手架缺失**，不是模型能力不足：
1. **认知死循环**：连续几轮试相似方向、收益递减，自己跳不出局部最优。
2. **停滞**：干完一块就输出总结、等用户反馈；外表 session 还活着、轮询还在跑，实际工作已停。日志显示这比崩溃更常见。
3. **运行时脆弱**：上下文压缩静默打断循环；关 session 会带走寄生其上的定时器；失败默认无人察觉。

本协议每一条机制都针对上述失败模式。

## 1. 五条行为约束（无人值守的硬纪律）

1. **Zero interaction**：运行中不打断用户——不进 Plan Mode、不用提问工具、不以问题结尾。持续工作直到用户喊停。歧义自己解决并写入日志（`level=decision`）。
   - ⚠️ **例外**：本项目 §自动交叉验证协议的四个研究判断节点 + 人类 gate（开新方向/定验收/批大算力/拍最终结论）仍要走——但走法是**拉 Codex 评审、把分歧记日志并按规则升级**，不是停下来等用户。重大分歧才升级人类。
2. **Ready means execute**：最常见的隐性违规是"准备好了再问要不要提交"。准备的目的就是执行；提交、重交、修复、起监控都是无需确认的例行操作。
3. **Callback means report-alive**：上下文压缩后循环会静默死。每次回调第一件事是更新自己的 `last_seen`，再检查存活；发现失败立即重启并记录。
4. **Persist state to files**：所有进展写入 `state/` 文件，不靠对话记忆。**每轮开 fresh session，只注入精选状态，绝不用 resume。**
5. **Guardian / worker separation**：心跳巡逻对非自己的任务只能做三件事——查活、重启、nudge；不读其数据、不改其状态文件、不替它向用户汇报。

## 2. 架构

```
┌── Orchestrator (当前 session / durable cron) ──┐
│ 读状态文件 → 检测停滞 → 注入新方向          │
└────┬──────────────┬──────────────┬────────┘
  [Task A]       [Task B]       [Task C]   ← 各自独立 fresh session
```

核心决策：
- **执行与评估分离**：干活的 agent 不评判自己的进展；停滞判定由编排层基于**量化指标**做。研究判断的对错由 **`@codex-reviewer`（Codex 外部）** 把关。
- **fresh session 优于 resume**：上下文累积是认知死循环的首因。每轮全新上下文，状态经文件注入。
- **方向多样性强制**：每轮开工前读 `directions_tried.json`，新方向必须不同于所有历史。

## 3. 状态文件（落盘，跟实验产物一起放 ceph，见 PROJECT.md §3）

```
{task}/state/
├── task_spec.md            # 目标 / 里程碑 / 验收标准（开跑前由人类 + Codex 共同定）
├── progress.json           # {iteration, total_findings, status, stale_count}
├── findings.jsonl          # 累积发现（append-only）
├── directions_tried.json   # 已试方向（保证多样性）
├── iteration_log.jsonl     # 每轮小结
└── experiments/            # 每个实验一份：config(seed/git commit/超参) + metrics.jsonl + decision.md
{task}/logs/
├── work.jsonl              # work agent 决策（level=decision）
├── orchestrator.jsonl      # 编排事件
└── heartbeat.jsonl         # 看门狗事件
```
日志行格式：`{"ts":"...","source":"...","level":"info|warn|error|decision","event":"...","detail":"..."}`

## 4. 启动方式（本 harness 原生能力）

```
# 1. 初始化任务目录：写 state/task_spec.md（含可验收标准）+ 初始 progress.json
#    task_spec 的验收标准开跑前先过一遍 @codex-reviewer + 人类 gate。

# 2. 起编排循环（/loop 是本 harness 内置 skill）：
/loop 2h 巡检 {task}：(1) 读 progress.json；(2) 若 stale_count>=2 生成全新方向(对照 directions_tried.json)；
(3) 经 Agent tool 起一个 work agent（给明确目标+完成判据+工作目录+文件/行数上限）；
(4) 结果写回状态文件。Zero interaction。

# 3. 注册 durable 心跳看门狗（L1，跨 session 存活）：
durable cron 每小时巡逻：写时间戳；比对各 loop 的 last_seen 与 interval×3，超时则重启；
检查各任务 progress 是否 >2h 无更新，停滞则 nudge。Zero interaction。
```

## 5. 停滞检测与 pivot

| 机制 | 规则 |
|------|------|
| 停滞检测 | 某轮 0 新发现 或 指标回退 → `stale_count + 1` |
| 强制 pivot | `stale_count >= 2` → 改**结构性约束**(不是调参)；**`>= 4` → 升级人类** |
| 方向多样性 | 新方向必须不同于每个已试方向；停滞后注入扰动策略 |
| 单轮上限 | 单个 work session 上限 15 轮 或 30 分钟 |

> "改结构不改战术"：任务在一个框架内反复停滞时，决定性收益通常来自修正环境/结构约束本身，而非在原框架内更狠地调参。这一步是 §自动交叉验证协议的"实验规划/idea"节点——**先拉 @codex-reviewer 审新结构方向再投**。

## 6. 心跳看门狗（轻量两层 L1+L2）

| 层 | 形式 | 依赖 | 职责 |
|----|------|------|------|
| **L1** | durable cron，每小时 | 一个活着的交互 session | 查各 loop 的 last_seen、重启超时 loop、检测停滞并 nudge |
| **L2** | 业务循环 | 各自 session | 每次回调第一行更新自己的 last_seen |

任一层死掉可被另一层发现恢复。停滞检测：progress >2h 无更新且最后输出是问句 → 判停滞，起 nudge 子 agent；连续 3 次 nudge 无进展 → 判结构卡死，停止 nudge、换新方向重开。
> L0（常驻 shell guard，不依赖 session）暂不启用；真要 72h+ 无人值守再加。

## 7. 子 agent 调度模式

| 模式 | 用于 | 关键 |
|------|------|------|
| A 目标驱动 | research 迭代 | 注入已试方向、要求可验证发现、写回 findings.jsonl |
| B 并行探索 | 复杂子问题 | 一条消息内并发多个：调查 + 反驳 + 跨域类比 |
| C 实验运行 | 长算力作业 | 提交后立即**分钟级轮询**：自动诊断报错、修复、重交（正合 verl 多机训练，见 PROJECT.md §6/§7） |
| **D 验证(本项目升级)** | 每轮 QA | **用 `@codex-reviewer`（Codex 外部）审证据链**，替代原协议的框架内自评人格 |

子 agent prompt 必含：背景、可验证交付物、工作目录、文件/行数上限、完成判据。

## 8. 工程约束

1. 每轮最多 5 个大文件；单文件不超 300 行。
2. 状态经文件注入，不靠对话历史。
3. 每轮之间必须跑验证（test/compile/check）。
4. 引用类内容每 20 条核验一次，绝不攒着批量做。
5. 多个候选方向时，优先加多样性而非深挖单一方向。
6. 不可解的外部依赖失败要升级（完整报告 + 通知 owner + 轮询回复），绝不静默放弃。

## 9. 与本项目其他机制的关系

- **research-methodology skill**：判断层。本 skill 每个研究判断节点（确定分析/解读/规划/idea）的**对错标准与防自欺红线**走它。
- **@codex-reviewer（Codex GPT-5.5 xhigh）**：所有外部评审/D 模式验证的执行者，替代原协议自评。
- **superpowers**：brainstorming（定 task_spec/idea）、writing-plans（拆实验）、systematic-debugging（实验崩了找根因）。
- **PROJECT.md**：状态文件落 ceph、实验提交走 cluster-scripts、可复现纪律。
- **人类 gate**：开新方向、定 task_spec 验收标准、批大算力、拍最终结论。stale_count≥4 自动升级到这里。

## 10. 诚实的局限（采纳前必读）

1. 原协议自评分（8.0-8.6）来自**框架内模拟评审**，非外部质量保证——本项目已用 Codex 外部评审替代，但 Codex 也非真值，最终仍需人类 gate。
2. **LLM 仍会编造引用/数据**：协议把外部核验变成流程里的机械步骤，**不消除错误源**。结论必附原始曲线/数据。
3. 职责分离靠**协议约束、非模型自律**：去掉约束越界行为会回来——所以这套约束要写在 skill 里持续生效。
4. 在场记录的最长连续运行 72h（其间 6 次方向性人类输入，零操作干预）。**不要假设能完全无人**——方向性人类介入应保留。
