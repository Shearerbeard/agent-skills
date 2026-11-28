---
name: typed-holes
description: |
  Use when the user says "typed holes", "todo!() skeleton", "type
  skeleton", "compile the type surface", "skeleton commit", "design
  panel", "golden frame", "whole-frame snapshot", "pre-failing test",
  "write the tests first from the spec", "byte-identical refactor", or
  asks how to start a Rust feature, module, or rewrite before writing
  any behavior. Also use when planning how to split Rust work across
  cards, issues, or delegated executors. Contains the two-layer
  practice: Layer 1 lays the full Rust type surface with todo!() bodies
  that must pass cargo check and clippy as their own commit, gated by a
  one-reviewer skeleton review; Layer 2 writes whole-frame golden tests
  from the spec so they fail on arrival. Also covers the #[expect]
  marker convention, the type-to-business-rule map, the coverage
  manifest, fill order, and the known limits of both signals. Pair with
  rust-design for the type modeling itself, rust-quality for
  anti-patterns in the filled bodies, and gate-probes at each gate.
compatibility: claude-code opencode
license: MIT
---

# Typed Holes and Layered Red-Green

Two layers of machine-tracked holes, filled over time.

- **Layer 1, type-checked surface.** Lay the full type surface with
  `todo!()` bodies. The compiler and clippy verify the design composes
  before any behavior exists. What is still missing is tracked by the
  hole inventory, not by the compiler: `todo!()` is a diverging panic, so
  it type-checks wherever it sits.
- **Layer 2, test-runner-tracked.** Write whole-frame golden tests from
  the spec before the implementation, so they fail on arrival. The test
  runner tracks the rendering and semantics holes the compiler cannot
  see.

The layers are complements, not alternatives. The compiler checks the
surface while the hole inventory tracks the behavior holes; the failing
golden tests track rendering holes. A module without rendered output uses
Layer 1 with spec-derived behavioral tests. Layer 2 adds whole-frame
goldens where emitted artifacts exist; snapshot testing alone is not the
complete practice.

Everything needed to run the practice is in this skill. A companion
repo of templates and worked exemplars augments it where the user has
one; its absence changes nothing about the workflow below. When
writing or updating that companion repo's playbook, templates, or
exemplars, load this skill first and treat it as the canonical statement
of the practice; divergence between the two is a defect in the
companion, not a fork of the practice.

## Execution flow

```text
skeleton (compile/build) -> skeleton review -> optional types U
  -> tests (compile, then named red) -> test review
  -> fills (green + no assigned holes) -> implementation review
  -> integration (candidate checks + review any new code)
  -> optional approved Frontier -> final U
```

The coordinator judges every review return and routes repairs or API/scope
friction to the owning stage. The board process owns review-slot and
delivery accounting; standalone work names its coordinator explicitly.

## Dispatch inputs and acceptance

Before authoring, name the specification path/revision and its approval
state, the exact input commit, allowed edits and required checks. Missing
or unapproved requirements return to the spec owner; a skeleton is not a
way to infer a product specification. Verify the assigned agent can load
this skill through its effective permissions. An executor without skill
loading receives the complete applicable rules quoted in its brief.
Installation alone does not establish access.

Assign test authoring and integration-source repairs to named roles before
fills start, with reviewers eligible for every actual author in the range.
Git integration by the board owner is distinct from authoring glue or
conflict resolutions. A takeover updates authorship and reviewer eligibility.

Map each test to the bodies it requires. Group or order coupled fills so
each assignment can pass its named tests; a test needing a later body stays
in the intermediate red set. Record that set and remaining holes separately
from new failures. The exact
Clippy command must agree with that partition: remove the todo allowance
only when no remaining authorized holes fall under the command's scope.
The complete candidate must pass all required checks.

The test assignment names who maintains skeleton lint markers made obsolete
by new test references and either authorizes that mechanical edit or returns
it to the type owner. New compile/lint failures remain unexpected even when
hole failures are authorized.

## Shared assignment boundary

Every assignment names the approved specification revision, input commit, allowed files, permitted edits, required checks, and expected outputs. Test authors and fillers stop the affected work if completion requires changing the approved API, invariants, dependencies, visibility, fixture access, or assignment scope. Return the smallest failing usage, expected contract, observed blocker, and decision needed to the coordinator. Do not add a workaround, weaken expectations, or reinterpret the brief to make it pass. The coordinator determines whether the repair belongs to the skeleton owner, test owner, filler, or operator, and authorizes the revised scope before redispatch. Independent in-scope work may continue.

## Test-author assignment

