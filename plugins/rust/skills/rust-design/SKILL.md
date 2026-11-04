---
name: rust-design
description: |
  Use when writing, designing, or reviewing Rust types and domain
  models — "design a Rust type", "model this in Rust", "Rust ADT",
  "state machine", "make illegal states unrepresentable",
  "constrained type", "typestate", "newtype design", "sketch type
  signatures", "write a Rust struct", "review these Rust types".
  Also use when authoring `.rs` files that introduce a new struct
  or enum, or when deciding a public API shape.
  Contains ADT-first workflow, constrained type patterns, railway-
  oriented programming, match semantics, 6-step clone avoidance,
  and ownership restructuring. Pair with rust-quality for anti-pattern
  prevention and rust-review for gate-level review.
compatibility: claude-code opencode
license: MIT
---

# Rust Design — Type-Driven Workflow

Model the domain with types before writing any logic. Enums represent
states and variants. Structs hold data within a single state. Newtypes
enforce semantic boundaries.

Compile and lint the type surface before any implementation;
`serde(untagged)` over enums with overlapping field shapes is a class of
bug you catch at the typing stage by inspection, because serde picks the
first matching variant in declaration order at runtime and the compiler
never flags it.

A skeleton that lands ahead of its bodies enters with a module-level
`#![allow(dead_code)]`, removed per phase as the holes fill. Prefer
`#[expect(...)]` wherever the lint will stop firing once the body lands,
so the marker warns when it stops being true; sweep the markers by hand
anyway, because one still-unused parameter keeps it quiet. `typed-holes`
carries the full workflow: skeleton commit and design panel, plus the
golden frames that track what the compiler cannot see.

## ADT-First Design

Make illegal states unrepresentable. Every `Option<T>` field and boolean
flag is a potential illegal-state leak. Replace with enum variants or
separate types.

```rust
// BAD: optional fields that only make sense in specific states
struct Connection { state: State, socket: Option<TcpStream>, error: Option<String> }
// GOOD: each variant carries exactly the data it needs
enum Connection { Idle, Connected { socket: TcpStream }, Failed { error: String } }
```

**State machines as type transitions.** Each state is a separate type.
Transitions consume one state and return the next. The compiler enforces
the valid transition graph — no path from Draft to Shipped.

```rust
struct DraftOrder { items: Vec<Item> }
struct PlacedOrder { items: Vec<Item>, placed_at: Instant }
impl DraftOrder { fn place(self) -> Result<PlacedOrder, OrderError> }
impl PlacedOrder { fn ship(self, tracking: TrackingId) -> ShippedOrder }
```

**Eliminate boolean flags with separate types.** "Verified" is a state
transition, not a property.

```rust
// BAD: caller must remember to check the flag
struct UserEmail { address: String, verified: bool }
// GOOD: type proves it
struct UnverifiedEmail(String);
struct VerifiedEmail(String);
impl UnverifiedEmail { fn verify(self, token: &Token) -> Result<VerifiedEmail, VerifyError> }
```

**"At least one required" via enum variants**, not runtime assertions.

```rust
// BAD: what if both are None?
struct Contact { email: Option<Email>, phone: Option<Phone> }
// GOOD: nonexistent combinations are unconstructable
enum Contact { EmailOnly(Email), PhoneOnly(Phone), Both { email: Email, phone: Phone } }
```

## Constrained Types

Between raw primitives and domain types: constrained types — private inner
field, single validating constructor. Downstream code trusts the value
without re-checking.

- **Newtype alias** (public field): when ANY value of the inner type is valid
  (UserId vs TeamId distinction).
- **Constrained type** (private field): when only SOME values are valid
  (email format, non-zero port, bounded range).

```rust
pub struct EmailAddress(String); // private field
impl EmailAddress {
    pub fn new(raw: &str) -> Result<Self, ValidationError> { /* validate */ }
    pub fn as_str(&self) -> &str { &self.0 }
}
impl TryFrom<&str> for EmailAddress { /* delegates to new() */ }
```

Implement `Display`, `AsRef<str>`, `Deref` (read-only). Never `DerefMut`
— bypasses the constraint. `#[cfg(test)]` helper constructors are fine;
never expose them outside `#[cfg(test)]`.

## Railway-Oriented Programming

`Result<T, E>` is a two-track railway. `?` switches to the failure track.
Never manually `match` on `Result` just to propagate — re-implementing `?`.

**Validation pipelines**: compose constrained types with `?`.

**Error composition**: errors compose via `#[from]` on wrapper enums, not matched-and-rewrapped.

```rust
// BAD: gauntlet of if-checks
if input.email.is_empty() { return Err(/*...*/); }
if !input.email.contains('@') { return Err(/*...*/); }
// GOOD: each type owns its validation
let email = EmailAddress::try_from(input.email.as_str())?;
let pass = Password::try_from(input.password.as_str())?;
```

**From-based error conversion** at boundaries. `impl From<io::Error> for
StorageError` — then `?` auto-converts. Don't let low-level errors leak
into domain signatures. Map at the boundary with `.map_err()`.

**Dead-end functions** ride on `.inspect()` / `.inspect_err()` — observe
without switching tracks.

**Option → Result**: `.ok_or_else(|| Error::NotFound(id))?` joins the
Option track onto the Result track.

## Match Semantics

`match` is for branching, not unwrapping. Use combinators for linear
`Option`/`Result` chains.

```rust
// BAD: nested match pyramid
match opt { Some(v) => match v.parse() { Ok(x) => ..., Err(e) => ... }, None => ... }
// GOOD: combinator chain
opt.map(|v| v.parse()).transpose()?.unwrap_or(default)
```

- `if let` for single-variant extraction with trivial `else`.
- Reserve `match` for 3+ variant enums with distinct logic per arm.
- Extract match arms >5 lines into named functions.
- Never `_` wildcard on owned enums — the exhaustiveness check is a feature.
- Never match on booleans. Use `if`/`else`.

## Ownership and Clone Avoidance

`.clone()` is a design decision, not a compiler fix. When the borrow
checker rejects, try IN ORDER before `clone()`:

1. Restructure scope — reorder let-bindings or split statements.
2. Take a reference — `&T` or `&mut T` instead of moving.
3. Narrow the borrow — extract borrowed access into a smaller scope.
4. `Cow<'_, T>` when you sometimes need owned, sometimes borrowed.
5. `Arc<T>` / `Rc<T>` for genuinely shared ownership.
6. Clone — with a brief comment explaining why.

**Defaults**: function params start as `&str` (not `String`), `&[T]` (not
`Vec<T>`). Only upgrade to owned when the function stores or moves the
value. Use `impl AsRef<str>` or `Into<String>` for public APIs.

**Split borrowing via destructuring** when methods need `&mut self` and
field reads:

```rust
// BAD: borrow checker conflict
fn update(&mut self) { let t = self.config.threshold; self.data.retain(|x| x > &t); }
// GOOD: destructure to get disjoint borrows (works for Copy fields too)
fn update(&mut self) { let Self { config, data, .. } = self; data.retain(|x| x.exceeds(&config.threshold)); }
```

## General Idioms

- `#[must_use]` on functions whose return value shouldn't be silently ignored.
- `#[non_exhaustive]` on public enums/structs that may grow variants.
- Derive order: `Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default, Serialize, Deserialize`. Skip what you don't need.
- Avoid `Box<dyn Error>` in library code. Use a concrete error enum.
- Early return / `?` over deeply nested blocks. The happy path is the least-indented code.


