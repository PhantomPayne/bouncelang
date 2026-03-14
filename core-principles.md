# Bouncelang: Core Principles

This document is the **North Star** for Bouncelang's design. Every feature, syntax choice, error message, and library decision must be measured against the principles here. When writing a new spec section and asking "should we add a lint rule for this?", "how can the LSP help here?", "is this good for DST?", or "is this the right DX?" — this document gives you a clear, actionable answer.

---

## 1. The One Question: Would a New User Understand This?

Bouncelang leans **practical and familiar over academic and powerful**. C-style over ML-style. This is the primary filter for every decision.

> **Could a competent developer who has never used Bouncelang understand what this does within 30 seconds?**

If the answer is no, the design needs rethinking — regardless of how elegant or powerful it is.

### What This Means in Practice

**Prefer named arguments over positional when order is non-obvious.**

```bounce
// ❌ What does `true` mean here? What order are these in?
connect("localhost", 5432, true, false)

// ✅ Self-documenting at the call site
connect(host: "localhost", port: 5432, tls: true, pooling: false)
```

**Prefer `try`/`catch`-shaped error handling over monadic chains.**

```bounce
// ❌ Familiar to Haskell programmers. Unfamiliar to most developers.
fetch_user(id)
    |> and_then { u => fetch_posts(u.id) }
    |> map_err { e => ApiError { source: e } }

// ✅ Familiar shape. Any developer from C++, Go, Kotlin, Swift can read this.
try {
    let user = fetch_user(id)
    let posts = fetch_posts(user.id)
    build_response(user, posts)
} catch {
    NotFoundError => respond_404()
    NetworkError => respond_503()
}
```

**Prefer consistent syntax even if a special case would be slightly more powerful.**

Every inconsistency is a new rule to memorize. Bouncelang accepts a small power reduction to avoid a large learning cost. The language should feel like it has one grammar, not many.

**Prefer explicit over implicit at the boundaries that matter.**

Inside a function, inference is fine. At a public API boundary (`pub fn`), effects and significant constraints should be visible. A developer reading a function signature at a call site shouldn't need the LSP to understand what it does or what it requires.

### The Corollary

A feature that is powerful but that nobody uses because nobody understands it has **negative value**. It adds complexity to the language, creates confusion, and makes error messages harder to write. If only experts understand it, only experts will use it correctly — and everyone else will misuse it or avoid the language.

This is why Bouncelang has no implicit coercions, no surprise inference across module boundaries, no magic method names, and no "unless you know the trick" syntax.

---

## 2. DX Is a First-Class Feature

**Developer experience** is not polish applied after the language is designed — it is a design constraint from the start. The LSP, formatter, error messages, and tooling are part of the language spec, not afterthoughts.

When specifying any feature, the question is not just "is the semantics correct?" It is: "what does it feel like to use this?"

### 2a. Errors Should Teach, Not Just Reject

A compiler error is a teaching moment. It should:

1. Say exactly what went wrong
2. Show the context (what you wrote, what was expected)
3. If possible, trace where the problem originated
4. If possible, suggest how to fix it

**Effect propagation errors show the full call chain:**

```
Error: Function `handle_request` uses Network but the `api` world does not provide it.

  Raised by:
    handle_request   (src/handlers.bounce:14)
    → fetch_user     (src/services.bounce:31)
    → Network.get    (std/network:8)

  Fix: Add `Network` to the world's provided effects, or handle it in a mock handler for testing.
```

**Missing handler errors show what's needed and where:**

```
Error: World `test` is missing a handler for `Network`.

  Required by: `fetch_user` (src/services.bounce:31)
  
  Quick fix: Add a mock Network handler
    handle Network = MockNetwork { base_url: "http://test.local" }
```

**Type errors show what you have and what was expected, with context:**

```
Error: Type mismatch in call to `create_order`.

  Expected: { user_id: UserId, amount: Decimal }
  Got:      { user_id: Int, amount: Float }

  Note: `UserId` is a nominal type — `Int` does not implicitly satisfy it.
        Use `UserId(raw_id)` to construct a `UserId`.
```

The error message is part of the language. Writing a good error message is required, not optional.

### 2b. The LSP Is Part of the Design

Every significant language feature must have a corresponding **LSP story**. When specifying a new feature, ask:

