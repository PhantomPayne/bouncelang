# Generics and Type System

**Status:** Draft — evolved from design review discussion
**Related:**
- [atoms-and-unions.md](file:///Users/tom/projects/bouncelang/docs/spec/atoms-and-unions.md) — atoms, unions, pattern matching
- [methods-and-packages.md](file:///Users/tom/projects/bouncelang/docs/spec/methods-and-packages.md) — UFCS, companions, exports
- [error-handling.md](file:///Users/tom/projects/bouncelang/docs/spec/error-handling.md) — errors, Raise effect

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

### `let mut` — Mutable Bindings

`mut` on a binding means the variable name can be reassigned. The values themselves never change:

```bounce
let mut list = LinkedList.empty()
list = list |> push(1)    // rebind 'list' to a new value
list = list |> push(2)    // rebind again
```

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

## 6. Open Design Questions

### `:true/:false` Unification

Under investigation: using `:true | :false` as a universal positive/negative pattern, replacing `Option<T>` with `:true { value: T } | :false`, and `T?` as sugar. Promising but needs more validation with generics and pipelines.

### `if let` Syntax

Likely: `if let value = expr { ... }` for pattern matching in conditionals. Details depend on the `:true/:false` question.

### Higher-Kinded Types

Deferred to V2. V1 uses concrete generic types only.
