# Methods, Packages, and Companion Functions

**Status:** Draft — evolved from design review discussion
**Supersedes:** `docs/plans/2026-03-11-impl-resolution.md` (impl blocks dropped in favor of UFCS)
**Related:**
- [worlds-and-handlers.md](file:///Users/tom/projects/bouncelang/docs/spec/worlds-and-handlers.md) — worlds, handlers, config
- [modules-and-imports.md](file:///Users/tom/projects/bouncelang/docs/spec/modules-and-imports.md) — sub-modules, imports, stdlib, WASM linking

---

## 1. Core Decision: No `impl`, Everything Is Functions (UFCS)

Bouncelang has no `impl` blocks. Methods are regular functions where the first parameter is `self`. The dot-call syntax `value.method()` desugars to `method(value)`.

```bounce
type User = { name: String, email: Email }

fn display(self: User) -> String { "{self.name} <{self.email}>" }
fn activate(self: User) -> User { { ...self, active: true } }
```

Both call styles are equivalent:
```bounce
user.display()       // method syntax — desugars to display(user)
display(user)        // function syntax
user |> display      // pipeline syntax
```

### 1.1 The `self` Keyword

`self` is a required keyword for UFCS eligibility. Only functions with a parameter literally named `self` can be called with dot syntax:

```bounce
fn display(self: User) -> String { ... }   // ✅ user.display() works
fn display(user: User) -> String { ... }   // ❌ user.display() does NOT work
                                           //    must call as display(user)
```

This makes intent explicit. The function author opts into method-call syntax by naming the parameter `self`.

### 1.2 Why Not `impl`

| Concern | `impl` blocks | UFCS |
|---|---|---|
| Salsa cache invalidation | Adding an `impl` anywhere invalidates method resolution for the whole package | Method resolution is a file-local name lookup |
| Orphan rules | Needed to prevent conflicting impls | Not needed — name collisions resolved by imports |
| Concepts to learn | `impl`, orphan rules, explicit import activation | Functions, imports |
| Operator overloading | Via `impl Add for T` | Via `Add<T>` interface on `nominal` types |

---

## 2. `package.bounce` — Public API Without File Paths

`package.bounce` declares the package's public API and code-analysis rules. It does **not** reference internal file paths — the compiler scans the package to find declarations. See [worlds-and-handlers.md](file:///Users/tom/projects/bouncelang/docs/spec/worlds-and-handlers.md) for the build-target side (`world`).

```bounce
package {
    name: accounts
    version: "1.0.0"
    license: MIT

    deps {
        std: "1.2"
        http-client: "^2.0"
        json: "^1.5"
    }

    dev_deps {
        test-utils: "^2.0"
    }

    lints {
        std/deprecated: :warn
        std/unused: { default: :error, test: :off }
    }
}

// Structural types — transparent data
export type UserRequest
export type TransferError

// Nominal types — named wrappers (transparent, type-safe)
export nominal type Email with { validate, domain }

// Opaque types — hidden internals, factory + API only
export opaque type User with { display, activate, validate, create, find }
export opaque type Transaction with { display, total, line_items }

// Standalone functions (not companions — must be imported explicitly)
export fn calculate_interest
export fn validate_email

// Test-only exports (visible only in .test.bounce files)
test fn mock_user
test fn assert_valid_transaction
```

### 2.1 Visibility Tiers

| Tier | Syntax | Behavior |
|---|---|---|
| **structural type** | `export type T` | Transparent data, always constructible |
| **nominal type** | `export nominal type T with { f }` | Named wrapper, transparent, type-safe |
| **opaque type** | `export opaque type T with { f, g }` | Hidden internals, factory-only, `with` controls API |
| **companion** | `with { f, g }` on a type export | `f` and `g` auto-import when `T` is imported |
| **explicit** | `export fn f` | Must be imported by name |
| **test** | `test fn f` | Only visible in `.test.bounce` files |

Within a file, the existing visibility rules apply:
- **Unmarked:** file-private
- **`internal`:** package-visible
- **`pub`:** exported (but only if listed in `package.bounce`)

### 2.2 Companion Rules

- Companion functions must take `self: T` where `T` is the named type
- Functions with anonymous structural `self` types (e.g., `self: { name: String }`) cannot be companions
- If two files in the same package define the same companion function, the compiler errors: *"ambiguous export — `display` for `User` defined in both `models.bounce` and `views.bounce`"*
- The convention (not enforced) is one type per file

### 2.3 No File Paths, No Re-Exports, No Wildcards

- **No file paths:** `package.bounce` says *what* is public, not *where* it lives. Reorganize files freely without touching `package.bounce`.
- **No re-exports:** Consumers import types from the package that defines them. If `accounts` uses `User` from `auth`, consumers import `User` from `auth` directly.
- **No wildcard imports:** All imports are explicit. The LSP auto-import handles ergonomics.

---

## 3. Import Syntax

```bounce
import { User, Transaction } from accounts
// User's companions (display, activate, validate) are now in scope.
// Transaction's companions (display, total, line_items) are now in scope.

import { calculate_interest } from accounts
// Standalone function — must be imported explicitly.
```

### 3.1 Conflict Resolution

If two imports bring the same function name into scope for the same `self` type, the compiler errors at the call site:

```bounce
import { User } from accounts          // companions: display
import { display } from pretty-print   // also display(self: User)

user.display()
// ERROR: ambiguous — `display` for `User` imported from both
//        `accounts` and `pretty-print`.
// Fix: use explicit call syntax or alias the import.
```

Resolution via aliasing:
```bounce
import { User } from accounts
import { display as pretty_display } from pretty-print

user.display()         // accounts' version (companion)
pretty_display(user)   // pretty-print's version (explicit)
```

Companion functions always win over non-companion imports in autocomplete ranking, but the compiler still errors on true ambiguity.

---

## 4. Operator Overloading

Operators desugar to interface method calls. Operator overloading requires `nominal` types.

```bounce
interface Add<Rhs = Self> {
    fn add(self, rhs: Rhs) -> Self
}

nominal type Amount = { value: Int, currency: String }

fn add(self: Amount, rhs: Amount) -> Amount {
    Amount { value: self.value + rhs.value, currency: self.currency }
}
fn add<Int>(self: Amount, rhs: Int) -> Amount {
    Amount { value: self.value + rhs, currency: self.currency }
}

let total = Amount(100, "USD") + Amount(50, "USD")   // ✅ Amount is nominal
let scaled = Amount(100, "USD") + 50                  // ✅ Add<Int> for Amount
```

### 4.1 Why Nominal Only

Structural types match by shape. Width subtyping means `{ x: Int, y: Int, z: Int }` matches `{ x: Int, y: Int }`. This creates ambiguous operator dispatch:

```bounce
type Point2D = { x: Float, y: Float }
fn add(self: Point2D, rhs: Point2D) -> Point2D { ... }

let p3d = { x: 1, y: 2, z: 3 }
p3d + p3d   // matches Point2D's add — silently drops z!
```

Requiring `nominal` for operators prevents this. Structural types use pipeline functions:
```bounce
type Point3D = { x: Float, y: Float, z: Float }

fn add_points(self: Point3D, rhs: Point3D) -> Point3D { ... }
p3d |> add_points(other_p3d)
```

**The mental model:** `nominal` types are "primitive-like" values where identity matters (money, dates, IDs, coordinates). Operators express domain semantics on these. Structural types are data bags — they compose via functions and pipelines, not operators.

---

## 5. Interface Satisfaction

Interfaces are satisfied structurally — by having the matching function in scope.

```bounce
interface Display { fn display(self) -> String }

fn print_item(item: Display) { log(item.display()) }
```

`User` satisfies `Display` wherever `display(self: User)` is in scope. Since companions auto-import with the type, a companion `display` means **`User` always satisfies `Display`** wherever `User` is usable.

If `display` is not a companion (it's in `explicit`), then `User` only satisfies `Display` in files that explicitly import `display`.

---

## 6. Library vs Application

No explicit flag needed. The distinction is structural:

| | Library | Application |
|---|---|---|
| Has `package.bounce` | Yes | Yes |
| Has `world` | No | Yes |
| Can `bounce check` | Yes | Yes |
| Can `bounce test` | Yes (uses default test world) | Yes |
| Can `bounce build` | Produces package artifact | Produces WASM component |
| Can `bounce run` | No | Yes |
| Can `bounce publish` | Yes | Typically no |

Libraries define types, functions, and handlers that applications wire together via worlds. See [worlds-and-handlers.md](file:///Users/tom/projects/bouncelang/docs/spec/worlds-and-handlers.md) for full world documentation.

### Effect Annotations

| Context | Effects | Why |
|---|---|---|
| World entry points | **Inferred** — verified against world handlers | The world IS the contract |
| Library exports (`pub fn`) | **Explicit** — required by compiler | API stability. Adding an effect is a breaking change |
| Internal functions | **Inferred** — shown as LSP inlay hints | No annotation tax on internal code |

---

## 7. LSP Features for UFCS

### Semantic Highlighting
- `self` parameters highlighted distinctly (e.g., italic or unique color) to signal UFCS eligibility

### Autocomplete on Dot
When typing `user.`:
1. **Companion functions** shown first with a "companion" badge — always available
2. **Imported functions** with matching `self: User` shown next
3. **Unimported functions** from dependencies shown last with "auto-import" action

### Quick Actions
- **"Show companions"** — cursor on a type name, see all companion functions from `package.bounce`
- **"Make companion"** — cursor on a function with `self: T`, add it to `package.bounce`'s companion list

### Inlay Hints
- **Inferred effects** on function calls and definitions
- **Pipeline step types** — hover any `|>` to see the type flowing through
- **Self type** — on dot-calls, show the resolved `self` type when it's structural

### Auto-Import
- Selecting an unimported function from autocomplete inserts the import statement automatically
- Companion functions never need manual import — they come with the type
