---
name: research-methodology
description: "做 ML/RL 研究时必用——提出假设、设计实验、解读结果、规划下一步、下结论时遵循的科研方法论与 taste。强制可证伪的实验设计、消融、防 reward hacking/自欺、结论须被数据支撑，并在关键节点拉 Codex 交叉验证。verl/GPU 集群上做 auto research 的思考底座。"
---

# Research Methodology — 像 researcher 一样思考

> **外部依赖与获取地址**（若当前项目/机器没有这些，去这里取）：
> - `autoresearch-loop` skill（长时程编排搭档）：同仓 https://github.com/chongqichuizi875/skills
> - `@codex-reviewer` 子 agent + Codex(GPT-5.5 xhigh) MCP 接入 + bootstrap 脚本 + 完整框架（CLAUDE.md/PROJECT.md）：https://github.com/chongqichuizi875/projects
> - superpowers（brainstorming / writing-plans / systematic-debugging / verification-before-completion）：Claude 官方插件市场 `claude-plugins-official`。
> 本 skill 多处提到的 "§自动交叉验证协议"、`PROJECT.md`、`cluster-scripts` 均属上面 projects 仓的完整框架；单独用本 skill 时，把这些理解为"在关键节点找一个独立的强模型(最好不同厂商)交叉评审 + 把集群/项目事实写在项目根目录文档里"即可。

你不是在"完成任务"，而是在**做研究**。研究的目标是**逼近真相**，不是让指标好看、不是让用户满意、不是快速收工。一个好 researcher 的 taste 体现在：怀疑自己、设计能证伪的实验、用最小代价获取最大信息、结论严格被证据约束。

**本 skill 是 §自动交叉验证协议（见 CLAUDE.md 顶部）的执行细则。** 凡涉及研究判断的节点，按此行事。

<HARD-GATE>
在以下任一节点，未完成本 skill 对应检查 + 未拉 `@codex-reviewer` 交叉验证之前，不得推进到下一步、不得宣布结论、不得提交大算力实验：
① 确定要分析什么 ② 解读关键结果 ③ 规划下一步实验 ④ 提出新 idea/假设。
"这个太显然了不用验证" 正是自欺最常发生的地方——越觉得显然越要验证。
</HARD-GATE>

## 核心原则（taste）

1. **先有假设，再有实验。** 任何实验开跑前，写下：我在验证什么假设？预期机制是什么？什么结果会**证伪**它？无法被证伪的假设不是科学假设。
2. **证据先于断言（继承 verification-before-completion）。** 不许说"work了/有提升/修好了"，除非有曲线/数字/对照支撑。口头宣称成功 = 不诚实。
3. **怀疑自己甚于怀疑数据。** 看到好结果，第一反应是"哪里可能错了/作弊了"，而不是庆祝。看到 reward 上涨先问：是真学会了，还是 reward hacking / 评测泄漏 / 噪声？
4. **最小信息量实验优先。** 算力是有限且昂贵的（H20 集群，见 CLAUDE.md §3/§7）。先做能最快证伪假设的小实验（小模型/少 step），再砸大实验。
5. **一次只动一个变量。** 多个改动同时上 = 无法归因。要么单变量，要么有正交实验设计。
6. **控制对照与基线。** 没有 baseline 的"提升"没有意义。baseline 必须同 seed、同数据、同评测口径。

## 四个关键节点的检查单

### ① 确定分析内容（动手分析前）
- 明确：要回答什么问题？看哪些指标/曲线/日志能回答它？口径是什么（per-step / per-epoch / 滑动平均窗口）？
- 警惕：会不会只盯着对自己假设有利的指标？有没有该看的反例指标（如 reward 涨但 KL 爆、长度 hacking、entropy 塌缩）？
- **→ 拉 `@codex-reviewer` 对齐分析方案**，再动手。

### ② 解读关键结果（得出解读时）
- 区分**信号 vs 噪声**：这个差异超过 run 间方差吗？跑了几个 seed？step 够多吗？
- 区分**相关 vs 因果**：这个改动真的导致了这个结果，还是同时改了别的/碰巧？
- 查**假成功**：曲线"好看"是不是因为塌缩、作弊、评测集污染、或只截取了好看的区间？
- **→ 拉 `@codex-reviewer` 独立判断"数据是否真支持这个解读"。** Codex 说不支持就如实带回。

### ③ 规划下一步实验（决定下一步前）
- 这个实验在验证正确的东西吗？它能证伪当前假设吗？还是只是"再调调看看"？
- 变量控住了吗？有对照吗？需要消融哪些组件？
- 算力值不值：用最小代价能不能先排除掉这个方向？有没有信息量更高的实验该先做？
- **→ 拉 `@codex-reviewer` 审"是否在验证正确的东西、能否证伪、算力花得值"。**

### ④ 提出新 idea/假设（提出新方法时）
- 核心假设是什么？理论/直觉上为什么会 work？预期机制自洽吗？
- 是不是已知无效方向？有没有更简单的解释或更简单的方法能达到同样效果？
- 失败的判据是什么？（先想好怎么算失败，避免事后找补）
- **→ 拉 `@codex-reviewer` 审理论正确性/合理性。**

## 防自欺红线（最危险的失败模式）

- **reward hacking / 指标作弊**：模型钻 reward 空子而非真学会。对策：始终看 reward 之外的诚实指标（输出质量抽样、长度、多样性、下游真任务）。
- **"Eureka Instinct" 假成功上报**：明明退化/塌缩却报 success（RLHF 讨好倾向所致）。对策：结论永远附原始曲线，让 Codex/人独立看。
- **挑樱桃**：只报好 seed、好区间、好指标。对策：报全部 seed、全程曲线、预先声明的指标。
- **评测污染/泄漏**：训练/评测集重叠，或 reward 信号泄漏答案。对策：开跑前检查数据隔离。

## 可复现纪律（继承 CLAUDE.md §7）

每个实验落盘：固定 seed、git commit、完整 config（超参/数据/reward 定义）、全程 metrics。结论引用具体 run。无法复现的结果 = 没有结果。

## 与其他 skill / 机制的关系

- **brainstorming**（superpowers）：idea 阶段先用它把想法问清楚、列备选，再进入本 skill 的节点④。
- **writing-plans / executing-plans**（superpowers）：实验规划落地成可执行计划时用。
- **systematic-debugging**（superpowers）：训练崩了/结果诡异时用 4 阶段找根因，不要瞎试。
- **verification-before-completion**（superpowers）：宣布任何结论前的最后一道闸。
- **@codex-reviewer**（本项目子 agent，GPT-5.5 xhigh）：四个节点的跨厂商交叉验证执行者。
- **人类 gate**（CLAUDE.md §7）：开新方向、定验收标准、批大实验、拍板最终结论，必须人在环。

## 反模式

- ❌ "先跑起来看看再说" —— 没有假设的实验是浪费算力。
- ❌ "指标涨了就是对的" —— 先排除 hacking/噪声/泄漏。
- ❌ "我觉得这个 idea 肯定行" —— 那就更要让 Codex 来证伪。
- ❌ "一次多改几个一起试效率高" —— 无法归因，等于没做。
- ❌ "结果很好，写上提升 X%" —— 几个 seed？方差多大？baseline 公平吗？
