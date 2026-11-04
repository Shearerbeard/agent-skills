---
name: prose-corpus
description: |
  Use when the user says "prose feedback", "capture this prose", "corpus
  this", "save this as a prose sample", or flags a draft as too verbose,
  overly prescriptive, or AI-sounding without asking for a rewrite. Also
  self-triggers before humanizer materially rewrites external prose (a
  commit message, ticket, or doc draft), and when the user hand-rewrites
  a draft the session produced: the pre-rewrite version is the sample.
  Snapshots the prose verbatim into boardkit's bench corpus inbox for
  the prose-reviewer bench. Not for process friction (use
  process-feedback) and not a rewrite skill (that is humanizer).
compatibility: claude-code opencode
license: MIT
---

# Prose corpus capture

Feed the prose-reviewer bench (boardkit card S10) with real specimens
of the failure it measures: external prose that drifted verbose,
overly prescriptive, or AI-sounding. A sample is perishable - the
wince-worthy draft is unreconstructable a week later - so capture
happens in the moment, before any rewrite improves it away.

## What counts as a sample

External prose written on the user's behalf: commit messages, GitHub
issue and PR text, release notes, checked-in docs, drafts of any of
these. The valuable version is the one BEFORE correction:

- When humanizer is about to materially rewrite such a draft, snapshot
  the pre-rewrite text first, then proceed with the rewrite.
- When the user hand-rewrites or heavily edits a draft the session
  produced, the session's original is the sample; the user's edit can
  ride along in the note line as the implied target.
- When the user flags existing prose (theirs to share, in a repo or
  ticket), capture it as found.

Ordinary assistant replies are not samples. Neither is prose the user
never flagged and no skill rewrote - do not trawl for specimens.

## Where it goes

The corpus inbox lives in the boardkit checkout at
`bench/prose/corpus-inbox/`, and its README is self-describing: read
the format section there before writing. One file per sample, named
`YYYY-MM-DD-short-slug.md`, a short provenance frontmatter block
(source, date, artifact type, optional note), then the prose verbatim.
Never trim, fix, or reflow the body - the bench needs the raw
artifact, tells and all. Prose lint does not apply to the sample body
for the same reason; it applies to the note line only.

The inbox is the only path this skill writes in the boardkit repo.
When boardkit is not checked out locally, save the sample in the
consuming repo's scratch space or the session scratchpad and tell the
user it needs hand delivery.

## Cautions

- Material under a never-publish rule (private governance artifacts,
  rust-holes content) stays out entirely; org-tracker material may be
  captured and is flagged for a keep-or-purge ruling at S10's user
  gate, as the inbox README states.
- One sample per file, no compilations. If a session produced three
  bad drafts, that is three files.
- If the sample also reveals process friction (a skill misfired, a
  gate was skipped), that finding is separate: file it through
  process-feedback, not here.

## Sunset

This skill is a stopgap. When card S10 closes with its golden corpus
frozen, the inbox stops accepting bulk intake and this skill retires
or narrows to rare-specimen capture; the S10 close-out decides which.
