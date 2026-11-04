---
name: docs-bustest
description: Use when reviewing, auditing, or updating documentation — checks whether a repo's docs are good enough for both a new human contributor and a cold AI agent to pick up the project without the maintainer. Use when editing README, CLAUDE.md, or any project docs, when completing work that changes public interfaces, or when asked to review docs quality. Also known as a "bus test" — could someone take over if you disappeared?
compatibility: claude-code opencode
license: MIT
---

# Documentation Bus Test

Review a repo's documentation as if the maintainer disappeared tomorrow. Two audiences must be able to pick it up cold:

1. **A human contributor** — someone who knows the domain but not this project
2. **A cold AI agent** — a fresh session with zero prior context

This is not a style guide. It checks whether the information exists, is findable, and isn't duplicated across audiences.

## Principle: one fact, one place

README, CLAUDE.md, and any handoff docs serve different readers but overlap heavily. When both a human and an agent need the same info (build commands, env vars, architecture), it belongs in the public doc (README). CLAUDE.md should reference it, not restate it. Duplicated facts drift apart — that's a P1.

## The Test

Score each item pass/fail. Report the count and any P1/P2 findings.

### 1. Orient (can they understand what this is?)

- [ ] **One-liner**: README opens with what it does in one sentence, no jargon
- [ ] **Quick start**: copy-pastable install + run, under 5 steps
- [ ] **Structure**: where code lives, key modules/crates/packages named
- [ ] **Architecture**: how components connect (diagram, prose, or both)

### 2. Operate (can they run it?)

- [ ] **Env setup**: all required env vars, keys, and dependencies listed
- [ ] **Build + test**: exact commands, not "run the tests"
- [ ] **Config**: what knobs exist, what they default to, where they live
- [ ] **Deploy**: how it gets to production (CI, Docker, Helm, manual — whatever applies)

### 3. Decide (can they understand why things are this way?)

- [ ] **Decision log**: ADRs, design docs, or a DECISIONS.md — anything explaining *why*
- [ ] **Constraints**: known limitations, tradeoffs, things that don't work
- [ ] **Changelog**: what changed and when (auto-generated or manual)

### 4. Contribute (can they change it safely?)

- [ ] **Dev workflow**: branch naming, commit conventions, PR process
- [ ] **Test strategy**: what's tested, how, what coverage looks like
- [ ] **CI/CD**: what runs on push/PR, how to read failures
- [ ] **Code quality**: linting, formatting, type checking commands

### 5. Agent discoverability (can a cold session find what it needs?)

- [ ] **CLAUDE.md exists**: project purpose, build/test commands, key conventions
- [ ] **Current status**: what's working, what's broken, what to work on next — findable without reading git log
- [ ] **Handoff docs not stale**: if handoff/session docs exist (HANDOFF.md, session logs, etc.), verify they reflect current state — stale handoff docs are worse than none
- [ ] **Single roadmap**: one canonical TODO/roadmap, not competing files with conflicting priorities
- [ ] **Cross-references resolve**: CLAUDE.md → README, ADR index → ADR files, README → LICENSE — all links land
- [ ] **No duplication**: CLAUDE.md doesn't restate what README already covers (build commands, env vars, architecture) — it references it

### 6. Content quality (is what exists trustworthy?)

- [ ] **No stale claims**: README says X, code does Y → P1 finding
- [ ] **No jargon walls**: internal labels, acronyms, or shorthand without definition
- [ ] **Dated or versioned**: reader can tell if docs are current

## Scoring

- **21-24 pass**: bus-test ready — a human or agent can be productive immediately
- **16-20 pass**: survivable — gaps will slow them down but won't block them
- **10-15 pass**: fragile — significant tribal knowledge not captured
- **Under 10**: bus factor is 1

## Severity

- **P1**: docs contradict code, a required step is missing, or the same fact is stated in two places with different values
- **P2**: info exists but is hard to find, outdated, or scattered
- **P3**: nice-to-have improvements (better examples, diagrams, cross-links)

## Report Format

```
## Bus Test: <repo name>
Score: X/24 (<rating>)

### P1 (blocking)
- <file>: <what's wrong, one line>

### P2 (friction)
- <file>: <what's wrong, one line>

### P3 (polish)
- <file>: <suggestion, one line>

### Agent discoverability
- CLAUDE.md: <exists/missing/stale>
- Handoff docs: <current/stale/missing/none expected>
- Competing roadmaps: <yes/no — list files if yes>
- Duplication: <list any facts restated across files>

### Missing docs (by Diataxis quadrant)
- Tutorial: <exists/missing — guided learning path>
- How-to: <exists/missing — task recipes>
- Reference: <exists/missing — complete API/config surface>
- Explanation: <exists/missing — design rationale>
```

## After editing docs

- **Coherence check**: re-read the full document after edits. LLMs edit in diff-mode and produce sections that look good in isolation but create disjointed flow, repeated context, or contradictory statements when read end-to-end. The file must read as a coherent narrative, not a patchwork of additions.
- **Prose lint**: for changed checked-in docs prose, invoke `prose-lint` on the changed files or sections. Skip code blocks, generated content, schemas, exact API signatures, config examples, and intentional bad-prose examples unless the user asks to lint them.
- **Humanizer**: after prose-lint findings are addressed, invoke `humanizer` on changed docs prose before committing. Use the same skip list as prose linting. For audit-only docs reviews with no docs edits, use `humanizer` only when the user asks for voice, tone, or style feedback.

## In the review chain

`gate-probes` routes to this skill for checked-in docs changes. Run `gate-probes` first for universal scope and reviewability checks before applying this bus test.

## References

- [CNCF TechDocs Analysis Criteria](https://github.com/cncf/techdocs/blob/main/docs/analysis/criteria.md)
- [Diataxis Framework](https://diataxis.fr/)
- [Good Docs Project Templates](https://thegooddocsproject.dev/template)
