# Error Handling

**Status:** Draft — evolved from design review discussion
**Related:**
- [atoms-and-unions.md](atoms-and-unions.md) — union types, pattern matching
- [worlds-and-handlers.md](worlds-and-handlers.md) — worlds, effect handlers
- [methods-and-packages.md](methods-and-packages.md) — `pub fn` effect annotations

---

## 1. Core Principle: Errors Are Effects

Errors are not return values. They are effects — specifically, the `Raise<E>` effect. Functions that can fail perform `Raise`. Nothing returns `Result`. There is no `?` operator. There is no `catch` keyword.

```bounce
// This function raises an error — it does NOT return Result
fn parse_int(s: String) -> Int {
    // ... on failure:
    raise(ParseError { input: s, expected: "integer" })
}
// compiler infers: Raise<ParseError>
```

Effects propagate naturally. No annotation needed on internal functions:

```bounce
fn process(input: String) -> User {
    let id = parse_int(input)       // Raise<ParseError> propagates
    let user = fetch_user(id)       // Raise<NetworkError> propagates
    user
}
// compiler infers: Raise<ParseError>, Raise<NetworkError>
```

---

## 2. The `error` Keyword

Errors are declared with `error` — a nominal type that the compiler knows is intended for use with `Raise`:

```bounce
error NetworkError = { url: String, status: Int, message: String }
error TimeoutError = { url: String, timeout_ms: Int }
error ParseError = { input: String, position: Int, expected: String }
error ValidationError = { field: String, message: String }
```

### What `error` Gives You

- **Nominal identity** — two errors with the same fields are still distinct types
- **Auto-derived Display** — generates human-readable message from fields
- **Auto-derived `to_json`** — structured JSON with error name, fields, source location
- **Stack trace attachment** — compiler records source location at `raise()` call sites
- **LSP integration** — "Show all errors this function can raise" as a first-class query

### Error Grouping

Related errors can be grouped under a parent error:

```bounce
error ApiError =
    | NetworkError { url: String, status: Int, message: String }
    | TimeoutError { url: String, timeout_ms: Int }
    | RateLimitError { retry_after_ms: Int }
```

The group name works in `pub fn` signatures — adding a new member doesn't change the signature:

```bounce
pub fn fetch_user(id: Int) -> User
    with Raise<ApiError>
```

---

## 3. `try` Blocks — Handling Errors

`try` is the primary error-handling construct. It calls an expression and pattern matches on success or failure, with exhaustiveness checking:

```bounce
try fetch_user(42) {
    user => process(user)
    NetworkError { message, ... } => {
        Logger.warn("Network issue: {message}")
        retry()
    }
    TimeoutError { ... } => retry_with_backoff()
}
```

### Exhaustiveness

The compiler knows all `Raise` effects on the expression. Missing an error is a compile error:

```bounce
try fetch_user(42) {
    user => process(user)
    NetworkError { ... } => retry()
    // ❌ compile error: unhandled error: TimeoutError
    //    raised by: fetch_user → http_get (services/http.bounce:28)
}
```

### Catch-All With Re-Raise

Handle some errors, let others bubble up:

```bounce
fn get_user(id: Int) -> User {
    try fetch_user(id) {
        user => user
        NetworkError { ... } => retry(id)
        _ => raise    // re-raise TimeoutError — compiler infers it on get_user
    }
}
```

`_ => raise` means "I acknowledge these errors exist but I'm not handling them here." The compiler infers the unhandled errors as effects on the enclosing function.

### Success Arm Can Raise

The success arm is normal code — its effects propagate:

```bounce
fn process(id: Int) -> Report {
    try fetch_user(id) {
        user => generate_report(user)    // Raise<ReportError> propagates
        NetworkError { ... } => default_report()
    }
}
// compiler infers: Raise<ReportError> (from success arm)
```

### Matching Grouped Errors

Match specific errors within a group, or use the group name as a catch-some:

```bounce
try fetch_user(42) {
    user => use(user)
    NetworkError { status: 503, ... } => retry()     // specific variant + field match
    NetworkError { ... } => fail()                    // any NetworkError
    ApiError { ... } => handle_any_api_error()        // any member of the group
}
```

---

## 4. Rethrowing Different Errors

Handle a low-level error, raise a domain-level error:

```bounce
error UserNotFound = { id: Int }

fn get_user(id: Int) -> User {
    try fetch_from_db(id) {
        user => user
        DatabaseError { ... } => raise(UserNotFound { id: id })
    }
}
// compiler infers: Raise<UserNotFound> (not Raise<DatabaseError>)
```

### Adding Context

Use spread to add context to an existing error:

```bounce
fn fetch_user(id: Int) -> User {
    try http_get("/users/{id}") {
        response => parse(response)
        NetworkError { url, status, message } => raise(NetworkError {
            url,
            status,
            message: "fetching user {id}: {message}",
        })
    }
}
```

---

## 5. Errors in Pipelines

### Effects Propagate Naturally

```bounce
input
    |> parse_data        // Raise<ParseError>
    |> validate          // Raise<ValidationError>
    |> generate_report   // Raise<ReportError>
// all three Raise effects propagate to the caller
```

