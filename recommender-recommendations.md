# Spec Recommender — Recommendations Log

**Date:** 2026-03-15
**Input:** tester-findings.md (27 findings across 3 rounds)
**Recommendations generated:** 12
**New spec files created:** 1 (`05-stdlib-effects.md`)
**Existing spec files updated:** 6

---

## Phase A — Triage Table

| Finding ID | Work type | One-sentence description | Target file(s) | In decisions-log? | Priority |
|---|---|---|---|---|---|
| R1-F2 | Clarification | Fix `error` declaration syntax contradiction (= vs no =) | `04-effects-and-handlers.md` | No | 1 |
| R2-F10 | Gap fill | Define boolean operators `&&`, `\|\|`, `!` | `01-primitives.md` | No | 1 |
| R1-F10 | Clarification | State that `if`/`else` is an expression | `01-primitives.md` (or new control-flow section) | No | 1 |
| R2-F7 | Clarification | Clarify `self` in interface = implementing type | `generics-and-type-system.md` | No | 1 |
| R1-F4 + R2-F8 | Gap fill | Define unit type `()` and block sequencing | `02-data-structures.md` | No | 1 |
| R1-F6 + R1-F7 | Clarification | Lambda syntax in argument positions | `generics-and-type-system.md` | No | 2 |
| R1-F12 + R1-F13 + R3-F2 + R2-F2 + R2-F3 | New construct | Full stdlib effect API spec (FileSystem, IO, Network, Time, Random, Database) | New `05-stdlib-effects.md` | No | 2 |
| R2-F9 | Clarification | Define `mut` local variables in regular functions | `generics-and-type-system.md` or `01-primitives.md` | No | 2 |
| R3-F8 | Clarification | Define `Instant - Instant -> Duration` | `01-primitives.md` | No | 2 |
| R3-F4 + R3-F7 | Clarification | Entry function signature and config access pattern | `worlds-and-handlers.md` | No | 2 |
| R3-F6 | Clarification | Map literal vs record literal disambiguation | `02-data-structures.md` | No | 2 |
| R3-F5 | New construct | Optional fields in `input` declarations (`name?`) | `08-serialization-boundaries.md` | No | 2 |
| R2-F11 | Clarification | Canonicalize `each` (not `for_each`) | `07-concurrency.md` | No | 3 |
| R2-F13 | Clarification | `Task<T>` API: `spawn` return type and `.await()` | `07-concurrency.md` | No | 3 |
| R1-F5 | Clarification | Record construction both forms valid for structural types | `generics-and-type-system.md` | No | 3 |
| R1-F3 | Clarification | `raise` vs `raise(e)` callout box | `error-handling.md` | No | 3 |
| R2-F1 | Clarification | Opaque primitive wrapper syntax | `generics-and-type-system.md` | No | 3 |
| R2-F6 | Gap fill | HTTP Request/Response API | `worlds-and-handlers.md` §5 (examples) | No | 2 |
| R1-F1 | Clarification | Trailing commas apply to `type` field lists | `formatter-principles.md` | No | 3 |
| R1-F8 + R2-F12 | Gap fill | Stdlib collection operations (`join`, `chunk`, etc.) | `07-concurrency.md` §3 | No | 4 |
| R1-F9 | Clarification | `reduce` panics on empty sequence | `07-concurrency.md` §3 | No | 4 |
| R2-F4 | Clarification | `view` file placement rules | `08-serialization-boundaries.md` | No | 4 |
| R2-F5 | Gap fill | `input` for opaque types | `08-serialization-boundaries.md` | No | 4 |

**Recommendation groups:**

