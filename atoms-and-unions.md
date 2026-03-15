# Atoms, Unions, and Pattern Matching

**Status:** Draft — evolved from design review discussion
**Related:**
- [methods-and-packages.md](methods-and-packages.md) — UFCS, companions, operators (nominal types)
- [worlds-and-handlers.md](worlds-and-handlers.md) — worlds, handlers, config
- [modules-and-imports.md](modules-and-imports.md) — sub-modules, imports, stdlib
- [02-data-structures.md](02-data-structures.md) — intersection tags, truthiness model

---

## 1. Core Concept: Atoms Are Typed Enum Variants

Atoms are **not** open-ended runtime values (unlike Elixir/Erlang). They are typed, closed enum variants. Every atom belongs to a known union type. The compiler checks exhaustiveness.

```bounce
type OrderStatus = :pending | :processing | :complete | :cancelled

let status: OrderStatus = :pending     // ✅ valid
let status: OrderStatus = :unknown     // ❌ compile error — :unknown is not in OrderStatus
```

Atoms are Bouncelang's answer to discriminated unions. Where TypeScript uses `{ type: 'success' } | { type: 'fail', reason: string }`, Bouncelang uses `:success | :fail { reason: String }` — the atom name IS the discriminator, first-class.

---

## 2. Syntax

### Bare Atoms

```bounce
:ok
:error
:pending
:north
```

Always prefixed with `:`. This distinguishes atoms from variable names and type names.

### Atoms With Data

Atoms can carry named fields — like a discriminator + anonymous record:

```bounce
:ok { value: T }
:redirect { url: String, permanent: Bool }
:error { code: Int, message: String }
```

Multiple fields are allowed. Fields are structurally typed.

### Union Types

Unions are pipe-separated sets of atom variants:

```bounce
type Toggle = :on | :off

type HttpResponse =
    | :ok { status: Int, body: String }
    | :redirect { url: String }
    | :not_found

type Shape =
    | :circle { radius: Float }
    | :rect { width: Float, height: Float }
    | :triangle { a: Float, b: Float, c: Float }
```

---

## 3. Formatter Rules

### Leading Pipe

The formatter always adds a leading `|` on the first variant when multi-line. Every variant line is syntactically identical — clean diffs, easy to comment out:

```bounce
type HttpResult =
    | :ok { status: Int, body: String }
    | :redirect { url: String }
    | :not_found
    | :error { code: Int, message: String }

// Comment out during debugging:
type HttpResult =
    | :ok { status: Int, body: String }
    // | :redirect { url: String }
    | :not_found
    | :error { code: Int, message: String }
```

### Single-Line vs Multi-Line

**Rule:** if ANY variant carries data, the formatter always multi-lines:

```bounce
// All bare atoms — single line
type Toggle = :on | :off
type Direction = :north | :south | :east | :west

// Any variant has data — always multi-line, leading pipes
type HttpResponse =
    | :ok { status: Int, body: String }
    | :redirect { url: String }
    | :not_found                              // bare, but formatted multi-line with siblings

type Shape =
    | :circle { radius: Float }
    | :rect { width: Float, height: Float }
    | :point                                  // bare, but formatted multi-line with siblings
```

---

## 4. Inline Unions

Unions can be used inline without a named type:

```bounce
fn toggle(state: :on | :off) -> :on | :off
fn check(input: String) -> :valid | :invalid
```

### Lint Rule: Inline Complexity

The lint `std/inline-union-complexity` measures union complexity, not just variant count:

```bounce
// ✅ Fine — 6 bare atoms, low complexity
fn handle(event: :click | :hover | :focus | :blur | :keydown | :keyup) -> ()

// ✅ Fine — 2 variants with data
fn fetch() -> :ok { data: String } | :err { message: String }

// ⚠️ Lint warning — 3+ variants with data, extract a type
fn process() -> :ok { data: List<User> } | :partial { data: List<User>, errors: List<Error> } | :fail { reason: String }
// Fix: type ProcessResult = :ok { ... } | :partial { ... } | :fail { ... }
```

Bare atoms are cheap (complexity = 1 each). Data-carrying atoms are expensive (complexity = 1 + field count). The lint fires when total complexity exceeds a threshold.

---

## 5. Structural Subtyping

Union types follow structural subtyping rules. A union with fewer variants is a subtype of a union with more variants:

```bounce
type SmallSet = :a | :b
type BigSet = :a | :b | :c | :d

// SmallSet is a subtype of BigSet
fn handle_big(value: BigSet) -> ()

let x: SmallSet = :a
handle_big(x)    // ✅ every SmallSet value is a valid BigSet value
```

