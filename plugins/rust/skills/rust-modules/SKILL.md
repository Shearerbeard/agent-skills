---
name: rust-modules
description: |
  Use for any Rust module work: planning, creating, splitting, moving, discussing,
  or reviewing modules and files: mod declarations, pub use facades, crate
  structure, mod.rs cleanup, type co-location, and naming. Pair with rust-quality
  for code patterns inside modules.
compatibility: claude-code opencode
license: MIT
---

# Rust Module Layout

Rules for file organization. For code quality within modules, see `rust-quality`. Do not load `rust-review` unless reviewing a diff.

## In the review chain

`gate-probes` routes to this skill for Rust module layout changes. Run `gate-probes` first for universal scope and reviewability checks before applying module layout rules.

## File layout (new code)

No `mod.rs`. Use the modern sibling pattern:

```text
src/
├── lib.rs
├── billing.rs          ← declares `mod invoice;`
└── billing/
    └── invoice.rs
```

## Re-exports (facade pattern)

Internal trees can nest freely. Public API must be flat. Re-export in the parent:

```rust
// billing.rs
mod invoice;
pub use invoice::{Invoice, InvoiceState};
```

Callers write `use crate::billing::Invoice`, not `use crate::billing::invoice::Invoice`.

## Co-locate related types

Tightly coupled types share a file. `Invoice`, `InvoiceState`, `InvoiceError` all belong in `invoice.rs` — not split across `models.rs`, `enums.rs`, `errors.rs`.

## Anti-stuttering

Module already provides namespace context. Don't repeat it:

- `billing::Invoice` not `billing::BillingInvoice`
- `config::Source` not `config::ConfigSource`

## When to split

- A file exceeds ~400 lines AND contains distinct domains
- You need a privacy boundary (internal helpers hidden from the rest of the crate)
- Do NOT split just for size — a cohesive 500-line module beats three fragmented ones

## Legacy code

If an existing codebase uses `mod.rs`, match that pattern when editing. Only modernize on explicit refactor requests — not drive-by fixes.