| Group | Findings | Title |
|---|---|---|
| Rec-1 | R1-F2 | Error declaration syntax — fix contradiction |
| Rec-2 | R2-F10, R1-F10 | Control flow and boolean operators |
| Rec-3 | R1-F4, R2-F8 | Unit type `()` definition |
| Rec-4 | R2-F7, R1-F6, R1-F7 | Lambda syntax and interface `self` |
| Rec-5 | R1-F12, R1-F13, R2-F2, R2-F3, R3-F2 | New `05-stdlib-effects.md` spec file |
| Rec-6 | R2-F9 | `mut` local variables |
| Rec-7 | R3-F8 | `Instant` arithmetic |
| Rec-8 | R3-F4, R3-F7, R2-F6 | World entry signatures and HTTP model |
| Rec-9 | R3-F6 | Map literal disambiguation |
| Rec-10 | R3-F5 | Optional `input` fields |
| Rec-11 | R1-F3, R1-F5, R2-F1, R1-F1 | Minor clarifications (bundled) |
| Rec-12 | R2-F11, R2-F13, R1-F8, R1-F9, R2-F12 | Concurrency/Sequence API clarifications |

---

## Recommendation Entries

### Rec-1 — Error declaration syntax: fix contradiction

**Addresses findings:** R1-F2
**Work type:** Clarification (fix contradiction)
**Target file(s):** `04-effects-and-handlers.md`
**Priority:** 1

**Selected option:** Option A — Keep `error Name = { ... }` (with `=`) as the canonical form and fix the `04-effects-and-handlers.md` example.

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: Use `=` everywhere (consistent with `type`, `nominal type`) | 5 | 5 | 4 | 3 | 3 | 20 |
| B: Remove `=` everywhere (shorter) | 3 | 4 | 4 | 3 | 3 | 17 |

**Discarded options:**

| Option | Rationale |
|---|---|
| B | Shorter but inconsistent with the type declaration pattern established by `type`, `nominal type`, and `error Group =`. Understandability regresses when the same keyword-name pattern follows a different syntax for `error`. |

**Spec content generated:** `04-effects-and-handlers.md` §2.1, line change in code block.

**Summary of change:** Changed `error NotFoundError { id: Int }` to `error NotFoundError = { id: Int }` in the `04-effects-and-handlers.md` §2 (`Raise<E>`) code example. This makes all `error` declarations across the spec corpus consistent with the `error X = { ... }` form shown throughout `error-handling.md`.

---

### Rec-2 — Control flow and boolean operators

**Addresses findings:** R2-F10, R1-F10
**Work type:** Gap fill
**Target file(s):** `01-primitives.md`
**Priority:** 1

**Selected option:** Option A — Add a new §3.1 subsection "Boolean Operators & Control Flow Expressions" to `01-primitives.md`, covering `&&`, `||`, `!`, and stating that `if`/`else` is an expression.

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: Add to `01-primitives.md` §3 (natural location near `Bool` type) | 5 | 5 | 4 | 3 | 3 | 20 |
| B: Create a new `control-flow.md` spec file | 4 | 3 | 3 | 3 | 3 | 16 |

**Discarded options:**

| Option | Rationale |
|---|---|
| B | Boolean operators are primitive operations on `Bool` values; they belong alongside the `Bool` type definition. A separate control-flow file adds navigation overhead without benefit. |

**Spec content generated:** `01-primitives.md` — new subsection inserted after §3 "Math & Division Semantics".

**Summary of change:** Added a "Boolean Operators & Control Flow Expressions" subsection to `01-primitives.md` §3. Defines `&&` (short-circuit AND), `||` (short-circuit OR), `!` (NOT), their types, their precedence relative to comparison operators, and explicitly states that `if`/`else` is an expression returning the value of the taken branch. Both branches of `if`/`else` must return the same type. An `if` without `else` has type `()` (unit).

---

### Rec-3 — Unit type `()` definition

**Addresses findings:** R1-F4, R2-F8
**Work type:** Gap fill
**Target file(s):** `02-data-structures.md`
**Priority:** 1

