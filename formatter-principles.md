# Formatter Principles

**Status:** Draft — canonical reference for all formatting decisions
**Related:** All spec documents reference formatter behavior — this document defines the principles they follow.

> **The Bouncelang formatter is not optional.** It runs on save, on commit, and on publish. There is no configuration. One format, one style, zero debates. Like `gofmt`, but stricter.

---

## 1. No Configuration

There is no `.bouncefmt.toml`. No `--style` flag. No options.

The formatter produces one canonical output for any given input. Two developers formatting the same code always get identical results. This eliminates all style discussions and ensures every file in every project follows the same conventions.

---

## 2. No Hard Line Length

The formatter does NOT enforce a character-count line limit. Instead, it uses **structural expansion rules** — formatting decisions are based on the number and kind of items, never on character count.

This means adding one character to a variable name **never** causes cascading reformats. Either the whole construct fits its structural rule, or it expands.

There is a soft guideline (~100 chars) the formatter uses as a hint for edge cases, but it never forces a mid-expression line break.

---

## 3. Vertical Space Is Cheap

Prefer vertical expansion over horizontal compression. Screens are wide but eyes scan vertically. Short, stacked lines are faster to read than long, dense lines.

```bounce
// ✅ Formatter output — vertical
fn create_user(
    name: String,
    email: Email,
    role: :admin | :user | :guest,
) -> User with Database, Raise<ValidationError> {
    // ...
}

// ❌ Not this — horizontal crunch
fn create_user(name: String, email: Email, role: :admin | :user | :guest) -> User with Database, Raise<ValidationError> {
```

---

## 4. Leading Operators

Binary operators at the **start** of continuation lines, not the end. The operator is the first thing your eye hits, telling you immediately what's happening:

```bounce
// ✅ Leading operators — immediately see the logic structure
let eligible =
    user.age >= 18
    && user.verified
    && !user.banned
    || user.role == :admin

// ✅ Leading pipes — see each variant
type HttpResponse =
    | :ok { status: Int, body: String }
    | :redirect { url: String }
    | :not_found

// ✅ Leading pipeline — see each transformation step
users
    |> filter { u => u.active }
    |> sort_by { u => u.name }
    |> map(display)

// ❌ Not trailing — operator buried at end of line
let eligible =
    user.age >= 18 &&
    user.verified &&
    !user.banned ||
    user.role == :admin
```

---

## 5. Consistency Over Compactness

When a construct CAN fit on one line but is part of a pattern where similar constructs don't, expand it to match.

```bounce
// ✅ All multi-line — consistent even though :point could fit inline
type Shape =
    | :circle { radius: Float }
    | :rect { width: Float, height: Float }
    | :point
```

**The rule:** if ANY item in a group forces expansion, ALL items expand. Applied to:
- Union variants (any has data → all multi-line)
- Match arms (any arm body expands → all expand)
- Function params, handler args, record fields

---

## 6. Minimize Git Diffs

### Trailing Commas — Always

Adding an item to a list changes one line, not two:

```bounce
type User = {
    name: String,
    email: Email,
    role: Role,        // trailing comma always
}
```

### Leading Operators — Also a Diff Win

Adding a union variant is a one-line diff:

```bounce
type Status =
    | :pending
    | :active
    | :complete
    // + | :cancelled  ← one line added, no other lines touched
```

### No Column Alignment on Changing Structures

Record fields, function params, and deps change frequently. Column alignment causes cascading diffs:

```bounce
// ❌ Column-aligned — adding `database_url` reformats every line
type Config = {
    name:         String,
    email:        Email,
    database_url: String,   // new longest name → all lines reformatted
}

// ✅ No alignment — adding a field is a one-line diff
type Config = {
    name: String,
    email: Email,
    database_url: String,   // only this line added
}
```

### Column Alignment Only for Stable, Read-Heavy Structures

Match arms and handler wiring are set up once and read many times. Alignment aids scanning:

```bounce
// ✅ Match arms — align arrows (stable, read-heavy)
match status {
    :pending    => process(order)
    :processing => wait()
    :complete   => archive(order)
    :cancelled  => refund(order)
}

// ✅ Handler wiring — align `with` (stable, read-heavy)
handle Network  with WasiHttp
handle Database with Postgres(config.database_url)
handle Logger   with StdoutLogger(level: :info)
```

---

## 7. Preserve Intentional Blank Lines

Blank lines are logical grouping signals. The formatter preserves them (up to one consecutive blank line):

```bounce
fn process_order(order: Order) -> Receipt {
    let validated = validate(order)
    let priced = calculate_totals(validated)

    let receipt = charge(priced)
    let confirmation = send_email(receipt)

    confirmation
}
```

Multiple consecutive blank lines are collapsed to one. No blank lines are added or removed.

---

## 8. Import Ordering

