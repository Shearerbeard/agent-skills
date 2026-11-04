---
name: git-commit
description: Use when writing a commit message or preparing a git commit. Covers conventional commit format, message structure, prose quality, and mandatory user review before committing. If a github-workflow skill is available, pair with it for branch naming and PR conventions.
compatibility: claude-code opencode
license: MIT
---

# Git Commit Conventions

Run `gate-probes` on the staged diff before writing the message, if it has not already run for this diff.

## Format

```
<type>(<scope>): <short description>

<body explaining what and why>

<optional footer>
```

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

Scope is optional. When used, it names the module or area changed: `feat(parser):`, `fix(auth):`.

The short description is imperative mood, lowercase, no trailing period: "add retry logic", not "Added retry logic."

## Body

The body explains *what* changed and *why*, not *how*. Wrap at 72 characters. Use a blank line between the subject and body.

Keep the body brief; details live in the diff:

- Do not restate the diff per file
- Never count tests or claim "has tests"; the review process owns that
- Do not narrate the symptom the change fixes
- Do not replay the task history; describe the result, not the session

For multi-point changes, use a plain list:

```
feat: add --json flag and exit codes to sync status command

Add machine-readable output for CI/CD workflows:

- Add --json flag for structured JSON output
- Exit code 0 when up to date
- Exit code 1 when changes detected
- Exit code 2 on error
```

## Forbidden content

Never add sign-off trailers or AI attribution to commits unless the user explicitly requests them or the repository requires them (for example, DCO sign-off):

- No `Signed-off-by` trailers
- No `Co-authored-by` lines with AI names
- No `Generated with` or `Generated-by` footers
- No AI signatures or attribution of any kind
- No robot emoji taglines

Commits contain only: type, description, optional body, and optional footers (`Ref:`, `Fixes:`).

## Prose quality

Before presenting the commit message, invoke `prose-lint` on the message body to catch AI tells. If the body reads like AI-generated text, invoke `humanizer` to rewrite it. Short single-line commits without a body skip this step.

## User review

Always present the full commit message to the user before committing. Show the exact text that will be committed. Do not commit without explicit user approval. Do not batch multiple commits without individual review of each message.

## Validation

If the project has a `.commitlintrc`, `.commitlintrc.js`, `.commitlintrc.json`, or `commitlint.config.js`, run commitlint after committing to verify the message format:

```bash
npx commitlint --edit
```

If commitlint fails, amend the message and re-validate. Do not skip this step or ask whether to run it.

If no commitlint config exists in the project, skip validation silently.

Repos with the ai-tells-commits Vale gate (wired to COMMIT_EDITMSG) enforce the test-count, file-listing, and overexplanation rules deterministically.