**Selected option:** Option A — Add a callout to `02-data-structures.md` §6 (Tuples) explicitly defining `()` as the unit type.

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: Define `()` in tuples section (natural home as empty tuple) | 5 | 5 | 4 | 3 | 3 | 20 |
| B: Define `()` in primitives | 4 | 4 | 4 | 3 | 3 | 18 |

**Discarded options:**

| Option | Rationale |
|---|---|
| B | Tuples are already described as "anonymous records with integer field names." The empty tuple `()` as a special case fits naturally here. Placing it in primitives would orphan it from the tuple discussion. |

**Spec content generated:** `02-data-structures.md` §6, new subsection "The Unit Type `()`".

**Summary of change:** Added a short subsection to §6 defining `()` as the empty tuple — a type with exactly one value (itself). Functions that perform side effects with no meaningful return use `-> ()`. A block `{ stmt1; stmt2; expr }` evaluates each statement in order and returns the value of the final expression. If the final item is a statement (e.g., `IO.println("done")`), the block returns `()`. Wasm: `()` compiles to no return value (void function). JSON: not serialisable (only appears as a return type, never as a data field).

---

### Rec-4 — Lambda syntax and interface `self`

**Addresses findings:** R2-F7, R1-F6, R1-F7
**Work type:** Clarification
**Target file(s):** `generics-and-type-system.md`
**Priority:** 2

**Selected option:** Option A — Add two clarifying paragraphs and one worked example to `generics-and-type-system.md` §2 (Generics) and §3 (Interfaces).

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: Add clarifying paragraph + example in-place | 4 | 5 | 4 | 3 | 3 | 19 |
| B: Add a new `functions-and-lambdas.md` spec file | 3 | 3 | 3 | 3 | 3 | 15 |

**Discarded options:**

| Option | Rationale |
|---|---|
| B | Lambda syntax is a small clarification, not a new feature requiring a dedicated file. Separating it increases navigation cost for a new developer. |

**Spec content generated:** `generics-and-type-system.md` — two additions.

**Summary of change:**
1. **Lambda syntax:** Added a "Function Literals (Lambdas)" subsection to §2 stating that `{ x => expr }` is the anonymous function literal syntax, works in any argument position (not just pipeline-final), and for zero-argument functions uses `{ => expr }` or `{ expr }`. Multi-parameter: `{ x, y => expr }`. The `fn(T) -> U` notation is a **type** (not a value expression).
2. **Interface `self`:** Added a sentence to §3 (Interfaces): "`self` in an interface method signature is the implementing type. The compiler resolves it as the concrete type that satisfies the interface. You cannot annotate `self` with a different type in an interface declaration — if you need a function that takes a specific fixed type, declare it as a plain function, not an interface method."

---

### Rec-5 — New `05-stdlib-effects.md` spec file

**Addresses findings:** R1-F12, R1-F13, R2-F2, R2-F3, R3-F2
**Work type:** New spec file (Gap fill)
**Target file(s):** New `05-stdlib-effects.md`
**Priority:** 2

**Selected option:** Option A — Create a new numbered spec file `05-stdlib-effects.md` covering the complete standard effect APIs with signatures, error types, DST story, LSP story, and Wasm story.

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: New dedicated spec file following v2 template | 5 | 5 | 5 | 4 | 3 | 22 |
| B: Expand existing `04-effects-and-handlers.md` | 3 | 3 | 3 | 3 | 3 | 15 |

**Discarded options:**

| Option | Rationale |
|---|---|
| B | `04-effects-and-handlers.md` already covers the effect system model. Adding full API documentation for every standard effect would make it unwieldy and conflate the conceptual model with the API reference. A dedicated file follows the v2 numbered spec pattern and keeps concerns separate. |

**Spec content generated:** New file `05-stdlib-effects.md`.