- What **completions** does this enable?
- What **inlay hints** help users understand what the compiler knows?
- What **diagnostics** catch common mistakes?
- What **code actions** (quick fixes) can we offer?

This is not aspirational. It is a checklist. See §7 for the formal version.

**Examples from the existing specs:**

- `select()` auto-completes the named sequence arguments from sequences in scope (see `07-concurrency.md`)
- `match` arms auto-generate from `select` union tags — the LSP knows the union shape and offers to fill all arms
- Effect annotations on `pub fn` are offered as a **quick-fix** when the inferred effects differ from what's written
- The TOCTOU warning on `State` (read-then-update on the same handle) fires as a diagnostic with a suggested refactor

The LSP's job is to make the compiler's knowledge visible to the developer. Inferred types, inferred effects, inferred finiteness tags — all of this should surface as inlay hints so that developers can learn the system by writing code, not by reading documentation.

### 2c. The Formatter Is Part of the Language

One canonical format. No configuration. No style debates. Formatting rules are specified alongside syntax — they are not a separate tool bolted on afterward.

The formatter does not enforce a character count limit. It uses **structural expansion rules**: either a construct fits on one line, or all its elements expand. This makes reformatting predictable and rename-safe — adding one character to a name never cascades into unexpected reformats.

See `formatter-principles.md` for the full specification.

```bounce
// ✅ Formatter output — one argument: inline
let result = transform(input)

// ✅ Formatter output — many arguments: expand all
let result = create_user(
    name: "Alice",
    email: "alice@example.com",
    role: :admin,
)

// ✅ Leading operators — immediately see the logic structure
let eligible =
    user.age >= 18
    && user.verified
    && !user.banned
```

### 2d. `bounce upgrade` — Migrations as a First-Class Citizen

Breaking changes happen. The tooling handles them. Codemods, not apologies.

When Bouncelang or the standard library introduces a breaking change, `bounce upgrade` ships a codemod that mechanically transforms existing code. Users run one command, review the diff, and continue. The alternative — reading migration guides and manually updating hundreds of call sites — is not acceptable.

This applies to:

- Renamed standard library functions
- Changed argument order (with named arguments, this is rare)
- Deprecated syntax being removed
- Effect set changes on `pub fn` in the standard library

Versioned stdlib modules (`std/http@v2`) allow gradual migration — the old API stays available until explicitly removed, giving users a migration window.

### 2e. Lint Rules and Refactor Targets Are User-Extensible

Teams have domain-specific conventions that generic tooling can't anticipate. Bouncelang's LSP architecture allows users to define **custom lint rules** and **refactor targets** in Bouncelang itself. These are discovered by the LSP automatically and surface through the same machinery as built-in rules — same diagnostic format, same code action UX, same configuration system.

This means a team can enforce "controllers must not import models directly" or "all public error types must have a `user_message` field" with the same tooling quality as a built-in rule.

The lint rule system needs its own spec file. This principle establishes it as a core design requirement, not an extension mechanism.

---

## 3. Consistency Is Kindness

**Inconsistency is a hidden tax** on every user of the language. When two things look similar, they should behave similarly. When two things behave differently, they should look different.

Consistency is not about aesthetics. It is about reducing the cognitive overhead of learning and using the language. A user who understands one part of the language should be able to predict how another part works.

### One Way to Call Functions

Dot syntax, pipe syntax, and direct call are all equivalent. There is no "method" vs "function" distinction that changes the calling convention:

```bounce
// All three are identical — the compiler picks based on style context
let result = transform(list, f)       // direct call
let result = list.transform(f)        // dot syntax (UFCS)
let result = list |> transform(f)     // pipe syntax
```

See `methods-and-packages.md` for the full UFCS specification.

### One Way to Handle Errors

`try`/`catch` is the one error-handling construct. There is no `?` operator, no `Result.map`, no `and_then` chain. One mechanism means one thing to learn, one thing to search for, one thing to audit.

```bounce
// The only way to handle a Raise — no alternatives, no shortcuts
try {
    let user = fetch_user(id)
    process(user)
} catch {
    NotFoundError => respond_404()
    NetworkError => respond_503()
}
```

### One Way to Define Data

Records for structural data, opaque types for identity boundaries. No classes, no separate tuple syntax that differs from destructuring, no structs-vs-objects distinction:

```bounce
// Structural data — identified by shape
type Point = { x: Float, y: Float }

// Identity boundary — identified by name and origin
export type UserId = opaque Int
```

