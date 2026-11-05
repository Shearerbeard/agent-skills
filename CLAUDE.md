# CLAUDE.md - Personal Skills Library

Model-invoked skills for Claude Code, OpenCode, Codex, and Pi, registered as a local marketplace (`my-skills`).

`README.md` owns orientation: the skill catalog, how skills chain, installation, quality-gate commands, repo structure, adding a skill, and the conventions for writing skill descriptions. Reference it rather than restating it here; when those facts change, update `README.md`.

## Repo conventions

- `plugins/` is the source of truth: one plugin per domain, skills at `plugins/<plugin>/skills/<name>/SKILL.md`.
- After any `plugins/` change, run the quality gates (README "Quality Gates") and reinstall with `./bin/install-skills /path/to/this/repo`. OpenCode, Codex, and Pi read `~/.agents/skills/`; Claude reads the marketplace source directly.
- Pre-commit hooks run on every commit: betterleaks (secrets), asciicheck (Unicode artifacts), check-prose (Vale on markdown diffs), vale-commit-msg (commit message quality). Install with `./bin/setup-hooks`.
- Behavior checks live in `docs/internal/testing/skill-test-matrix.md`. Score auto-loading separately from manual invocation; model-driven skill routing is not deterministic.
- `docs/internal/sessions/`, `docs/research/`, `docs/proposals/`, and `feedback/` are gitignored local working documents. Keep them out of commits; never delete them.
- `feedback/` holds session retros on skill triggering and performance. `feedback/README.md` owns the directory naming and frontmatter conventions.

## Bundling Scripts with Skills

If a skill needs a helper script, place it in `plugins/<plugin>/bin/`. Claude Code adds this directory to `$PATH` when the plugin is enabled, so scripts can be referenced by name from any project directory.

Do NOT put scripts inside `skills/<name>/scripts/`; that path only resolves from within this repo.

See: [Plugin structure docs](https://code.claude.com/docs/en/plugins)

## Prose Linting

README "Prose Linting" covers setup (`vale sync`), the pinned style package, and the prose-lint/humanizer split. Working notes for sessions in this repo:

Common checks for changed prose:
```bash
vale --no-global --output=JSON README.md plugins/docs/skills/prose-lint/SKILL.md
printf '%s' "$draft" | vale --no-global --ext=.md --path=.git/COMMIT_EDITMSG --output=JSON
printf '%s' "$docstring" | vale --no-global --ext=.md --ignore-syntax --output=JSON
```

Use `prose-lint` for Vale workflows. Do not lint generated content, code blocks, schemas, exact API signatures, config examples, or intentional bad-prose examples unless the user asks. When invoked standalone, `humanizer` runs its own Vale pre-pass (step 0) so deterministic checks apply even without an upstream `prose-lint` call.

## Planning

Before non-trivial code work, load `plan-discipline`. It enforces the scope interview, verification framing, blast-radius scan, gate placement, and review checkpoints. It is the single source of truth for planning workflow; do not duplicate its rules here.

Audit rounds follow a standing pattern (adopted 2026-07-03): the round's execution plan lives at `docs/proposals/<date>-<topic>-execution-plan.md` (gitignored working doc) with a temporary "Active execution plan" pointer section at the top of this file. Every session updates the plan's SESSION STATE block before it ends; remove the pointer section when the round's closing actions complete.

See: [Skills docs](https://code.claude.com/docs/en/skills), [Plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)

## Archive Convention

**When archiving:** move to `_archive/<descriptive-name>/`, never delete.

**When creating a new skill:** check `_archive/` for related prior work (see README "Archive"). Adapt to the current format, don't copy wholesale.
