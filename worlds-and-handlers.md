# Worlds, Handlers, and Configuration

**Status:** Draft — evolved from design review discussion  
**Related:**
- [methods-and-packages.md](methods-and-packages.md) — UFCS, companions, `package.bounce` exports
- [modules-and-imports.md](modules-and-imports.md) — sub-modules, imports, stdlib, WASM linking

---

## 1. The Two Files

Every Bouncelang project has a root `package.bounce`. Sub-directories can also have `package.bounce` files acting as module barrels (see [modules-and-imports.md](modules-and-imports.md)). Applications also have one or more `world` declarations inside `package.bounce`.

### `package.bounce` — About the Code

Properties of the codebase itself. Same regardless of how it's built or deployed.

```bounce
package {
    name: myapp
    version: "0.1.0"
    license: MIT

    deps {
        std: "1.2"
        http: "^1.0"
        pg: "^1.0"
    }

    dev_deps {
        test-utils: "^2.0"
    }

    lints {
        std/deprecated: :warn
        std/unused: { default: :error, test: :off }
        std/complexity: { max: 15 }
    }
}

// Public API — no file paths, compiler finds declarations
export opaque type User        with { display, activate, validate }
export opaque type Transaction with { display, total }
export fn calculate_interest
test fn mock_user
```

### `world` — About a Build Target

Properties that change per deployment. Entry point, handler wiring, config schema.

```bounce
world server {
    config {
        database_url: String
        api_base: String
        log_level: :debug | :info | :warn | :error = :info
    }

    entry app

    handle Http     with WasiHttp
    handle Database with Postgres(config.database_url)
    handle Logger   with StdoutLogger(level: config.log_level)
}
```

### Decision Principle

> **Would this be the same if you deployed to two different targets?**
> - Yes → `package.bounce`
> - No → `world`

| Belongs in `package` | Belongs in `world` |
|---|---|
| Name, version, license | Entry point |
| Dependencies | Handler wiring |
| Public API exports | Typed config schema |
| Lint rules | Codegen steps |
| Dev dependencies | Build optimization flags |
| WASM target version | — |

---

## 2. Named Handlers

Handlers are standalone declarations that provide an implementation for an effect. They live in regular `.bounce` files and can be exported via `package.bounce`.

```bounce
handler RealNetwork(base_url: String): Http {
    get(url)        => resume(http_get(base_url + url))
    post(url, body) => resume(http_post(base_url + url, body))
}

handler MockHttp: Http {
    get(url)        => resume(Bytes.from("mock response"))
    post(url, body) => resume(Bytes.from("ok"))
}

handler Postgres(conn_str: String): Database {
    query(sql, params) => resume(pg_execute(conn_str, sql, params))
}

handler InMemoryDb: Database {
    let store = mut Map.new()
    query(sql, params) => resume(mem_query(store, sql, params))
}
```

### Handler Properties

