# Skill Testing Guide

How to run a proper skill testing session. Covers both trigger testing (did the skill load?) and enforcement testing (did the model follow what the skill said?).

## When to Test

- After changing any SKILL.md description field
- After adding or removing intra-skill routing instructions
- After adding checklists or report formats to a skill
- Before running `./bin/install-skills` to push changes to editors
- After a real session reveals a skill miss (like session-008)

## Setup

### Fresh session per tool

Each editor needs a clean session. Do not reuse sessions across test cases — the model's context from earlier prompts will skew results.

### Model selection

Test with the model you actually use. Different models route skills differently:

- **Qwen 3.6 Plus** (opencode default for this user): more willing to load skills based on description matching
- **Claude Sonnet** (Claude Code default): responds from training for planning/review tasks, loads skills less often
- **Smaller models**: may skip skills entirely or follow checklists less reliably

Record which model was used for each test. Results are model-specific.

### Project selection

Use a real project with actual code, not a synthetic test repo. The model's behavior changes when there's real code to search, real files to edit, and real context to consider. Good candidates:

- This repo (agent-skills) for docs/prose skills
- A Rust project for rust-review, rust-quality, rust-modules
- A Python project for python-review, python-quality

### Install after gates pass

```bash
./bin/check-skills
./bin/check-prose
./bin/install-skills opencode
```

For Claude Code, no install needed — it reads from the marketplace plugin path.

## Running Tests

### Step 1: Trigger test

Use the prompts from the skill-test-matrix. Record whether the expected skill loads:

- `auto`: loaded without naming it
- `nudge`: loaded after a hint like "use the relevant skill"
- `manual`: only works with explicit invocation
- `miss`: never loaded

### Step 2: Enforcement test

If the skill loaded, check whether the model followed its instructions:

**For skills with checklists (plan-discipline):**
- Did the model complete all checklist items?
- Or did it just acknowledge the checklist and skip it?
- Look for the actual checkbox output or equivalent structured response

**For skills with report formats (gate-probes, docs-bustest):**
- Did the model produce the expected output format?
- gate-probes: table with PASS/FAIL per probe, gate verdict
- docs-bustest: score out of 24, P1/P2/P3 findings, Diataxis quadrant analysis

**For skills with intra-skill routing (rust-review, python-review):**
- Did the model load the next skill in the chain?
- rust-review should load rust-quality before applying probes
- python-review should load python-quality before applying probes
- gate-probes should route to language-specific review skills after universal probes

### Step 3: Record results

Fill in the skill-test-matrix table. Also note:

- What the model actually did (even if wrong)
- Any unexpected behavior
- Whether the model followed the skill's "why" explanations

### Step 4: Compare across models

Run the same prompts in different editors/models. Record differences. Claude Code often responds from training where opencode loads skills — both are valid, but the difference matters for skill design.

## What to Look For

### Good signs

- Skill loads on first prompt without naming it
- Model completes checklists item by item
- Model produces the expected report format
- Model chains to the next skill as instructed
- Model explains its reasoning using the skill's language

### Bad signs

- Skill never loads (trigger description gap)
- Skill loads but model ignores its instructions (enforcement gap)
- Model acknowledges the skill but does something different (attention gap)
- Model follows the checklist superficially without doing the actual work (checkbox theater)
- Model loads the skill but skips the intra-skill routing (routing gap)

### Checkbox theater

A specific failure mode for skills with checklists: the model prints all the checkboxes as complete without actually doing the work. Example:

```
- [x] Scope interview answered
- [x] Verification method stated
- [x] Blast radius identified
```

But the actual response has no scope interview answers, no verification plan, no blast radius analysis. The model treated the checklist as formatting, not as a work product.

To detect this: check whether each checklist item has actual content, not just a checkmark.

## Recording Results

Save session transcripts or notes under `docs/internal/testing/results/`. Name files by date and editor:

- `2026-06-03-opencode-qwen.md`
- `2026-06-03-claude-sonnet.md`

Include:

1. The prompt used
2. Whether the skill loaded (and how)
3. Whether the model followed the skill's instructions
4. What the model actually did
5. Any unexpected behavior
6. Model name and editor version

## Cross-Model Notes

Skills are model-facing. Different models will route differently. This is expected, not a bug. Design skills for the model you use most, but test in both editors to catch gaps.

Key differences observed so far:

- opencode loads skills more readily from description matching
- Claude Code responds from training for planning/review tasks
- Both load skills when explicitly invoked
- Smaller models may skip skills or follow checklists less reliably

See session-007 for detailed cross-platform comparison.