Partition authoring by coherent behavior and fixture prerequisites before
dispatch: ordinary contract tests and backend-boundary fixtures may need
different inputs. Return a new dependency, platform prerequisite or missing
seam to the owner before adopting a workaround. A changed implementation
shape returns for scope review; estimates are not line quotas.

Derive complete expected values from the approved spec and fixture inputs.
An append return reused as the expected read result can hide a defect shared
by both paths. Check returned values and persisted/replayed values against
independent expectations where the contract requires both.

Use the smallest scenarios that distinguish the named wrong implementations.
Share repetitive setup within a suite without building a parallel implementation.
Keep separate adapter and runtime checks when they reject different failures.
Record the spec-to-test/body map once, with exclusions and evidence links;
reference the spec instead of copying its narrative into tests and manifests.

## Layer 1: the typed-holes skeleton

### What lands

The full type surface, and nothing else:

- Real signatures, real derives, real `From`/`Into` impls, real leaf
  types. No placeholder `String` standing in for a type not yet written.
- `todo!()` appears **only** at real-behavior bodies: parsing, rendering,
  assembly, anything with a decision in it.
- Trivial accessors are implemented, not held open. A field read, or an
  `as_str` over an already-validated value, is not a behavior hole;
  leaving it as `todo!()` hides the type surface from the compiler
  instead of exposing it.
- Fallible constructors return `Result<_, YourError>` with the error enum
  fully written. An error type invented later is a type-surface change,
  which is the thing this step exists to catch early.

Design rules the surface must satisfy are `rust-design`'s: invalid states
unrepresentable, parse rather than validate, no bare domain value across
the public boundary (including through error payloads), public fields
only where every field is itself a validated type.

The boundary rule is scoped to domain values. A diagnostic payload in an
error variant may stay raw text where the variant name and its docs mark
it diagnostic-only and no domain logic branches on it: an unparsed
response body reported next to an unexpected status is the usual case.
Once a caller matches on the contents, it is a domain value and needs a
type.

### The skeleton commit

The skeleton must pass `cargo check` and `cargo clippy` before any body
is implemented, and it lands as **its own commit**. The commit is the
point: it makes the design panel's input a retrievable git object rather
than a conversation. An auditor can check out the skeleton and see
exactly what was reviewed.

Gate the skeleton commit on:

```
cargo check --workspace --all-targets
cargo clippy --workspace --all-targets -- -D warnings
cargo +nightly fmt --check
```

**Toolchain prerequisites.** `#[expect]` needs Rust 1.81 or newer; on an
older toolchain use `#[allow(unused_variables)]` with a tracking comment
naming the card or issue that fills the body. The `+nightly` form needs a
nightly toolchain installed; where only stable is available, run plain
`cargo fmt --check`.

### The `#[expect]` marker convention

Every function that has a `todo!()` body **and at least one parameter**
carries:

```rust
#[expect(unused_variables, reason = "todo!() body; filled by <card or issue>")]
```

Use `expect`, not `allow`. While the lint fires the two behave alike, and
once it stops firing `expect` warns, so the marker asks to be removed at
the point it stops being true. Two limits belong in the same breath,
because the marker is easy to oversell:

- **A zero-parameter hole has nothing to mark.** With no parameters there
  is no `unused_variables` to expect, the expectation is unfulfilled the
  moment it is written, and `unfulfilled_lint_expectations` fails the
  skeleton's own `-D warnings` gate. Leave those functions unmarked; the
  hole inventory tracks them.
- **A filled body can keep its own marker alive.** The expectation goes
  unfulfilled only when the filled body uses *every* parameter, so a body
  that lands while still ignoring one argument keeps a marker that looks
  legitimate. Each fill unit's checklist carries an explicit marker
  sweep; do not rely on the compiler to ask for one.

The module-level counterpart is `#![allow(dead_code)]` on skeleton entry,
removed in slices as the holes fill. Like the per-function marker, it
needs an explicit removal step in the fill plan.

### The hole inventory

`todo!()` is a diverging panic, not debt the compiler tracks. A skeleton
full of them passes check and clippy alike, which is what the skeleton
commit is for and also its blind spot. Track the holes explicitly, by one
of two routes.

- **Lint config available.** Enable `clippy::todo` at **warn** for the
  fill phase (a `[lints]` entry in `Cargo.toml`, or `#![warn(clippy::todo)]`
  at crate entry), and flip it to **deny** as the completion gate for the
  last fill unit. The lint is allow-by-default, and denying it from the
  skeleton commit onward would fail the skeleton's own clippy gate, so
  the order matters.