Imports are auto-sorted and grouped by origin, separated by blank lines:

```bounce
import { List, Map } from std/collections
import { format } from std/time

import { Router, Request, Response } from http
import { query } from pg

import { User, Transaction } from models
import { create_user } from services
```

Groups: std → deps → internal. Within each group, sorted alphabetically by source.

---

## 9. Deterministic

The formatter is a pure function: `format(format(code)) == format(code)`. Formatting already-formatted code changes nothing. The input's original formatting is irrelevant — the output depends only on the AST.

---

## 10. One Line or Fully Expanded

Constructs either fit on one line or expand with each item on its own line. **No partial wrapping.** No line breaking mid-expression.

```bounce
// ✅ Fits — one line
handle Logger with StdoutLogger(level: :info)

// ✅ Doesn't fit — every arg on its own line
handle Database with Postgres(
    host: config.db_host,
    port: config.db_port,
    database: config.db_name,
    user: config.db_user,
    password: config.db_password,
    pool_size: 10,
    timeout_ms: 5000,
)

// ❌ Never this — partial wrapping
handle Database with Postgres(host: config.db_host, port: config.db_port,
    database: config.db_name, user: config.db_user)
```

---

## 11. Structural Expansion Rules

Expansion is based on structure, not character count:

| Construct | Compact Form | Expanded Form | Expansion Trigger |
|---|---|---|---|
| Function params | `(a: Int, b: Int)` | Each param on its own line | > 2 params |
| Record fields | `{ name: String }` | Each field on its own line | > 1 field |
| Constructor args | `Postgres(host: x)` | Each arg on its own line | > 2 args |
| Pipelines | `a \|> b` | Each step on its own line | 2+ `\|>` operators |
| Union variants | `:a \| :b` | Leading pipe, each on own line | Any variant has data |
| Boolean chains | `a && b` | Leading operator, each on own line | 2+ `&&`/`\|\|` operators |
| Match arms | Short arms | Always multi-line | Always |

### Pipelines

```bounce
// 1 pipe operator — single line
users |> count

// 2+ pipe operators — always multi-line
users
    |> filter(active)
    |> count

users
    |> filter { u => u.active }
    |> sort_by { u => u.name }
    |> take(10)
    |> map(display)
```

### Match Arms — Expansion Cascades

If any arm body is too long for one line, all arms use the expanded form:

```bounce
// ✅ Short arms — compact, arrows aligned
match status {
    :pending  => process(order)
    :complete => archive(order)
}

// ✅ Any arm is long — all arms expand, body on next line
match event {
    :click { target, position } =>
        handle_click(target, position, viewport)
    :drag { start, end, modifier_keys } =>
        calculate_drag_vector(start, end)
            |> apply_modifiers(modifier_keys)
            |> update_canvas
}

// ✅ Body is a block — use braces
match result {
    :success { user } => {
        Logger.info("User created: {user.name}")
        send_welcome_email(user)
        user
    }
    :failure { reason } => {
        Logger.warn("Creation failed: {reason}")
        raise(reason)
    }
}
```

### Long Lambda in a Pipeline Step

If a single pipeline step's lambda is too long, expand just the lambda:

```bounce
users
    |> filter { u =>
        u.age > 18
        && u.verified
        && u.subscription != :expired
    }
    |> sort_by { u => u.created_at }
    |> take(10)
```

---

## 12. Function Signatures

### Short — Single Line

```bounce
fn add(a: Int, b: Int) -> Int { a + b }
```

### Medium — Params on Separate Lines

```bounce
fn create_user(
    name: String,
    email: Email,
    role: :admin | :user | :guest,
) -> User {
    // ...
}
```

### Long — Params + Effects on Separate Lines

```bounce
fn create_user(
    name: String,
    email: Email,
    role: :admin | :user | :guest,
) -> User
    with Database, Raise<ValidationError> {
    // ...
}
```

**Rule:** > 2 parameters → multi-line params. Effects always on their own line when params are multi-line.

---

## Summary of Principles

| # | Principle | Rationale |
|---|---|---|
| 1 | No configuration | Zero debates, universal consistency |
| 2 | No hard line length | Structural rules, not character counting |
| 3 | Vertical space is cheap | Eyes scan vertically faster |
| 4 | Leading operators | Logic structure visible at line start |
| 5 | Consistency over compactness | If any item expands, all expand |
| 6 | Minimize git diffs | Trailing commas, leading operators, no alignment on changing structures |
| 7 | Preserve blank lines | Developer's logical grouping is meaningful |
| 8 | Import ordering | Auto-sorted, grouped by origin |
| 9 | Deterministic | Idempotent, pure function of AST |
| 10 | One line or fully expanded | No partial wrapping, ever |
| 11 | Structural expansion rules | Triggers based on item count, not line length |
