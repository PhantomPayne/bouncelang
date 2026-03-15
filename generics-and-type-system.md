# Generics and Type System

**Status:** Draft — evolved from design review discussion
**Related:**
- [atoms-and-unions.md](atoms-and-unions.md) — atoms, unions, pattern matching
- [methods-and-packages.md](methods-and-packages.md) — UFCS, companions, exports
- [error-handling.md](error-handling.md) — errors, Raise effect

---

## 1. Three Tiers of Types

Bouncelang has three kinds of types, each with clear rules:

### `type` — Structural Data

Transparent records. Fields are the type. Always constructible, always spreadable.

```bounce
type Point = { x: Float, y: Float }
type UserRequest = { name: String, email: String, role: :admin | :user }
```

```bounce
let p = Point { x: 1.0, y: 2.0 }
let moved = { ...p, x: p.x + 1.0 }      // ✅ spread
let name = request.name                   // ✅ field access
```

Structural types are compared by shape — two types with the same fields are interchangeable.

### `nominal type` — Named Wrapper

A transparent type with a distinct identity. Same as its underlying type but not interchangeable with it. For type safety — preventing accidental mixing of semantically different values.

```bounce
nominal type Email = String
nominal type UserId = Int
nominal type Money = { amount: Int, currency: String }
```

```bounce
let e = Email("user@test.com")            // ✅ direct construction
let s: String = e                          // ❌ Email is not String
let m = Money { amount: 1000, currency: "USD" }   // ✅ transparent
let price = m.amount                       // ✅ field access
let doubled = { ...m, amount: 2000 }       // ✅ spread
```

Use nominal types when you want the compiler to prevent confusion — `UserId` vs `OrderId`, `Email` vs `String`. No invariant enforcement — if you need validation, use `opaque type`.

### `opaque type` — Abstract Data Type

Hidden internals, controlled API. Construction only through factory functions. Fields accessible only through the `with` list in exports.

```bounce
opaque type LinkedList<T> = {
    head: Node<T>?,
    size: Int,
}

type Node<T> = { value: T, next: Node<T>? }
```

```bounce
// Inside the defining module: full access
fn empty<T>() -> LinkedList<T> {
    LinkedList { head: :false, size: 0 }
}

fn push<T>(self: LinkedList<T>, item: T) -> LinkedList<T> {
    LinkedList {
        head: :true { value: Node { value: item, next: self.head } },
        size: self.size + 1,
    }
}

fn len<T>(self: LinkedList<T>) -> Int { self.size }
```

```bounce
// Outside the defining module:
let list = LinkedList.empty()             // ✅ factory
let list = LinkedList { head: ... }       // ❌ can't construct directly
list.head                                  // ❌ can't access internals
list.len()                                 // ✅ companion function
```

### Export Syntax

```bounce
// package.bounce

// Structural — transparent data (fields auto-exported)
export type UserRequest

// Nominal — transparent, named wrapper (fields auto-exported)
export nominal type Email with { validate, domain }
export nominal type Money with { add, display }

// Opaque — hidden internals, factory API (fields hidden)
export opaque type LinkedList<T> with { empty, push, pop, len, iter }
export opaque type Connection with { open, query, close }
```

For opaque types, the `with { }` list can include field names for read-only access:

```bounce
export opaque type Money with { amount, currency, new, add }
// amount and currency: read-only field access
// new, add: companion functions
// Direct construction: ❌ always factory-only for opaque types
```

### Summary

| Kind | Construct? | Access fields? | Spread? | Invariants? |
|---|---|---|---|---|
| `type` | ✅ direct | ✅ all | ✅ yes | No — pure data |
| `nominal type` | ✅ direct | ✅ all | ✅ yes | No — just type safety |
| `opaque type` | ❌ factory only | `with` list only | ❌ no | Yes — hidden internals |

**Record construction forms:** For structural (`type`) and nominal types, two equivalent
construction forms are valid:

```bounce
type Point = { x: Float, y: Float }

// Form 1 — TypeName { fields }
// The compiler verifies all required fields against the named type.
let p1 = Point { x: 1.0, y: 2.0 }

// Form 2 — structural literal with type annotation on the binding
// The compiler infers the shape from context.
let p2: Point = { x: 1.0, y: 2.0 }
```

Both forms are correct. `TypeName { }` is preferred when you want the compiler to check field
completeness at the construction site. `{ }` with an annotation is preferred when the type is
clear from context (e.g., a function return type or a known parameter type).