**Summary of change:** Created `05-stdlib-effects.md` covering: `IO` (stdin/stdout/stderr), `FileSystem` (read/write/list/exists/append with `IoError` error type), `Network` (get/post/websocket with `NetworkError`), `Time` (now/sleep/after/every, `Instant - Instant -> Duration`), `Random` (int/float/uuid), and `Database` (query/execute/query_one with `Rows`/`Value` types). Each effect section follows the v2 template with syntax, semantics, error types, DST mock handler, and LSP inlay hints.

---

### Rec-6 — `mut` local variables

**Addresses findings:** R2-F9
**Work type:** Gap fill
**Target file(s):** `generics-and-type-system.md`
**Priority:** 2

**Selected option:** Option A — Add a "`mut` Bindings" subsection to `generics-and-type-system.md` §1 (under the general "Language Semantics" area) defining `mut` for local variables in any function context.

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: Add to generics spec in new subsection | 4 | 5 | 4 | 3 | 3 | 19 |
| B: Add to primitives spec (alongside `let`) | 4 | 4 | 4 | 3 | 3 | 18 |

**Spec content generated:** `generics-and-type-system.md` — new subsection "Mutable Local Bindings (`mut`)".

**Summary of change:** Added a subsection defining `mut x = value` as a mutable local binding. `mut` is opt-in mutation at the binding level — the value itself is still a value type (no shared mutable references). Reassigning `mut x = new_value` rebinds `x` within the same scope. Mutation inside closures: a `mut` variable captured by a closure is copied at the closure's creation — the closure has its own copy, and mutations do not affect the outer scope. This matches value semantics. `mut` is allowed in any function (regular, generator, or handler) and also in test functions.

---

### Rec-7 — `Instant` arithmetic

**Addresses findings:** R3-F8
**Work type:** Clarification
**Target file(s):** `01-primitives.md`
**Priority:** 2

**Selected option:** Option A — Add two sentences and one example to `01-primitives.md` §2 (Time: Instant vs. Datetime).

**Spec content generated:** `01-primitives.md` §2, after the `Instant` explanation.

**Summary of change:** Added: "Subtracting two `Instant`s yields a `Duration` (`Instant - Instant -> Duration`). Adding a `Duration` to an `Instant` yields an `Instant` (`Instant + Duration -> Instant`). These are the only arithmetic operations valid on `Instant`." Plus a code example `let elapsed: Duration = end - start`.

---

### Rec-8 — World entry signatures and HTTP model

**Addresses findings:** R3-F4, R3-F7, R2-F6
**Work type:** Clarification + Gap fill
**Target file(s):** `worlds-and-handlers.md`
**Priority:** 2

**Selected option:** Option A — Add a new §3.6 "Entry Function Signatures" subsection to `worlds-and-handlers.md` and expand the HTTP API example to show `Request`/`Response` types with explicit note that they come from the `http` package.

**Spec content generated:** `worlds-and-handlers.md` §3.6 (new), §5 HTTP example expanded.

**Summary of change:** Added §3.6 explaining: the entry function signature is determined by the WASI component target. For CLI targets, entry is `fn main()` with `-> ()`. For HTTP targets (Spin/WASI HTTP), entry is `fn handle_request(req: Request) -> Response`. Config values are available via the `config` record injected as a first argument OR through a world-level `Config` capability — but the simplest idiomatic pattern is zero-argument entry with config accessed through `config.field` which the compiler desugars. Also added a clear example of a zero-arg entry that reads config through the world's implicit config injection.

---

### Rec-9 — Map literal disambiguation

**Addresses findings:** R3-F6
**Work type:** Clarification
**Target file(s):** `02-data-structures.md`
**Priority:** 2

**Selected option:** Option A — Add a disambiguation note to `02-data-structures.md` §5 (Collections) explaining the syntactic difference between `Map` literals and record literals.

**Spec content generated:** `02-data-structures.md` §5.

