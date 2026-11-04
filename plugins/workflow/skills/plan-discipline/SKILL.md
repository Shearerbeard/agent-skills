---
name: plan-discipline
description: |
  Your planning discipline FOR plan mode: the user's required data-gathering
  depth, staging, and review gates. Load it whenever you plan - it layers on
  plan mode, it does NOT replace it. In Claude Code especially, EnterPlanMode
  is the container and this skill is the contents; entering plan mode without
  loading this means planning without the user's gates. Trigger it the moment
  you decide to plan - or the user says "plan this out", "scope this",
  "minimal V1", "vet assumptions", "don't assume", "plan with gates", "where
  should the user review?", or "what gates should this have?" - on any
  feature, refactor, migration, redesign, new module, public-interface, or
  3+ file change. It carries the scope interview, blast-radius scan,
  building-blocks search, and the gate ladder into the plan. Skip single-file
  edits, typo fixes, and pure read/answer requests.
compatibility: claude-code opencode
when_to_use: |
  Load this WITH plan mode, never instead of it - it is the preferences layer
  for HOW to plan (data-gathering depth, staging, gates), not a separate
  planner. The moment you decide to plan, or enter plan mode, on a multi-file
  implementation, refactor, migration, redesign, public-interface change, or
  new module, load this first. Entering plan mode without it means planning
  without the user's gates: scope interview, evidence check, verification
  framing, blast-radius scan, and the gate-probes review boundary.
license: MIT
---

# Plan Discipline

This is the user's discipline layer FOR plan mode - the preferences for HOW to plan (gathering, staging, gates). Load it WITH plan mode, never instead of it. Plan mode is the container; this skill is the contents.

## Pre-flight checklist

Before any file edit, complete this checklist. Do not proceed until all items are answered:

- [ ] **Scope interview answered**: core problem, audience, success criteria, explicit non-goals
- [ ] **Verification method stated**: how will this be verified? Include smoke test plan for the happy path
- [ ] **Blast radius identified**: which files change, which docs update, what tests exist, what coverage is needed
- [ ] **Existing building blocks searched**: grep/glob for reusable functions before implementing
- [ ] **Delegation inventory taken (before the plan is written)**: read the harness's agent or subagent configuration and record which executors and reviewers exist and what model each one is pinned to. Agent names drift from the models underneath them, so read the configuration rather than trusting the name. Do this before allocating work: the rule that a reviewer's model must differ from the author's shapes the plan itself, deciding who may write each stage and who is left able to review it, rather than being checked after the diff exists. If the plan depends on a particular reviewer, confirm it is reachable and authenticated during planning, not at the gate
- [ ] **Gate-probes invoked**: if this is a gate boundary (plan review, pre-implementation checkpoint), run `gate-probes` first

## Gate-probes routing

Before any file edit at a gate boundary, load `gate-probes` — it runs universal scope control, duplication, reviewability, and residual risk checks that this skill does not cover. Without it loaded, you will miss sprawl, god modules, and incoherent edits that look correct in isolation but break module flow.

## Scope interview

For non-trivial tasks, ask before building:
- What is the core problem this solves?
- Who is it for?
- What does success look like?
- What should this NOT do?

Summarize back before writing code.

## Review Gates

Every non-trivial plan must define which gates apply at each stage. Gates are
checkpoints the agent must pass before proceeding. Some gates pull the user in;
the plan must say so.

### Gate types

**Gate S — Self-review (agent, deterministic)**
- Load skill `gate-probes` first, then run its universal probes against the staged diff
- Run deterministic tools: lint, fmt --check, type check, build, test
- Fix all failures before proceeding
- Skip tools that can't run yet (e.g., tests before implementation) — note why

**Gate A — Agent second opinion (context-isolated)**
- Dispatch a fresh subagent with no prior implementation context
- Give it the diff, the plan, and the success criteria
- It must find issues or explicitly sign off
- Its model must differ from the model that authored the work under review; the pre-flight delegation inventory is what makes that allocation possible, so a reviewer this gate depends on was already named and confirmed reachable at planning time
- Combine its findings with Gate S results

