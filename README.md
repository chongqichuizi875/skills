# skills

A collection of personal [Claude Code](https://docs.claude.com/en/docs/claude-code) skills.

## What's a skill?

A skill is a folder with a `SKILL.md` (YAML frontmatter + instructions) that Claude Code can
load on demand to perform a specialized task. See the
[skills docs](https://docs.claude.com/en/docs/claude-code/skills) for details.

## Install

Clone into your user-level skills directory so the skills are available in every project:

```bash
git clone https://github.com/chongqichuizi875/skills.git /tmp/skills
mkdir -p ~/.claude/skills
cp -R /tmp/skills/*/ ~/.claude/skills/
```

(Or symlink individual skill folders into `~/.claude/skills/`.)

## Skills

### `research-methodology`

像 researcher 一样思考的方法论与 taste（**判断层**）：先假设后实验、证据先于断言、怀疑自己甚于数据、
最小信息量实验优先、一次只动一个变量、必须有对照基线；提出 idea / 解读结果 / 规划实验 / 下结论四个
关键节点的检查单；防自欺红线（reward hacking / 假成功 / 挑樱桃 / 评测污染）。强制在关键节点拉一个
独立的强模型（最好不同厂商）做交叉验证。面向 ML/RL 研究，但 taste 通用。

### `autoresearch-loop`

长时程（数小时~数天）无人值守自动 research 的**编排层**协议（改编自 Deli_AutoResearch 开放协议，
victorchen96.github.io/auto_research，并以跨厂商外部评审替代其框架内自评）。提供：状态全落盘 + 每轮
fresh session（防认知死循环）、停滞检测与强制 pivot、方向多样性、实验提交后分钟级轮询、轻量两层心跳
看门狗、五条无人值守行为约束。与 `research-methodology` 搭配：前者管"怎么不死地跑数天"，后者管
"每一步对不对"。需要 GPU 集群跑 RL/ML 长实验、Claude Code 的 `/loop` 与 Agent tool。

> 这两个 skill 与一套更完整的研究框架（`@codex-reviewer` 跨厂商交叉评审子 agent、Codex MCP 接入、
> bootstrap 脚本、CLAUDE.md/PROJECT.md 分层文档）配套，完整框架在
> **https://github.com/chongqichuizi875/projects** 。单独使用本仓两个 skill 也可以，缺失的资源按各
> SKILL.md 顶部"外部依赖与获取地址"去取。

### `infra-verification`

写系统/基础设施代码（GPU runtime、checkpoint/restore、分布式通信、存储 I/O）时判断**「测够了没有」**
的方法论。核心论点：端到端跑通只证明「这台机器上、一切顺利时」是对的，证明不了三类东西——
**没被执行的分支**（→ 静态类型检查，量 delta 不量绝对值）、**只在失败时走的分支**
（→ 错误注入单元测试，不走全链路）、**换个环境才走到的分支**（→ 环境能力探测 + 显式 skip）。
另含 A/B 性能测量口径（同起点复位、冷 L2、对 oracle 比而非两个近似互比、形状分布加权、
先断言有限性再比较）与「结论必须带边界、负面结果照写」。
附 [postmortem.md](infra-verification/postmortem.md)：一次真实 PR 的复盘——337 个测试全绿、lint 干净，
审阅者仍找出三个真 bug，逐个拆解它们分别属于哪一类、为什么结构性地碰不到。

### `gpu-box-ops`

在共享多卡 GPU 机器上做 kernel / 系统级 infra 的**工程实践**：占卡 lease 礼仪与还卡
（`flock ... sleep infinity` 会留孤儿，要杀进程组）、跑完清理残留进程与 /dev/shm、
远程迭代回路（base64 绕开多层引号转义、`git ls-files | tar` 增量同步、后台跑 + 轮询日志、
`trap` 保证还卡），以及只有在真机上才会撞到的坑：CRIU 版本门禁与 rseq / `//deleted` 挂载 /
必须把 criu 自己的 stderr 带出来、NCCL 在独立 netns 里的三个环境变量、
容器里 torch 没装 numpy、国内源与代理（`Proxy CONNECT aborted` 时先试直连）。

### `setup-latex-vscode-mac`

Set up a local LaTeX authoring environment on macOS that mirrors the Overleaf experience —
edit a `.tex` project in VSCode, auto-recompile on save, and side-by-side PDF preview.
Installs the TeX distribution + the LaTeX Workshop extension, writes the per-project build
config, and pre-empts the common macOS gotchas (GUI apps not inheriting `PATH`, Gatekeeper
quarantine, broken webview preview). Handles Chinese / `ctex` documents (XeLaTeX). Ships a
`settings.template.json` you adapt per project.
