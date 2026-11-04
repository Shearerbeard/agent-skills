---
name: mermaid
description: Use when asked to view, render, preview, export, or open a Mermaid diagram; when working with .mmd files; or when Markdown contains Mermaid code blocks. Render the diagram with mermaid-view and report the output path.
compatibility: claude-code opencode
license: MIT
---

# Mermaid Diagram Viewer

Render a Mermaid diagram and open it in the browser.

## Usage

1. Identify the file (from `$ARGUMENTS` or the most recently discussed `.mmd` or mermaid-containing `.md` file)
2. Run: `mermaid-view <file-path>`
3. Report the output path and confirm it opened

Options: `--theme dark|default|forest|neutral`, `--format svg|png|pdf`, `--scale N`, `--width N`

SVG is the default and the right pick almost always — it scales, so a large
flowchart stays readable. Use `--format png` only when the target needs a
raster: Slack, GitHub comments, slide decks. Pair PNG with `--scale 3` (and
`--width` if the diagram is wide) or the result is too coarse to read.

A `.md` input renders every `` ```mermaid `` block in the file; multiple
diagrams get an HTML index page.

## Requirements

- `@mermaid-js/mermaid-cli` (`brew install mermaid-cli` — provides `mmdc`)
- Google Chrome (used by mmdc for rendering)
- The script itself is installed from dotfiles
  (`Configs/shell/.local/bin/mermaid-view` → `~/.local/bin`), not from this
  repo. Edit it there.
