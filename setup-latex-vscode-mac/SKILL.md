---
name: setup-latex-vscode-mac
description: Set up a local LaTeX authoring environment on macOS that mirrors the Overleaf experience — edit a .tex project in VSCode, auto-recompile on save, and see a side-by-side PDF preview. Use this when the user wants to compile LaTeX locally (especially a resume/CV or a Chinese-language document using the ctex package), is moving off Overleaf, or is setting up a fresh Mac and asks Claude to "configure VSCode + LaTeX". Covers installing the TeX distribution, the VSCode LaTeX Workshop extension, the per-project build config, and fixing the common macOS gotchas (PATH not inherited by GUI apps, Gatekeeper quarantine, broken webview preview).
user-invocable: true
---

# Set up LaTeX + VSCode on macOS (local Overleaf)

## Goal

A local workflow that feels like Overleaf: open a `.tex` project in VSCode, edit, hit save,
and the PDF auto-recompiles and updates in a side-by-side preview. This skill gets that
working on a Mac from scratch, including a few macOS-specific pitfalls that reliably bite.

## What "done" looks like

1. A TeX engine (`xelatex`, `latexmk`) is installed and works from the command line.
2. VSCode has the **LaTeX Workshop** extension (`James-Yu.latex-workshop`).
3. The project has a `.vscode/settings.json` that compiles with the right engine on save and
   previews the PDF in a tab.
4. The user can edit → save → see the PDF update, no manual steps.

Always **verify by compiling from the command line first** — that isolates "the toolchain
works" from "VSCode is wired up correctly". Only then debug the editor integration.

## Step 0 — Survey what's already there

Don't assume a clean machine. Check before installing anything:

- TeX engines: `which xelatex latexmk pdflatex` and `ls /Library/TeX/texbin` (MacTeX's bin dir).
- Package manager: `which brew`.
- VSCode: is the app in `/Applications`? Is the `code` CLI on PATH (`which code`)?
- The project: find the `.tex` files and any `.cls`/`.sty`. If the user has a `.zip` (e.g. an
  Overleaf export), unzip it into the project root and work from the extracted sources.

## Step 1 — Pick the engine (this matters)

Inspect the main `.tex` preamble. **If it uses `\usepackage{ctex}` or otherwise needs CJK /
custom system fonts, you must use XeLaTeX (or LuaLaTeX) — `pdflatex` will not work.** This is
the single most common reason a Chinese resume "won't compile". Default to **XeLaTeX** for
anything with Chinese; it's also a safe default for modern fontspec-based documents.

## Step 2 — Install the TeX distribution

Recommend the **full distribution** (MacTeX) unless the user is space-constrained: it bundles
every common package plus CJK fonts, so Chinese documents work with zero extra package
juggling. The trade-off is a large download (several GB).

```
brew install --cask mactex          # full; or `brew install --cask basictex` for the minimal one
```

**Critical gotcha — the install needs a password and a real terminal.** The final step shells
out to `sudo installer …`. If you run `brew install --cask mactex` in a non-interactive /
background context, it downloads the whole package fine but then **fails at the sudo step**
("a terminal is required to read the password"). The multi-GB download is *not* wasted — it
stays in Homebrew's download cache. When this happens, hand the install step to the user:
have them run the `sudo installer -pkg <cached-pkg> -target /` command (or just double-click
the cached `.pkg`) so they can type their password. Find the cached pkg under Homebrew's
downloads cache. After install, a new shell (or `eval "$(/usr/libexec/path_helper)"`) is
needed for the CLI tools to land on PATH.

If you chose BasicTeX instead, expect to `sudo tlmgr install …` extra packages (latexmk, ctex,
the CJK font packages, etc.) — more fiddly for Chinese; prefer the full distribution when in doubt.

## Step 3 — Install VSCode + the extension

- If VSCode isn't in `/Applications`, install it there. If the user downloaded it via a browser,
  it carries a Gatekeeper **quarantine** flag (see Pitfalls).
- Install the `code` CLI so terminal launches inherit the shell PATH (this sidesteps the #1
  pitfall below). Symlink VSCode's bundled `code` binary onto a directory already on PATH, or
  use VSCode's "Shell Command: Install 'code' command in PATH".
- Install the extension: `code --install-extension James-Yu.latex-workshop`. The extension
  lives in the user profile (`~/.vscode/extensions`), so it survives reinstalling the app.

## Step 4 — Write the per-project build config

Create `.vscode/settings.json` in the project. The shape that works:

- A single **recipe** that runs one **tool**; the tool calls `latexmk` with the chosen engine
  flag (`-xelatex` for CJK).
- Set `latex-workshop.latex.recipe.default` to **`"first"`**, not the tool name. The default
  must name a *recipe*, and `"first"` just picks the first recipe in the list — robust and
  avoids the "Failed to resolve build recipe" error from naming mismatches.
- Auto-build on save and preview in a tab:
  `"latex-workshop.latex.autoBuild.run": "onSave"`, `"latex-workshop.view.pdf.viewer": "tab"`.
- Send build artifacts to a subdirectory (e.g. an out dir) and auto-clean aux files so the
  source folder stays tidy.
- **Make the toolchain findable from a GUI launch** — see the PATH pitfall. Set
  `latex-workshop.latex.path` to the TeX bin directory, give the tool's `command` the absolute
  path to `latexmk`, and put the TeX bin dir on the tool's `env.PATH` (latexmk shells out to
  the engine, so it needs it too).

