---
name: mermaid
description: Use when asked to view, render, preview, export, or open a Mermaid diagram; when working with .mmd files; or when Markdown contains Mermaid code blocks. In tmux, show it in the session with mermaid-pop (o keeps a live split, q closes); use mermaid-view for the browser page or file exports.
compatibility: claude-code opencode
license: MIT
---

# Mermaid Diagram Viewer

Show a Mermaid diagram in the current tmux session (default), or render it
to the browser and files.

## Quick view (default): mermaid-pop

1. Identify the file (from `$ARGUMENTS` or the most recently discussed
   `.mmd` or mermaid-containing `.md` file). Write any inline source to a
   `.mmd` temp path first; the script takes file paths, not stdin.
2. Run: `mermaid-pop <file-path>`
3. On exit 0, say the popup is up: the user reads the box-drawing render and
   presses `q` (or Esc) to close, or `o` to keep it in a split that
   re-renders on every change to the file. On non-zero exit, relay stderr
   (usually `sketch` missing or a render error) and present in chat instead.

- Needs tmux and `sketch` (tuios.dev/sketch). Outside tmux the script prints
  the text render, so it also doubles as an agent-readable fallback.
- A `.md` input draws every `` ```mermaid `` block in the file.

## Browser and export: mermaid-view

Run: `mermaid-view <file-path>` (opens the render in the browser)

Options: `--theme dark|default|forest|neutral`, `--format svg|png|pdf`,
`--scale N`, `--width N`

SVG is the default and the right pick almost always — it scales, so a large
flowchart stays readable. Use `--format png` only when the target needs a
raster: Slack, GitHub comments, slide decks. Pair PNG with `--scale 3` (and
`--width` if the diagram is wide) or the result is too coarse to read.

Routing: quick look in-session → `mermaid-pop <file>`; navigable or
shareable → `mermaid-view <file>`; persistent export →
`mermaid-view --format <file>`.

## Requirements

- mermaid-pop needs `sketch`: CachyOS/Arch `yay -S tuios-sketch-bin`,
  macOS `brew install --cask gaurav-gosain/tap/tuios-sketch`, or
  `GOBIN=~/.local/bin go install tuios.dev/sketch/cmd/sketch@latest`.
- mermaid-view needs `@mermaid-js/mermaid-cli`
  (`brew install mermaid-cli` — provides `mmdc`) and Google Chrome.
- The scripts are installed from dotfiles
  (`Configs/shell/.local/bin/{mermaid-view,mermaid-pop}` → `~/.local/bin`),
  not from this repo. Edit them there.
