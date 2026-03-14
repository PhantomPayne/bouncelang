# Spec v2 layer 4: Effects & Handlers

This document defines Bouncelang's effect system — the three-layer model, effect declarations, implicit propagation, the `pure` function annotation, handler wiring, and the automatic test sandboxing model.

---

## 1. The Three-Layer Model

Bouncelang has three distinct layers for managing control flow and external interaction. Understanding which layer a concept belongs to is critical for writing correct, testable code.

### Layer 1: Primitives (Built-in, compiler special-cases)

| Primitive | Purpose | Compilation |
|---|---|---|
| `Raise<E>` | Structured error control flow | Branches (no suspension) |
| `Panic` | Unrecoverable failure | WASM trap |
| `Alloc` | Memory allocation | Runtime concern (OOM = trap) |

Primitives are not effects. They do not suspend, do not cross WASI boundaries, and cannot be denied by worlds. They are always available in all code, including `pure fn`.

### Layer 2: Effects (WASI boundaries, Fiber suspension)

| Effect | WASI Interface | Purpose |
|---|---|---|
| `Network` | `wasi:http` | HTTP requests, WebSockets |
| `FileSystem` | `wasi:filesystem` | File read/write, directory operations |
| `IO` | `wasi:cli` | Stdin/stdout/stderr |
| `Time` | `wasi:clocks` | Current time, timers, sleep |
| `Random` | `wasi:random` | Random number generation |
| `Concurrency` | WASI 0.3 async | Task spawning, channels, scheduling |

Effects are the boundary between Bouncelang code and the host runtime. They require Fiber suspension because the host controls when I/O completes. **You cannot define new effects in pure Bouncelang** — effects bridge to host capabilities via WASI interfaces.

### Layer 3: Functions (Everything internal)

All internal application logic is plain functions. No `service` keyword, no implicit propagation for internal DI. If you need dependency injection, pass functions or records of functions as parameters. Import is the default wiring mechanism.

### Why This Split Matters

| Question | Answer | Layer |
|---|---|---|
| Does it cross a WASI boundary? | Yes → Effect | 2 |
| Is it a control flow primitive? | Yes → Primitive | 1 |
| Is it internal application logic? | Yes → Function | 3 |

The compiler uses this split for optimization:
- **Primitives** compile to inline instructions (branches, traps). Zero overhead.
- **Effects** compile to Fiber suspension/resumption. Necessary overhead for host interaction.
- **Functions** compile to direct or indirect calls. No suspension, no special handling.

---

## 2. Primitives

### Raise<E>

`Raise<E>` is Bouncelang's structured error mechanism. It is NOT an effect — it compiles to branches, not Fiber suspension. See `error-handling.md` for full syntax.

Key properties:
- Always available, even in `pure fn`.
- Propagates implicitly through function calls and pipelines.
- Caught by `try`/`catch` blocks.
- At the WASM level, it compiles to conditional branches and tagged return values.

```bounce
error NotFoundError { id: Int }

fn get_user(id: Int) -> User {
    let row = db_lookup(id)
    if row == :false {
        raise NotFoundError { id }
    }
    parse_user(row)
}

// Raise propagates through pipelines naturally
ids |> map { id => get_user(id) } |> collect
// If any get_user raises, the whole pipeline raises
```

### Panic

`Panic` represents an unrecoverable failure. It compiles to a WASM `unreachable` instruction (trap). The host runtime catches the trap and terminates the component.

```bounce
fn unreachable_code() -> Never {
    panic("This should never happen")
}
```

- Cannot be caught by `try`/`catch`.
- Triggers `defer` and `using` cleanup before the component terminates.
- In DST mode, panics are captured and reported as test failures with full stack traces.

### Alloc

Memory allocation is a runtime concern, not an effect. Heap allocation happens transparently when creating strings, lists, maps, or any dynamically-sized value.

- If the WASM linear memory is exhausted, the runtime traps (OOM panic).
- Guest code does not "handle" allocation failures.
- In DST mode, the test runner can inject artificial OOM traps to test host resilience.

---

## 3. Effects

### What is an Effect?

An effect is a declaration of operations that cross the WASI boundary. It defines the Bouncelang-side API for interacting with a host capability. The effect declaration does not contain implementation — the implementation lives in the host runtime.

### Declaring Effects

Standard effects are declared in the standard library. Third-party effects are declared in packages that also ship the WASM component implementing the WASI interface.

