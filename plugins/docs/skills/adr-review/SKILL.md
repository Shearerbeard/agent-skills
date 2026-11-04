---
name: adr-review
description: Use when writing, reviewing, or revising an architecture decision record - an ADR, MADR, design-decision note, a docs/adr/ file, or a DECISIONS.md entry. Checks decision structure (context, RFC-2119 drivers, considered options, chosen outcome, honest consequences), forward-compatible dependency links, and status lifecycle, then re-verifies that the record's time-sensitive premises still hold (premise-freshness). Chains prose-lint then humanizer on the prose before commit. For GitHub issues and PRs use github-workflow; this covers decision records.
compatibility: claude-code opencode
license: MIT
---

# ADR Review

Review an architecture decision record so the decision is justified, the record
is complete and honest, and its premises still hold. An ADR captures *why* a
choice was made at a point in time. Reviewing one differs from reviewing prose
or code: you are checking that the reasoning survives scrutiny and that the world
it assumed is still real.

This is not an ADR generator. It strengthens a draft or an existing record; it
does not emit boilerplate.

## What a sound ADR has

Grounded in MADR (Markdown Any Decision Record) structure. Score each section
present or absent; flag the weak ones.

### Structure

- [ ] **Title states the decision** in plain language ("Human-in-the-loop
  approval gating for agent tool calls"), not a vague topic ("HITL stuff").
- [ ] **Status, date, deciders** present. Status is one of proposed, accepted,
  deprecated, or superseded-by-ADR-X.
- [ ] **Context and problem statement** names the forces, who is blocked, and the
  constraints that bound the choice.
- [ ] **Decision drivers** are the criteria that pick between options, each tied
  to a concrete need or stakeholder.
- [ ] **Considered options** are at least two genuine candidates, not one option
  and two strawmen.
- [ ] **Decision outcome** names the chosen option plainly and connects it back
  to the drivers.
- [ ] **Consequences** cover both the positive and the negative.
- [ ] **Links** reach the issue or technical story, related and superseded ADRs,
  any spike, and the design note when detail is deferred.

### Drivers separate hard from soft with RFC-2119

- [ ] Non-negotiable constraints use MUST or MUST NOT ("the gate MUST fail
  closed").
- [ ] Preferences use SHOULD or SHOULD NOT, with the condition that would flip
  them ("routing SHOULD be config; revisit if one deployment ever serves both
  shapes at once").
- [ ] A driver a reader cannot test or disagree with is too vague. Sharpen it.

### Options are real and the rejections say why

- [ ] Each rejected option records the specific reason it lost, in its own terms.
- [ ] Options evaluated in depth are kept distinct from secondary ones mentioned
  but not pursued (MADR's "considered" versus "alternative" split).
- [ ] A partially-kept option is marked as such ("kept as Route A, rejected as
  the whole story"), not silently dropped.

### Consequences are honest

- [ ] Negative consequences are listed, not just benefits. A record with only
  upside is under-reviewed.
- [ ] Known gaps are named as gaps with an owner: a follow-up ADR, an issue, or a
  roadmap line ("neither ingress nor egress authenticates; a named gap for the
  roadmap").
- [ ] Failure modes are stated: what happens on timeout, denial, disconnect, the
  unhappy path.

### Forward compatibility

- [ ] Work deferred to another decision links to the ADR or issue that owns it
  ("durable parking is the #209-gated successor, not this work").
- [ ] Contracts the ADR introduces are shaped so the deferred work attaches
  without reworking callers, and the ADR says so.

### Status lifecycle

- [ ] An accepted ADR is not silently rewritten when the decision later changes.
  A new ADR supersedes it and the two link to each other.
- [ ] A superseded or deprecated ADR points forward to its replacement.

## Premise-freshness check

ADRs encode premises that were true when written and quietly rot. On any review
or revision, re-verify each premise against the current code and decisions:

- **Stated-absent facts** ("no durable cross-request state exists yet", "the
  server has no auth layer today"). Grep the code or check the referenced issue.
  If the fact has changed, the decision may no longer hold.
- **Pinned references** ("a spike on branch X @ commit Y", "builds on PR #N").
  Confirm the branch, commit, or PR still exists and still says what the ADR
  claims.
- **Dependency status** (an ADR that "depends on #209 for durable parking").
  Check the dependency's current status; one that shipped, stalled, or was
  rejected changes this ADR's standing.
- **Quantitative claims** (counts, limits, "single-pod until", version numbers).
  Re-confirm against reality.

Report each premise as still-true, changed, or unverifiable. A changed premise
is at least a P1: the decision rests on it.

## Severity

- **P1**: a missing decision outcome or driver, an option set with no real
  alternative, a consequence section with no downsides, or a premise that has
  changed since writing.
- **P2**: vague drivers (no RFC-2119, untestable), a rejected option with no
  stated reason, a deferred dependency with no link, or a stale date with no
  review note.
- **P3**: a title that could be sharper, missing cross-links, or formatting drift
  from the house ADR style.

## Report format

```
## ADR Review: <title>
Status: <proposed/accepted/...>   Structure: X/8 sections

### P1 (blocking)
- <section>: <what is wrong, one line>

### P2 (friction)
- <section>: <what is wrong, one line>

### P3 (polish)
- <section>: <suggestion, one line>

### Premise freshness
- <premise>: still-true | CHANGED (<what changed>) | unverifiable (<why>)
```

## After editing the ADR

- **Coherence check**: re-read the whole ADR end to end. Decision records get
  edited section by section and drift into restating context or contradicting an
  earlier driver. It must read as one argument.
- **Prose lint**: invoke `prose-lint` on the changed prose. Skip code blocks,
  schemas, wire formats, config tables, and the RFC-2119 keywords themselves -
  Vale may flag MUST or SHOULD as shouting, which is intentional ADR vocabulary,
  so suppress with a recorded justification rather than rewording.
- **Humanizer**: once the prose-lint findings are handled, invoke `humanizer` on
  the changed prose before committing. An ADR is checked-in, externally-read
  text.

## In the review chain

`gate-probes` routes here for ADR and design-decision changes. Run `gate-probes`
first for the universal scope and reviewability checks, then apply this review.
For GitHub issues and PRs, that path is the github-workflow skill, not this one.

## References

- [MADR - Markdown Any Decision Records](https://adr.github.io/madr/)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
- [Diataxis: Explanation](https://diataxis.fr/explanation/) - an ADR is
  explanation-quadrant documentation