If any step raises, the pipeline aborts and the error bubbles up. No special syntax needed.

### Pipeline Recovery Functions

Standard library functions for inline recovery, built on `try`:

```bounce
// or_default — use a fallback value on any error
user_id
    |> fetch_profile
    |> or_default(empty_profile)

// or_else — use a fallback function on any error
user_id
    |> fetch_profile
    |> or_else { default_profile(user_id) }

// retry — retry on failure
data
    |> send_to_server
    |> retry(times: 3, backoff: :exponential)
```

These are not keywords — they are functions:

```bounce
fn or_default(body: -> T with Raise<E>, fallback: T) -> T {
    try body() {
        value => value
        _ => fallback
    }
}
```

### Per-Item Error Handling in Lists

For "try each item, keep going on failure":

```bounce
// Keep successes, drop failures
user_ids |> filter_map { id =>
    try fetch_user(id) {
        user => :some { value: user }
        _ => :none
    }
}

// Collect both successes and failures
user_ids |> map { id =>
    try fetch_user(id) {
        user => :ok { value: user }
        error => :err { error: error }
    }
}
```

No special `catch` keyword — just `try` inside a lambda.

---

## 6. Panic vs Raise

Two kinds of failure, with a clear boundary:

| | Panic | Raise |
|---|---|---|
| For | Bugs — programmer errors | Expected failures — bad input, network down |
| Examples | Index out of bounds, division by zero, assertion failure | Parse error, timeout, validation error |
| Recovery | Not recoverable in application code | Handled via `try` blocks |
| Mechanism | WASM trap | `Raise<E>` effect |
| In tests | Test world catches panics → test failure | Normal `try` handling |
| In DST | Panic is a reproducible finding (seed recorded) | Normal effect handling |

```bounce
// Panic — bugs
let x = list[100]             // out of bounds → panic
let y = 1 / 0                 // division by zero → panic
panic("invariant violated")   // explicit

// Raise — expected failures
fn parse(s: String) -> Int    // Raise<ParseError>
fn fetch(url: String) -> Data // Raise<NetworkError>
```

### Arithmetic Overflow

Overflow panics in release mode. Use explicit checked operations when needed:

```bounce
let x = Int.max + 1                      // panic
let wrapped = Int.wrapping_add(Int.max, 1)   // wraps to Int.min
let checked = Int.checked_add(Int.max, 1)    // Option<Int> — :none on overflow
```

DST catches overflow panics and records the seed — the panic IS the finding. You fix the code, not the test.

---

## 7. Top-Level Error Handling

World-level handlers catch errors that bubble past all `try` blocks:

```bounce
handler ErrorReporter(sentry_dsn: String): Raise {
    raise(error) => {
        Logger.error(error |> to_json)
        Sentry.capture(sentry_dsn, error)
        resume(default_error_response(error))
    }
}

world production {
    entry app
    handle Network  with WasiHttp
    handle Database with Postgres(config.database_url)
    handle Raise    with ErrorReporter(config.sentry_dsn)
}
```

### JSON Error Logging

Errors auto-serialize to structured JSON via `to_json`:

```json
{
  "error": "NetworkError",
  "fields": {
    "url": "/api/users/42",
    "status": 503,
    "message": "service unavailable"
  },
  "source": {
    "file": "services/api.bounce",
    "line": 42,
    "function": "fetch_user"
  },
  "trace": [
    "services/api.bounce:42 fetch_user",
    "controllers/users.bounce:18 get_user",
    "app.bounce:7 handle_request"
  ]
}
```

The `error` keyword gives the compiler enough information to generate this automatically. No manual serialization.

### Layering

```
Application code:  try { } — handles specific, expected errors
World handler:     handle Raise — catches unhandled errors, logs, reports
WASM runtime:      panics trap — test world catches for graceful failure
```

---

## 8. `try` vs `handle`

`try` is sugar for `handle Raise<E>`. They coexist:

| Construct | For | Syntax |
|---|---|---|
| `try` | Error handling specifically | `try expr { success => ..., Error { ... } => ... }` |
| `handle` | Any effect (Logger, Network, etc.) | `handle Effect { body } with { op => ... }` |

`try` only works with `Raise`. `handle` works with any effect. Use `try` for errors — it reads better.

---

## 9. Effect Annotations on `pub fn`

Internal functions infer all effects. Library exports require explicit annotations:

```bounce
// Internal — effects inferred, shown as LSP inlay hints
fn fetch_user(id: Int) -> User {
    // inferred: Raise<NetworkError>, Raise<TimeoutError>
}

// Library export — effects explicit
pub fn fetch_user(id: Int) -> User
    with Raise<ApiError> {
    // ...
}
```

Adding a new error to an error group doesn't change the `pub fn` signature. But adding a new `Raise` of a different group IS a breaking change — the compiler enforces this.

---

## 10. Testing Errors

```bounce
test fn parse_rejects_letters() {
    try parse_int("abc") {
        value => fail("expected error, got {value}")
        ParseError { expected, ... } => {
            assert_eq(expected, "integer")
        }
    }
}

// Or with a helper:
test fn parse_rejects_letters() {
    assert_raises<ParseError> { parse_int("abc") }
}
```