```bounce
// Standard library declaration (std/network)
effect Network {
    fn get(url: String) -> Response
    fn post(url: String, body: Bytes) -> Response
    fn websocket(url: String) -> WebSocket
}

// Standard library declaration (std/time)
effect Time {
    fn now() -> Instant
    fn local_timezone() -> Timezone
    fn sleep(duration: Duration)
    fn after(duration: Duration) -> Sequence<Instant> & :finite
    fn every(interval: Duration) -> Sequence<Instant> & :infinite
}

// Standard library declaration (std/filesystem)
effect FileSystem {
    fn open(path: String) -> File
    fn read(path: String) -> String
    fn write(path: String, content: String)
    fn list(path: String) -> List<String>
    fn exists(path: String) -> Bool
}
```

### Using Effects

Effect operations are called using dot syntax on the effect name:

```bounce
let response = Network.get("https://api.example.com/users")
let now = Time.now()
let content = FileSystem.read("/etc/config.json")
```

When an effect operation is called:
1. The Bouncelang runtime suspends the current Fiber.
2. Control passes to the WASI host.
3. The host performs the I/O operation.
4. The host resumes the Fiber with the result.
5. Execution continues after the effect call.

The developer writes sequential code. The suspension is invisible.

### Third-Party Effects

Packages can declare new effects by shipping both the effect declaration AND a WASM component implementing the corresponding WIT interface.

```bounce
// In a database package (e.g., "bouncelang/postgres")
effect Database {
    fn query(sql: String, params: List<Value>) -> Rows
    fn execute(sql: String, params: List<Value>) -> Int
    fn transaction(f: pure fn() -> T) -> T
}
// The package also ships a WASM component compiled from Rust
// that implements the WIT interface for these operations.
```

Consumers use it like any standard effect:

```bounce
import Database from "bouncelang/postgres"

fn get_user(id: Int) -> User {
    let rows = Database.query("SELECT * FROM users WHERE id = $1", [id])
    parse_user(rows.first!)
}
```

The world block wires the effect to its handler (see §6).

---

## 4. Effect Propagation

### Implicit Propagation

Effects propagate implicitly through the call graph. If function `a` calls `Network.get()`, then `a` requires `Network`. If function `b` calls `a`, then `b` also requires `Network`. No annotation needed at any level.

```bounce
// These functions all implicitly require Network.
// No "requires Network" annotation anywhere.

fn fetch_user(id: Int) -> User {
    let response = Network.get("https://api.example.com/users/{id}")
    parse_user(response.body)
}

fn fetch_all_users() -> List<User> {
    let response = Network.get("https://api.example.com/users")
    parse_users(response.body)
}

fn build_report() -> Report {
    let users = fetch_all_users()    // implicitly requires Network
    let posts = fetch_all_posts()    // implicitly requires Network
    compile_report(users, posts)
}
```

### Why No Annotations?

