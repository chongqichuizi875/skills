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

### `setup-latex-vscode-mac`

Set up a local LaTeX authoring environment on macOS that mirrors the Overleaf experience —
edit a `.tex` project in VSCode, auto-recompile on save, and side-by-side PDF preview.
Installs the TeX distribution + the LaTeX Workshop extension, writes the per-project build
config, and pre-empts the common macOS gotchas (GUI apps not inheriting `PATH`, Gatekeeper
quarantine, broken webview preview). Handles Chinese / `ctex` documents (XeLaTeX). Ships a
`settings.template.json` you adapt per project.