A ready-to-adapt template lives in `assets/settings.template.json` next to this skill — copy it
and substitute the real TeX bin path and `.tex` filename. Keep paths generic in any committed
copy; don't hardcode a home directory with a username if it can be avoided.

## Step 5 — Verify

1. Compile from the command line first: run `latexmk` with the engine flag and an out dir on
   the main `.tex`. Confirm a PDF is produced. (Optional: `brew install poppler` to render the
   PDF to PNG and eyeball it — handy for checking CJK glyphs actually appear.)
2. Then in VSCode: reload the window so new settings load, open the `.tex`, hit the build
   (green arrow) or just save, and open the side preview (the SyncTeX/preview command).
3. If either fails, work the Pitfalls below in order.

## Pitfalls (macOS-specific, in rough order of frequency)

**`spawn latexmk ENOENT` / recipe can't find the engine.** A GUI-launched VSCode (opened from
Finder/Dock) does **not** inherit your shell's PATH, so it can't find MacTeX's binaries even
though the terminal can. Fixes (do all three for robustness): set `latex-workshop.latex.path`
to the TeX bin dir, use an **absolute path** for the tool's `command`, and add the TeX bin dir
to the tool's `env.PATH`. Launching VSCode via the `code` CLI from a terminal also works because
it inherits PATH — but the config fix is better because it works regardless of how it's launched.

**`Failed to resolve build recipe: <name>`.** `recipe.default` was set to a *tool* name instead
of a *recipe* name. Set it to `"first"` (or to an actual recipe's `name`).

**Preview shows "Could not register service worker: InvalidStateError" — and even image
(PNG) preview is broken.** This is not a LaTeX problem; it's VSCode's webview cache gone bad
(common when a freshly installed app reuses an old, stale user profile). Fully quit VSCode
(Cmd+Q, not just close the window), then delete the cache dirs under
`~/Library/Application Support/Code/` — `Cache`, `CachedData`, `Code Cache`, `GPUCache`, and
`Service Worker`. These are caches; deleting them loses no settings, extensions, or workspaces.
Reopen VSCode and they rebuild clean. If it persists, also clear the Gatekeeper quarantine
(below) — a quarantined app's helper processes can be sandboxed in ways that break the webview.

**App "won't open" / Gatekeeper.** An app downloaded via a browser carries
`com.apple.quarantine`. Easiest fixes are GUI: right-click the app → **Open** (then confirm),
or **System Settings → Privacy & Security → "Open Anyway"**. The CLI equivalent is
`sudo xattr -rd com.apple.quarantine "<App.app>"` — but that needs the user's password, so
hand it to them rather than trying to run it from a sandboxed context. If the app already opens
fine, this step is unnecessary.

**Permissions.** Anything touching `/Applications` or `/Library` (the `sudo installer`, clearing
quarantine) needs the user's password and can't be done from a non-interactive/sandboxed tool
context. Recognize these and ask the user to run the one command in their own terminal, rather
than retrying and failing.

## Notes on reuse

The point of this skill is repeatability: on a fresh Mac, follow Steps 0→5 and the user gets a
working local Overleaf without rediscovering the PATH/quarantine/webview traps. Keep the
committed config generic — substitute machine-specific absolute paths at setup time, don't bake
them into shared files.
