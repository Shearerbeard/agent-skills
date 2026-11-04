---
name: collaborating-with-antigravity
description: Delegate plan, prose, spec, architecture, product, and marketing review to the Google Antigravity (agy) CLI, through the agy MCP tools from Claude Code or a JSON-bridge script from other harnesses. Also covers sandboxed execution in a worktree and long agent loops that should not block the main conversation. Not for code review, which goes to the opencode CLI reviewers or the codex CLI. agy spend is budget-gated, so get explicit user approval once per session before dispatching.
license: MIT
---

# Collaborating with Antigravity

The agy bridge (`scripts/agy_bridge.py`) is a thin JSON wrapper around the
Google Antigravity (`agy`) CLI plus an optional `gemini` CLI fallback. It
produces stable `BridgeResponse` envelopes so this skill can drive
Antigravity sessions deterministically.

## Artifact class and routing

This skill serves one artifact class: **plan, prose, spec, architecture,
product, and marketing review**. agy handles human language more
naturally than Anthropic and Anthropic-inspired models, and those
artifacts are judged on exactly that.

Work outside that class is not this skill's to place. Hand it back to
the delegating-work router when a board process installs one; with no
router present, follow the repo's review-tooling document. Code review
in particular is out of scope here, and its destinations are the
OpenCode CLI reviewers (`opencode-cli`) and the codex CLI (`codex-cli`).
Naming a destination is not the same as picking the lane, and picking
the lane stays the router's job. Where a repo's review-tooling document
routes prose review somewhere else, that document decides, not this
file.

## Preflight, in order

Both steps run before this session's **first** dispatch, in this order.

1. **Confirm the spend approval is logged.** agy spend is budget-gated.
   The session needs explicit per-session user approval for agy spend,
   recorded in the session itself, before anything is dispatched. An
   approval carried over from an earlier session does not count, and
   neither does an inference from the user asking for a review.
2. **Run the `agy_doctor` probe and read its report.** It confirms the
   agy version, the auth state, whether the `gemini` CLI fallback is
   present, and the session-store path. Do not trust any install-state
   claim written into this file; where the two disagree, the report is
   authoritative.

Outcome rule: a probe reporting auth trouble, an exhausted quota, or
config that is missing means the reviewer is unreachable. The dispatch
does not happen. The gate it would have served stays open as a
deferral, and that deferral is surfaced at the next user gate.

## MCP or bridge

Two frontends, one backend. Pick by what your harness can reach:

- **From Claude Code, use the agy MCP tools.** They implement a bounded
  job model: `agy_start` returns a job id immediately, then `agy_status`,
  `agy_read`, `agy_result`, and `agy_cancel` work against that id. Every
  call takes an explicit `timeout`, and jobs are recorded in an on-disk
  session store. That job record is why agy stays reliable where a
  client-side wait would strand the work.
- **Use `scripts/agy_bridge.py` from harnesses that cannot reach Claude
  Code's MCP servers**: an OpenCode or codex session, or a plain shell
  script.

Both drive the same agy-mcp backend, so this is a transport choice, not
a capability difference.

## When to use

- You need a **second opinion** on a plan, spec, architecture note, or
  other language-shaped artifact.
- You are delegating **execution** of a change and want it kept in
  isolation (opt into a worktree). This is execution delegation, not
  review, and it stays behind the budget gate.
- You need to **run a long agent loop** (planning, large refactors) without
  blocking your own conversation: start a background job and poll.
- You want code Antigravity proposes **run in a sandbox** so its side
  effects are contained. Again isolation, not a review verdict, and
  budget-gated like the rest.

Do not use for trivial one-shot questions - the round-trip overhead is
not worth it. Prefer direct work for those.

## Quick start

```bash
# One-shot synchronous call (ask mode, no write):
python scripts/agy_bridge.py \
  --cd "/path/to/project" \
  --PROMPT "Explain the auth flow in src/auth/"
```

Output is a single JSON line on stdout: `{"success": true, "SESSION_ID":
"...", "agent_messages": "...", "adapter": {...}, ...}`.

## Modes

`--mode` controls the agent persona and downstream safety policy:

| Mode | Use it for | Worktree? | Writes? |
|------|-----------|-----------|---------|
| `ask` (default) | Q&A, code reading, design discussion | no | no |
| `plan` | Multi-step planning, breakdown | no | no |
| `prototype` | Generate diffs to hand back (execution delegation, not review) | optional | no |
| `review` | Review of a plan, prose, or spec artifact | no | no |
| `execute` | Make file edits in the workspace | opt-in | requires `--allow-write` |
| `browser` | Interactive browsing / research | no | no |
| `long` | Multi-hour agent loop, expect to poll status | no | no |

`execute` needs `--allow-write` to opt in to mutations. Keep
`worktree_default` off in `~/.config/agy-mcp/config.toml` unless a run
genuinely needs isolation, and turn it on per call with `--worktree
true` when it does. A machine-wide default-on leaves a branch and a
directory behind after every execute run, whether or not that run
wanted one (see references/security.md for the settings).

When a run does create a worktree, nothing auto-cleans it;
references/prompt-patterns.md carries the cleanup duty next to the
execute prompt scaffold.

## Multi-turn

Capture `SESSION_ID` from the first response, then pass it back:

```bash
# Turn 1
python scripts/agy_bridge.py --cd "/proj" --PROMPT "Analyse src/auth/"
# → {"SESSION_ID": "abc-123", "agent_messages": "..."}

# Turn 2 (continues the same conversation)
python scripts/agy_bridge.py --cd "/proj" --SESSION_ID abc-123 \
  --PROMPT "Now propose a refactor."
```

## Long jobs (start / status / result / read / cancel)

For tasks that exceed a single Claude turn, use the supervisor surface
via the MCP tools `agy_start` / `agy_status` / `agy_result` /
`agy_read` / `agy_cancel`.
See `references/usage.md` for full examples.

## Capability detection

The bridge advertises adapter capabilities in every response under
`"adapter"`. Trust those over your prior assumptions - `agy` does not
stream tokens, so `supports_streaming=false` is normal; `gemini` does,
so the fallback path returns finer-grained events.

## Output protocols

`--output-protocol claude` (default) emits events shaped like Claude
Code stream-json. `--output-protocol codex` emits OpenAI Codex
exec-json. `--output-protocol raw` returns the canonical event envelope
unchanged. Pick the one that matches your downstream parser.

## Safety floor

- Secrets are scrubbed from every response and log via `SafetyPolicy`.
- `--allow-write` is required for any mutation; safety policy denies
  destructive prompts even with the flag.
- Long jobs expose `exit_code` and timing on the `JobRecord` returned
  by `agy_status`.

## Detailed references

- `references/usage.md` - full CLI flag reference, MCP tool surface,
  long-job patterns, exit codes.
- `references/prompt-patterns.md` - proven prompt scaffolds for `ask`,
  `plan`, `prototype`, `review`, and `execute` modes.
- `references/security.md` - threat model, secret handling, denylist,
  worktree behaviour, audit log layout.
