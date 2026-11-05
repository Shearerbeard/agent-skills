# Personal Skills Library

Model-invoked skills for Claude Code, OpenCode, Codex, Pi, and Kimi Code. Registered as a local marketplace (`my-skills`) for Claude Code, and installed to `~/.agents/skills/` for OpenCode, Codex, Pi, and Kimi Code.

## Skills

| Skill | Plugin | Trigger | Purpose |
|---|---|---|---|
| `python-quality` | python | writing `.py` | Toolchain (uv/ruff) + code patterns + LLM anti-patterns |
| `python-review` | python | reviewing Python | 7 Python-specific gate probes |
| `rust-quality` | rust | writing `.rs` | Anti-pattern catch list (clone escapes, wildcards, transitive deps) |
| `rust-async` | rust | async, tokio, Send+Sync | Cooperative scheduling, Send+'static, spawn_blocking vs rayon, async Drop |
| `rust-design` | rust | writing/designing/reviewing types | ADT-first workflow, constrained types, railway programming, 6-step clone avoidance |
| `rust-review` | rust | reviewing Rust | 7 Rust-specific gate probes |
| `typed-holes` | rust | typed holes, `todo!()` skeleton, golden frames | Two-layer practice: compiler-tracked skeleton + design panel, then goldens written to fail first |
| `rust-modules` | rust | Rust module layout | Modern file layout, re-exports, type co-location |
| `docs-bustest` | docs | docs review / updates | 24-item bus test for documentation quality |
| `adr-review` | docs | writing / reviewing an ADR | MADR rubric + premise-freshness; chains prose-lint then humanizer |
| `prose-lint` | docs | Vale / prose lint | Deterministic AI-tell checks for docs and emitted prose |
| `humanizer` | docs | publishable prose | Remove AI writing patterns from user-facing text |
| `mermaid` | docs | `.mmd` files / diagrams | Render and open Mermaid diagrams |
| `plan-discipline` | workflow | planning / design / scope | Verification-first, scope interview, blast radius |
| `gate-probes` | workflow | commit / review / PR | 7 universal quality probes + surgical discipline |
| `skill-retro` | workflow | "retro my skills" | Audit how skills triggered and performed in a session; files retros under `feedback/` |
| `git-commit` | workflow | committing / commit messages | Conventional commit format, prose quality chain, mandatory user review |
| `rebase-hygiene` | workflow | explicit invocation only | Fetch-first rebase ritual: pre-vet conflicts, pre-planned resolutions, verify the push |
| `rust-repo-doctor` | workflow | explicit invocation only | Audit Rust repos, gates, boards, skill installs, wiki docks, and local overrides across a machine |
| `opencode-cli` | workflow | delegating code work / review to OpenCode | CLI-only delegation: staging recipe, model pins from config, `--agent` trap, deadlines, validity rule |
| `codex-cli` | workflow | "codex review" / second opinion / adversarial review | Un-gated codex delegation: sandboxed invocation, caller-owned deadline, prompt contract, validity rule, no model pins |
| `collaborating-with-antigravity` | workflow | delegating plan / prose / spec / architecture review to agy | Budget-gated Antigravity delegation: ordered spend-and-doctor preflight, MCP or bridge transport, long-job supervision, the mode table |
| `process-feedback` | workflow | upstream toolkit friction | Record a process finding locally and file it as a GitHub issue on the upstream repo |
| `design-system` | design | brand colors, design tokens, visual identity, theme | Three-pass token authoring (philosophy, tokens, critique) into a harness-neutral store; anti-slop calibration |
| `web-artifact` | design | infographic, flow chart, shareable visual page | Self-contained HTML artifacts published to GitHub Pages or Netlify, with config registry and refresh deltas |

Two more plugins ship LSP configs rather than skills: `vale-lsp` (Vale prose diagnostics) and `haskell-lsp` (Haskell Language Server).

## How Skills Chain

Skills are designed to compose. A single prompt can load 2-3 skills when the content spans multiple domains.

### Rust stack

```
User prompt → rust-design   (type modeling, ADTs, constrained types)
            → rust-async    (Send+Sync, spawn, cooperative scheduling)
            → rust-quality  (anti-pattern catch list)
            → rust-review   (gate checklist, loaded at review time)
            → rust-modules  (file layout, triggered on mod/split/create)
```

Common combos:
- "design an async connection pool with proper types and Send bounds" → `rust-design` + `rust-async`
- "create a new billing module with Invoice state machine, avoid clone escapes" → `rust-design` + `rust-modules` + `rust-quality`
- "review this PR for clone issues and wildcard matches" → `rust-review` → `rust-quality` (loaded internally)

### Plan stack