**Opaque primitive wrappers:** An opaque type can wrap a single primitive:

```bounce
opaque type UserId = Int
opaque type Email = String
```

This is distinct from `nominal type` (which is transparent) — external code cannot see the
underlying `Int`. The opaque wrapper is constructed and accessed only through factory functions
listed in the `with` clause. Zero cost at runtime — it compiles to the underlying type.

---

## 2. Generics

### Type Parameters

Functions and types can be generic over type parameters:

```bounce
fn first<T>(list: List<T>) -> T?
fn map<T, U>(list: List<T>, f: fn(T) -> U) -> List<U>
fn zip<A, B>(a: List<A>, b: List<B>) -> List<{ first: A, second: B }>

type Pair<A, B> = { first: A, second: B }
type Tree<T> = :leaf | :node { value: T, left: Tree<T>, right: Tree<T> }
```

### Type Inference

Hindley-Milner style. Explicit `<T>` is rarely needed — the compiler infers from usage:

```bounce
// Inferred — no annotation needed
let numbers = [1, 2, 3]                          // List<Int>
let doubled = numbers |> map { x => x * 2 }      // List<Int>
let names = users |> map { u => u.name }          // List<String>

// Explicit when ambiguous
let empty = List<Int>.empty()
```

### Constraints

Constraints are only needed when a generic function must call a behavior on `T` — sorting needs comparison, map keys need hashing:

```bounce
// Inline constraint — simple cases
fn sort<T: Ord>(list: List<T>) -> List<T>
fn deduplicate<T: Eq>(list: List<T>) -> List<T>

// where clause — multiple or complex constraints
fn group_by<T, K>(list: List<T>, key: fn(T) -> K) -> Map<K, List<T>>
    where K: Hash + Eq

fn sort_and_display<T>(list: List<T>) -> List<String>
    where T: Ord + Display
```

Structural typing handles everything else — if a function takes `{ name: String }`, any type with a `name: String` field works. No constraint needed.

### Function Literals (Lambdas)

`fn(T) -> U` is a **type annotation** for a function value. The **literal** syntax for anonymous
functions is `{ params => expr }` — a block with a `=>` separating the parameter list from the
body.

```bounce
// Single parameter
let double = { x => x * 2 }                       // fn(Int) -> Int (inferred)

// Multiple parameters
let add = { x, y => x + y }                       // fn(Int, Int) -> Int

// Zero parameters (thunk)
let greeting = { => "Hello!" }                     // fn() -> String
// or just a block (when it captures nothing):
let greeting = { "Hello!" }

// Multi-line body — last expression is the return value
let describe = { score =>
    if score >= 80 { "pass" }
    else if score >= 60 { "borderline" }
    else { "fail" }
}
```

Lambdas work in **any argument position**, not just the final trailing argument:

```bounce
// Lambda as a non-trailing argument
fn apply_twice<T>(f: fn(T) -> T, value: T) -> T { f(f(value)) }

let result = apply_twice({ x => x * 2 }, 3)       // result == 12

// Pipeline position (most common)
scores |> filter { s => s.value > 80 } |> map { s => s.player }
```

The `fn(T) -> U` type annotation describes what a lambda argument must look like. You do not write
`fn(x) => expr` as a value — the `{ x => expr }` form is the only anonymous function syntax.

### What We Don't Have

No type-level computation. The type system is simple and predictable:

- ❌ No conditional types (`T extends U ? A : B`)
- ❌ No mapped types (`{ [K in keyof T]: ... }`)
- ❌ No `infer` keyword
- ❌ No template literal types
- ❌ No index signatures (`{ [key: string]: T }`) — use `Map<String, T>`
- ❌ No `as const` — atoms are already literal types, values are already immutable
- ❌ No higher-kinded types in V1

---

## 3. Interfaces

Interfaces define behavioral contracts. They are the **only** reason `where` constraints exist — structural typing handles everything shape-based.

### Definition

```bounce
interface Display {
    fn display(self) -> String
}

interface Ord {
    fn compare(self, other: Self) -> :less | :equal | :greater
}
```

**`self` in interface methods means the implementing type.** When you write `fn display(self) -> String`
in an interface, `self` is a placeholder for "the type that satisfies this interface." The compiler
substitutes the concrete type at the call site. You cannot annotate `self` with a specific type
inside an interface declaration — if you need a function that takes a fixed type, declare it as a
plain function, not an interface method.