This means unions are compatible based on their shape, not their name. A function accepting `BigSet` also accepts any `SmallSet` value — structural subtyping.

---

## 6. Pattern Matching

### Exhaustiveness Checking

The compiler verifies all variants are handled:

```bounce
match order.status {
    :pending    => process(order)
    :processing => wait()
    :complete   => archive(order)
    // ❌ compile error: unhandled variant :cancelled
}
```

### Destructuring Data

Pattern matching destructures atom data inline:

```bounce
match result {
    :ok { value }                  => use(value)
    :err { error }                 => log(error)
}

match shape {
    :circle { radius }             => Float.pi * radius * radius
    :rect { width, height }        => width * height
    :triangle { a, b, c }          => heron(a, b, c)
}
```

### Nested Patterns

Patterns can match on specific field values:

```bounce
match response {
    :ok { status: 200, body }          => parse(body)
    :ok { status: 301, ... }           => follow_redirect()
    :ok { status, ... } if status > 399 => handle_error(status)
    :error { message }                  => log(message)
}
```

### Pipeline Matching

Match works in pipelines:

```bounce
order
    |> validate
    |> match {
        :ok { order }  => process(order)
        :err { error } => log_and_default(error)
    }
    |> save
```

### Wildcard and Rest

```bounce
match value {
    :ok { value, ... }  => value     // ignore extra fields
    _                   => default   // wildcard — matches anything
}
```

---

## 7. Prelude Unions

These union types are always in scope — no import needed:

```bounce
// Bool is a union of two atoms
type Bool = :true | :false
```

> **`Option<T>` and the extended truthiness model.** Bouncelang defines a fixed set of truthy and
> falsy atom tags for use in `if` conditions:
>
> | Category | Tags |
> |---|---|
> | Truthy | `:true`, `:some`, `:ok` |
> | Falsy | `:false`, `:none`, `:error` |
>
> Any value whose type is `(T & truthy_tag) | falsy_tag` can be used directly in an `if` condition.
> In the truthy branch, the tag is stripped and the inner `T` is accessible directly.
>
> This powers three standard optionality/result patterns:
>
> ```bounce
> // Option<T>  =  T?  =  (T & :true) | :false
> let name: String? = user.nickname
> if name {
>     Terminal.println("Hello, {name}")   // name: String (tag stripped)
> }
>
> // Result<T, E>  =  (T & :ok) | (E & :error)
> let result: (User & :ok) | (DbError & :error) = Database.find_user(id)
> if result {
>     serve_user(result)            // result: User (tag stripped)
> } else {
>     Log.error("{result.message}") // result: DbError (tag stripped)
> }
>
> // Nullable<T>  =  (T & :some) | :none  (alternative alias — e.g., for JSON null)
> let value: (String & :some) | :none = json_field
> if value {
>     process(value)    // value: String
> }
> ```
>
> Flow-sensitive narrowing in `if` and `match` gives the compiler proof of tag presence without any
> runtime cost. The tag is compile-time metadata only.
>
> **`match` also narrows:** In a `match` arm, the compiler strips the matched tag from the type of
> the binding:
>
> ```bounce
> match name {
>     :false => Terminal.println("no name")
>     name   => Terminal.println("Hello, {name}")    // name: String (tag stripped)
> }
> ```
>
> See [01-primitives.md §3](01-primitives.md) for the full truthiness, `if`/`else`, and `if let`
> design decisions.

> **`Result<T, E>` — no prelude type alias, but the pattern is first-class.** The truthiness
> model naturally supports a `Result`-like type as `(T & :ok) | (E & :error)`. Projects can
> declare `type Result<T, E> = (T & :ok) | (E & :error)` and use it with `if`. No stdlib
> `Result` alias is currently planned — the `try` / `Raise<E>` model covers the same ground
> without a wrapper type.

---

## 8. Atoms in Records

Atoms as record field values work naturally:

```bounce
type Order = {
    id: Int,
    status: :pending | :processing | :complete | :cancelled,
    priority: :low | :medium | :high,
}

let order = Order {
    id: 42,
    status: :pending,
    priority: :high,
}
```

The `status:` and `priority:` fields are typed as inline unions. Pattern matching on the record works:

```bounce
match order {
    { status: :complete, ... }   => archive(order)
    { priority: :high, ... }     => expedite(order)
    _                            => enqueue(order)
}
```

---

## 9. Tags — Adopted in V2

Intersection tags (`String & :email`, `Connection & :authenticated`, `Sequence<T> & :finite`) are
a **core part of v2**. They are compile-time, zero-cost metadata attached to any type that create
an intersection type `Base & :Tag`.