```
User prompt → plan-discipline   (scope interview, blast radius, gate placement)
            → gate-probes       (exit check at each gate: sprawl, duplication, coherence)
                                   → rust-review / python-review (language-specific)
```

### Async concerns

`rust-async` is separate from `rust-design`. Design triggers ("type", "enum", "struct") fire `rust-design`. Async triggers ("tokio", "spawn", "Send+Sync") fire `rust-async`. A prompt with both triggers loads both.

## Writing Skill Descriptions

The `description` field in SKILL.md frontmatter is model-facing routing text. Claude Code and OpenCode both rely on model judgment over skill metadata; words like "triggered", "Triggers", "auto-triggered", and "activates" have no special runtime meaning. Claude Code can also use `when_to_use`; OpenCode ignores that field, so keep critical routing phrases in `description`.

Follow these conventions:

- **Lead with concrete user phrases**: "Use when the user says 'design a Rust type', 'model this in Rust'", not "Triggers when working with Rust."
- **List keywords and verbs the user actually types**: "async function", "tokio", "spawn a task". The model matches against these.
- **Front-load trigger conditions, then describe content**: trigger phrases first, then "Contains X, Y, Z."
- **Mention sibling skills for pairing**: "Pair with rust-quality during implementation."
- **Avoid abstract state language**: "entering plan mode", "at commit boundaries". The trigger system can't observe internal state transitions.

