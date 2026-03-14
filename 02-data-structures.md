# Spec v2 layer 2: Data Structures

This document defines how primitives are composed into complex structures in Bouncelang. It covers records, unions, collections, and the ergonomic tools used to manipulate them while adhering to strict value semantics.

---

## 1. Records & Structural Typing

A Record is a named collection of fields. Bouncelang uses **Structural Typing** by default. This means that two records with the exact same field names and types are considered identical by the compiler, regardless of what they are named.

### Syntax
```bounce
type Point = {
    x: Int
    y: Int
}

let p1: Point = { x: 10, y: 20 }
```

### Type Constraints (Structural Subtyping)
Functions can accept any record that *contains* the required fields, even if the record has extra fields.
```bounce
// This function accepts ANY record that has at least `x` and `y`
fn distance(p: { x: Int, y: Int }) -> Float { ... }

type Player3D = { x: Int, y: Int, z: Int, health: Int }
let hero: Player3D = { x: 1, y: 1, z: 0, health: 100 }

// This is perfectly valid because Player3D structurally satisfies { x, y }
let d = distance(hero) 
```

### Default Values
To reduce construction boilerplate, fields can define default values.
```bounce
type SearchConfig = {
    recursive: Bool = false
    max_depth: Int = 10
    target: String
}

// Only `target` is required at construction
let cfg: SearchConfig = { target: "/" }
```

---

## 2. Opaque Types (Encapsulation)

**Opaque Types** are used when you need a hard identity boundary and want to hide internal implementation details. Unlike structural records, Opaque types are defined by their **name** and **package origin**.

### Syntax & Privacy
Opaque types are **private-by-default**. This enforces the "Black Box" pattern, where a package exposes an API but hides the data layout.

```bounce
export type User = opaque {
    id: String
    internal session_token: String
    pub name: String
}
```

*   **Boundary:** Fields are only visible within the defining package. External code cannot read `internal` fields.
*   **Construction:** External code cannot construct an Opaque type structurally if any fields are private.
*   **Zero-Cost Wrappers:** Opaque types can also wrap primitives (e.g., `export type Key = opaque String`).

---

## 3. Intersection Tags (Refinement & Truthiness)

**Tags** are zero-cost, compile-time metadata attached to any type. They create an **Intersection Type** (`Base & :Tag`). 

### Tags as "Atoms of Evidence"
A tag is more than just a label; it is a piece of **evidence** that follows a value and enables specific logic.
*   **Identity:** A `:point` tag on a record identifies its domain for operator dispatch (e.g., `+`).
*   **Refinement:** A `:validated` tag on a string promises it has passed through a validator.
*   **Truthiness:** Bouncelang uses the `:true` and `:false` tags to drive all control flow.

### The "Direct" Option & Result Model
By leveraging Tags, Bouncelang eliminates the need for "unwrapping" optional values or results.

1.  **Booleans:** `Bool` is just `:true | :false`.
2.  **Options:** `Option<T>` is `(T & :true) | :false`.
3.  **Results:** `Result<T, E>` is `(T & :true) | (E & :false)`.

Because `:true` is a tag, a "True Option" **is** the underlying value. There is no `.value` wrapper.

### Flow-Sensitive Narrowing (Pinning)
When a value is checked via an `if` or `match`, the compiler "narrows" its type within that branch.

```bounce
let name: Option<String> = ... // (String & :true) | :false

if name {
    // Inside here, `name` is narrowed to `String & :true`
    // The `:true` tag acts as a "Proof of Presence."
}
```

**Does the tag stay?**
Yes. Keeping the `:true` tag is beneficial—it allows the compiler to guarantee that a value is non-optional in subsequent logic without needing a separate "NonNullable" type system. The tag is zero-cost at runtime, so it doesn't "clutter" the execution memory.

**Evidence Decay:**
If you modify a narrowed value (e.g., `name = name + "!"`), the tag is **dropped**. This forces you to re-validate or handle potential optionality again if the mutation could have invalidated the previous check.

---

## 4. Atoms & Unions (The Foundation of Choices)

An **Atom** (e.g., `:loading`, `:success`) is a unique, zero-cost symbol. In Bouncelang, Atoms are the building blocks of both **Choices (Unions)** and **Evidence (Intersections)**.

### Unions
A **Union** is a type definition that allows a value to be one of several Atom structures.
```bounce
type LoadingState<T> = 
    | :loading
    | :success { data: T }
    | :error { code: Int }
```

### Pattern Matching
The compiler enforces exhaustive checking on Union types.
```bounce
match state {
    :loading => render_spinner()
    :success { data } => render_data(data)
    :error { code } => render_error(code)
}
```

---

## 5. Collections: Lists & Maps

Collections are built-in, generically typed data structures that utilize Structural Sharing for O(log N) mutation performance.

*   **`List<T>`**: An ordered sequence of elements. `[1, 2, 3]`.
*   **`Map<K, V>`**: A hash map where `K` must fulfill the `Hash` trait. `{ "key": 42 }`.

---

## 6. Tuples & Recursive Types

### Tuples
Tuples are anonymous, ordered sequences of types. They are sugar for anonymous records with integer field names (`{ 0: T, 1: U }`).
```bounce
let pair: (Int, String) = (1, "Alice")
```

### Recursive Types
Unions natively support recursion for trees and linked structures.
```bounce
type IntTree = :node { value: Int, left: IntTree, right: IntTree } | :leaf
```

---

## 7. Companion Functions (`with`)

When defining a type (Opaque or Tagged), you can associate specific functions as **Companions**. These functions are automatically imported when the type is used, enabling the dot-call method syntax via UFCS.

```bounce
export type Email = String & :email with { validate, domain }
```

---

## 8. Ergonomics: Deep Spread (`...`)

Updating deeply nested pure data uses the `->` path arrow:
```bounce
let new_state = { ...state, player->stats->hp: 0 }
```
This syntax also extends to Atoms. If a state is currently `:error`, a spread like `{ ...state, :success->data: x }` safely no-ops and returns the original `:error`.

---

## 9. Compiling & WebAssembly (Wasm)

*   **Records/Tuples** map to Wasm `record` types.
*   **Unions/Booleans** map to Wasm `variant` types.
*   **Tags & Opaque Wrappers** are erased at runtime; they compile to their base representation.

---

## 10. LSP / DX (Developer Experience)

*   **Evidence Tracking:** The LSP visually distinguishes "Evidence" (tagged values) from raw values.
*   **Privacy Boundaries:** The LSP refuses to autocomplete `internal` fields of Opaque types outside their package.
*   **Deep Spread Autocomplete:** Autocompletions intelligently query nested field types through the `->` arrow.
*   **Annotation Discovery:** Hovering over fields shows `@help` text attached via the `@{}` syntax.