```bounce
// ✅ Correct — self is the implementing type
interface Parseable {
    fn parse(bytes: Bytes) -> Self   // Self refers to the implementing type
}

// ❌ Wrong — self: Bytes means "the implementing type IS Bytes"
// interface Parseable {
//     fn parse(self: Bytes) -> T   // this won't work as intended
// }
```

### Structural Satisfaction

No `implements` declaration. If a matching function exists in scope, the type satisfies the interface:

```bounce
interface Display { fn display(self) -> String }

// User satisfies Display because this companion exists:
fn display(self: User) -> String { "{self.name} <{self.email}>" }

// The compiler sees: User satisfies Display
// No declaration needed
```

### Auto-Derivation

The compiler auto-derives common interfaces from type fields. Explicit implementations override:

```bounce
type User = { name: String, email: Email, age: Int }
// Auto-derived: Eq, Hash, Debug, Serialize, Deserialize
// Not auto-derived: Display, Ord (require domain-specific logic)

fn display(self: User) -> String {
    "{self.name} <{self.email}>"
}
```

### Core Interfaces

#### Representation

```bounce
interface Display { fn display(self) -> String }     // string interpolation, user-facing
interface Debug { fn debug(self) -> String }          // debug output, auto-derived
```

`"{user}"` calls `display`. Debug output in test failures and logging calls `debug`.

#### Comparison

```bounce
interface Eq { fn eq(self, other: Self) -> Bool }
interface Ord { fn compare(self, other: Self) -> :less | :equal | :greater }
interface Hash { fn hash(self) -> Int }
```

`==` calls `eq`. `<`, `>`, `<=`, `>=` call `compare`. `Map` and `Set` require `Hash + Eq`.

#### Arithmetic (Nominal Types Only)

```bounce
interface Add { fn add(self, rhs: Self) -> Self }
interface Sub { fn sub(self, rhs: Self) -> Self }
interface Mul { fn mul(self, rhs: Self) -> Self }
interface Div { fn div(self, rhs: Self) -> Self }
interface Neg { fn neg(self) -> Self }
```

`+` calls `add`, `-` calls `sub`, etc. Only meaningful on nominal types — structural records don't get operator overloading.

```bounce
nominal type Money = { amount: Int, currency: String }

fn add(self: Money, rhs: Money) -> Money {
    assert(self.currency == rhs.currency)
    Money { amount: self.amount + rhs.amount, currency: self.currency }
}

let total = price + tax    // calls add(price, tax)
```

#### Collections

```bounce
interface Iterable<T> { fn iter(self) -> Iterator<T> }
interface Iterator<T> { fn next(mut self) -> T? }
interface Sized { fn len(self) -> Int }
interface Index<K, V> { fn get(self, key: K) -> V? }
```

These enable custom data structures to work with pipelines, `for` loops, and collection operations. `List`, `Map`, `Set` implement them in the stdlib. User-defined collections implement them to integrate with the ecosystem.

#### Serialization

```bounce
interface Serialize { fn serialize(self) -> Bytes }
interface Deserialize { fn deserialize(bytes: Bytes) -> Self }
```

One standard way to serialize. Auto-derived from fields. Format (JSON, MessagePack, etc.) is determined by the handler:

```bounce
world api {
    handle Serialize with JsonSerializer
}
```

---

## 4. Value Semantics and Mutability

### Everything Is a Value

All data in Bouncelang is immutable. There are no references, no aliasing, no shared mutable state.

```bounce
let list1 = LinkedList.empty()
let list2 = list1 |> push(1)     // list1 unchanged, list2 is a new value
let list3 = list2 |> push(2)     // list2 unchanged, list3 is a new value
```

### Mutable Local Bindings (`mut`)

`mut` on a binding means the variable name can be reassigned. The **values** themselves never
change — `mut` is a property of the binding, not the value.

```bounce
// `let mut` and `mut` are equivalent — `let` is optional.
let mut counter = 0
counter = counter + 1
counter = counter + 1   // counter is now 2

// Shorthand without `let` (common in generators and loops):
mut total = 0
total = total + item
```

`mut` is valid in any function context: regular functions, generators, handlers, and test
functions. Mutations inside a closure capture the value at the time the closure is created —
the closure has its own copy, so mutations inside a closure do not affect the outer binding:

```bounce
mut count = 0
let increment = { => count = count + 1 }   // captures a COPY of count
increment()
increment()
// count is still 0 — the closure mutated its own copy
```

To share mutable state across closures and concurrent tasks, use `State<T>` from the
`Concurrency` effect (see `07-concurrency.md`).

### `mut` Fields — In-Place Update Sugar