- **Lint config not available.** Keep a deterministic inventory instead.
  `grep -rn 'todo!(' src/` at the skeleton commit is the baseline; each
  fill unit re-runs it and accounts for every line that changed, so no
  hole outlives the card that owned it.

### The type-to-business-rule map

Every public type in the skeleton maps to **one** business rule, with the
invalid state it forbids named explicitly. The map lives in a `DESIGN.md`
co-located with the module, alongside a visibility and seam table (what
the module reaches into, at what visibility, which accessors are
test-only) and the named residual risks.

A type that cannot be given a one-line business rule is a signal, not a
formatting problem: it is usually two types, or a bag with no invariant
to defend.

## Skeleton review

One fresh Rust reviewer checks the compiled skeleton commit against the
approved spec. The same reviewer covers both question sets below. The
coordinator records dispositions and routes repairs before tests or fills.
The card or DAG may include a user gate for types and invariants.

Invalid-states questions:

- Which invalid states can these types still represent? Try to construct
  a well-typed, meaningless value. Cross-field combinations count.
- Does any type encode a state production cannot reach? An always-`Some`
  option or an unreachable variant is surface that tests will cover while
  proving nothing.
- Does any bare `String`, `usize`, or unbounded collection cross the
  public boundary, including through error variants and serialization
  derives? Raw diagnostic text in an error variant is the one allowed
  exception, when its docs mark it diagnostic-only and nothing branches
  on it.
- Does every public type map to exactly one business rule? Two rules mean
  two types; none means a bag.
- Do fallible constructors return `Result`, with downstream code taking
  only already-valid types? A validate-then-use path anywhere is a
  finding.
- Do the serde derives mean what the types mean? Check every derive whose
  behavior is not obvious from the type.
- Are public fields confined to structs where every field is itself a
  validated type?

Logic and seams questions:

- Spec-to-type mismatches: a spec rule with no type enforcing it, or a
  type enforcing what the spec does not say.
- Ordering and sequencing the spec fixes and the types leave free, or fix
  wrongly.
- Seam errors: what the module reaches into, at what visibility, and
  whether the design record's visibility table is accurate.
- Coverage claims the types cannot support.
- Synthetic states: fixture or builder types that assemble combinations
  production never produces.
- Error modeling: does the enum distinguish the failure modes a caller
  acts on differently, and collapse the ones it must not?

The reviewer returns numbered findings, each marked blocking or minor,
each with the repair the reviewer would make. An empty return is a failed
review; a clean review says PASS explicitly.

**Findings are numbered, and each carries its disposition individually.**
An accepted finding's disposition is the repair; a rejected finding's is
the reason. Aggregate counts ("8 findings, all addressed") are a failure
of the practice, because no auditor can reconstruct which repair answered
which finding. The ledger row per finding is: number, finding,
disposition, repair or reason, author model, reviewer model. Keep it
where an auditor will find it, in the module's `DESIGN.md` or the card's
review directory if a board is in use.

Keep reviewers independent of every actual author of the skeleton, tests,
fills, and integration code they review. Record authors and reviewers so
that independence can be checked.

Skeleton review is required, not optional. In its founding use it caught a
fixture type encoding a production-unreachable state and a number of
logic issues, before any body existed to debug.

## Layer 2: golden frames written to fail first

### The shape

Expect-style golden tests of **whole rendered frames**, not string
fragments. Where the unit under test emits a request envelope, a rendered
prompt, a serialized message, or any other complete artifact, the test
snapshots the complete artifact: every surface of it, in one snapshot.

The failure mode this replaces is the accumulation of string assertions,
one unit of work at a time. Twenty tests each asserting one substring
cost more to maintain than one snapshot of the whole frame, and they
leave the gaps between the substrings uncovered and unnamed.

### Written from the spec, failing on arrival

Write the goldens **before** the implementation, from the spec, so they
fail on arrival. That is what makes them Layer 2: the failing test is the
hole, the same way `todo!()` is the hole in Layer 1. A golden written
after the implementation records what the code does; a golden written
before it records what the code should do.

Practically: accept no snapshot until the body it covers is implemented.
Run the suite with snapshot auto-acceptance disabled in CI and at every
gate, so an unreviewed accepted snapshot cannot pass.

### Behavioral evidence for every fill

Name an owner for spec-derived tests before implementation, including work
with no rendered artifact. Derive expected results from the specification,
never from the response under test. First-red failure on a `todo!()` proves
reachability only. For a critical limit, exclusion or representation
conversion, name a plausible wrong implementation and retain evidence that
the owning test rejects it. Select critical counterexamples rather than
requiring mutation of every assertion.