- Handlers are first-class declarations with a name and the effect they satisfy
- They can take constructor arguments (e.g., `Postgres(conn_str)`)
- They can hold local mutable state (e.g., `InMemoryDb`'s `store`)
- They are exportable via `package.bounce` and importable by consumers

---

## 3. Worlds in Detail

### 3.1 Config Block

The `config` block declares a typed struct. Values are provided at runtime by the CLI.

```bounce
world api {
    config {
        database_url: String                                    // required
        redis_url: String                                       // required
        log_level: :debug | :info | :warn | :error = :info      // default
        max_retries: Int = 3                                    // default
    }
}
```

The CLI resolves config values. The language doesn't prescribe the source:

```bash
bounce run --world api                        # reads from env vars
bounce run --world api --env .env             # reads from .env file
bounce run --world api --config prod.yaml     # reads from YAML
bounce run --world api --set log_level=debug  # inline override
```

Resolution order (last wins): defaults → config file → env file → env vars → CLI flags.

Config values are available in the world via `config.field_name`. They are **not** an effect — they are resolved at startup before the effect system exists.

### 3.2 Entry Point

Each world declares one entry point. The entry function needs **no effect annotations** — the compiler infers its full transitive effect set and verifies it against the world's handlers.

```bounce
world server {
    entry app
    handle Http     with WasiHttp
    handle Database with Postgres(config.database_url)
}
// Compiler infers: `app` transitively requires Network, Database, Logger
// Network  — ✅ handled
// Database — ✅ handled
// Logger   — ❌ compile error:
//   "entry `app` requires effect `Logger` (used by `log_request`
//    in routes.bounce:42) but no handler is declared in world `server`"
```

### 3.3 Handler Declarations

Each `handle` line wires an effect to a named handler:

```bounce
world production {
    config { database_url: String }

    entry app
    handle Http     with WasiHttp
    handle Database with Postgres(config.database_url)
    handle Logger   with StdoutLogger(level: :info)
    handle Time     with SystemTime
}
```

Handlers are constructed in declaration order. **This only matters for future DI scenarios** — in V1, handlers are independent.

### 3.4 World Composition

Worlds can extend a base world. Child declarations override parent declarations for the same effect.

```bounce
world base {
    handle Database with Postgres(config.database_url)
    handle Logger   with StdoutLogger(level: config.log_level)
}

world server extends base {
    config {
        database_url: String
        api_base: String
        log_level: :debug | :info | :warn | :error = :info
    }
    entry app
    handle Http    with WasiHttp
}

world worker extends base {
    config {
        database_url: String
        redis_url: String
        log_level: :debug | :info | :warn | :error = :info
    }
    entry process_jobs
    handle Queue with RedisQueue(config.redis_url)
}

world cli extends base {
    config {
        database_url: String
        log_level: :debug | :info | :warn | :error = :debug  // louder default
    }
    entry cli_main
    handle Logger with StdoutLogger(level: :debug)  // override base
}
```

A base world without an `entry` cannot be built or run directly.

### 3.5 Inline Handle Overrides

Application code can locally override a world handler for a specific scope:

```bounce
fn process_sensitive_data(data: Data) {
    handle RedactedLogger {
        process(data)    // Logger effect uses RedactedLogger here
    }
    // back to the world's Logger handler outside this block
}
```

Handlers form a stack — inline `handle` pushes, scope exit pops. This is standard algebraic effect handler semantics.

### 3.6 Entry Function Signatures

The entry function's **signature** is determined by the WASI component model target, not by the
world declaration itself. Different component types expect different signatures:

| Component type | Entry signature | Notes |
|---|---|---|
| CLI (`wasi:cli/command`) | `fn main() -> ()` | Reads args via `Terminal.read_args()` |
| HTTP handler (`wasi:http/incoming-handler`) | `fn handle_request(req: Request) -> Response` | `Request` and `Response` from `std/http` |
| Background worker (queue trigger) | `fn process(job: Bytes) -> ()` | Format is queue-specific |
| Cron / timer trigger | `fn tick(at: Datetime) -> ()` | |

The `bounce build --world name` command selects the component model target based on the world's
handler set and any explicit target annotation. The compiler validates the entry signature against
the expected target:

```bounce
// ❌ Compile error: world `server` targets wasi:http but entry `main` has signature fn() -> ()
world server {
    entry main         // wrong signature for HTTP target
    handle Http    with WasiHttp
}

// ✅ Correct: handle_request has the right signature for HTTP
world server {
    entry handle_request
    handle Http    with WasiHttp
}
```

**Config access inside the entry function:** Config values are available inside any function
called from the world's entry, not just in handler wiring expressions. The config is injected as
a module-level record named `config` — any function in the package can read `config.field_name`
without importing it explicitly:

```bounce
// world declaration
world server {
    config {
        api_base: String
        log_level: :debug | :info | :warn | :error = :info
    }
    entry handle_request
    handle Http    with WasiHttp
    handle Logger  with StdoutLogger(level: config.log_level)
}

// app.bounce — config is available without any import
fn handle_request(req: Request) -> Response {
    let base = config.api_base      // ✅ reads from world config
    // ...
}
```

---

## 4. Effect Annotations — Where They're Required

| Context | Effects | Why |
|---|---|---|
| World entry points | **Inferred** — verified against handlers | The world IS the contract |
| Library exports (`pub fn` in `package.bounce`) | **Explicit** — required by compiler | API stability. Adding an effect is a breaking change |
| Internal functions | **Inferred** — shown as LSP inlay hints | No annotation tax on internal code |

For library exports, the compiler infers effects and the LSP offers a quick-fix to make them explicit:

```bounce
// You write:
pub fn fetch_user(id: UserId) -> User { ... }

// LSP inlay hint: with Http, Raise<ParseError>
// Quick-fix action: "Make effects explicit" →

pub fn fetch_user(id: UserId) -> User
    with Http, Raise<ParseError> { ... }
```

The explicit annotation lists **the effects this function produces** — callers' effects bubble up naturally through inference.

---

## 5. Real-World Examples

### HTTP API (Spin)

```bounce
// package.bounce
package {
    name: my-api
    deps { std: "1.2", http: "^1.0", pg: "^1.0" }
}

export fn handle_request

world spin {
    config {
        database_url: String
    }
    entry handle_request
    handle Http     with WasiHttp
    handle Database with SpinSqlite("default")
}
```

```bounce
// app.bounce
fn handle_request(req: Request) -> Response {
    let router = Router.new()
        |> get("/users",      list_users)
        |> get("/users/{id}", get_user)
        |> post("/users",     create_user)

    router.dispatch(req)
}
```

```bounce
// handlers/users.bounce
fn list_users(req: Request) -> Response {
    let users = Database.query("SELECT * FROM users", [])
    Response.ok(users |> to_json)
}
```

Build: `bounce build --world spin` generates a WASM component **and** a `spin.toml` with typed variables, permissions, and trigger configuration inferred from the world.

### CLI Tool

```bounce
world cli {
    config {
        verbose: Bool = false
    }
    entry main
    handle Log       with ConsoleLog
    handle Terminal  with StdTerminal
    handle FileSystem with RealFileSystem
}
```

```bounce
fn main() {
    let args = Terminal.read_args()

    args.files
        |> map { path => FileSystem.read_file(path) }
        |> map(transform)
        |> each { result => Terminal.println(result) }
}
```

### Background Worker

```bounce
world worker extends base {
    config {
        database_url: String
        redis_url: String
    }
    entry process_jobs
    handle Queue with RedisQueue(config.redis_url)
}
```

```bounce
fn process_jobs() {
    Logger.info("Worker starting")

    Queue.subscribe("jobs") { job =>
        match job.type {
            :send_email      => send_email(job.payload)
            :generate_report => generate_report(job.payload)
        }
    }
}
```

### Test World

```bounce
world test {
    config {
        test_timeout_ms: Int = 5000
    }
    entry test_runner
    handle Database with InMemoryDb
    handle Http     with MockHttp
    handle Time     with SimulatedTime
    handle Queue    with InMemoryQueue
}
```

No test-specific handler boilerplate in test files. Individual tests can still override inline:

```bounce
test fn email_sends_on_transfer() {
    let spy = spy(Http)
    handle MockHttp(spy) {
        transfer(from: alice, to: bob, amount: 100)
    }
    assert(spy.called(:post, times: 1))
}
```

---

## 6. Library vs Application

No flag needed. The distinction is structural:

| | Library | Application |
|---|---|---|
| Has `package.bounce` | Yes | Yes |
| Has `world` | No | Yes |
| Can `bounce check` | Yes | Yes |
| Can `bounce test` | Yes (uses default test world) | Yes |
| Can `bounce build` | Produces package artifact | Produces WASM component |
| Can `bounce run` | No | Yes |
| Can `bounce publish` | Yes | Typically no |

Libraries define types, functions, and handlers that applications wire together via worlds.

---

## 7. DST / Testing

### The Test World Is the DST Contract

The `world test` block is the primary DST configuration mechanism. It replaces all real-world
effect handlers with deterministic test doubles:

```bounce
world test {
    config {
        test_timeout_ms: Int = 5000
    }
    entry test_runner
    handle Database with InMemoryDb          // deterministic in-memory state
    handle Http     with MockHttp         // no real HTTP, explicit mock configuration
    handle Time     with SimulatedTime       // virtual clock — freeze and advance
    handle Random   with SeededRandom        // deterministic PRNG from seed
    handle Queue    with InMemoryQueue       // synchronous delivery
    handle Log      with CapturedLog
    handle Terminal with MockTerminal          // captures stdout/stderr, feeds mock stdin
}
```

### Virtual Time in World Tests

`SimulatedTime` implements the `Time` effect with a virtual clock. All `Time.now()`, `Time.sleep()`,
`Time.after()`, and `Time.every()` calls use the virtual clock:

```bounce
test fn rate_limiter_resets_after_window() {
    Time.freeze(@2026-03-13T12:00:00Z)

    let limiter = RateLimiter.new(max: 10, window: 1m)
    range(0, 10) |> for_each { _ => limiter.consume() }

    // Window not elapsed yet — next consume should fail
    try limiter.consume() {
        _ => fail("expected rate limit error")
        RateLimitError { ... } => assert(true)
    }

    // Advance past the window
    Time.advance(61s)

    // Limiter should reset
    limiter.consume()    // no error
}
```

### Fault Injection via Handler Overrides

Inline `handle` overrides inject failures at specific effect boundaries without needing a full
alternative world:

```bounce
test fn retries_on_network_failure() {
    let call_count = s.state(0)

    handle FailFirstNetwork(call_count) {
        fetch_with_retry(url: "https://api.example.com/data")
    }

    assert_eq(call_count.value, 2)    // one failure + one success
}

handler FailFirstNetwork(count: State<Int>): Http {
    get(url) => {
        count.update { n => n + 1 }
        if count.read { n => n } == 1 {
            raise(HttpError { url, status: 500, message: "injected failure" })
        }
        resume(http_get(url))
    }
}
```

### Deterministic Scheduling

The `Concurrency` effect uses a deterministic scheduler in test mode. Given the same seed, task
interleaving is reproducible. A failing test records its seed in the output — re-running with the
same seed reproduces the failure exactly:

```
Test failed: no_race_condition (seed: 4294967297)
  Re-run with: bounce test --seed 4294967297 no_race_condition
```

---

## 8. LSP / DX (Developer Experience)

| Checklist Item | Behavior |
|---|---|
| **Completions** | In a `world` block, after `handle `, the LSP suggests all effects in the transitively required set for the `entry` function, sorted by unsatisfied-first. After `with `, the LSP suggests all `handler` declarations in scope for the selected effect. Inside a `config { }` block, field types are autocompleted from available types. |
| **Inlay hints** | On the `entry` declaration, the LSP shows the full transitive effect set inferred for the entry function. This lets the developer see at a glance which `handle` lines are needed. On each `handle` line, the LSP shows the handler's constructor arguments and their types as inlay hints. |
| **Diagnostics** | Missing handler: "Effect `Logger` is required by `log_request` (routes.bounce:42) but no handler is declared in world `server`." Unknown handler: "`WasiHttp` is not in scope — import it from a dep or define a handler." Config type mismatch: "`database_url` expects `String`, but world `staging` provides `Int`." Unused handler: "Effect `Queue` is handled but not used by any function in `app`'s call graph." |
| **Quick fixes** | "Add missing handler" — inserts a `handle Effect with ?` stub for each unsatisfied effect. "Generate config schema" — creates the `config { }` block with all fields inferred from `config.field` usages in the entry function. "Extract world" — cursor on a set of `handle` lines, extracts them into a named `world base { }` for composition. |
| **Hover / go-to-definition** | Hovering a handler name (e.g., `WasiHttp`) shows the effect it satisfies and its constructor arguments. Go-to-definition navigates to the `handler` declaration. Hovering `config.field_name` shows the field type and its default value. |
| **Semantic highlighting** | `world` and `handler` keywords receive distinct tokens. Effect names in `handle` lines (`Network`, `Database`) receive the `type` semantic token. Handler names (`WasiHttp`, `Postgres`) receive the `class` token. Config field names receive the `property` token. |
