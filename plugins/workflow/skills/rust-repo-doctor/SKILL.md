---
name: rust-repo-doctor
description: |
  Use only when the user explicitly invokes `rust-repo-doctor`; never
  auto-load. Audits Rust repositories and their machine-local development
  wiring across one or more checkout roots. Checks each repo's declared
  build, Clippy, test, and formatting gates; Boardkit boards and skill
  deployment; wiki-kit docks and rendered skills; rust-holes and typed-holes;
  Git/upstream state; entry files; and project-local overrides. Produces one
  status table and a separate list of changes required in other repositories.
compatibility: claude-code opencode
disable-model-invocation: true
---

# Rust repo doctor

Run this skill only when the user names it. It is a read-only audit unless the
user includes **fix-local** in the invocation.

## Scope

Use checkout roots supplied by the user. With no roots, use the current Git
repository and existing `~/dev` and `~/workspace` directories. Do not scan the
whole home directory. Skip `.git`, `target`, `.venv`, `node_modules`, vendor,
and cache trees.

Inventory:

- Git roots containing `Cargo.toml`.
- Separate board homes found through `.boardkit/manifest.toml`, `boardkit.toml`,
  and `boardkit boards --json`.
- Wiki deployments and consumers found through `wiki.toml` and
  `.wiki/manifest.toml`.
- The local Boardkit, agent-skills, wiki-kit, and rust-holes
  checkouts when present.

Deduplicate by resolved Git root. Never treat a worktree as a second project,
but report its branch separately when it is part of the requested roots.

## Safety rules

- Read each repo's `AGENTS.md`; if absent, read `CLAUDE.md`. Then read the
  build/test section of its README and its CI, Makefile, justfile, or
  pre-commit configuration. Repository commands override generic commands.
- Do not pull, checkout, reset, clean, stash, commit, push, start containers,
  migrate data, or edit tracked files.
- `git fetch --prune` may run when network access is available because an
  unfetched "up to date" claim is not useful. Record auth or network failure
  as `BLOCKED`, not as repository failure.
- Run live or ignored tests only when their declared services are already
  healthy and the tests are documented as non-destructive. Otherwise report
  them as `NOT RUN` with the exact prerequisite.
- A command that was not run is never a pass. Keep `FAIL`, `BLOCKED`, `WARN`,
  `NOT RUN`, and `N/A` distinct.

## 1. Git and entry files

For every repo, report:

- branch, HEAD, upstream, ahead/behind counts after fetch, and working-tree
  state;
- linked-worktree status where applicable;
- presence and direction of `AGENTS.md`, `CLAUDE.md`, and `GEMINI.md` shims;
- local `.claude/settings*.json` plugin disables; and
- repo-local `.agents/skills` or `.claude/skills` that shadow global skills.

Do not call a dirty tree unsafe by itself. Name each pre-existing path and keep
it separate from failures introduced by a check.

## 2. Rust gates

Run the repo's declared commands exactly, including its pinned toolchain,
features, `--locked`, and lint allowances. Cover these columns when the repo
provides them:

1. format;
2. build or check, including all targets/features when declared;
3. Clippy with the repo's warning policy;
4. unit, integration, doctest, example, and already-available backend gates;
5. language-specific companion checks such as Ruff.

If no gate is documented, use this conservative baseline and label it
`fallback`, not repository policy:

```sh
cargo fmt --check
cargo build --locked --all-targets
cargo clippy --locked --all-targets -- -D warnings
cargo test --locked
```

Do not repair source failures during the audit. Capture the shortest useful
failure signature and the command that reproduced it.

## 3. Boardkit

Locate Boardkit from `BOARDKIT_HOME`, a sibling checkout, or a path the user
supplied. Fail loud if more than one candidate disagrees.

For every resolved board:

```sh
export BOARDKIT_HOME=/resolved/path/to/boardkit
uv run --project "$BOARDKIT_HOME" boardkit check
uv run --project "$BOARDKIT_HOME" boardkit doctor
```

Also run `boardkit boards --json` where a family manifest exists. Report the
resolved board root, unreachable external rows, errors, warnings, and skips.
A doctor exit zero with warnings is `PASS (warnings)`, never `clean`.

Verify both Boardkit skills, board-hygiene and delegating-work:

- source exists under Boardkit's plugin;
- the `~/.agents/skills/.manifest.boardkit` copies byte-match source; and
- `board@boardkit` is installed and enabled when the Claude CLI is present.

Search for project-local copies with the same names. A local copy wins in that
repo, so compare it with source and label deliberate forks separately from
stale shadows.

## 4. Rust-holes

A complete result requires:

- a current, cleanly readable rust-holes checkout;
- `bin/check` passing;
- the canonical `typed-holes` skill present in agent-skills;
- the `~/.agents/skills/typed-holes` copy matching source; and
- the Claude `rust` plugin copy matching source when Claude is installed.

The seeded-defect script documents a BSD `sed` assumption. On GNU/Linux, test
it in a temporary copy with only `sed -i ''` changed to `sed -i`; never patch
the checkout just to run the probe. Report the native portability issue and
the translated result separately.

## 5. Wiki-kit

Wiki-kit skills are project-scoped before a deployment's adoption gate. Do
not require or install `garden`, `handoff`, `morning`, or session-feedback
globally.

For the kit checkout, run its declared Ruff and pytest gates. For each wiki,
run `wiki-doctor.py`. For each consumer dock, run `wiki-dock.py status` and
verify `.wiki/rendered-skills.json` against the rendered files.

A consumer intended for Claude and Agent Skills tools needs all four skills
under both `.claude/skills` and `.agents/skills`. Compare rendered content with
the current kit after substituting the recorded kit path. Report legacy or
pre-adoption deployments; never migrate them during an audit.

## 6. Personal skill deployment

When agent-skills is present:

- run `bin/check-skills` and `bin/check-install`;
- compare every name in `.manifest.my-skills` with its source tree;
- validate and list Claude plugins when the CLI exists; and
- report same-name files in project or user skill directories that can shadow
  marketplace copies.

Version equality is not content equality. Compare files even when the plugin
manager says a version is current.

## Optional fix-local mode

Only when the user explicitly includes **fix-local**:

- reinstall agent-skills and Boardkit through
  `agent-skills/bin/install-skills`;
- refresh ignored wiki skill renders for complete, already-adopted docks; and
- add missing machine-local Boardkit overlays or checkout aliases after showing
  the proposed path.

Never use this mode to edit tracked files, migrate a wiki, rewrite prose,
change dependency versions, or cross a board gate. Put those items in the
external-changes list.

## Report

Start with this table, one row per repo or separate board home:

| Repository | Git | Build | Clippy/lint | Tests/live | Boardkit | Wiki-kit | Rust-holes | Doctor | Verdict |
|---|---|---|---|---|---|---|---|---|---|

Use `100%` only when all expected source and installed files were counted and
byte-compared. Follow the table with:

1. **Local repairs made** - machine-only changes from fix-local, or `none`.
2. **External repository changes required** - repository, file or surface,
   failure, and the smallest next action.
3. **Not run** - commands and prerequisites.
4. **Residual warnings** - future incompatibilities, dirty trees, board
   warnings, intentional local forks, and stale plugin-version metadata.

End with one verdict: `SAFE`, `SAFE WITH WARNINGS`, or `NOT SAFE`. A build pass
does not override a Clippy, test, board-doctor, or deployment failure.