Plan mode hooks exist in Claude Code but are buggy: EnterPlanMode hook output is ignored (claude-code#41051), and user-initiated `/plan` doesn't fire hooks (claude-code#15660). Invoke `plan-discipline` manually when its hard blockers matter; plan-mode prompts often make the skill look redundant to the model.

## Frontmatter Fields

Skills use the [Agent Skills spec](https://agentskills.io/specification) frontmatter: `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`. Two non-spec fields are used deliberately and are **honored by Claude Code but silently ignored by OpenCode**:

- `when_to_use` (`plan-discipline`) - Claude Code routing hint. OpenCode ignores it, so its content is also folded into `description`.
- `disable-model-invocation` (`rebase-hygiene`) - Claude Code suppresses auto-loading. OpenCode ignores it, so the skill also states "never auto-load" in its `description` as an in-band backstop.

`bin/_check-frontmatter.py` is the single validator for the spec limits OpenCode enforces (description <= 1024 chars, `name` lowercase-alphanumeric-hyphen <= 64, `compatibility` <= 500) and warns on any other unknown key. Both `install-skills` (install-time gate) and `check-skills` (standalone gate) call it, so the two can't drift.

## CLAUDE.md Guidelines

`CLAUDE.md` files (project-level and repo-level) should route to skills, not duplicate them:

```markdown
## Planning

Before non-trivial code work, load `plan-discipline`. It enforces the scope
interview, verification framing, blast-radius scan, gate placement, and review
checkpoints. It is the single source of truth for planning workflow; do not
duplicate its rules here.
```

This pattern keeps the skill as the single source of truth. Duplicating skill rules in CLAUDE.md causes the model to follow the CLAUDE.md version (which is always active) and never load the skill, missing gate types, placement rules, and templates.

## Review Gate Types

`plan-discipline` defines five gate types for structuring a plan. Each stage in the plan template specifies which gates apply and whether the user must be pulled in:

| Gate | Who | What |
|------|-----|------|
| **S** Self-review | Agent | Run deterministic tools (lint, fmt, type check, build, test). Fix all failures. |
| **A** Agent second opinion | Context-isolated subagent | Review diff with fresh eyes. No prior implementation context. Find issues or sign off. |
| **M** Manual testing | Agent | Exercise the feature, report behavior, verify success criteria. |
| **U** User review | User | Pause and present diffs + findings. Highlight highest-risk changes. Await approval. |
| **T** User testing | User | Prescribe specific manual testing steps. Be explicit; don't assume the user knows how to test. |

## Installation

The installer registers the marketplace with Claude Code and flat-copies skills to `~/.agents/skills/` for [OpenCode](https://opencode.ai/docs/skills/), [Codex](https://developers.openai.com/codex/skills/), [Pi](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/skills.md), and [Kimi Code](https://github.com/MoonshotAI/kimi-cli). All four read `~/.agents/skills/` as a global user-scope location per the [Agent Skills standard](https://agentskills.io/specification); Kimi Code's parser accepts the same frontmatter spellings (`when_to_use`, `disable-model-invocation`) via its metadata aliases.

```bash
./bin/install-skills /path/to/this/repo
```

This does two things:
1. **Claude Code**: registers the directory marketplace with `claude plugin marketplace add` and installs/enables all plugins; if the Claude CLI is unavailable, it falls back to `~/.claude/settings.json`
2. **Agent tools**: copies skills to `~/.agents/skills/` with a per-source manifest (`.manifest.<marketplace-name>`)

To install only specific skills from a repo (e.g., a shared marketplace where you only want certain skills):
```bash
./bin/install-skills /path/to/employer/repo --skills github-workflow
```

Options:
- `--skills name1,name2` - only copy named skills to `~/.agents/skills/`
- `--claude-only` - only register Claude marketplace and plugins
- `--agents-only` - only copy to `~/.agents/skills/`

Per-source manifests prevent cross-pruning when multiple repos share `~/.agents/skills/`. Third-party skills (installed via npx or other tools) are never touched.

## Quality Gates

### Pre-commit Hooks (Recommended)

Install once per checkout to run all checks automatically on commit:

```bash
brew install pre-commit betterleaks
./bin/setup-hooks
```

Hooks run on every commit:
* **betterleaks**: secret scanning (BPE-based, 98.6% recall)
* **asciicheck**: non-ASCII artifact detection (smart quotes, em dashes, non-breaking spaces)
* **check-prose**: Vale prose quality on markdown diffs (skips SKILL.md and code blocks)
* **vale-commit-msg**: commit message quality (ai-tells-commits)

Run all checks: `pre-commit run --all-files`

### Manual Checks

Run these before installing changed skills:

```bash
./bin/check-skills
./bin/check-install
./bin/check-prose
```

Use `docs/internal/testing/skill-test-matrix.md` for manual Claude Code and OpenCode behavior checks. Treat auto-loading as a behavior to measure, not a deterministic contract.

## Prose Linting

This repo uses Vale with `tbhb/vale-ai-tells` (pinned to v1.13.1 in `.vale.ini`) for deterministic checks on AI-writing patterns. Run `vale sync` once per checkout to download the pinned style packages into `.vale/`; downloaded package directories are ignored by git.

`prose-lint` prefers a project's own `.vale.ini`; without one, Vale falls back to the global `~/.vale.ini` (dotfiles-managed). If Vale is not installed or `vale sync` fails, the skill reports that prose linting was skipped and does not try to install anything.

Use `prose-lint` for mechanical findings. Use `humanizer` after that when changed docs or outgoing prose need a rewrite.

## Adding a Skill

1. Create `plugins/<plugin>/skills/<name>/SKILL.md` with frontmatter (`name`, `description`)
2. If new plugin: create `plugins/<plugin>/.claude-plugin/plugin.json` and add to `.claude-plugin/marketplace.json`
3. Run `./bin/install-skills /path/to/this/repo` to register with Claude and install to `~/.agents/skills/`

## Structure

```
agent-skills/
├── plugins/                     # Source of truth: one plugin per domain
│   ├── python/                  # python-quality, python-review
│   ├── rust/                    # rust-design, rust-async, rust-quality, rust-review, rust-modules
│   ├── docs/                    # docs-bustest, adr-review, prose-lint, humanizer, mermaid
│   ├── workflow/                # plan-discipline, gate-probes, skill-retro, git-commit, rebase-hygiene, rust-repo-doctor, opencode-cli, codex-cli, collaborating-with-antigravity, process-feedback
│   ├── design/                  # design-system, web-artifact
│   ├── vale-lsp/                # Vale LSP config (no skills)
│   └── haskell-lsp/             # Haskell LSP config (no skills)
├── .claude-plugin/
│   └── marketplace.json         # Marketplace registry
├── bin/
│   ├── install-skills           # Register Claude marketplace + install to ~/.agents/skills/
│   ├── _check-frontmatter.py    # Shared Agent Skills spec-limit validator (install + check)
│   ├── check-skills             # Static skill/frontmatter checks
│   ├── check-install            # Temp-home install checks
│   └── check-prose              # Vale/prose lint smoke checks
├── docs/
│   └── internal/testing/        # Manual Claude/OpenCode test matrix
├── feedback/                    # Session retros (gitignored local working docs)
└── _archive/
    ├── legacy-slash-commands/   # v1 slash commands (2025-11)
    └── collaborating-with-opencode-mcp/  # retired MCP-first OpenCode skill (2026-07)
```

## License and Attribution

MIT, © 2026 Mike Shearer. See [LICENSE](LICENSE).

These skills are meant to be adapted. If you re-author one, keep a Lineage
block naming the source repo and skill path. Include the commit you started
from and a line on what you changed. [NOTICE](NOTICE) has the format.

## Archive

`_archive/legacy-slash-commands/` contains v1 slash commands from before Claude Code had skills. Some have useful patterns worth mining; check before building from scratch.