### Testing Panics

The test world installs a panic handler. Panics become test failures, not crashes:

```bounce
test fn division_by_zero_panics() {
    assert_panics { 1 / 0 }
}
```

---

## 11. LSP / DX (Developer Experience)

| Checklist Item | Behavior |
|---|---|
| **Completions** | Typing `raise(` suggests all `error` types in scope. Inside a `try` block, after the success arm, the LSP suggests all `Raise<E>` types inferred on the expression, as pattern stubs with field names. |
| **Inlay hints** | On every internal `fn`, inlay hints show the inferred `Raise<E>` effects: `fn process(id: Int) -> User  // inferred: Raise<ParseError>, Raise<NetworkError>`. On `pub fn`, missing explicit effect annotations are shown as hints with a "Make explicit" quick-fix. |
| **Diagnostics** | Missing arm in `try`: "Unhandled error: `TimeoutError`, raised by `fetch_user → http_get` (services/http.bounce:28)". Using `?` operator: "`?` is not available — use `try` blocks". Using `catch` keyword: "`catch` is not available — use `try` with error arms". |
| **Quick fixes** | "Add missing error arms" — inserts stubs for all unhandled errors, with field destructuring and a `// TODO` comment. "Make effects explicit" — adds `with Raise<...>` annotation to a `pub fn`. "Extract error group" — wraps multiple `Raise<X>` errors into a named `error Group = \| X \| Y`. |
| **Hover / go-to-definition** | Hovering `raise(NetworkError { ... })` shows the full error type including auto-derived JSON structure. Hovering a `try` expression shows the full set of error arms expected. Go-to-definition on an error name navigates to its `error` declaration. |
| **Semantic highlighting** | `error` declarations are highlighted as type declarations. `raise` calls are highlighted distinctly (e.g., as a control flow keyword) to signal "this exits the current function path". `try` arms are highlighted like match arms. |

---

## 12. Compiling & WebAssembly (Wasm)

`Raise<E>` is a Layer 1 primitive — it compiles to **conditional branches**, not Fiber suspension.
There is zero overhead from the async/WASI machinery.

### Compilation model

```bounce
fn get_user(id: Int) -> User {
    let row = db_lookup(id)
    if row == :false {
        raise NotFoundError { id }
    }
    parse_user(row)
}
```

Compiles to approximately:

```wat
;; Wasm (conceptual — actual output uses tagged return values)
(func $get_user (param $id i64) (result i32 i64)
  ;; i32 is the discriminant: 0 = success, 1 = NotFoundError
  ;; i64 carries either the User ptr or the error struct ptr
  (local $row i64)
  (local.set $row (call $db_lookup (local.get $id)))
  (if (i64.eq (local.get $row) (i64.const 0))  ;; :false check
    (then
      ;; return NotFoundError tag + error struct
      (return (i32.const 1) (call $alloc_not_found_error (local.get $id)))))
  ;; return success tag + user ptr
  (i32.const 0) (call $parse_user (local.get $row))
)
```

The tagged return pattern means each `Raise<E>` adds one `i32` discriminant to the return ABI. The
compiler collapses multiple errors into a single discriminant field — `Raise<A | B | C>` is one
`i32`, not three.

### `error` structs at the Wasm boundary

`error` types are emitted as standard `record` types in WIT. Auto-derived `to_json` compiles to
a serialization function over those fields — no reflection, no runtime type tags.

```wit
// error NetworkError = { url: String, status: Int, message: String }
record network-error { url: string, status: s64, message: string }
```

### Panic vs Raise at the Wasm level

| Mechanism | Wasm instruction | Host behavior |
|---|---|---|
| `Raise<E>` | Tagged `return` (branch) | Caller handles the tagged value |
| `Panic` | `unreachable` | Runtime traps; host catches the trap |



| Concept | Decision |
|---|---|
| Error declaration | `error` keyword — nominal, auto-Display, auto-JSON |
| Error grouping | Union syntax: `error ApiError = \| NetworkError \| TimeoutError` |
| Raising | `raise(error)` — performs `Raise<E>` effect |
| Handling | `try expr { success => ..., ErrorType { ... } => ... }` |
| Exhaustiveness | Compiler checks all possible errors in `try` |
| Re-raise | `_ => raise` — unhandled errors propagate, inferred |
| Rethrowing | `raise(DifferentError { ... })` in a try arm |
| Pipeline behavior | Effects propagate naturally, no special syntax |
| Pipeline recovery | `or_default`, `or_else`, `retry` — stdlib functions |
| Annotation tax | Internal: inferred. `pub fn`: explicit, use error groups |
| Panic | Bugs only — overflow, OOB, assertions. WASM trap. |
| Top-level handling | World-level `handle Raise` for logging/reporting |
| `Result` type | Not a language primitive. Construct in `try` arms if needed |
| `?` operator | Removed — not needed |
| `catch` keyword | Removed — `try` + stdlib functions cover all cases |