Effect annotations (like Koka's `fun fetch() : <network, raise<e>> response`) add noise to every function signature. Since Bouncelang effects are always WASI boundaries (a small, well-known set), the compiler can infer them reliably. The LSP shows inferred effects as inlay hints, giving you the documentation without the syntax burden.

### Pub Boundary Tracking

At package boundaries (`pub fn`), the compiler tracks which effects a function requires as part of its public API signature. This tracking is automatic — the developer does not annotate.

When publishing a package (`bounce publish`), the semver checker compares effect requirements:
- **Adding** an effect to a `pub fn` = **breaking change** (major version bump required).
- **Removing** an effect from a `pub fn` = **minor change** (compatible).

This prevents a common class of breaking changes: a library update silently starts requiring Network access, breaking consumers in sandboxed environments.

### LSP Inlay Hints

The LSP shows effect requirements as unobtrusive inlay hints:

```bounce
fn build_report() -> Report {            // hint: [Network, FileSystem]
    let users = fetch_all_users()        // hint: [Network]
    let template = load_template()       // hint: [FileSystem]
    compile_report(users, template)
}
```

Hovering over any function call shows the full effect chain.

---

## 5. Pure Functions

A `pure fn` guarantees that a function performs no effects. The compiler enforces this transitively — a pure function can only call other pure functions and effect-free operations.

### What Pure Allows and Disallows

| Allowed in `pure fn` | Disallowed in `pure fn` |
|---|---|
| All computation | Effect calls (Network, FileSystem, etc.) |
| `Raise<E>` (error control flow) | `Time.now()`, `Random.int()` |
| `Panic` (unrecoverable) | `IO.print()`, `FileSystem.read()` |
| Memory allocation | `Concurrency.scope { }` |
| Calling other pure functions | Calling non-pure functions |

`Raise<E>` is allowed because it's a primitive (compiles to branches), not an effect.

### Syntax

```bounce
// Annotate a function as pure
pure fn add(a: Int, b: Int) -> Int {
    a + b
}

// Pure closures
let f: pure fn(Int) -> Int = { x => x * 2 }

// Compile error: pure function calls an effect
pure fn bad_example() -> Int {
    let n = Random.int(0, 100)    // ERROR: Random is an effect
    n
}

// Compile error: pure function calls non-pure function
fn fetch() -> String { Network.get("...").body }

pure fn also_bad() -> String {
    fetch()    // ERROR: fetch is not pure (requires Network)
    //         //        cannot call non-pure function from pure context
}
```

### Where Pure is Enforced

**State closures** (see `07-concurrency.md`): The `State.read` and `State.update` methods require `pure fn` closures. This prevents blocking the coordinator task and eliminates deadlocks structurally.

```bounce
fn read<R>(self: State<T>, f: pure fn(T) -> R) -> R
fn update(self: State<T>, f: pure fn(T) -> T) -> Unit
```

**Library authors**: Any function can be marked `pure` to guarantee it has no side effects. This is valuable for functions used in pipelines, map operations, and any context where purity is important.

**Generic constraints**: The `pure` annotation can be used as a constraint on function parameters:

```bounce
// This function accepts any mapping function, but it must be pure
fn transform<T, U>(items: List<T>, f: pure fn(T) -> U) -> List<U> {
    items |> map(f)
}
```

### Inference

The compiler does not infer `pure` automatically. It must be explicitly annotated. This is intentional — purity is a contract the developer opts into, not a property that silently changes when someone adds an effect call.

However, the LSP can suggest adding `pure` to functions that are already pure, via a code action.

---

## 6. Handlers

Effects are wired to implementations via handlers in world blocks. A handler connects a Bouncelang effect declaration to a concrete WASI implementation.

### World Block Syntax

```bounce
world cli {
    handle Network with WasiHttp
    handle FileSystem with WasiFilesystem
    handle IO with WasiCli
    handle Time with WasiClocks
    handle Random with WasiRandom
    handle Concurrency with WasiAsync

    config {
        timeout: Duration = 30s
        host: String = "localhost"
    }

    entry main
}
```

### Handler Types

**Standard WASI Handlers**: Map directly to standardized WASI interfaces. These are provided by the runtime (wasmtime, Spin, etc.).

```bounce
handle Network with WasiHttp
handle FileSystem with WasiFilesystem
```

**Parameterized Handlers**: Some handlers accept configuration:

```bounce
handle Database with Postgres(config.db_url)
handle FileSystem with WasiFilesystem(root: "/app/data")
```

**Package-Provided Handlers**: Third-party packages ship WASM components that implement effect interfaces. The handler name comes from the package:

```bounce
import Database from "bouncelang/postgres"

world api_server {
    handle Database with postgres.WasmHandler(config.db_url)
}
```

### Inline Handler Overrides

Within a scope, you can temporarily override a handler using `with handle`:

```bounce
fn process_safely() {
    with handle FileSystem = ReadOnlyFilesystem {
        // Inside here, FileSystem.write() raises PermissionError
        process_files()
    }
}
```

This is primarily useful for restricting capabilities in specific code paths, and for testing (see §7).

### Handler Resolution

When a function calls an effect operation (e.g., `Network.get(url)`):
1. The runtime looks up the handler for `Network` in the current world.
2. If an inline `with handle` override is active, it takes precedence.
3. The handler bridges the call to the WASI interface.
4. The host runtime processes the operation and returns the result.

If no handler is installed for an effect that the code requires, this is a compile error — the world must provide handlers for all effects used by the entry point's call graph.

```bounce
world cli {
    // Missing: handle Network
    entry main    // ERROR if main's call graph uses Network
}
```

---

## 7. Testing: Automatic Effect Sandboxing

Since every effect crosses a WASI boundary, the test runner can always intercept them. **Tests are automatically sandboxed** — no real network calls, no real file writes, no real clock access.

Each standard effect provides built-in test utilities that match its natural usage pattern. No `handle Effect = MockEffect(...)` boilerplate needed.

### Network (MSW / VCR style)

```bounce
test fn fetches_users() {
    // Configure mock responses — request matching
    Network.mock(:get, "https://api.example.com/users",
        response: { status: 200, body: "[{\"id\": 1}]" },
    )

    let users = fetch_users()
    assert(users.len() == 1)

    // Assert what was called
    assert(Network.requests == [
        :get { url: "https://api.example.com/users" },
    ])
}

test fn handles_network_error() {
    Network.mock(:get, "https://api.example.com/users",
        response: { status: 500, body: "Internal Server Error" },
    )

    try {
        fetch_users()
        assert(false, "should have raised")
    } catch {
        ApiError => assert(true)
    }
}
```

Unmatched network calls in tests raise `UnmockedEffectError` — preventing accidental real HTTP requests.

### Time (Freeze / Advance style)

```bounce
test fn timeout_fires() {
    Time.freeze(@2026-03-13T12:00:00Z)

    let timer = start_timer(duration: 5m)

    Time.advance(3m)
    assert(timer.is_active)

    Time.advance(3m)
    assert(timer.is_expired)
}

test fn scheduled_job_runs_at_midnight() {
    Time.freeze(@2026-03-13T23:55:00Z)

    let job = schedule_daily_job()

    Time.advance(10m)  // crosses midnight
    assert(job.ran_count == 1)
}
```

`Time.freeze()` stops the clock at a specific instant. `Time.advance()` moves it forward deterministically. `Time.now()` always returns the frozen/advanced time.

### FileSystem (Virtual FS style)

```bounce
test fn reads_config() {
    FileSystem.mock({
        "/etc/app/config.json": "{\"debug\": true}",
        "/etc/app/secrets.env": "API_KEY=test123",
    })

    let config = read_config("/etc/app/config.json")
    assert(config.debug == true)

    // File writes go to the virtual FS
    FileSystem.write("/tmp/output.txt", "result")
    assert(FileSystem.read("/tmp/output.txt") == "result")
}
```

The virtual filesystem starts with only the files you declare. Reads/writes to undeclared paths raise `FileNotFound`.

### Random (Seeded style)

```bounce
test fn deterministic_shuffle() {
    Random.seed(42)

    let result = shuffle([1, 2, 3, 4, 5])
    assert(result == [3, 1, 5, 2, 4])

    // Same seed, same result — always
    Random.seed(42)
    let result2 = shuffle([1, 2, 3, 4, 5])
    assert(result2 == result)
}
```

### IO (Captured style)

```bounce
test fn cli_output() {
    IO.mock_stdin(["yes", "Alice"])

    run_interactive_prompt()

    assert(IO.stdout_lines == [
        "Continue? [y/n]",
        "Enter name:",
        "Hello, Alice!",
    ])
}
```

### Concurrency (Deterministic Scheduler)

In test mode, `Concurrency` uses a deterministic scheduler controlled by the test seed. See `07-concurrency.md` §14 for details.

```bounce
test fn no_race_condition(seed: Int) with [sim: seed(seed)] {
    // Task ordering is deterministic given the seed
    let result = Concurrency.scope { s =>
        let c = s.state(0)
        let tasks = range(0, 100) |> map { _ =>
            s.spawn { c.update { n => n + 1 } }
        } |> collect
        tasks |> for_each { t => t.join() }
        c.value
    }
    assert(result == 100)
}
```

### Custom Effect Test Utilities

Third-party packages that declare effects should also ship test utilities following the same pattern:

```bounce
// From the "bouncelang/postgres" package
test fn queries_users() {
    Database.mock(
        :query, "SELECT * FROM users WHERE id = $1",
        params: [1],
        returns: [{ id: 1, name: "Alice" }],
    )

    let user = get_user(1)
    assert(user.name == "Alice")
    assert(Database.queries.len() == 1)
}
```

Package authors provide mock utilities that match their effect's natural usage pattern.

### Unmocked Effects

If test code calls an effect operation that hasn't been configured, the test runner raises `UnmockedEffectError` with a clear message:

```
Test Error: Unmocked effect call
  Network.get("https://api.example.com/data")

  Add a mock: Network.mock(:get, "https://api.example.com/data", response: ...)
```

This prevents tests from accidentally performing real I/O.

---

## 8. The `effect` Keyword

### Declaring Effects

The `effect` keyword declares a set of operations that cross a WASI boundary. It is similar to an `interface` but with different semantics — effect operations suspend the Fiber and delegate to the host.

```bounce
effect EffectName {
    fn operation_a(params...) -> ReturnType
    fn operation_b(params...) -> ReturnType
}
```

### Effects vs Interfaces

| Property | `effect` | `interface` |
|---|---|---|
| Operations | Suspend Fiber, cross WASI boundary | Regular function calls |
| Implementation | Host runtime (WASM component) | Bouncelang code (structural satisfaction) |
| Testability | Auto-sandboxed, mock utilities | Regular mocking (pass different functions) |
| Definable in Bouncelang | No (must have WASI backing) | Yes |
| World wiring | Required (`handle Effect with Handler`) | Not applicable |
| `pure fn` restriction | Blocked | Allowed |

### Effect Operations vs Regular Methods

Effect operations look like static method calls but behave differently:

```bounce
// Effect operation — suspends Fiber, crosses WASI boundary
let response = Network.get(url)

// Regular function call — no suspension, no WASI
let parsed = parse_json(response.body)
```

The compiler knows which is which based on whether the target is an `effect` or a regular type/module.

---

## 9. Standard Effects Reference

### Network

```bounce
effect Network {
    fn get(url: String) -> Response
    fn post(url: String, body: Bytes) -> Response
    fn put(url: String, body: Bytes) -> Response
    fn patch(url: String, body: Bytes) -> Response
    fn delete(url: String) -> Response
    fn request(req: Request) -> Response
    fn websocket(url: String) -> WebSocket
    fn listen(addr: String) -> Listener
}
```

Maps to `wasi:http/outgoing-handler` and `wasi:http/incoming-handler`.

### FileSystem

```bounce
effect FileSystem {
    fn open(path: String) -> File
    fn read(path: String) -> String
    fn read_bytes(path: String) -> Bytes
    fn write(path: String, content: String)
    fn write_bytes(path: String, content: Bytes)
    fn append(path: String, content: String)
    fn list(path: String) -> List<DirEntry>
    fn exists(path: String) -> Bool
    fn remove(path: String)
    fn create_dir(path: String)
    fn metadata(path: String) -> FileMetadata
}
```

Maps to `wasi:filesystem/types` and `wasi:filesystem/preopens`.

### IO

```bounce
effect IO {
    fn print(msg: String)
    fn println(msg: String)
    fn eprint(msg: String)
    fn eprintln(msg: String)
    fn read_line() -> String
    fn lines() -> Sequence<String> & :infinite
    fn args() -> List<String>
    fn env(key: String) -> String?
}
```

Maps to `wasi:cli/stdin`, `wasi:cli/stdout`, `wasi:cli/stderr`, `wasi:cli/environment`.

### Time

```bounce
effect Time {
    fn now() -> Instant
    fn datetime_now() -> Datetime
    fn local_timezone() -> Timezone
    fn sleep(duration: Duration)
    fn after(duration: Duration) -> Sequence<Instant> & :finite
    fn every(interval: Duration) -> Sequence<Instant> & :infinite
}
```

Maps to `wasi:clocks/monotonic-clock` and `wasi:clocks/wall-clock`.

### Random

```bounce
effect Random {
    fn int(min: Int, max: Int) -> Int
    fn float() -> Float
    fn bytes(count: Int) -> Bytes
    fn bool() -> Bool
    fn choice<T>(items: List<T>) -> T
    fn shuffle<T>(items: List<T>) -> List<T>
}
```

Maps to `wasi:random/random`.

### Concurrency

See `07-concurrency.md` for full specification. The `Concurrency` effect provides:

```bounce
effect Concurrency {
    fn scope<T>(f: fn(ScopeHandle) -> T) -> T
    fn timeout<T>(duration: Duration, f: fn() -> T) -> T?
    fn checkpoint()
    fn unshielded<T>(f: fn() -> T) -> T
}
```

Maps to WASI 0.3 Component Model Async.

---

## 10. Compilation & WebAssembly

### Effect Calls Compile to Fiber Suspension

When the compiler encounters an effect operation:
1. The current function's state is saved to the Fiber stack.
2. A WASI import call is emitted.
3. Control passes to the host runtime.
4. When the host completes the operation, the Fiber is resumed.
5. The result is available as the return value of the effect call.

```
// Bouncelang source:
let response = Network.get(url)

// Compiles to (conceptual WASM):
call $wasi_http_outgoing_request  // WASI import
// Fiber suspends here
// Host processes HTTP request
// Fiber resumes with response
```

### Effect Requirements in WIT

Each effect maps to a WIT import. The compiler generates a WIT world that declares all imports required by the program:

```wit
// Generated by the Bouncelang compiler
world my-app {
    import wasi:http/outgoing-handler;
    import wasi:filesystem/types;
    import wasi:clocks/monotonic-clock;
    import wasi:random/random;

    export run: func() -> result;
}
```

If a world block denies an effect, the corresponding WIT import is not generated. This means the compiled WASM component physically cannot access the denied capability — it doesn't have the import.

### Pure Functions Compile to Regular WASM

`pure fn` functions never contain WASI imports or Fiber suspension points. They compile to standard WASM functions with no special handling.

### Raise Compiles to Branches

`Raise<E>` does not use Fibers. It compiles to tagged return values and conditional branches:

```
// Bouncelang source:
fn get_user(id: Int) -> User {
    if id < 0 { raise InvalidId { id } }
    lookup(id)
}

// Compiles to (conceptual):
// Returns (tag: i32, value: User | InvalidId)
// tag 0 = success, tag 1 = InvalidId
// Caller checks tag and branches accordingly
```

---

## 11. LSP / DX (Developer Experience)

### Effect Inlay Hints

The LSP shows which effects each function requires as subtle inlay hints:

```bounce
fn build_report() -> Report {            // [Network, FileSystem]
    let users = fetch_all_users()        // [Network]
    let template = load_template()       // [FileSystem]
    compile_report(users, template)      // (pure)
}
```

### Pure Function Indicators

Functions annotated `pure` are visually marked. The LSP also shows when a function is effectively pure (could be annotated `pure`) via a suggestion code action.

### Effect Completions

After typing an effect name and `.`, the LSP suggests available operations:
- `Network.` → `get`, `post`, `put`, `patch`, `delete`, `request`, `websocket`, `listen`
- `Time.` → `now`, `datetime_now`, `sleep`, `after`, `every`

### Diagnostics

- **Pure violation**: `Cannot call 'Network.get' from pure context. Effect calls are not allowed in pure functions.`
- **Missing handler**: `World 'cli' does not handle effect 'Database'. Add: handle Database with ...`
- **Breaking change**: `Adding 'Network' requirement to pub fn 'process' is a breaking change (was pure).`

### Go-to-Definition

Clicking on an effect operation (e.g., `Network.get`) navigates to the effect declaration, showing the full operation signature and documentation.

### Test Mock Suggestions

When writing a test that calls a function requiring effects, the LSP suggests adding mock configurations:

```
Hint: 'fetch_users' requires Network.
      Add: Network.mock(:get, "...", response: ...)
```

---

## 12. World & Sandboxing (Brief)

Effects are wired and constrained in world blocks. Full specification in `09-worlds-and-sandboxing.md`.

### Key Points

- Worlds declare which effects are available via `handle` clauses.
- World types (published by frameworks) can `deny` effects — consumers cannot override denials.
- If an effect is denied, code using that effect fails at compile time.
- Denied effects produce WASM components that physically lack the corresponding imports.

```bounce
// Framework-published world type
export world type game_script_sandbox {
    sandbox {
        allow [Time, Random]
        deny [Network, FileSystem, IO, Concurrency]
    }
}

// User code targeting the sandbox
world my_mod extends game_script_sandbox {
    // Cannot add: handle Network with WasiHttp
    // ERROR: Network is denied by game_script_sandbox
    entry tick
}
```

---

## 13. Summary

### The Model

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Functions                                         │
│  Regular code. No suspension. Pass as parameters for DI.    │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Effects                                           │
│  WASI boundaries. Fiber suspension. Handled in worlds.      │
│  Network, FileSystem, IO, Time, Random, Concurrency         │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Primitives                                        │
│  Built-in. Always available. Raise (branches), Panic (trap) │
└─────────────────────────────────────────────────────────────┘
```

### Key Rules

1. **Effects = WASI boundaries.** You cannot define new effects in pure Bouncelang.
2. **Effects propagate implicitly.** No annotations on callers. LSP shows hints.
3. **`pure fn` = no effects.** Enforced by compiler. Raise and Panic are allowed (they're primitives).
4. **Tests are auto-sandboxed.** The test runner intercepts all effects. Each effect has idiomatic mock utilities.
5. **Worlds wire effects to handlers.** Missing handlers = compile error. Denied effects = no WASI import in binary.
6. **Semver tracking.** Adding an effect to a `pub fn` is a breaking change. The publish tool catches this.
