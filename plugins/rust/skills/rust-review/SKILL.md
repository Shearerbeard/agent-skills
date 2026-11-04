---
name: rust-review
description: |
  Use for Rust reviews: .rs diffs, PRs with Rust changes, Cargo.toml changes,
  and clippy/pre-commit checks. Run gate-probes first for universal checks, then
  these Rust-specific probes. Always load before presenting Rust code review
  findings to the user.
compatibility: claude-code opencode
license: MIT
---

# Rust Review Gates

Before applying probes, load `rust-quality` — it contains the anti-pattern rules and preferred patterns you must check against. Without it loaded, you will miss clone escapes, speculative fallbacks, and weak error modeling that are invisible from training data alone. If `gate-probes` has not already run for this diff, run it first for universal checks. Then apply these Rust-specific probes against the diff.

If the diff changes public docs, public API doc comments, README content, release notes, or PR prose, invoke `prose-lint` on changed prose only. For doc comments, pass the changed text via stdin. Use `humanizer` only for prose that will be checked in, published, or sent on the user's behalf.

## Rust probes

1. **Clone escape?** — Did we `clone()` to satisfy the borrow checker instead of restructuring ownership?
2. **Wildcard enum?** — Any `_ =>` that will silently absorb future enum variants?
3. **Transitive deps?** — Any `use` of a crate not in `[dependencies]`? Compiling via transitive dep is fragile.
4. **Redundant closure?** — Any `.map_err(|e| Variant(e))` where `Variant` has `#[from]` and `.map_err(Variant)` works?
5. **Type complexity suppressed?** — Any `#[allow(clippy::type_complexity)]` that should be a type alias?
6. **Single-arm match?** — Any `match` with one arm + `_ => {}` that should be `if let`?
7. **Nested match over combinators?** — Nested `match` on `Option`/`Result` where `.map()`, `.and_then()`, `.map_err()` would be flatter and clearer?
8. **Comment noise?** — Any comment that restates an identifier, type, or the next line, narrates a change ("previously", "now uses"), or describes behavior owned by other code?
