---
name: skill-retro
description: |
  Use when the user asks for a skill retro, says "retro time", "retro my
  skills", "vet my skills", or asks how skills performed or triggered this
  session. This reviews the skills, not the work: do not use for code review,
  sprint or project retros, or postmortems of the work itself.
compatibility: claude-code opencode
license: MIT
---

# Skill Retro

Audit how marketplace skills triggered and performed against the actual session
log, and file vetted correctives under the skills marketplace repo's feedback/
directory. Retro sessions usually run in the project under review, not in the
marketplace repo: resolve feedback/ against the marketplace checkout, not the
current working directory.

## Process

1. Ground every finding in the actual conversation log and the skill files on
   disk. Reconstruct which skills fired and which should have fired, and verify
   each cited skill's frontmatter and description by reading the file. Never
   assert a skill's trigger text from memory.

2. Classify each trigger finding by channel:
   - Entry-time phrase match. The user's message wording matches the skill
     description at task start. Only works at entry.
   - Mid-flow in-body cross-references. Once inside a loaded skill's flow, the
     skill list is no longer in fresh context; only imperative "load skill X"
     lines inside the loaded body fire at sub-task boundaries.
   - Artifact/action boundaries. Descriptions naming a file type or an action
     (about to commit, about to write a checked-in doc) fire at tool-use
     moments, unless the niche is already filled by CLAUDE.md or memory rules.
   - Deterministic hooks. Harness-level hooks are the only guaranteed channel.
     They are Claude-Code-only and must be a silent no-op elsewhere.

   Name the anti-pattern where it appears: redundancy suppression. When
   CLAUDE.md or memory duplicates a skill's content, the model satisfies the
   rule from the always-on copy and never loads the skill.

3. Vet findings with the user before writing any document, including the
   destination directory and the file naming. Do not write first and ask after.

4. File the retro under the marketplace repo's feedback/ directory (retro
   contents stay untracked; only the convention README is published):
   - One directory per session: feedback/<YYYY-MM-DD>-<harness>-<topic-slug>/
     where harness is claude-code, opencode, codex, or antigravity; same-day
     collisions take a -2 suffix.
   - The usual artifact is skill-retro.md; add plan-retro.md and
     transcript.md when applicable.
   - Frontmatter on every retro file: date, harness, agent (model id),
     session_event (handoff event id, if one exists), workstreams, repo.
   feedback/README.md carries the human-readable version of this convention.

5. Run every proposed corrective through the portability checklist:
   - Skill bodies stay tool-neutral: say "load skill X" and "ask the user",
     never a Claude-Code tool name.
   - Keep the compatibility frontmatter intact.
   - Mark Claude-Code-only mechanisms (hooks, settings.json) as
     Claude-Code-only; their absence elsewhere must be a no-op, never a
     broken instruction.
