---
name: process-feedback
description: |
  Use when the user says "file process feedback", "record process friction",
  "send feedback to boardkit", or names friction whose fix belongs in an
  upstream toolkit or process repo (boardkit, rust-holes, another kit this
  repo consumes) rather than in the current repo. Records the finding as a
  session-feedback entry and files it as a GitHub issue on the upstream
  repo. Not for skill-trigger retros (use skill-retro) and not for code
  review findings.
compatibility: claude-code opencode
license: MIT
---

# Process feedback

Record friction with an upstream toolkit or process repo so its maintainer
can act on it, without editing that repo's content from a consumer session.
Two writes: a local record under the skills marketplace repo's feedback/
directory, and a GitHub issue on the upstream repo.

## Ground rules

- Ground the finding in the actual session: what happened, where, and what
  the friction cost. Never assert from memory.
- Vet the finding with the user before writing either copy. Show the issue
  title and body before filing; the user approves the exact text.
- The upstream repo is touched in exactly one way: one new issue through
  `gh issue create`. Never edit its templates, code, docs, or files, and
  never scaffold anything in it. If the upstream has no reachable issue
  tracker, keep the local record only and tell the user it needs hand
  delivery.
- An entry proposes; the upstream maintainer disposes. Do not phrase the
  issue as a commitment that the kit will change.

## The local record

File the finding under the marketplace repo's feedback/ directory using
the convention feedback/README.md owns: one directory per session,
feedback/<YYYY-MM-DD>-<harness>-<topic-slug>/, harness vocabulary
claude-code, opencode, codex, or antigravity, same-day collisions taking a
-2 suffix. The artifact for this skill is process-feedback.md, with the
standard frontmatter (date, harness, agent, session_event, workstreams,
repo) and the finding in prose: what happened, why it is upstream
material, and the candidate fix if one is apparent. Retro sessions run in
the consuming project, not the marketplace repo: resolve feedback/ against
the marketplace checkout.

## The upstream issue

File the finding as an issue on the upstream repo:

```sh
gh issue create --repo <owner>/<repo> \
  --title "process feedback: <short slug>" \
  --body <body>
```

The body opens with the local record's frontmatter values (date, harness,
agent, workstreams, repo) as a fenced yaml block, then the finding in
prose: what happened, why it is upstream material, and the candidate fix
if one is apparent. End with a source line pointing at the local record's
path, so the two copies cross-reference. Resolve <owner>/<repo> from the
upstream checkout's origin remote when one exists locally; otherwise ask
the user. For boardkit that is Shearerbeard/boardkit.

`gh` must be authenticated for the target repo, and a failed create is a
failed mirror - never a reason to edit files in an upstream checkout.

## After writing

Read the local record back, and confirm the issue landed by the URL `gh`
returns. Nothing in an upstream checkout changes; this skill makes no
writes there at all, including during its own error handling. Where a gate
or commit follows, prose lint applies to the entry like any other
checked-in prose.
