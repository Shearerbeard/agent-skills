---
name: design-system
description: >-
  Use when creating or evolving a visual design system - choosing a palette,
  typography scale, spacing rhythm, and one signature element that artifacts
  and pages then consume - or when asked for brand colors, design tokens, a
  theme, or a visual identity for a project or team. Produces a small token
  store (philosophy, tokens.json, tokens.css) under
  ~/.local/share/design-system/ that the web-artifact skill and any other
  generator can consume. Also use when reviewing a design for AI-default
  slop: the cream-plus-serif-plus-terracotta look, the
  near-black-plus-acid-accent look, purple gradients, excessive centered
  layouts, uniform rounded corners, Inter by default.
compatibility: claude-code opencode kimi
license: MIT
---

# Design System

Author a small, persistent token system that every visual artifact in a project
draws from, so a run of infographics, flow charts, and pages reads as one
product instead of a pile of one-offs. The output is three files in a
harness-neutral store, not a website: other skills (starting with
`web-artifact`) do the rendering.

Work model-agnostically. The process below is a thinking discipline, not a
toolchain; it runs the same under Claude Code, OpenCode, and Kimi.

## The token store

One directory per system, in a location every harness can reach:

    ~/.local/share/design-system/<system-name>/
        philosophy.md    why the system looks the way it does (the manifesto)
        tokens.json      the machine-readable tokens (shape below)
        tokens.css       CSS custom properties, generated from tokens.json

Keep the store out of any single repo: artifacts in many projects may consume
one system. If the user wants a system versioned with a project instead, put
the three files in that repo (for example design/tokens.json) and record the
absolute path - name alone cannot find a project-local store - in the
consuming artifact's config.

`tokens.css` is generated, never hand-edited: after changing tokens.json,
regenerate it so the two cannot drift. Say so in a header comment in the CSS.

## Process: philosophy, tokens, critique

**Pass 1 - philosophy.** Before any hex value, write philosophy.md: a short
manifesto naming the subject, the audience, the mood, and what this system
refuses to look like. Distinctive choices come from the subject's own world -
its materials, instruments, and vernacular - not from a mood board of other
software. Three to six sentences is plenty; this file exists so future
revisions (and future sessions) can tell a deliberate choice from a drift.

**Pass 2 - tokens.** Derive a compact token system from the philosophy:

- **Color**: named hex values with roles, each in a light and a dark
  variant. The consumer contract needs these roles: ink, paper, accent,
  accent2, muted, warn, danger, border, surface (code/table background).
  Name them for their role in the subject's world internally if you like,
  but tokens.json keys stay these role names.
- **Type**: typefaces for 3 roles - a characterful display face used with
  restraint, a complementary body face, and a utility face for captions and
  data - plus a modular scale (ratio and 5-7 steps with px sizes).
  Prefer system or freely-hostable stacks: a web artifact inlines
  everything, so a licensed web font that cannot be embedded is a trap.
- **Space**: a 4px base grid, spacing steps as multiples, and the corner
  radius the system allows.
- **Signature**: the single memorable element - a texture, a rule style, a
  numeral treatment, a motion beat - that embodies the philosophy. Spend the
  boldness here and keep everything around it quiet.

**Pass 3 - critique.** Read the plan back against the brief and the slop list
below. If any choice is what you would have produced for any similar brief,
revise it and note what changed and why in philosophy.md. Only then write the
token files.

## The slop list (calibration)

AI-generated design clusters around a few defaults. Each is legitimate when
the brief asks for it; none should be spent by accident:

- warm cream background, high-contrast serif display, terracotta accent
- near-black background with a single acid-green or vermilion accent
- broadsheet layout with hairline rules and dense newspaper columns
- purple gradients, excessive centered layouts, uniform rounded corners,
  Inter as the default face

The brief's own words always win, including when they ask for one of these.

## tokens.json shape

Keep it flat and role-named; this exact shape is what `web-artifact`
consumes. Every color role must appear in both light and dark:

    {
      "name": "harbor-log",
      "version": 3,
      "color": {
        "light": { "ink": "#...", "paper": "#...", "accent": "#...",
                   "accent2": "#...", "muted": "#...", "warn": "#...",
                   "danger": "#...", "border": "#...", "surface": "#..." },
        "dark":  { "...": "same nine roles, dark values" }
      },
      "type": {
        "display": { "stack": "...", "weight": 700 },
        "body":    { "stack": "...", "weight": 400 },
        "utility": { "stack": "...", "weight": 500 },
        "scale":   { "ratio": 1.25, "steps": { "s": 13, "m": 16, "l": 20, "xl": 25, "xxl": 31 } }
      },
      "space": { "base": 4, "radius": "8px",
                 "steps": { "1": 4, "2": 8, "3": 12, "4": 16, "6": 24, "8": 32 } },
      "signature": "..."
    }

Bump version on every change and log one dated line per change at the bottom
of philosophy.md. Consumers pin nothing; they take the latest, so a version
bump plus the changelog line is how a refresh explains why a page's look
moved.

## tokens.css generation

Emit CSS custom properties under :root, dark values inside a
prefers-color-scheme media query. The variable names are the contract -
consumers reference exactly these:

    /* GENERATED from tokens.json (harbor-log v3) - edit the JSON, not this file. */
    :root {
      --ink: ...; --paper: ...; --accent: ...; --accent2: ...;
      --muted: ...; --warn: ...; --danger: ...; --border: ...; --surface: ...;
      --radius: ...;
      --font-display: ...; --font-body: ...; --font-utility: ...;
      --weight-display: ...; --weight-body: ...; --weight-utility: ...;
      --text-s: ...; --text-m: ...; --text-l: ...; --text-xl: ...; --text-xxl: ...;
      --space-1: 4px; --space-2: 8px; --space-3: 12px; --space-4: 16px;
      --space-6: 24px; --space-8: 32px;
    }
    @media (prefers-color-scheme: dark) {
      :root { --ink: ...; /* ...all nine colors, dark values... */ }
    }

The generated header carries the system name and version so a consumer can
credit the exact tokens a page was built with.

Keep selectors to :root only - no element styling - so the file composes
with any template. Colors go in both schemes; type, weight, and space
variables are scheme-independent, so declare them once.

The signature token is guidance for the artifact author, not a CSS value:
`web-artifact` reads tokens.json (or philosophy.md) for it and expresses it
by hand in the page - the CSS carries only what custom properties can carry.

## Consuming and reviewing

- When `web-artifact` (or any generator) builds a page, it reads the named
  system, inlines tokens.css, and maps the template's variables onto these
  roles. If no system is named, the generator asks which to use or offers to
  run this skill first.
- When asked to review a design against a system, check that every color and
  type choice traces to a token; a value that matches no token is either a
  bug or a proposal to extend the system - make the author pick.
- Prose in philosophy.md should go through `prose-lint` (and `humanizer` if
  flagged) when those skills are installed; under a harness without them,
  give the file one plain-language read-through instead.