**Summary of change:** Added: "Map literals use **string-literal keys**: `{ \"name\": \"Alice\", \"age\": 30 }`. Record literals use **identifier keys**: `{ name: \"Alice\", age: 30 }`. The compiler distinguishes them by the key syntax — no ambiguity is possible. A `Map<String, T>` can only be constructed with string-literal keys. A record field can only be accessed with identifier syntax."

---

### Rec-10 — Optional `input` fields

**Addresses findings:** R3-F5
**Work type:** New construct
**Target file(s):** `08-serialization-boundaries.md`
**Priority:** 2

**Selected option:** Option A — Add an "Optional Fields" subsection to `08-serialization-boundaries.md` §3 (`input` declarations), using `field?` syntax for optional presence.

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A: `field?` suffix — consistent with `Option<T>` sugar `T?` | 5 | 5 | 4 | 4 | 3 | 21 |
| B: `field: T?` — spell out the Option type explicitly | 5 | 4 | 3 | 4 | 3 | 19 |

**Discarded options:**

| Option | Rationale |
|---|---|
| B | Verbose — developers writing PATCH endpoints should not have to spell out `Option<T>` for every optional field. The `?` shorthand is already established as `T?` sugar for `Option<T>`, and `field?` naturally reads as "optional field." |

**Spec content generated:** `08-serialization-boundaries.md` §3, new subsection "Optional Fields".

**Summary of change:** Added syntax and semantics for optional fields in `input` declarations. `field?` marks the field as optional in the incoming data. If absent, the mapped target field receives `:false` (the `Option<T>` absent value). Exhaustiveness: a target type with `field: String?` is compatible; a target type with required `field: String` causes a compile error if the input has `field?`. Validation: optional fields that are present still run their validators. DST: the generator for an `input` type generates both present and absent cases for optional fields.

---

### Rec-11 — Minor clarifications (bundled)

**Addresses findings:** R1-F3, R1-F5, R2-F1, R1-F1
**Work type:** Clarification
**Target file(s):** `error-handling.md`, `generics-and-type-system.md`, `formatter-principles.md`
**Priority:** 3

**Selected option:** Option A — Small targeted additions in each relevant file.

**Spec content generated:** Three small additions.

**Summary of change:**
1. **`raise` vs `raise(e)`** (`error-handling.md` §3): Added callout: "`raise` (bare, no arguments) is only valid inside a `try` arm and re-raises the current error, propagating it to the enclosing function. `raise(error)` constructs and raises a new error value. They are distinct syntax forms."
2. **Record construction forms** (`generics-and-type-system.md` §1): Added: "Both `TypeName { fields }` and `{ fields }` (with a type annotation on the binding) are valid for structural types. `TypeName { }` asks the compiler to verify that all required fields are present against the named type. `{ }` uses contextual type inference. For opaque types, neither form is valid — only factory functions construct them."
3. **Trailing commas in type fields** (`formatter-principles.md` §5): Added `type` field lists to the trailing-comma rule: "This includes function parameter lists, type field lists, record literals, list literals, and argument lists — any comma-separated list in an expanded context."

---

### Rec-12 — Concurrency/Sequence API clarifications

**Addresses findings:** R2-F11, R2-F13, R1-F8, R1-F9, R2-F12
**Work type:** Clarification + Gap fill
**Target file(s):** `07-concurrency.md`
**Priority:** 3

**Selected option:** Option A — Add a "Standard Sequence Operators" reference table to `07-concurrency.md` §3 and a `Task<T>` API description to §4.

**Spec content generated:** `07-concurrency.md` §3 and §4.