**Gate M — Manual testing by agent**
- Agent exercises the feature using realistic inputs
- Reports behavior: what worked, edge cases hit, open questions
- Never passes if the agent can't verify the success criteria

**Gate U — User review**
- Pause and present: staged diffs, Gate S/A results, what changed and why
- Highlight highest-risk changes (new interfaces, data migrations, auth/permission)
- Do NOT proceed until user confirms

**Gate T — User testing**
- Prescribe specific manual testing steps the user must perform
- List what to test, how to verify, what edge cases to try
- Do not assume the user knows how to test — be explicit
- Output a required handout with six parts:
  - temp config diff (copy-paste runnable, outside the repo)
  - exact run command
  - reference prompt
  - expected observations in order (observable behaviors — never "verify it works")
  - failure signatures (what a bad result looks like)
  - revert steps

### Gate placement rules

| Stage | Gates | User pulled in? |
|-------|-------|-----------------|
| After each commit / chunk of work | S → A | No |
| Public API / interface changes | S → A → U | **Yes** |
| Data model / schema / migration changes | S → A → U | **Yes** |
| Auth / permission / security changes | S → A → U → T | **Yes — mandatory** |
| Before integration or merge | S → A → M → U | **Yes** |
| After full implementation | S → A → M → T | **Yes** |

**Gate economy.** Intermediate coding stages are S → A only. Place a 🛑 USER GATE
only where this table requires one (interface/schema, auth, integration or merge,
final implementation) — typically toward the end. A plan that stops for the user
on every stage is as wrong as one with no gates; it buries the gate that matters.

If a stage triggers user involvement, the plan template must include the 🛑
USER GATE marker. The agent must STOP and present at that gate — never push past
without user approval.

### Implementation shape

- For 5+ stage refactors: delegate per-stage work to sub-agents. Main agent owns
  sequencing, gates, git, memory.
- Integration tests at each stage boundary, not bundled at end
- At each gate boundary, load `gate-probes` first, then applicable
  language-specific review skills (rust-review, python-review)

## Plan Template

After completing the pre-flight checklist, produce the plan in this format.

**Render every stage's gates as the `[ ] Gate X:` checklist below, each with
named runnable checks. A prose `Gates: S → A → U` line is a scan summary, not the
gate rendering — a stage carrying only that line reads as gateless and is
non-compliant.** Give each stage a Goal and a Done-when so the gates have
something to verify.

```
## Plan: <task summary>

### Scope
- Core problem: ...
- Audience: ...
- Success: ...
- Non-goals: ...

### Verification
- Smoke test: ...
- Deterministic checks: ...

### Blast Radius
- Files to change: ...
- Existing building blocks: ...
- Test coverage gaps: ...

### Implementation Stages

#### Stage 1: <description>
- Goal: ...
- Changes: ...
- Gates: S → A
- [ ] Gate S: load skill `gate-probes`, then <deterministic checks to run>
- [ ] Gate A: agent review of diff
- Done when: ...

#### Stage 2: <description> 🛑 USER GATE
- Goal: ...
- Changes: ...
- Gates: S → A → U
- [ ] Gate S: <checks>
- [ ] Gate A: agent review
- [ ] Gate U: present diffs + findings, highlight highest-risk changes, await approval
- Done when: ...

#### Stage 3: <description> 🛑 USER TEST
- Goal: ...
- Changes: ...
- Gates: S → A → M → T
- [ ] Gate S: ...
- [ ] Gate A: ...
- [ ] Gate M: agent exercises feature, reports behavior
- [ ] Gate T: manual testing steps for user
- [ ] Gate T handout: config diff, run command, reference prompt, expected observations, failure signatures, revert steps
- Done when: ...

### Rollback
- If stage N fails: <how to revert>
```
