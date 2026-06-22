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

### `setup-latex-vscode-mac`

Set up a local LaTeX authoring environment on macOS that mirrors the Overleaf experience —
edit a `.tex` project in VSCode, auto-recompile on save, and side-by-side PDF preview.
Installs the TeX distribution + the LaTeX Workshop extension, writes the per-project build
config, and pre-empts the common macOS gotchas (GUI apps not inheriting `PATH`, Gatekeeper
quarantine, broken webview preview). Handles Chinese / `ctex` documents (XeLaTeX). Ships a
`settings.template.json` you adapt per project.