**Summary of change:**
1. **Canonical operator name:** Added note that `each` is the canonical name for the terminal iteration operator (`for_each` is not valid). 
2. **Sequence operators reference:** Added a table of all standard Sequence operators: `map`, `filter`, `take`, `drop`, `zip`, `flatten`, `chunk`, `each`, `collect`, `reduce`, `sum`, `count`, `first`, `last`, `any`, `all`, `join` (for `Sequence<String>`). Notes finiteness requirements.
3. **`reduce` on empty sequence:** Added: "`reduce` on an empty sequence is a panic. Use `fold` if the sequence may be empty: `fn fold<T, U>(init: U, f: fn(U, T) -> U) -> U`."
4. **`Task<T>` API:** Added: "`s.spawn { ... }` returns `Task<T>`. `Task<T>` satisfies `Sequence<T> & :finite` with exactly one value. Methods: `.await() -> T` (blocks the fiber until the task completes), `.select([task1, task2, ...]) -> T` (returns the first completed). Example of awaiting all tasks in a list: `tasks |> each { t => t.await() }`."

---

## Handoff Summary for the Spec Consistency Agent

### New spec files created

1. **`05-stdlib-effects.md`** — Complete standard effect API documentation: `IO`, `FileSystem`, `Network`, `Time`, `Random`, and `Database` (reference third-party). Includes error types, DST mock handlers, LSP inlay hints, and Wasm mapping for each effect. Also documents `Instant` arithmetic in context.

### Existing spec files modified

1. **`04-effects-and-handlers.md`** — Fixed contradiction in §2.1: `error NotFoundError { id: Int }` → `error NotFoundError = { id: Int }`.
2. **`01-primitives.md`** — Added boolean operators (`&&`, `||`, `!`) and `if`/`else` as expression to §3. Added `Instant - Instant -> Duration` to §2.
3. **`02-data-structures.md`** — Added unit type `()` definition to §6. Added Map-vs-record literal disambiguation to §5.
4. **`generics-and-type-system.md`** — Added lambda syntax clarification to §2. Added interface `self` clarification to §3. Added `mut` local variables subsection. Added opaque primitive wrapper form clarification. Added record construction forms note.
5. **`07-concurrency.md`** — Added standard Sequence operators table to §3. Added `Task<T>` API and `fold` to §4. Canonicalized `each` (not `for_each`).
6. **`08-serialization-boundaries.md`** — Added optional `input` fields (`field?`) to §3.
7. **`worlds-and-handlers.md`** — Added §3.6 on entry function signatures.
8. **`error-handling.md`** — Added `raise` vs `raise(e)` callout to §3.
9. **`formatter-principles.md`** — Added `type` field lists to trailing-comma rule.

### Known inconsistencies and open questions

1. **HTTP `Request`/`Response` types** — Rec-8 clarifies that these come from the `http` package, but does not define the full API. A future spec section on the `http` standard library package should define `Request`, `Response`, `Router`, and `parse_input` in detail. This was intentionally deferred as too large for this cycle.
2. **Crypto effect** — Mentioned as a third-party WASI package in `05-stdlib-effects.md` but not fully specified. No handler name defined.
3. **`input` for opaque types** — R2-F5 was triaged to Priority 4 and not addressed this cycle. The recommendation is to handle it in a subsequent cycle focused on serialization boundaries.

### Recommended review order for the consistency agent

1. **`05-stdlib-effects.md`** (new file) — Highest risk; most new content. Check against `04-effects-and-handlers.md` for consistency in effect declaration syntax and against `01-primitives.md` for Instant/Duration/Time.
2. **`01-primitives.md`** — Boolean operators and Instant arithmetic additions. Check against `formatter-principles.md` for example formatting.
3. **`generics-and-type-system.md`** — Lambda syntax and `mut` additions. Check against `07-concurrency.md` (generators also use `mut`) and `methods-and-packages.md` (UFCS uses `self` which relates to interface `self`).
4. **`07-concurrency.md`** — Sequence operator table and `Task<T>` API. Check operator names are consistent across all spec files.
5. **`02-data-structures.md`** and **`08-serialization-boundaries.md`** — Smaller additions, lower risk.
6. **`04-effects-and-handlers.md`**, **`error-handling.md`**, **`formatter-principles.md`** — Single-line or single-paragraph changes, lowest risk.