At first-red, a control requiring unassigned production edits remains pending
with its owning test and the later control owner/check named. Execute selected
mutants in a disposable worktree once the required bodies exist, before
claiming that evidence passed. A pending control is not a successful rejection
and must remain visible at the applicable acceptance gate.

When a storage conversion collapses distinct domain values, cover the exact
boundary, the first value beyond it, empty input and a colliding stored
value. If normal ingress cannot construct that row, use a backend-local
fixture and explain why it is needed. Distinguish a storage-representable
boundary fixture from a synthetic state invented for production logic.
Whole-frame goldens remain for emitted artifacts; behavioral tests own
downstream semantics.

Review this evidence inside the applicable code review, not an extra panel.
Keep source inspection, owner-executed checks and reviewer-executed checks
distinct. A permission denial or missing prerequisite is an unverified
check, not a defect in the diff or a passing runtime check.

### The coverage manifest

Commit a manifest that maps every surface and branch to the fixture
covering it, and explicitly names the surfaces it leaves uncovered.

The exclusion rows are where the value is. A manifest listing only
covered rows is a table of contents; a manifest whose exclusion rows
carry reasons and name the tests that still own those paths is a coverage
argument. Every exclusion row states either why the surface needs no
coverage or which other test owns it.

The manifest also states the **scope of the identity claim**: which
surfaces a passing snapshot proves unchanged. Be explicit that a green
snapshot suite proves the artifact is unchanged, and never that behavior
downstream of the artifact is unchanged.

### Normalization

Nondeterministic bytes (timestamps, hash-map iteration order) need
normalization before snapshotting. Two rules keep normalization from
silently absorbing real drift:

- **Location-anchored, not text-global.** Rewrite at the structural
  position that produces the nondeterminism (this message, at this
  offset, inside this named span), operating on the structured artifact
  before it is flattened into the snapshot document. A global regex over
  flattened text can rewrite a payload byte that happens to look like a
  marker.
- **Audited.** Before any pass rewrites anything, assert that the
  artifact contains exactly the occurrences the passes expect, and panic
  otherwise. A marker appearing in a payload, or a marker missing where
  the builder should have emitted one, is a loud failure and never a
  silent skip.

### Byte-identity mode for refactor work

Provide an assertion mode that asserts the rendered frame is
byte-identical to the committed baseline, and use it as the acceptance
gate for refactor work that must not change output.

**Prove the mode before trusting it**, with both controls:

- a positive control: a real refactor that changes no behavior leaves the
  whole suite green;
- a negative control: a deliberate one-byte change fails exactly the
  snapshots that cover it and leaves the rest green.

A mode proven only by the positive control cannot be distinguished from a
mode that asserts nothing. Record both transcripts in the design record.

## Dispatch units

Delegation follows the same split: the skeleton is the expensive design
step and the fills are cheap, so a stronger model lays the type surface
and smaller models fill bodies against green-when-done criteria. Two
cautions. API-convention drift accumulates across executors even when
every per-unit gate passes (inherent `as_str` with no `AsRef` impl,
function-local consts, `as_str().to_owned()` chains, render-level
placeholders papering over source-level optionality), because per-unit
gates cannot see a cross-unit trend; bake the conventions into executor
briefs and the review checklist. And an executor outside a harness that
loads skills receives only its brief, so for those executors the brief
quotes the applicable unit below instead of naming skills. Fill the
angle brackets and send the skeleton, test-only, or fill unit matching
the assignment, together with the shared API conventions.

> **Skeleton unit.**
>
> Inputs: <approved spec path/revision and approval>, <input commit>,
> <allowed files>, <exact checks>. Stop for missing inputs or scope changes.
>
> 1. Lay the full type surface: real signatures, derives, `From`/`Into`
>    impls, leaf types. No placeholder `String` for a type not yet
>    written.
> 2. `todo!()` only in bodies with real behavior. Implement trivial
>    accessors and field reads.
> 3. Make invalid states unrepresentable: enums over flags, newtypes over
>    bare `String` or `usize`, bounded or non-empty collections where the
>    domain demands them.
> 4. Parse, do not validate. Fallible constructors return `Result` with
>    the error enum complete; downstream code takes only valid types.
> 5. No bare domain value crosses the public boundary, error payloads
>    included. Raw text is allowed only in a variant documented as
>    diagnostic-only that no caller branches on.
> 6. Fields are public only where every field is itself a validated type.
>    A cross-field invariant means private fields behind a constructor.
> 7. Every `todo!()` function that takes a parameter carries
>    `#[expect(unused_variables, reason = "todo!() body; filled by <ref>")]`.
>    Do not mark zero-parameter functions. Module entry carries
>    `#![allow(dead_code)]`.
> 8. Deliver a design record next to the module mapping every public type
>    to one business rule and the invalid state it forbids, plus a
>    visibility and seam table and named residual risks.
> 9. The skeleton is its own commit, green under
>    `cargo check --workspace --all-targets`,
>    `cargo clippy --workspace --all-targets -- -D warnings`, and
>    `cargo fmt --check` (the `+nightly` form where the repo pins
>    nightly rustfmt options). Implement no body in this unit.

