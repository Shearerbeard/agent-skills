---
name: opencode-cli
description: |
  Drive OpenCode from an external harness (Claude Code, codex, a script)
  through the opencode CLI, for code writing and second-model code review.
  Use when the user says "delegate to opencode", "opencode review",
  "second-model code review", "run opencode", "opencode CLI", or names an
  opencode reviewer such as the glm or kimi reviewer. Contains the staging
  recipe, model selection from config, the --agent trap, permission and
  deadline handling, and the validity rule for a delegated review. Code
  only; for plan, prose, spec, architecture, product, or marketing review
  use the `collaborating-with-antigravity` skill instead. Never for a
  session already running inside OpenCode, which dispatches its
  subagents in-session instead.
compatibility: claude-code opencode
license: MIT
---

# OpenCode CLI Delegation

Delegating code work to OpenCode from outside an OpenCode session. Check
the three preconditions before anything else.

## Preconditions

**1. You must not already be an OpenCode session.** A native OpenCode
session dispatches its own pinned subagents in-session and must never
shell out to the opencode CLI. A nested server breaks per-session cost
capture and violates the invariant that a review is never nested inside a
delegation. If that is you, stop here and follow the machine's global
opencode instruction file, `~/.config/opencode/AGENTS.md` (authored in
the dotfiles opencode group; if it is not deployed on this machine yet,
apply these three rules directly):

- Read your own agent pins before routing. Agent names do not imply
  model families, so check the `model:` field rather than the filename.
- Dispatch the pinned subagents with your own in-session task tool.
  Never invoke the opencode CLI from inside an opencode session.
- When a subagent cannot read a path, stage the packet into a `.review/`
  directory in the working directory and name the staged paths. Never
  fall back to the CLI because a read was refused.

**2. Pre-vet the reviewer before any gate depends on it.** Run a throwaway
echo prompt against the exact model id you plan to use, and read the
current pins (see Model selection). A reviewer that is unreachable, out of
credit, or mis-pinned fails silently at the moment you need it most.

```bash
cd /tmp && timeout 120 opencode run -m <provider>/<model> "Reply with: OK"
```

**3. This skill's artifact class is code**: code writing and code review
through the opencode CLI. Work outside that class is not this skill's to
place. Hand it back to the delegating-work router when a board process
installs one; with no router present, follow the repo's review-tooling
document. Plan reviews, prose, specs, architecture, product, and
marketing review land in `collaborating-with-antigravity`, but naming
that destination is not the same as picking the lane.

Within code, which reviews go to the opencode pinned reviewers and
which to the codex CLI (`codex-cli`) is the repo review-tooling
document's call; read it before dispatching either.

## Model selection

There is no model table in this skill on purpose - a stale table is what
retired its predecessor. Read the pins at routing time:

- `~/.config/opencode/agent/*.md` - each agent file's `model:` field
- the agent block in `~/.config/opencode/opencode.json`

Agent names do not imply model families. A file called rust-reviewer may
be pinned to any provider. The config is the only source of truth.

Never pass a router id of the form accounts/fireworks/routers/NAME; those
bill extra. Pass the plain model path, .../models/NAME.

## The invocation recipe

Stage first, then run. Never point a reviewer at scattered paths.

1. Create a dedicated directory and copy in everything the reviewer needs:
   the card or spec, the diff (a full-range.diff for a multi-commit range),
   and the prompt file.
2. `cd` into that directory.
3. Run, naming the staged files in the prompt:

```bash
cd /path/to/staging
timeout 600 opencode run -m <provider>/<model> \
  "Read prompt.md, spec.md and full-range.diff in this directory. ..."
```

Add `--format json` when you want the tool-call trace rather than plain
text. Flags verified present on 1.18.4: --command, -c/--continue,
-s/--session, --fork, --share, -m/--model, --agent, --format
{default,json}, -f/--file, --title, --attach, --dir, --port, --variant,
--thinking, -i/--interactive, --auto.

### The --agent trap

`opencode run --agent rust-reviewer` does not do what it reads like.
Subagent-only agents are not CLI-addressable, and the CLI silently falls
back to a different agent rather than erroring. You get a review from a
model you did not choose. Select the model with `-m` and put the reviewer
persona in the prompt.

### Why staging is mandatory

Reads outside the working directory are gated by OpenCode's
external_directory permission. It asks by default, and in a headless run
there is nobody to answer, so it auto-rejects. A run that ends in
permission rejections with no verdict is a failed delegation, not a pass.
Staging removes the permission question entirely.

## Deadlines and the stall protocol

`opencode run` has no timeout flag, so the caller owns the deadline. Always
wrap it (`timeout 600 opencode run ...`) and pick a bound you are willing
to wait out.

If it stalls, switch tools rather than retrying blind. Two runs have been
killed at the deadline, one at 10 minutes and one at 50; a stall record
like that is exactly what a repo's review-tooling document weighs when
it picks its reviewers.

## Validity rule

A delegation counts only if a verdict came back. Each of these is a FAILED
delegation, never a clean pass:

- an empty return
- an exit-0 run that produced no final text
- any run whose output carries no explicit verdict

Headless runs have exited 0 with no review text at all. Before recording a
result, confirm there is a final text event and grep the stream for
auto-rejection messages.

## Review-quality requirements

Put these in the prompt, or the output will not be gradeable:

- numbered findings, each labelled BLOCKING or MINOR
- a file:line reference on every finding
- a final verdict line

Zero findings is a valid result. Record it as an explicit PASS.

## Reviewer differs from author

A model never grades its own diff. If a model wrote the code, a different
model reviews it. Where the repo ships its own delegation document, that
document overrides the defaults here - read it first.

Pair with `gate-probes` for the universal checks, and `rust-review` or
`python-review` for language-specific probes once the findings are back.