`mut` on a record field is sugar for "create a new value with this field changed":

```bounce
type Counter = { mut count: Int }

let mut c = Counter { count: 0 }
c.count += 1
// Desugars to: c = Counter { ...c, count: c.count + 1 }
```

The semantics are always "new value." The compiler optimizes to in-place mutation when safe (single ownership, no aliasing).

### No Aliasing Guarantee

The compiler guarantees a value has exactly one owner:

```bounce
let mut a = LinkedList.empty() |> push(1) |> push(2)
let b = a            // a is MOVED to b — a is no longer usable
// a |> push(3)      // ❌ compile error: a has been moved
```

Because there is no aliasing, the compiler can always mutate in place when an operation consumes the only reference. The programmer writes functional code; the compiler generates imperative code.

### Persistent Data Structures

Standard collections use structural sharing internally:

```bounce
let list1 = [1, 2, 3, 4, 5]
let list2 = list1 |> push(6)
// list2 shares the [1, 2, 3, 4, 5] with list1 — only the new tail is allocated
```

The compiler handles this automatically. Developers never think about it.

---

## 5. Recursive Types

Recursive types are supported. The compiler inserts necessary indirection (heap allocation) automatically:

```bounce
type Tree<T> = :leaf | :node { value: T, left: Tree<T>, right: Tree<T> }
type Json = :null | :bool { value: Bool } | :number { value: Float } | :string { value: String }
    | :array { items: List<Json> } | :object { fields: Map<String, Json> }
```

No `Box`, no manual heap allocation. The compiler determines what needs indirection.

---

## 6. V2 Decisions

### `:true/:false` Unification — Adopted

The v2 spec adopts the unified truthiness model. `Option<T>` is `(T & :true) | :false`. `T?` is
syntactic sugar. This eliminates all special `Option` unwrapping — the value IS the option when
present — and makes `if`, `match`, and flow-sensitive narrowing work uniformly across booleans,
optionals, and tagged results.

See [02-data-structures.md §3](02-data-structures.md) for the full model, including evidence
decay and the interaction with nested updates.

### `if let` Syntax — Resolved by `match`

Explicit `if let` syntax is not needed. `match` handles all pattern matching in conditionals, and
flow-sensitive narrowing in plain `if` covers the common "is it present?" case for `Option<T>`.

```bounce
// Flow-sensitive narrowing in if — covers the common case
if name {
    Terminal.println("Hello, {name}")    // name narrowed to String & :true
}

// match — for when you need the :false branch or complex patterns
match name {
    :false => Terminal.println("no name")
    name   => Terminal.println("Hello, {name}")
}
```

### Higher-Kinded Types — Deferred

Higher-kinded types remain deferred beyond v2. The current generic system (concrete type
parameters with constraints) covers all standard use cases in the spec corpus. HKTs will be
reconsidered when real-world Bouncelang code demonstrates a genuine need.

---

## 7. DST / Testing

### Generics Are DST-Transparent

Generic functions and types have no special DST behavior. Because all values are immutable and
all effects are tracked, generic code is inherently testable — a generic `fn sort<T: Ord>` has
no effects, so it works identically in test and production with no handler configuration.

### Property-Based Testing With Generics

The standard `gen` generator works with generic types:

```bounce
test fn sort_is_idempotent<T: Ord>(list: List<T>) with [gen(List<Int>)] {
    let sorted = sort(list)
    assert_eq(sorted, sort(sorted))    // sorting a sorted list gives the same list
}

test fn map_preserves_length<T, U>(list: List<T>, f: pure fn(T) -> U) with [
    gen(List<Int>),
    gen(pure fn(Int) -> String = { n => n |> to_string }),
] {
    assert_eq(list.len(), list |> map(f) |> len())
}
```

### `pure fn` in DST

`pure fn` constraints are especially useful in DST because they guarantee no hidden I/O. The
`State<T>` closures in `07-concurrency.md` use `pure fn` to prevent blocking the coordinator
task — this makes concurrent state updates deterministic under the DST scheduler.

### Value Semantics and Reproducibility

Because all data is immutable and all mutation is via rebinding or structural update, any sequence
of operations on a generic type is reproducible from the same input. There is no aliasing,
no hidden mutation — the DST invariant "same seed → same execution" holds trivially for all
generic code.

---

## 8. LSP / DX (Developer Experience)

