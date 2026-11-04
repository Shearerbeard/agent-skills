---
name: codex-cli
description: |
  Drive the codex CLI from an external harness (Claude Code, OpenCode, a
  script) for a second-model read. Use when the user says "codex review",
  "second opinion via codex", "adversarial review", "drive codex", "codex
  exec", or names codex as the reviewer for a diff, plan, or card.
  Contains the sandboxed invocation, caller-owned deadlines, the prompt
  contract that makes a review gradeable, the validity rule for a
  delegated review, and the staging recipe for material outside the
  working repo. Pins no model on purpose. Where a repo ships a
  review-tooling document, that document decides which role codex serves.
compatibility: claude-code opencode
license: MIT
---

# codex CLI Delegation

Driving the codex CLI from another harness for a second-model read. The
command is the small part; the prompt contract and the validity rule are
what decide whether the result counts.

## Role

Which role codex serves in a repo is that repo's review-tooling
document's call; read it before dispatching. Bindings such a document
commonly makes: ad-hoc adversarial review, and the doc-card prose
reviewer. A property worth knowing when the document weighs its
options: no per-session spend approval gates a codex dispatch.

Where neither a review-tooling document nor a delegating-work router
is installed, codex is a workable second opinion - that is this
skill's degrade path, not a topology decision it gets to make. This
skill owns **how** to drive codex, never which lane gets it.

## Artifact class and routing

The class this skill serves is the codex transport itself: adversarial
review of whatever artifact the caller hands it, plus any role a repo's
review-tooling document binds to codex.

Work that belongs to another transport is not this skill's to place.
Hand it back to the delegating-work router when a board process installs
one; with no router present, follow the repo's review-tooling document.
Code writing and code review through OpenCode live in `opencode-cli`;
plan, prose, spec, architecture, product, and marketing review live in
`collaborating-with-antigravity`. Naming those destinations is not the
same as picking between them.

## Invocation

```bash
codex exec --sandbox read-only '<review prompt>'
```

Run it in the background. A review is a read, so the read-only sandbox
is the right default and there is no reason to widen it.

`codex exec` carries no timeout flag, so the CALLER owns the deadline.
Use the timeout mechanism your own harness gives a background command.
Do not reach for GNU `timeout`: stock macOS does not ship it, and a
recipe that assumes it will fail on the machine where you need it.

Locate the binary with `command -v codex` rather than hardcoding a
path. Installs move it.

## Model

Never pinned here. codex prints the model it ran in its run header, and
the ledger records what actually ran. A recipe that names a model id
goes stale the first time the default moves, and from then on it
misreports the review that happened. Read the header.

## Prompt contract

codex reads repo files itself. Give it the program context and the file
PATHS; never paste file contents into the prompt.

The prompt has to demand:

- numbered findings, each labelled BLOCKING or MINOR
- a file and line reference on every finding
- exactly one verdict line, `VERDICT: PASS` or `VERDICT: FAIL`, or the
  caller's own equivalent pair

Without those three, the output is not gradeable and the delegation
cannot be recorded against a gate.

## Validity rule

A delegation counts only if a verdict came back. Each of these is a
FAILED delegation, never a clean pass:

- an empty return
- an exit-0 run that produced no final text
- any run whose output carries no explicit verdict

On a stall, switch tools rather than retrying blind. A second blind run
against a stalled reviewer costs the same wall clock and teaches you
nothing.

Zero findings is a valid result. Record it as an explicit PASS.

## Staging

When the reviewer needs material from outside the repo it runs in, copy
that material into a directory inside the working repo and name the
staged paths in the prompt. A path the sandbox cannot reach yields a
review of nothing, and the run still exits 0.

For a review of a multi-file change, point at a pre-generated diff or
review packet rather than requiring git access. A reviewer's permission
profile may not cover running git at all, and a generated packet is
also the artifact the ledger can cite later.

Pair with `gate-probes` for the universal checks, and `rust-review` or
`python-review` for language-specific probes once the findings are back.