The `:true` and `:false` tags power the entire truthiness and `Option<T>` model (see §7 above).
The `:finite` and `:infinite` tags are used by `Sequence<T>` to enforce at compile time which
operations require a bounded sequence. The `:validated` pattern enables evidence-carrying types.

```bounce
// Tags are additive — they compose
let validated: Email & :validated = validate(raw_string)

// Tags are zero-cost — erased at runtime
// Tags are compile-time evidence — the compiler tracks them through the call graph

// Sequences carry finiteness tags
fn collect<T>(self: Sequence<T> & :finite) -> List<T>    // only finite sequences can be collected
fn fibonacci() -> Sequence<Int> & :infinite               // can never be collected

// Structural records and nominal types can both be tagged
let point: Point & :origin = { x: 0, y: 0 }
let conn: Connection & :authenticated = authenticate(raw_conn)
```

For the full tag model — intersection types, flow-sensitive narrowing, evidence decay — see
[02-data-structures.md §3](02-data-structures.md).

---

## 10. DST / Testing

### Property-based Testing Over Unions

Union types compose naturally with the property-based test generator:

```bounce
test fn exhaustive_order_status(status: OrderStatus) with [gen(OrderStatus)] {
    // Generator produces all variants — compiler verifies exhaustiveness at compile time
    match status {
        :pending    => assert(can_process(status))
        :processing => assert(is_active(status))
        :complete   => assert(is_terminal(status))
        :cancelled  => assert(is_terminal(status))
    }
}
```

### Atoms in DST State Machines

Atoms are the natural building block for DST state machine assertions. Because union exhaustiveness
is compile-time enforced, no state transition is accidentally unhandled in the test:

```bounce
test fn order_state_machine(seed: Int) with [sim: seed(seed)] {
    let order = create_order()

    // Deterministic simulation drives the state machine
    Concurrency.scope { s =>
        let state = s.state(order)
        simulate_order_lifecycle(s, state)

        let final = state.read { o => o.status }
        match final {
            :complete  => assert_receipt_exists(order.id)
            :cancelled => assert_refund_issued(order.id)
            _          => fail("order in unexpected terminal state: {final}")
        }
    }
}
```

### No Special DST Behavior

Atoms and unions have no special DST behavior beyond what the general type system provides.
They are pure compile-time and value-semantic constructs with no I/O, no time dependency, and
no concurrency concerns — they always behave identically in test and production.

---

## 11. LSP / DX (Developer Experience)

| Checklist Item | Behavior |
|---|---|
| **Completions** | Typing `:` after `type Foo =` offers completions for all atoms defined in the current file and imported modules. Typing `:` inside a `match` arm suggests atoms from the matched union type. |
| **Inlay hints** | For inline unions (`fn toggle(s: :on \| :off)`), the LSP shows the inferred type on hover without needing a named `type` alias. For `T?` sugar, inlay hints show the full `(T & :true) \| :false` expansion on hover. |
| **Diagnostics** | Missing `match` arm: "Unhandled variant `:cancelled` in match on `OrderStatus`". Incorrect atom: "`:unknown` is not a member of `OrderStatus`". Inline union complexity lint: "3+ data-carrying variants — extract a named type". |
| **Quick fixes** | "Add missing arms" — inserts all unhandled variants as stubs. "Extract inline union to named type" — creates a `type` alias and replaces the inline usage. |
| **Hover / go-to-definition** | Hovering an atom (`:pending`) shows the union type it belongs to. Go-to-definition navigates to the `type` declaration. |
| **Semantic highlighting** | Atoms (`:pending`, `:error`) receive a distinct semantic token (e.g., `enumMember`) separate from type names and variable names. The leading `:` is highlighted as part of the atom, not as a punctuation character. |

---

## 12. Compiling & WebAssembly (Wasm)

Atoms and unions map directly to the WebAssembly Component Model's `variant` type:

```
// Bouncelang
type HttpResponse =
    | :ok { status: Int, body: String }
    | :redirect { url: String }
    | :not_found
```

```wit
// Generated WIT
variant http-response {
    ok(ok-fields),
    redirect(redirect-fields),
    not-found,
}
record ok-fields { status: s64, body: string }
record redirect-fields { url: string }
```

**Bare atoms** (no data) compile to `variant` cases with no associated type — zero payload, one discriminant byte.

**Atoms with data** compile to `variant` cases with an associated `record` — one discriminant byte plus the record fields.

**Intersection tags** (`:true`, `:finite`, `:validated`) are erased entirely at the Wasm boundary — they are compile-time metadata only. The underlying type is emitted as-is.