| Checklist Item | Behavior |
|---|---|
| **Completions** | After typing `<`, the LSP suggests known type parameters in context. After `where`, the LSP suggests available interfaces (`Ord`, `Hash`, `Display`, etc.) with documentation. After `fn f<T: `, the LSP autocompletes constraint names. |
| **Inlay hints** | On every variable binding, the inferred type is shown: `let doubled = numbers \|> map { x => x * 2 }  // List<Int>`. On generic calls, inferred type arguments are shown: `sort(users)  // <User>`. |
| **Diagnostics** | Missing constraint: "Function `sort` requires `T: Ord`, but `User` does not satisfy `Ord` — add `fn compare(self: User, other: User) -> :less \| :equal \| :greater` to satisfy the constraint". Type mismatch in generic: shows the expected and actual types with the full generic substitution applied. |
| **Quick fixes** | "Generate `Display` implementation" — when a type is missing an interface the call site needs, inserts a stub implementation. "Make parameter `pure`" — when a function is already pure, offers to annotate it. |
| **Hover / go-to-definition** | Hovering a type variable (`T`) inside a generic function shows all constraints on it. Hovering an interface name shows all types in scope that satisfy it. Go-to-definition on an interface navigates to the `interface` declaration. |
| **Semantic highlighting** | Type parameters (`T`, `K`, `V`) receive a distinct semantic token (e.g., `typeParameter`). Interface names in `where` clauses receive the `interface` token. `pure` keyword on function types is highlighted distinctly. |

---

## 9. Compiling & WebAssembly (Wasm)

### Monomorphization

Bouncelang uses monomorphization for generics — the compiler generates a concrete implementation
for each distinct set of type arguments used. `sort<Int>`, `sort<String>`, and `sort<User>` are
three distinct functions in the Wasm output.

This means generic code has zero runtime overhead — no boxing, no vtables, no type tags at
runtime. The cost is code size, mitigated by tree-shaking (only used instantiations are emitted).

### `opaque type` at the Wasm Boundary

Opaque types export their `with` list as the WIT interface. Internal fields are hidden — the Wasm
component model enforces the boundary at the binary level, not just the language level:

```bounce
// package.bounce
export opaque type Connection with { open, query, close }
```

Generates a WIT interface where only `open`, `query`, and `close` are exported. No internal field
accessor is present in the WIT. An external Wasm consumer cannot bypass the `opaque` boundary even
through low-level Wasm tooling.

### `nominal type` at the Wasm Boundary

Nominal types are transparent at the Wasm level — they compile to their underlying type. The
nominal distinction exists only in the Bouncelang type system:

```bounce
nominal type Email = String
```

Compiles to `string` in WIT. A Wasm consumer receives a plain string. The `Email` vs `String`
distinction is enforced within Bouncelang code only.

### Value Semantics and Move Optimization

All values are immutable. The compiler uses move semantics to optimize — when a value has exactly
one owner and is passed to a function that consumes it, the compiler generates an in-place mutation
rather than a copy. This is invisible to the programmer but produces efficient Wasm output without
requiring explicit ownership annotations.

### Recursive Types

Recursive types (`Tree<T>`, `Json`) require heap allocation for the recursive field. The compiler
inserts `ref` (heap indirection) automatically at recursive positions. No `Box<T>` or explicit
heap annotation is visible in Bouncelang source code.

---

## 10. Serialization

### Auto-Derived Interfaces

The compiler auto-derives `Eq`, `Hash`, `Debug`, and `Serialize`/`Deserialize` from fields for
all `type` and `nominal type` declarations. No annotation needed:

```bounce
type Point = { x: Float, y: Float }
// Auto-derived: Eq, Hash, Debug, Serialize, Deserialize

nominal type UserId = Int
// Auto-derived: Eq, Hash, Debug, Serialize, Deserialize
// NOT interchangeable with plain Int in function signatures
```

`opaque type` auto-derives `Debug` for internal use only — the debug output includes all fields.
For external serialization, only fields listed in the `with` export list are available. Custom
serialization for opaque types is defined via a `view` declaration (see
[08-serialization-boundaries.md](08-serialization-boundaries.md)).

### Overriding Derivation

An explicit function definition overrides the auto-derived version:

```bounce
type User = { name: String, email: Email, age: Int }
// Auto-derived: Debug, Eq, Hash, Serialize

// Override display (not auto-derived, requires domain logic)
fn display(self: User) -> String { "{self.name} <{self.email}>" }

// Override serialize to exclude sensitive fields
fn serialize(self: User) -> Bytes {
    Json.encode({ name: self.name, email: self.email })
    // deliberately omitting age
}
```
