---
name: prose-lint
description: Run before committing or posting any prose written on the user's behalf — commit messages, PR and issue text, release notes, checked-in docs/README/skill prose, Slack/email drafts, and public docstrings or doc comments. Also use when asked to run Vale, prose lint, style lint, or to check AI tells/slop. Reports Vale findings and assigns each a disposition (fix, suppress with recorded justification, or promote to config); never rewrites — for the rewrite use humanizer.
compatibility: claude-code opencode
license: MIT
---

# Prose Lint

Run deterministic prose checks with Vale. This skill reports findings and suggested fixes; it does not rewrite prose. Use `humanizer` after fixes when changed prose will be checked in or sent on the user's behalf. Never post replies or comments on the user's behalf; produce a draft or talking points — the user owns his own correspondence.

## Preconditions

1. Check whether Vale is available: `vale --version`.
2. If Vale is missing, report that prose linting was skipped. Do not install Vale unless the user asks.
3. Vale natively resolves `.vale.ini` from the project root → `~/.vale.ini` (global). No `--config` or `--no-global` needed — just run `vale` from the project directory. The global `~/.vale.ini` is managed by dotfiles and provides the ai-tells and local styles used here.
4. If synced styles are missing, run `vale sync`. If sync fails, report the failure and continue without prose linting.

## Input modes

### File mode

Use when prose already exists in files:

```bash
vale --output=JSON README.md CLAUDE.md docs/**/*.md plugins/**/*.md
vale --output=JSON plugins/python/skills/python-quality/SKILL.md
```

Prefer changed files or changed prose sections. Do not lint the whole repo unless the user asks for a full prose audit.

### Markdown stdin mode

Use when the agent is drafting Markdown-like prose before it exists in a file:

```bash
printf '%s' "$draft" | vale --ext=.md --output=JSON
```

Use for PR descriptions, issue comments, release notes, Slack/email drafts, and proposed README/docs paragraphs.

### Commit-message stdin mode

Use the commit-message rules by associating stdin with the commit-message path:

```bash
printf '%s' "$message" | vale --ext=.md --path=.git/COMMIT_EDITMSG --output=JSON
```

### Raw prose snippet mode

Use for small snippets that are not Markdown, such as public docstrings or API comments:

```bash
printf '%s' "$docstring" | vale --ext=.md --ignore-syntax --output=JSON
```

## Reporting

- Parse JSON findings by file, line, span, check, severity, match, and message.
- Prioritize `error` and `warning`; include `suggestion` findings when they point to a clear fix and are not noisy.
- For large output, summarize the top repeated issues instead of dumping raw JSON.
- Do not rewrite entire files automatically from a large Vale report. Suggest deterministic small fixes, then summarize remaining style suggestions for the user.
- Ignore code blocks, generated content, schemas, exact API signatures, config examples, and intentional examples of bad prose unless the user asks to lint them.
- `plugins/docs/skills/humanizer/**` is excluded from file-mode Vale because it contains intentional bad-prose examples. For changes to its non-example prose, lint the changed text with stdin mode.

## Disposition

Every error-level finding gets exactly one of three dispositions. An error with no disposition is unfinished work; scrolling past is not an option.

1. Fix the prose. The finding is right; change the text. The diff is the record.

2. Suppress inline with a recorded justification. The rule is right in general but wrong for this span. Vale's comment parser is strict — `<!-- vale Style.Rule = NO : reason -->` does not suppress (verified by test 2026-06-10). Put the justification in a separate adjacent comment:

```markdown
<!-- vale ai-tells.RuleName = NO -->
<!-- vale-reason: quoting reviewed text verbatim -->
The flagged span.
<!-- vale ai-tells.RuleName = YES -->
```

The reason comment is required — a suppression without one fails review. Suppressions are greppable: `grep -rn "vale-reason" .`

3. Promote to config. The finding is wrong everywhere in the repo (a recurring false positive). Edit the `local` style override or add a per-glob section in `.vale.ini`. Never edit files under a synced package directory such as `.vale/ai-tells/` — `vale sync` overwrites them. Durable overrides live in the `local` style with the package rule disabled in `.vale.ini` (`ai-tells.RuleName = NO` plus a `local/RuleName.yml` copy carrying the change).

Warning-level findings are judged against context; do not force them through this protocol — that would generate its own noise. The report to the user must state which disposition each error received.