### One Naming Convention — Always

| Category | Convention | Examples |
|---|---|---|
| Atoms | `:lowercase` | `:ok`, `:true`, `:not_found` |
| Types | `PascalCase` | `User`, `HttpResponse`, `UserId` |
| Functions & variables | `snake_case` | `fetch_user`, `max_retries` |
| Effects | `PascalCase` | `Network`, `FileSystem`, `Time` |

The formatter enforces these. There are no exceptions, no configuration options.

### One Format

The formatter decides. Not the team lead. Not the style guide. Not a linter config. If two developers format the same file, they get identical output, always.

### Syntactic Consistency and Learning Speed

A user who learns how `match` works on a union can immediately apply that knowledge to `match` on a `select` result, on an `Option`, on an error in a `try` block — because they all have the same shape:

```bounce
// Union match
match response {
    :ok { body } => process(body)
    :error { message } => log_error(message)
}

// select result match — identical pattern
for event in select(timer: heartbeat, message: inbox) {
    match event {
        :timer { t } => send_heartbeat(t)
        :message { msg } => handle(msg)
    }
}

// Option match — identical pattern
match find_user(id) {
    user & :true => greet(user)
    :false => respond_404()
}
```

**The "any forces all" formatter rule**: if any item in a collection needs expansion (because it's too long, or it contains nested structure), all items expand. Compact exceptions create inconsistency. Consistent visual patterns are easier to scan:

```bounce
// ❌ Mixed — hard to scan
let config = { host: "localhost", port: 5432,
    tls: true, max_connections: 100 }

// ✅ Any forces all — uniform visual rhythm
let config = {
    host: "localhost",
    port: 5432,
    tls: true,
    max_connections: 100,
}
```

---

## 4. The Type System Serves the Programmer

Types catch bugs and communicate intent. They are not an obstacle course. The type system works for the developer, not the other way around.

### Inference by Default, Explicit at Boundaries

Internal functions infer everything. `pub fn` requires explicit effect annotations because those are API contracts — they affect consumers and must be visible without running the compiler.

```bounce
// Internal — fully inferred, no annotations required
fn build_report(user_ids: List<Int>) {
    let users = user_ids |> map(fetch_user) |> collect   // Network inferred
    let template = load_template("report.html")           // FileSystem inferred
    render(template, users)
}

// Public API — effects explicit because callers need to know
pub fn fetch_user(id: Int) -> User
    with Network, Raise<NotFoundError> {
    // ...
}
```

The LSP shows inferred types and effects as inlay hints so internal code is still self-documenting:

```bounce
fn build_report(user_ids: List<Int>) {    // hint: [Network, FileSystem]
    let users = ...                        // hint: List<User>
    let template = ...                     // hint: Template
}
```

### Structural Typing for Data, Nominal for Identity

A record is defined by its shape. If two records have the same fields, they are interchangeable. This makes composition natural and eliminates ceremony:

```bounce
// Any record with at least `x` and `y` satisfies this — no explicit interface needed
fn distance(p: { x: Float, y: Float }) -> Float {
    (p.x * p.x + p.y * p.y).sqrt()
}

type Player = { x: Float, y: Float, health: Int }

let hero = Player { x: 3.0, y: 4.0, health: 100 }
distance(hero)  // ✅ Player structurally satisfies { x: Float, y: Float }
```

`nominal type` exists for when two things with the same shape mean different things:

```bounce
export type UserId = opaque Int
export type OrderId = opaque Int

// These are now distinct types — Int does not satisfy either
fn get_user(id: UserId) -> User { ... }
fn get_order(id: OrderId) -> Order { ... }

// Compile error — UserId ≠ OrderId, even though both are opaque Int
get_user(order.id)
```

### Tags for Refinement, Not Ceremony

Tags (`T & :tag`) communicate facts about values without wrapping them in new types. They are zero-cost compile-time metadata:

```bounce
// :finite tells the compiler this sequence will end — no wrapper type needed
fn collect<T>(self: Sequence<T> & :finite) -> List<T>

// :validated tells downstream code this string passed validation
fn validate_email(s: String) -> String & :validated { ... }

// Calling collect on an infinite sequence is a compile error
fibonacci() |> collect
// Error: `collect` requires Sequence<Int> & :finite,
//        but `fibonacci()` returns Sequence<Int> & :infinite
```

The `Option<T>` type uses this directly: `Option<T>` is `(T & :true) | :false`. The value *is* the truth. No `.value` unwrapping, no `.get()` ceremony:

```bounce
let user: Option<User> = find_user(id)    // (User & :true) | :false

if user {
    // user is narrowed to User & :true — use it directly as User
    greet(user)
}
```

### Errors Are Effects, Not Return Types

`Raise<E>` propagates like any effect — no `?` operators, no wrapping, no unwrapping. The `try` block is the single handle point. Functions that can fail have cleaner signatures:

```bounce
// No Result<User, NotFoundError> — just User
fn get_user(id: Int) -> User {
    // compiler infers: Raise<NotFoundError>
    ...
}

// Errors propagate naturally through pipelines
user_ids |> map(get_user) |> collect
// If any get_user raises, the whole pipeline raises — no special handling
```

### Exhaustiveness Is a Feature

The compiler checks match arms, not you. Missing a union variant is a compile error, not a runtime surprise. This is what makes `match` trustworthy — you can refactor a union and the compiler tells you every place that needs updating:

```bounce
type Status = :pending | :active | :suspended | :deleted

fn describe(s: Status) -> String {
    match s {
        :pending => "Waiting for activation"
        :active => "Account is active"
        :suspended => "Account suspended"
        // Compile error: non-exhaustive match — `:deleted` not handled
    }
}
```

---

## 5. Safe by Default, Powerful When Needed

Bouncelang's defaults eliminate entire categories of bugs. Users opt into complexity when they need it, not by default.

### 5a. Value Semantics by Default

No shared mutable state by accident. Records are values. Copying a record does not share state:

```bounce
let original = { x: 1, y: 2 }
let modified = { ...original, x: 10 }

// original is unchanged — it is a value, not a reference
IO.println(original.x)   // 1
IO.println(modified.x)   // 10
```

Deep spread syntax makes "update a nested field" ergonomic without mutation:

```bounce
// Update deeply nested state without mutation
let next_state = { ...state, player->stats->hp: new_hp }
```

`mut` rebinding allows local computation to be written imperatively when it's clearer:

```bounce
fn fibonacci(n: Int) -> Int {
    mut a = 0
    mut b = 1
    for _ in range(0, n) {
        (a, b) = (b, a + b)
    }
    a
}
```

`mut` is a rebinding modifier on the *binding*, not the value. There is no in-place mutation of a record.

For concurrent shared state, `State<T>` is explicit and scoped (see `07-concurrency.md`).

### 5b. Effect System as Capability Control

Functions declare what they do (`Network`, `FileSystem`, `Concurrency`). The world decides which capabilities are available. Sandboxing is **structural** — the type system enforces it, not runtime checks:

```bounce
world cli {
    // This world provides FileSystem — but NOT Network
    // Any function requiring Network is a compile error in this world
    provide FileSystem = ReadOnly { root: "/data" }
}
```

A function that requires `Network` cannot be called in a world that doesn't provide it. This is checked at compile time, not runtime. There is no "accidentally calling a network function in a sandboxed context" class of bug.

See `04-effects-and-handlers.md` and `worlds-and-handlers.md` for the full specification.

### 5c. Panics for Bugs, Raise for Expected Failures

Two distinct failure modes with a clear boundary:

| Mechanism | Meaning | Compilation | Catchable? |
|---|---|---|---|
| `Raise<E>` | Expected failure — the caller should handle this | Conditional branches | Yes, via `try` |
| `panic` | Programming bug — this should never happen | WASM `unreachable` trap | No |

**Index out of bounds is a bug** — use `panic`. **Network timeout is an expected failure** — use `Raise`. Users never confuse the two because the language makes the distinction structural.

```bounce
// Expected failure — typed, catchable, documented
fn parse_config(path: String) -> Config {
    // compiler infers: Raise<ParseError>, Raise<NotFoundError>
    let content = FileSystem.read(path)
    parse_json(content)
}

// Programming bug — untypeable, uncatchable, terminates the component
fn get_first(list: List<T>) -> T {
    if list.is_empty() {
        panic("get_first called on empty list — this is a bug")
    }
    list[0]
}
```

In DST mode (see §6), panics are captured and reported as test failures with full stack traces, not as host termination.

### 5d. No Implicit Coercions

No implicit numeric widening. No implicit `ToString`. No implicit `null`. Every conversion is explicit and visible:

```bounce
let x: Int = 42
let y: Float = x          // Compile error — no implicit Int → Float
let y: Float = x.to_float()   // ✅ explicit

let message = "Count: " + items.count()   // Compile error — no implicit Int → String
let message = "Count: {items.count()}"    // ✅ explicit interpolation
```

This makes code **predictable and grep-able**. You can search for `.to_float()` and find every place the conversion happens. You cannot accidentally introduce precision loss or unexpected behavior through implicit coercions that are invisible in the source.

---

## 6. Deterministic Simulation Testing (DST) Is a Design Constraint

DST is not a testing library bolted on. It is a **design constraint that shapes language features**. When specifying any feature that involves time, randomness, concurrency, or I/O, ask:

> **"Can a user write a deterministic simulation test for this?"**

If the answer is "they can't" or "it would require mock objects and prayer", the design needs rethinking.

### What Makes DST Possible

All effects (`Network`, `FileSystem`, `Time`, `Concurrency`, `Random`) are replaceable via the world/handler system. The DST world substitutes test implementations that are fully deterministic:

```bounce
world sim {
    // Virtual time — Time.after(30s) advances instantly
    provide Time = VirtualTime { start: Datetime.parse("2026-01-01T00:00:00Z") }

    // Deterministic randomness — same seed → same sequence
    provide Random = SeededRandom { seed: 42 }

    // Fake network — controlled responses, injectable failures
    provide Network = FakeNetwork {
        routes: {
            "GET /users/1": { status: 200, body: test_user_json() }
            "GET /users/2": { status: 404 }
        }
    }

    // In-memory filesystem — no actual disk I/O
    provide FileSystem = MemoryFs {
        files: { "/config.json": test_config() }
    }
}
```

### Virtual Time

Code that relies on real wall-clock time is untestable. Real time makes tests slow, flaky, and non-deterministic. The `Time` effect makes real time an explicit capability — in a DST world, it is replaced with `VirtualTime`:

```bounce
// This test completes instantly — no actual 30-second sleep
test "session expires after timeout" {
    use sim {
        provide Time = VirtualTime { start: Datetime.parse("2026-01-01T00:00:00Z") }
    }

    let session = create_session(user_id: 1)
    Time.advance(31.minutes)     // Instant in VirtualTime
    assert session.is_expired()
}
```

### Deterministic Scheduling

Given the same seed, concurrent code interleaves the same way every time. Concurrent bugs become **reproducible findings**, not "it happens sometimes":

```bounce
test "producer consumer ordering" with seed: 42 {
    // The scheduler uses the seed to determine interleaving
    // Run with seed 42 → same interleaving every time
    Concurrency.scope { s =>
        let ch = s.channel<Int>(10)
        s.spawn { produce(ch, 1..100) }
        s.spawn { consume(ch) }
    }
}
```

### Fault Injection

The DST runner can inject failures at effect boundaries:

```bounce
test "handles network failure on 3rd request" with faults: {
    Network.get: fail_after(count: 2, error: NetworkError { message: "timeout" })
} {
    // The third call to Network.get raises NetworkError
    let result = try { batch_fetch_users([1, 2, 3]) } catch {
        NetworkError => :partial_failure
    }
    assert result == :partial_failure
}
```

### Seeds Make Failures Reproducible

A flaky test is a seed. `bounce test --seed 42` replays it exactly:

```
FAIL: test "concurrent state update" [seed: 7291038]
  Run `bounce test --seed 7291038 --test "concurrent state update"` to reproduce.
```

### The DST Test

When designing a feature, ask: "if a user wrote a DST test for the code that uses this feature, what would that look like?" Apply this when designing:

- Any time-based feature (timers, expiry, rate limiting)
- Any concurrent feature (locks, queues, event streams)
- Any I/O feature (file reads, HTTP calls, database queries)
- Any random or non-deterministic feature

If the test would require mocking frameworks, monkeypatching, or "just don't test that part", the feature design is resisting testability. Redesign so the world/handler system can intercept it.

---

## 7. The LSP Checklist

Every language feature, when added to the spec, must answer all of these questions. This is a **formal checklist**, not a suggestion. A spec section that does not address these is incomplete.

| Checklist Item | Question to Answer | Example |
|---|---|---|
| **Completions** | What does the LSP suggest in the context of this feature? Be specific. | After `s.` in a concurrency scope: `spawn`, `detach`, `state`, `channel`, `broadcast`, `registry`, `cancel` |
| **Inlay hints** | What types/effects/tags does the compiler know that the user didn't write? Show them. | `:finite`/`:infinite` on sequences; inferred effect sets on internal functions; `linked`/`detached` next to spawn calls |
| **Diagnostics** | What mistakes do users make with this feature? What can the compiler detect statically? | TOCTOU on State; unused spawn result; `collect` called on `:infinite` sequence |
| **Quick fixes / code actions** | What mechanical transformations can the LSP offer? | Make effects explicit on `pub fn`; move effect call outside `pure` closure; extract inline union to named type |
| **Hover / go-to-definition** | What information is most useful on hover for this construct? | Full effect set for a function; union shape for a `select` result; tag constraints on a type parameter |
| **Semantic highlighting** | Does this construct benefit from distinct visual treatment? | `spawn` vs `detach`; `pure` closure boundaries; cancellation points |

### Working Through the Checklist: `select`

As a concrete example, here is how `select` (from `07-concurrency.md`) satisfies the LSP checklist:

- **Completions**: After typing `select(`, the LSP suggests named sequence arguments from all `Sequence<T>` values in scope, each with its inferred element type.
- **Inlay hints**: The return type of `select(timer: heartbeat, message: inbox)` is shown as `Sequence<:timer { Instant } | :message { Msg }>`.
- **Diagnostics**: Warning if a sequence passed to `select` is `:finite` and the loop body doesn't handle the case where it completes.
- **Quick fixes**: "Generate match arms" — fills all branches of a `match` on the `select` result, using the union shape the compiler knows.
- **Hover**: Shows the full union type and which source produced each tag.
- **Semantic highlighting**: Each tag arm in the `match` is highlighted with the same color as the corresponding source sequence name.

---

## 8. The Lint Rule Question

When a pattern is problematic but not a type error, it should be a **lint rule**. When specifying a feature, ask:

> **"What incorrect-but-legal things can a user do with this that we should warn about?"**

Built-in lint rules (`std/`) are part of the language. User-defined lint rules are first-class. Both surface through the same LSP machinery — same diagnostic format, same severity levels, same quick-fix integration.

### When to Add What

| Mechanism | Threshold | Examples |
|---|---|---|
| **Compiler error** | Statically provably wrong — no legitimate use exists | Missing match arm; wrong type; unhandled effect in world |
| **Lint error** (default `:error`) | Almost certainly a mistake; legitimate uses are extremely rare | Unused variable; unreachable code after `return`; `pub fn` not exported from module |
| **Lint warning** (default `:warn`) | Often a mistake, but legitimate uses exist | Complexity threshold exceeded; inline union annotation with more than 3 variants |
| **Lint off by default** | Style preference; teams may want it, but it's not universal | Specific naming conventions beyond the enforced ones |

### Built-In Lint Rules

| Rule | Default | Description |
|---|---|---|
| `std/unused` | `:error` | Unused bindings, functions, imports, type parameters |
| `std/unreachable` | `:error` | Code after `return`, `panic`, or exhaustive match |
| `std/deprecated` | `:warn` | Calls to functions or types marked `@deprecated` |
| `std/boundaries` | `:warn` | Module dependency direction — configurable per module |
| `std/no-effects` | `:warn` | A module marked pure imports an effectful module |
| `std/inline-union-complexity` | `:warn` | Inline union annotation with more than 3 variants |
| `std/toctou` | `:warn` | Read-then-update on the same `State` handle in the same scope |
| `std/unused-task` | `:warn` | `spawn` result is never `join`ed or consumed |

### User-Defined Lint Rules

Teams define custom rules in Bouncelang itself. The LSP discovers them from the project's package configuration. A custom rule has the same format, severity levels, and quick-fix capability as a built-in rule.

This feature needs its own spec. The principle here: **domain-specific correctness rules deserve first-class tooling support**, not third-party linter plugins with degraded IDE integration.

Example use cases for custom lint rules:

- `org/no-raw-sql` — warn when SQL strings are not parameterized
- `org/model-purity` — error when the `models` module imports from `controllers`
- `org/error-user-message` — warn when a public `error` type lacks a `user_message: String` field

---

## 9. Practical Tradeoff Guide

Design decisions often involve genuine tradeoffs. When two good principles conflict, use this priority order:

### Priority Order

| Priority | Principle | Meaning |
|---|---|---|
| 1 | **Safety** | Does this prevent a class of bugs? A feature that makes unsafe code impossible is worth almost any other cost. |
| 2 | **Understandability** | Can a new user read this and understand it within 30 seconds? Complexity that doesn't pay for itself in safety must be justified by understandability. |
| 3 | **DX / Ergonomics** | Is this pleasant to write? Boilerplate that serves no safety or clarity purpose should be eliminated. |
| 4 | **Power / Expressiveness** | Can advanced users do more with this? Power that conflicts with safety or understandability requires a strong case. |
| 5 | **Performance** | Is this fast? Performance is never ignored, but it is the last thing to sacrifice other principles for. The compiler and runtime handle performance; the language spec does not compromise safety for it. |

### Anti-Priorities

These are explicitly **not** goals. When a proposal is justified by one of these, it should be rejected or redesigned:

| Anti-priority | Why it's not a goal |
|---|---|
| **Minimizing keystrokes** | Saving 3 keystrokes at the cost of clarity costs every reader of that code. Readers outnumber writers. |
| **Academic purity** | The type system serves the programmer, not the other way around. A theoretically elegant system that is confusing in practice is a failure. |
| **Feature parity** | We add features because they solve real problems, not to check boxes relative to other languages. |
| **Configuration options** | Every configuration option is a decision the user must make. We make decisions for them when we can. Fewer options = fewer disagreements = faster teams. |

### Worked Example: Named vs. Positional Arguments

**Proposal**: Allow positional arguments for functions where the argument order is obvious (e.g., `range(0, 10)`).

**Analysis**:

- Safety: Positional args with non-obvious order cause bugs (e.g., `connect("host", 5432, true)` — what is `true`?). Named args prevent transposition bugs. ↑ Safety for named.
- Understandability: `range(0, 10)` is readable; `create_user("Alice", "alice@example.com", :admin)` is not. Named args help at scale. ↑ Understandability for named (especially as argument count grows).
- DX: Typing `start: 0, end: 10` is more verbose. ↓ Ergonomics for named (slightly).
- Power: Positional enables shorter syntax in cases where order is obvious. ↑ Slightly.

**Decision**: Bouncelang uses named arguments for any function where the argument semantics are not obvious from position. The formatter makes named args visually lightweight. The LSP auto-completes argument names, eliminating most of the typing overhead. The safety and understandability gains dominate the ergonomics cost.

---

## Synthesis: The Original Principles

The following foundational principles from Bouncelang's original design are synthesized throughout this document. They are collected here for reference:

**Deep spread syntax over mutation** — updating deeply nested records should be ergonomic without mutation. `{ ...state, player->stats->hp: new_hp }` is the idiomatic pattern. See §5a.

**Lodash-like standard library** — functional transformations (`fold`, `filter_map`, `chunk`, `group_by`) are the path of least resistance. Imperative loops and manual accumulation are available but not the default. See §5a and `02-data-structures.md`.

**Expressions everywhere** — `if`/`else`, `match`, `try` all return values. Declarative bindings (`let x = if cond { a } else { b }`) are idiomatic. See §3.

**Unified `:true`/`:false` truthiness** — `Option<T>` is `(T & :true) | :false`. `Bool` is `:true | :false`. The same `if`, `match`, and `?.` patterns work uniformly across all three. See §4 and `02-data-structures.md` §3.

**Pragmatic early exits** — `return` is provided for early exits to avoid the arrow anti-pattern. Expression orientation is a default, not a dogma. See §1.

**Standard library as ecosystem vocabulary** — `Request`, `Response`, `Url`, `Datetime`, `Duration` live in the standard library. Third-party libraries use these types to interoperate. `std/json` and `std/regex` are batteries included. See §3 of the original and `modules-and-imports.md`.

**Worlds as declarative execution environments** — the world defines the capability sandbox and runtime hosting. Code focuses on logic; the world defines how it runs and what it can access. See §5b and `worlds-and-handlers.md`.

**Containerless composition via WASM** — worlds natively support WASM Component Linking, enabling sandboxed deployment to CLIs, servers, and serverless runtimes without container overhead.
