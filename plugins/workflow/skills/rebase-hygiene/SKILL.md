---
name: rebase-hygiene
description: |
  Use only when the user explicitly invokes it - never auto-load. The
  user's required rebase ritual for long-lived branches and worktrees:
  fetch before trusting any local state, dry-run the rebase and pre-vet
  the conflicting surface area, resolve conflicts per pre-planned
  resolutions rather than ad hoc, and verify the push actually reached
  the remote afterward.
compatibility: claude-code opencode
disable-model-invocation: true
license: MIT
---

# Rebase Hygiene

The user's rebase ritual. Run the four steps in order; do not start the
rebase itself until steps 1 and 2 are done.

## 1. Fetch first - never trust local state

- `git fetch origin` before any assessment. Never assume the local base
  is current without a fetch.
- In a worktree, confirm which checkout you are in (`git worktree list`)
  and that the branch tracks the remote you think it does
  (`git branch -vv`).

## 2. Pre-vet the conflicting surface before acting

- Enumerate both sides: `git log --oneline <base>..HEAD` and
  `git log --oneline HEAD..<base>`.
- Compute the overlap: files changed on both sides since the merge base
  (`git diff --name-only $(git merge-base HEAD <base>) HEAD` intersected
  with the same diff against `<base>`).
- Dry-run without touching the working tree:
  `git merge-tree --write-tree <base> HEAD` (git 2.38+). Conflict
  markers in its output are the real conflict list.
- Report the conflict surface before rebasing anything.

## 3. Resolve per pre-planned resolutions

- For each conflicting file, decide the resolution BEFORE starting:
  which side wins, or what the merged shape is, and why.
- Present the resolution plan to the user, then rebase and apply exactly
  those resolutions.
- If a conflict appears that was not pre-planned, stop and re-vet;
  never improvise mid-rebase.

## 4. Verify the push reached the remote

- Push rebased branches with `--force-with-lease`, never bare `--force`.
- Confirm the remote ref moved: `git fetch`, then
  `git rev-parse HEAD origin/<branch>` must match (or `git status`
  reports up to date). The push command exiting zero is not the check;
  the remote ref is.

## Failure modes this ritual exists to prevent

- Rebasing onto a stale base because the fetch was skipped.
- Hitting conflicts mid-rebase with no plan, then resolving ad hoc.
- Reporting "pushed" while the remote still shows the old head.