> **Test-only unit.**
>
> Inputs: <approved spec revision>, <reviewed skeleton commit>, <allowed tests,
> fixtures and mechanical lint markers>, <exact compile and first-red checks>.
>
> 1. Exercise the approved API through spec-derived tests. Edit only the
>    named tests, fixtures, and assigned lint markers; do not change
>    production types or fill behavior bodies.
> 2. Derive expectations from the specification and fixture inputs, not
>    responses under test. Use behavioral tests for semantics and
>    whole-frame goldens where an emitted artifact exists.
> 3. Compile the tests, then record the named expected-red set, failure
>    reasons, count, and bodies each test needs. A compile error is not
>    intended red; a hole panic proves reachability only. Retain the
>    critical counterexample evidence or explicitly pending control owner.
> 4. Return API friction, missing fixture access, new prerequisites, or
>    scope changes to the coordinator before adopting a workaround.
> 5. Return the tests and evidence to the coordinator for review before
>    a filler starts. Do not dispatch reviewers yourself.

> **Fill unit.**
>
> Inputs: <spec revision>, <input commit>, <named tests and required bodies>,
> <remaining holes and expected intermediate red set>, <exact checks>.
>
> 1. Fill only the bodies named here: <list>. Do not change signatures,
>    derives, types, or API workflows. Return a needed API change to the
>    coordinator rather than working around it.
> 2. Remove each filled function's obsolete #[expect] marker by hand;
>    sweep its assigned module-level lint markers as the surface goes live.
> 3. Preserve supplied tests, fixtures, expectations, and snapshots. A
>    suspected oracle defect returns to the coordinator for test-owner
>    disposition; do not change the test to fit the implementation.
> 4. Run the assigned compile/build, lint, formatting, and test checks,
>    with snapshot auto-acceptance disabled. Its named tests must pass;
>    distinguish authorized intermediate red from new failures.
> 5. Mechanically verify no todo!() remains in the completed assigned
>    body scope and account for authorized remaining holes outside it.
>    Green tests alone do not establish hole completion.
> 6. Return the changes and exact check evidence to the coordinator for
>    implementation review. Do not broaden the assignment or dispatch
>    reviewers yourself.

> **API conventions, all units** (drift accumulates across executors and
> is checked at review): `AsRef` impls rather than inherent `as_str` on
> wrappers over a borrowed form; no function-local consts; no
> `as_str().to_owned()` chains; never paper over source-level optionality
> with a render-level placeholder.

## Known limits

Record these as standing limits, and record future failures against them
as correctives rather than as refutations of the technique.

- **The compile surface misses partly-private ingress gaps.** A real
  defect of this class survived a green skeleton: the type surface was
  correct at the module boundary while an ingress path at a narrower
  visibility bypassed it. The compiler cannot see it, because nothing
  ill-typed is written. Agent review at the design panel caught it, which
  is one reason the panel is not optional. A deterministic full-surface
  oracle that enumerates every ingress path regardless of visibility is
  the open answer, not yet built.
- **Runtime semantic mismatches are unattested.** The technique has never
  been shown to catch a defect where the types compose correctly and the
  runtime meaning is wrong. Do not claim it does. When a defect of this
  class appears, record it as a corrective against the practice.
- **The value is downstream of the signals.** The practice is only as
  trustworthy as `cargo check`, clippy, and derive behavior; where a
  signal is weak, the practice inherits the weakness. Serde is the worked
  case. `serde(untagged)` tries the variants in declaration order and
  returns the first that deserializes, so an enum whose variants have
  overlapping field shapes can take a payload as the wrong variant: a
  subset variant declared first swallows the superset's payload. That is
  runtime deserialization behavior and the compiler never flags it. What
  the typing stage buys is that the risk is visible by inspecting the
  overlapping field shapes in the declaration, so the panel checks every
  derive whose behavior is not obvious from the type.
- **A green golden suite is not behavioral evidence.** It proves the
  artifact bytes are unchanged. Downstream work must not read it as proof
  that behavior, scores, or user-visible outcomes are unchanged.
