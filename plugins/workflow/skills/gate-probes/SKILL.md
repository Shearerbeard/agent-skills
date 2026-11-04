---
name: gate-probes
description: |
  Use before git commit, gh pr create, or when the user says "ready to commit",
  "review this diff", "check before I push", or "run the gates". Also use when
  the user says "ensure proper gates", "verifiable checks", "ready for
  implementation", "plan with gates", or "scope this change". Run these
  universal gates before language-specific review skills (python-review,
  rust-review). Checks scope control, duplication, reviewability, verification,
  and residual risks.
compatibility: claude-code opencode
license: MIT
---

# Gate Probes

Run at every commit boundary, review gate, or plan review. Language-specific skills add their own probes after these.

After the universal probes, route to the next applicable skill without repeating work already done for this diff:
- Python diffs (`.py`, `pyproject.toml`, `uv`, `ruff`, `pytest`, `click`): run `python-review`
- Rust diffs (`.rs`, `Cargo.toml`, `clippy`): run `rust-review`
- Rust module layout changes: run `rust-modules`
- ADR / decision-record changes (`docs/adr/`, `DECISIONS.md`): run `adr-review`
- Checked-in docs changes: run `docs-bustest`
- User-facing prose without a docs structure review need: run `prose-lint`; then run `humanizer` if the prose will be checked in, published, or sent on the user's behalf
- Writing the commit message itself: run `git-commit`

## Quality probes

1. Are we sprawling code unnecessarily?
2. Did we reimplement something that already exists in the codebase?
3. Are we building god modules?
4. Will a developer be able to review and follow what we wrote?

## Surgical discipline

5. Every changed line traces directly to the user's request
6. Unrelated findings: mention, don't fix

## Coherence check

7. Re-read modified files in full after editing — diffs that look correct in isolation can create duplicated logic, inconsistent naming, orphaned imports, or functions that no longer fit the module's flow

## Plan-specific probes

Use these probes when reviewing a plan or implementation spec (not a code diff):

8. Does each step have verifiable exit criteria?
9. Are dependencies between steps explicit?
10. Is there a rollback procedure if a step fails?
11. Are the success criteria testable, not subjective?

## Report format

Present findings as a table. Block the gate on any FAIL:

```
## Gate Probe Results

| # | Probe | Status | Evidence |
|---|-------|--------|----------|
| 1 | Code sprawl | PASS | 3 files changed, all trace to user request |
| 2 | Duplication | FAIL | Reimplemented X; existing function at src/util.rs:42 |
| ... | ... | ... | ... |

### Gate verdict: PASS / FAIL
- Blocking findings: <list any FAIL items>
- Next step: <route to language-specific skill or proceed>
```
