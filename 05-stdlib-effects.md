# Spec v2 layer 5: Standard Library Effects

**Status:** Draft — updated per PR review (see `decisions-log.md` Pipeline Cycle 3)
**Related:**
- [04-effects-and-handlers.md](04-effects-and-handlers.md) — effect system model, handler wiring, test sandboxing
- [worlds-and-handlers.md](worlds-and-handlers.md) — worlds, config, entry points
- [01-primitives.md](01-primitives.md) — `Instant`, `Duration`, `Datetime`, `Bytes`, `String`
- [07-concurrency.md](07-concurrency.md) — `Sequence<T>`, finiteness tags

---

## Overview

This document specifies the complete API for every **standard effect** that ships with Bouncelang.
The effect system model — how effects propagate, how handlers are wired, how pure functions work —
is covered in `04-effects-and-handlers.md`. This document focuses on the **API surface**: the
exact operations, their signatures, their error types, their DST mock handlers, and their LSP
behaviours.

Standard effects do not require an explicit import — they are available in any Bouncelang module
via the effect name. Third-party effects (like `Database`) require an explicit package import.

**Standard effects:**

| Effect | Purpose | Maps to |
|---|---|---|
| `Log` | Structured application logging | `wasi:cli/stdout` (log format) |
| `Terminal` | Interactive CLI I/O (stdin/stdout/stderr) | `wasi:cli/{stdin,stdout,stderr}` |
| `FileSystem` | File and directory operations | `wasi:filesystem/types` |
| `Http` | Outgoing HTTP/HTTPS requests | `wasi:http/outgoing-handler` |
| `WebSocket` | WebSocket connections | `wasi:http/outgoing-handler` (upgrade) |
| `Time` | Clocks, sleep, time-based sequences | `wasi:clocks` |
| `Random` | Secure random numbers and UUIDs | `wasi:random/random` |

---

## 1. Log

### What it is

`Log` is the effect for **structured application logging**. It is used by most apps: web services,
background workers, scheduled jobs, and CLI tools that write progress information. Unlike raw
stdout writes, `Log` emits structured key-value records that can be routed to log aggregators.

If your program is an interactive CLI tool that converses with a user via a terminal, use
`Terminal` instead.

### Operations

```bounce
effect Log {
    // Log a message at info level.
    fn info(msg: String) -> ()

    // Log a message at warn level.
    fn warn(msg: String) -> ()

    // Log a message at error level. Does NOT raise — use raise(SomeError) separately.
    fn error(msg: String) -> ()

    // Log a message at debug level.
    fn debug(msg: String) -> ()

    // Structured log with arbitrary key-value fields.
    fn info_fields(msg: String, fields: Map<String, String>) -> ()
    fn warn_fields(msg: String, fields: Map<String, String>) -> ()
    fn error_fields(msg: String, fields: Map<String, String>) -> ()
    fn debug_fields(msg: String, fields: Map<String, String>) -> ()
}
```

### Semantics

- `Log.error` writes an error-level log entry. It does **not** raise a Bouncelang error — if you
  want to both log and raise, do both explicitly:
  ```bounce
  Log.error("Payment failed: {e.message}")
  raise(PaymentError { order_id, reason: e.message })
  ```
- The production handler (`ConsoleLog`) formats log entries in JSON (default) or logfmt, writes
  to stdout, and prefixes each entry with an ISO 8601 timestamp.
- Log level is configured at the handler level in the world:
  ```bounce
  handle Log with ConsoleLog(level: :info)   // suppress :debug entries
  ```

### DST / Testing

The mock handler is `CapturedLog`:

```bounce
world test {
    entry test_runner
    handle Log with CapturedLog
}
```

`CapturedLog` stores all log entries in memory. Tests assert on captured output:

```bounce
test fn payment_logs_failure_on_bad_card() {
    let log = CapturedLog.new()
    handle log {
        process_payment(bad_card_order())
    }
    let entries = log.entries_at(:error)
    assert(entries |> any { e => e.msg |> String.contains("Payment failed") })
}
```

Tests that do not need to inspect log output can simply not declare a `Log` handler — any log
calls in that case are silently dropped (no world declaration needed if the test world omits it).

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `Log.` suggests all 8 operations. `_fields` variants show `Map<String, String>` hint. |
| **Inlay hints** | `// [Log]` on any function transitively calling `Log.*`. |
| **Diagnostics** | Calling `Log.error` without a corresponding `raise` in the same branch: Info hint "Did you mean to also raise an error here?" (suppressed if `raise` is present). |
| **Quick fixes** | None specific. |
| **Hover** | Clarifies that `Log.error` does not raise — distinguishes from `raise`. |
| **Semantic highlighting** | Standard effect namespace token. |

### Compiling & WebAssembly

`Log` maps to `wasi:cli/stdout`. Each call is a Fiber suspension. The handler formats the
structured record and writes it atomically. In DST/test mode, all calls are intercepted by
`CapturedLog` — no stdout I/O.

---

## 2. Terminal

### What it is

`Terminal` is the effect for **interactive CLI I/O** — writing to stdout/stderr and reading from
stdin. Use this for programs that have a conversational relationship with the user: command-line
tools, REPLs, interactive installers.

Most apps should use `Log` for output. `Terminal` is the exception, not the rule.

### Operations

```bounce
effect Terminal {
    // Write a string followed by a newline to stdout.
    fn println(msg: String) -> ()

    // Write a string without a trailing newline to stdout.
    fn print(msg: String) -> ()

    // Write a string followed by a newline to stderr.
    fn eprintln(msg: String) -> ()

    // Read a single line from stdin. Blocks until the user presses Enter.
    // Returns :false if stdin is closed (EOF).
    fn read_line() -> String?

    // Read all of stdin as a String. Blocks until EOF.
    fn read_all() -> String

    // Read all command-line arguments passed to the program.
    fn read_args() -> List<String>
}
```

### Semantics

- `println` and `print` perform no buffering beyond the OS stream.
- `read_line()` strips the trailing newline character (`\n` or `\r\n`).
- `read_line()` returns `:false` on EOF.
- None of these operations raise errors — a broken stream causes a runtime `Panic`.

### DST / Testing

The mock handler is `MockTerminal`:

```bounce
world test {
    entry test_runner
    handle Terminal with MockTerminal
}
```

`MockTerminal` allows seeding stdin lines and reading captured stdout:

```bounce
test fn cli_greets_user() {
    let term = MockTerminal.new()
    term.feed_line("Alice")         // simulate user typing "Alice" and pressing Enter

    handle term {
        let name = Terminal.read_line()
        Terminal.println("Hello, {name}!")
    }

    assert_eq(term.output(), "Hello, Alice!\n")
}
```

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `Terminal.` suggests all 6 operations. |
| **Inlay hints** | `// [Terminal]` on functions using it. |
| **Diagnostics** | Using `Terminal` in a web server entry function: Info hint "Most server apps use `Log` for output rather than `Terminal` — `Terminal` is for interactive CLI programs." |
| **Quick fixes** | None specific. |
| **Hover** | Shows signature and `Terminal` vs `Log` distinction. |
| **Semantic highlighting** | Standard effect namespace token. |

### Compiling & WebAssembly

`Terminal` maps to `wasi:cli/stdin`, `wasi:cli/stdout`, `wasi:cli/stderr`. Each read/write is a
Fiber suspension.

---

## 2. FileSystem

### What it is

`FileSystem` is the effect for reading and writing files on the host filesystem. It maps to the
`wasi:filesystem` WASI interface.

### The `Path` Type

All `FileSystem` operations accept `Path` values rather than raw strings. `Path` is a first-class
type with builder methods for construction and manipulation:

```bounce
// Construction
let root: Path = Path.from("/var/data")            // absolute path
let rel:  Path = Path.from("config/app.toml")      // relative path (resolved against handler root)

// Building paths — safe cross-platform join
let config: Path = root.join("config/app.toml")    // /var/data/config/app.toml
let logs:   Path = root.join("logs")               // /var/data/logs

// Inspection
let parent:   Path   = config.parent()             // /var/data/config
let name:     String = config.filename()           // "app.toml"
let ext:      String = config.extension()          // "toml"
let is_abs:   Bool   = config.is_absolute()        // true

// String conversion (for display/serialisation only)
let s: String = config.to_string()                 // "/var/data/config/app.toml"
```

`Path` values are always well-formed at the type level — you cannot construct a path with null
bytes or other invalid sequences. The handler enforces access control at runtime.

**Relative paths:** relative paths are resolved against the handler's `root` directory (configured
in the world). There is no global "current working directory" in Bouncelang — the handler root is
the effective root.

### Operations

```bounce
effect FileSystem {
    // Read the entire contents of a file as a UTF-8 string.
    // Raises FileSystemError if the file does not exist or is not readable.
    fn read(path: Path) -> String

    // Read the entire contents of a file as raw bytes.
    // Raises FileSystemError if the file does not exist or is not readable.
    fn read_bytes(path: Path) -> Bytes

    // Write a UTF-8 string to a file, replacing existing contents.
    // Raises FileSystemError if the path is not writable.
    fn write(path: Path, content: String) -> ()

    // Append a UTF-8 string to a file. Creates the file if it does not exist.
    // Raises FileSystemError if the path is not writable.
    fn append(path: Path, content: String) -> ()

    // List the entries in a directory. Returns paths (not just names).
    // Raises FileSystemError if the path is not a directory or is not readable.
    fn list(path: Path) -> List<Path>

    // Return true if a file or directory exists at the given path.
    fn exists(path: Path) -> Bool

    // Open a file for streaming line reads.
    // Raises FileSystemError if the file does not exist or is not readable.
    fn open_lines(path: Path) -> Sequence<String> & :finite

    // Open a file for streaming raw byte chunk reads.
    fn open_bytes(path: Path) -> Sequence<Bytes> & :finite

    // Open a file handle within a scoped block.
    // The file is automatically closed when the block exits — even if it raises.
    fn with_file<T>(path: Path, f: fn(File) -> T) -> T
}
```

**`with_file` for file handles:** When you need to pass a file handle around or perform mixed
reads and writes on the same file, use `with_file`:

```bounce
FileSystem.with_file(log_path) { file =>
    file.write_line("Starting job {job_id}")
    let result = do_work()
    file.write_line("Completed job {job_id}: {result}")
    // file is automatically closed here — even if do_work raised
}
```

`File` exposes:
```bounce
type File = opaque {
    fn read_line(self: File) -> String?
    fn read_all(self: File) -> String
    fn write_line(self: File, msg: String) -> ()
    fn write(self: File, bytes: Bytes) -> ()
    fn seek(self: File, pos: Int) -> ()
    fn position(self: File) -> Int
}
```

### Error Type

```bounce
error FileSystemError = {
    path: String          // path as string for display
    operation: :read | :write | :append | :list | :open | :stat
    message: String
}
```

> **Naming convention:** Each effect's errors are named `<EffectName>Error`.
> `FileSystem` → `FileSystemError`, `Http` → `HttpError`, `Database` → `DatabaseError`, etc.
> `IoError` is **not** used — it would be ambiguous about which I/O effect originated it.

All `FileSystem` operations (except `exists`) raise `FileSystemError` on failure.

### Handler Configuration and Sandboxing

The `WasiFileSystem` handler sandboxes the effect to a declared root directory. Code can only
access paths that are descendants of the configured root:

```bounce
world server {
    config { data_dir: String }
    handle FileSystem with WasiFileSystem(root: config.data_dir)
}
```

Relative paths like `Path.from("uploads/image.png")` resolve to `<data_dir>/uploads/image.png`.
Attempting to traverse above the root (e.g., `Path.from("../../etc/passwd")`) raises
`FileSystemError` at runtime.

### DST / Testing

The mock handler is `MockFileSystem`:

```bounce
// Option A: World-level default (for tests that don't need custom files)
world test {
    entry test_runner
    handle FileSystem with MockFileSystem
}

// Option B: Per-test inline handle (for tests that need to seed specific files)
test fn reads_config_file() {
    let fs = MockFileSystem.new()
    fs.add(Path.from("config.toml"), "timeout = 30\n")

    handle fs {   // overrides the world default for this test only
        let content = FileSystem.read(Path.from("config.toml"))
        assert_eq(content, "timeout = 30\n")
    }
}

test fn returns_error_for_missing_file() {
    let fs = MockFileSystem.new()    // empty — no files seeded
    handle fs {
        try FileSystem.read(Path.from("missing.txt")) {
            content => fail("expected error")
            FileSystemError { operation, ... } => assert_eq(operation, :read)
        }
    }
}
```

> **Reducing handler verbosity:** Tests that share the same file setup can use the world default
> handler without calling `.new()` inline. For tests needing custom seeding, the inline `handle`
> block covers just that test. You do not need both — use whichever form applies.

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `FileSystem.` suggests all operations. Path arguments show `Path` type hint. |
| **Inlay hints** | `// [FileSystem, Raise<FileSystemError>]` on functions using it. |
| **Diagnostics** | Passing a raw `String` as a `Path` argument: type error with quick-fix "Wrap in `Path.from(...)`". |
| **Quick fixes** | Wrap `FileSystem.read` in `try` → inserts `try` block with `FileSystemError` arm stub. |
| **Hover** | Each operation shows signature, which raises `FileSystemError`, and the `exists` exception. |
| **Semantic highlighting** | Standard effect namespace token. |

### Compiling & WebAssembly

`FileSystem` maps to `wasi:filesystem/types`. The handler root and preopened capabilities are
embedded in the compiled Wasm component manifest from the world declaration.

---

## 4. Http

### What it is

`Http` is the effect for **outgoing HTTP/HTTPS requests**. It is separate from `WebSocket` so that
worlds can grant one capability without the other (e.g., "this component may make HTTP calls but
may not open WebSocket connections").

### Operations

```bounce
effect Http {
    // Perform an HTTP GET request.
    // Raises HttpError on connection failure or status >= 400.
    fn get(url: String) -> Response

    // Perform an HTTP POST request with a raw bytes body.
    // Raises HttpError on connection failure or status >= 400.
    fn post(url: String, body: Bytes) -> Response

    // Perform an HTTP POST request with a JSON body (auto-serialised from a View).
    // Raises HttpError on connection failure or status >= 400.
    fn post_json<T>(url: String, body: View<T>) -> Response

    // Perform an arbitrary HTTP request using a Request builder.
    // NEVER raises HttpError for status >= 400 — returns the Response regardless.
    // Use this when you want to inspect 4xx/5xx responses manually.
    fn request(req: Request) -> Response
}
```

### The `Request` Builder

For requests that need headers, custom methods, or streaming bodies, use the `Request` builder:

```bounce
// Build a request step by step, then send it
let response = Http.request(
    Request.new(:post, "https://api.example.com/v1/orders")
        |> Request.header("Authorization", "Bearer {token}")
        |> Request.header("Idempotency-Key", idempotency_key)
        |> Request.json_body(order_view)
)

// Streaming: send a large body without buffering it all in memory
let response = Http.request(
    Request.new(:put, "https://upload.example.com/video")
        |> Request.header("Content-Type", "video/mp4")
        |> Request.body_stream(file_bytes_sequence)
)
```

`Request` companion functions:
```bounce
fn new(method: :get | :post | :put | :patch | :delete, url: String) -> Request
fn header(self: Request, key: String, value: String) -> Request
fn json_body<T>(self: Request, view: View<T>) -> Request
fn bytes_body(self: Request, body: Bytes) -> Request
fn body_stream(self: Request, body: Sequence<Bytes>) -> Request
```

### The `Response` Type

```bounce
type Response = {
    status: Int
    headers: Map<String, String>
    body: Bytes
    body_stream: Sequence<Bytes> & :finite   // lazy; reads body on first iteration
}

// Convenience companions
fn body_text(self: Response) -> String            // decode body as UTF-8
fn body_json(self: Response) -> Json              // parse body as JSON
fn ok(self: Response) -> Bool                     // status >= 200 && status < 300
```

**Streaming responses:** For large response bodies (file downloads, AI streaming, SSE), consume
`body_stream` rather than `body`:

```bounce
// Buffer the whole response (fine for small responses)
let response = Http.get("https://api.example.com/data")
let text = response.body_text()

// Stream a large response without buffering
let response = Http.request(Request.new(:get, "https://files.example.com/large.csv"))
response.body_stream |> each { chunk =>
    process_chunk(chunk)
}
```

### Error Type

```bounce
error HttpError = {
    url: String
    status: Int          // 0 if connection failed before receiving a response
    message: String
}
```

**Raise-on-error behaviour:**
- `Http.get` and `Http.post`/`Http.post_json` raise `HttpError` for status >= 400 and for all
  connection errors. This is the "raise_for_status" mode — it is the default for convenience.
- `Http.request` **never** raises for HTTP status codes. It returns the `Response` regardless.
  Use `response.ok` or check `response.status` explicitly. It still raises `HttpError` for
  connection failures (DNS, TLS, timeout).

```bounce
// Convenience functions: raise on 4xx/5xx
try Http.get("https://api.example.com/user/1") {
    response => parse_user(response.body_json())
    HttpError { status: 404, ... } => raise(UserNotFound { id: 1 })
    HttpError { message, ... } => raise(ServiceError { message })
}

// Request builder: manual status check
let response = Http.request(Request.new(:get, "https://api.example.com/user/1"))
if response.ok {
    parse_user(response.body_json())
} else {
    handle_error(response.status, response.body_text())
}
```

### URL Sandboxing

The `WasiHttp` handler supports an `allowed_origins` allowlist. Calls to origins not in the list
raise `HttpError` at runtime:

```bounce
world payment_service {
    handle Http with WasiHttp(
        allowed_origins: [
            "https://api.stripe.com",
            "https://api.github.com",
        ]
    )
}
```

The compiler emits a **compile-time warning** when a string-literal URL argument is not in the
allowlist (it cannot statically check dynamically constructed URLs):

```
Warning: Http.get called with literal "https://unknown-api.com" which is not in the Http
allowlist for world `payment_service`. This will raise HttpError at runtime.
  payments.bounce:42
```

### DST / Testing

The mock handler is `MockHttp`:

```bounce
world test {
    entry test_runner
    handle Http with MockHttp
}
```

Seeding responses in a test:

```bounce
test fn fetch_user_returns_parsed_data() {
    let mock = MockHttp.new()
    mock.on_get("https://api.example.com/users/1", Response {
        status: 200,
        headers: { "content-type": "application/json" },
        body: Json.encode({ id: 1, name: "Alice" }),    // Json.encode -> Bytes
        body_stream: Sequence.empty(),
    })

    handle mock {
        let response = Http.get("https://api.example.com/users/1")
        assert_eq(response.status, 200)
        assert_eq(response.body_json() |> Json.get_string("name"), "Alice")
    }
}
```

> `Json.encode(value)` returns `Bytes` and is the canonical way to construct JSON response bodies
> in test mocks. `Bytes.from_string(json_string)` works but is more verbose and error-prone.

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `Http.` suggests all operations. `Request.` suggests all builder methods. |
| **Inlay hints** | `// [Http, Raise<HttpError>]` on functions using `get`/`post`. `// [Http]` on functions using only `request`. |
| **Diagnostics** | Using `Http.get` without `try`: hint "Http.get raises HttpError for 4xx/5xx — consider wrapping in `try`." Using `Http.request` and ignoring `response.ok`: hint "Consider checking `response.ok` or `response.status`." |
| **Quick fixes** | Wrap `Http.get` in `try` with stub `HttpError` arm. |
| **Hover** | `Http.get` hover distinguishes raise behaviour vs `Http.request`. Shows which status codes trigger `HttpError`. |
| **Semantic highlighting** | Standard effect namespace token. |

### Compiling & WebAssembly

`Http` maps to `wasi:http/outgoing-handler`. The allowed-origins allowlist is embedded in the
component manifest as a network policy. Each `request` call is a Fiber suspension.

---

## 5. WebSocket

### What it is

`WebSocket` is the effect for **bidirectional WebSocket connections**. It is declared separately
from `Http` so that a world can permit outgoing HTTP calls without also permitting persistent
WebSocket connections.

### Operations

```bounce
effect WebSocket {
    // Open a WebSocket connection.
    // Raises WebSocketError if the handshake fails.
    fn connect(url: String) -> WsConnection
}
```

### The `WsConnection` Type

```bounce
type WsConnection = opaque {
    fn messages(self: WsConnection) -> Sequence<WsMessage> & :infinite
    fn send(self: WsConnection, msg: String) -> ()
    fn send_bytes(self: WsConnection, msg: Bytes) -> ()
    fn close(self: WsConnection) -> ()
}

type WsMessage =
    | :text   { data: String }
    | :binary { data: Bytes }
    | :close  { code: Int, reason: String }
```

### Error Type

```bounce
error WebSocketError = {
    url: String
    message: String
}
```

### Usage

```bounce
Concurrency.scope { s =>
    let conn = WebSocket.connect("wss://chat.example.com")
    conn.send("Hello!")

    conn.messages()
        |> take_until(s.shutdown_signal())
        |> each { msg =>
            match msg {
                :text { data } => handle_text(data)
                :binary { data } => handle_binary(data)
                :close { code, reason } => {
                    Log.info("connection closed ({code}): {reason}")
                    s.cancel()
                }
            }
        }
}
```

### DST / Testing

The mock handler is `MockWebSocket`:

```bounce
world test {
    entry test_runner
    handle WebSocket with MockWebSocket
}

test fn chat_client_handles_incoming_text() {
    let ws = MockWebSocket.new()
    ws.queue_message(:text { data: "Hello from server!" })
    ws.queue_close(code: 1000, reason: "done")

    handle ws {
        let conn = WebSocket.connect("wss://chat.example.com")
        conn.messages() |> take(1) |> each { msg =>
            match msg {
                :text { data } => assert_eq(data, "Hello from server!")
                _ => fail("unexpected message type")
            }
        }
    }
}
```

### Compiling & WebAssembly

`WebSocket` maps to `wasi:http/outgoing-handler` with a WebSocket upgrade header. Each message
send/receive is a Fiber suspension.

---

## 6. Time

### What it is

`Time` is the effect for reading the current time, sleeping, and producing time-based sequences.
It maps to `wasi:clocks`.

### Operations

```bounce
effect Time {
    // Return the current monotonic machine time as an Instant.
    fn now() -> Instant

    // Return the current calendar time as a Datetime in UTC.
    fn now_utc() -> Datetime

    // Return the host machine's local timezone.
    fn local_timezone() -> Timezone

    // Sleep for a fixed duration. Suspends the current Fiber.
    fn sleep(duration: Duration) -> ()

    // Return a finite sequence that yields one Instant after the given duration.
    fn after(duration: Duration) -> Sequence<Instant> & :finite

    // Return an infinite sequence that yields an Instant on every interval.
    // The sequence never stops — use `take` or a scope timeout to bound it.
    fn every(interval: Duration) -> Sequence<Instant> & :infinite
}
```

### Semantics

- `Time.now()` returns a **monotonic** clock value. It is guaranteed non-decreasing within the
  same process lifetime. It is not affected by NTP adjustments.
- `Time.now_utc()` returns the **wall clock**. It may go backward if the system clock is adjusted.
  Prefer `Time.now()` for measuring elapsed time.
- `Time.sleep(duration)` yields the Fiber to the scheduler — it does not block the OS thread.

### DST / Testing

The mock handler is `SimulatedTime`. It provides a virtual clock that starts at a configurable
epoch and only advances when explicitly instructed:

```bounce
world test {
    entry test_runner
    handle Time with SimulatedTime
}

test fn rate_limiter_resets_after_window() {
    let sim = SimulatedTime.new()
    sim.freeze(@2026-03-12T00:00:00Z)   // set the virtual clock

    handle sim {
        let limiter = RateLimiter.new(max: 5, window: 60s)
        range(0, 5) |> each { _ => limiter.consume() }

        sim.advance(61s)                // advance virtual clock by 61 seconds

        limiter.consume()               // should not raise after reset
    }
}
```

> **`sim.freeze` and `sim.advance` are called on the `SimulatedTime` handler instance, not on the
> `Time` effect.** The effect `Time` exposes only the operations that production code uses.
> Test control methods live on the handler.

`SimulatedTime.freeze(datetime)` and `SimulatedTime.advance(duration)` are available only in
test code. The handler is only wired in test worlds. Using them in production code is a
compile error (the `SimulatedTime` type is not available in production contexts).

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `Time.` suggests all operations. After `Time.after(`, the LSP suggests `Duration` literals. |
| **Inlay hints** | `// [Time]` on functions using `Time.now()` etc. |
| **Diagnostics** | Attempting to call `sim.freeze` or `sim.advance` in production code: compile error. |
| **Quick fixes** | None specific. |
| **Hover** | `Time.now()` clarifies it returns `Instant` (monotonic) vs `Time.now_utc()` (wall clock). |
| **Semantic highlighting** | Standard effect namespace token. |

### Compiling & WebAssembly

`Time` maps to `wasi:clocks/wall-clock` (for `now_utc`) and `wasi:clocks/monotonic-clock` (for
`now`, `after`, `every`). `Instant - Instant → Duration` compiles to a direct integer subtraction
on the underlying `u64` nanosecond representation.

---

## 7. Random

### What it is

`Random` is the effect for generating cryptographically secure random numbers and UUIDs. It maps
to `wasi:random/random`.

### Operations

```bounce
effect Random {
    // Generate a uniformly distributed random Int in [min, max) — half-open.
    fn int(min: Int, max: Int) -> Int

    // Generate a uniformly distributed random Float in [0.0, 1.0).
    fn float() -> Float

    // Generate a version 4 UUID (random bytes in UUID format).
    // Returns a lowercase hyphenated UUID string: "a3bb189e-8bf9-3888-9912-ace4e6543002"
    fn uuid4() -> String

    // Generate a version 7 UUID (timestamp-ordered, better for database primary keys).
    // Returns a lowercase hyphenated UUID7 string.
    // Implicitly uses the current wall time for the timestamp prefix.
    fn uuid7() -> String

    // Fill a Bytes buffer with random bytes.
    fn bytes(n: Int) -> Bytes
}
```

**UUID4 vs UUID7:**
- `uuid4()` — completely random bytes; globally unique; not sortable.
- `uuid7()` — timestamp-prefixed; globally unique; sortable by creation time. Preferred for
  database primary keys because ordered UUIDs reduce B-tree fragmentation.

**Why `Random` and not a separate `UuidGenerator` effect?** UUID4 is cryptographically random
bytes formatted as a UUID. UUID7 is a timestamp plus random bytes. Both require randomness. A
separate `UuidGenerator` effect would provide no additional sandboxing benefit (you'd need to
grant both). Keeping them together in `Random` reduces world declaration boilerplate.

### Semantics

- `Random.int(min, max)` panics if `min >= max`.
- `Random.float()` returns a value in `[0.0, 1.0)` — never exactly `1.0`.
- All `Random` operations use the WASI-provided secure random source — not seeded, always
  non-deterministic in production.
- For deterministic testing, use `SeededRandom`.

### DST / Testing

```bounce
world test {
    entry test_runner
    handle Random with SeededRandom
}
```

`SeededRandom` uses a deterministic PRNG seeded from the DST seed. Given the same seed, all
`Random` calls produce the same sequence. The seed is logged on test failure:

```
Test failed: load_balancer_distributes_evenly (seed: 8675309)
Re-run with: bounce test --seed 8675309 load_balancer_distributes_evenly
```

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `Random.` suggests all operations. |
| **Inlay hints** | `// [Random]` on functions using any `Random` operation. |
| **Diagnostics** | None specific. |
| **Quick fixes** | None. |
| **Hover** | `uuid4` vs `uuid7` distinction; half-open interval for `int`. |
| **Semantic highlighting** | Standard effect namespace token. |

### Compiling & WebAssembly

`Random` maps to `wasi:random/random`. No buffering — each call is a WASI import. For
high-frequency needs, call `Random.bytes(n)` and consume from the buffer.

---

## 8. Database (Reference Third-Party Effect)

The `Database` effect is **not** a standard Bouncelang effect — it is a reference third-party
effect defined here because it appears frequently in examples. Real projects use a specific
database package (e.g., `bouncelang/postgres`, `bouncelang/sqlite`) that declares its own effect.

This section documents the **reference Database effect** that those packages should conform to.

### Reference Declaration

```bounce
// From a database package (e.g., bouncelang/postgres)
effect Database {
    // Execute a query and return all matching rows.
    fn query(sql: String, params: List<Value>) -> Rows

    // Execute a query and return the first matching row, or :false if none.
    fn query_one(sql: String, params: List<Value>) -> Row?

    // Execute a non-returning statement (INSERT, UPDATE, DELETE).
    // Returns the number of rows affected.
    fn execute(sql: String, params: List<Value>) -> Int

    // Execute a function inside a database transaction.
    // If the function raises, the transaction is rolled back.
    // The callback is NOT pure — it calls the Database effect itself.
    fn transaction<T>(f: fn() -> T) -> T
}
```

> **Why `fn()` and not `pure fn()` in `transaction`?** The entire point of `transaction` is to
> run multiple database operations atomically. A `pure fn` cannot call effects — using it here
> would make `transaction` useless. The callback calls `Database.query`, `Database.execute`, etc.
> within the transaction boundary. The handler ensures all those calls participate in the same
> database transaction.

### Connection Pooling

Connection pooling is configured at the handler level in the world, not in the effect API. The
effect API is stateless — it does not expose connections:

```bounce
world production {
    config { database_url: String }
    handle Database with Postgres(
        config.database_url,
        pool_size: 10,
        idle_timeout: 30s,
    )
}
```

### Multiple Databases

When an application needs multiple databases, use effect aliasing with explicit imports:

```bounce
// In your package
import Postgres as PrimaryDb   from "bouncelang/postgres"
import Postgres as AnalyticsDb from "bouncelang/postgres"

fn save_order(order: Order) {
    PrimaryDb.execute("INSERT INTO orders ...", [...])   // primary
}

fn record_metric(event: Event) {
    AnalyticsDb.execute("INSERT INTO events ...", [...]) // analytics
}
```

World declaration:
```bounce
world production {
    config {
        primary_db_url:   String
        analytics_db_url: String
    }
    handle PrimaryDb   with Postgres(config.primary_db_url)
    handle AnalyticsDb with Postgres(config.analytics_db_url)
}
```

Each import alias becomes a distinct effect. The compiler tracks which alias is used in each
function and verifies the world provides a handler for each.

### Value Type

```bounce
type Value =
    | :null
    | :int   { v: Int }
    | :float { v: Float }
    | :text  { v: String }
    | :bytes { v: Bytes }
    | :bool  { v: Bool }
```

### Rows and Row Access

**Manual column access (escape hatch):**
```bounce
// Row exposes typed getter methods
type Row = opaque {
    fn get_int(self: Row, column: String) -> Int
    fn get_float(self: Row, column: String) -> Float
    fn get_string(self: Row, column: String) -> String
    fn get_bool(self: Row, column: String) -> Bool
    fn get_bytes(self: Row, column: String) -> Bytes
    fn get_datetime(self: Row, column: String) -> Datetime
    fn get_int_opt(self: Row, column: String) -> Int?
    fn get_string_opt(self: Row, column: String) -> String?
}
```

Column accessors panic if the column does not exist or the value cannot be cast. Use `_opt`
variants for nullable columns.

**`input` declaration (preferred):** Database rows are structured inputs. Use `input` declarations
to parse them declaratively — consistent with the rest of the serialisation model:

```bounce
// Declare how rows map to your internal type
input UserRow -> User {
    fields {
        id    as Int.positive
        name  as String.any
        email as Email.format
    }
}

// Usage: Row implements the Parseable interface required by parse_input
fn find_user(id: Int) -> User {
    try Database.query_one("SELECT id, name, email FROM users WHERE id = ?", [:int { v: id }]) {
        row => row |> parse_input(UserRow)     // declarative, validated
        :false  => raise(UserNotFound { id })
        DatabaseError { kind, ... } => raise(ServiceError { message: "db error: {kind}" })
    }
}
```

`parse_input(UserRow)` validates column values against the declared validators and maps them to
the `User` type. It raises `ValidationError` if any field fails validation.

### Error Type

```bounce
error DatabaseError = {
    sql: String
    message: String
    kind: :connection | :query | :constraint | :timeout | :transaction
}
```

### DST / Testing

```bounce
world test {
    entry test_runner
    handle Database with InMemoryDb
}

test fn list_users_returns_all_rows() {
    let db = InMemoryDb.new()
    db.seed_query(
        "SELECT * FROM users",
        [
            { "id": 1, "name": "Alice", "email": "alice@example.com" },
            { "id": 2, "name": "Bob",   "email": "bob@example.com"   },
        ],
    )
    handle db {
        let users = list_users()
        assert_eq(users |> count, 2)
    }
}
```

### LSP / DX

| Checklist Item | Behaviour |
|---|---|
| **Completions** | `Database.` suggests all operations. SQL strings get basic syntax highlighting inside string literals when the LSP detects a `Database.query` call. |
| **Inlay hints** | `// [Database, Raise<DatabaseError>]` on functions using Database. |
| **Diagnostics** | Passing a raw Bounce value (e.g., `Int`) as a `Value` parameter: "Expected `Value`, found `Int`. Wrap with `:int { v: ... }`." |
| **Quick fixes** | Auto-wrap parameter in `:int { v: ... }` or appropriate `Value` variant. |
| **Hover** | `row.get_string("col")` hover shows it panics on absent/wrong-type column. Suggests `input` declaration as the preferred parsing approach. |
| **Semantic highlighting** | Standard effect namespace token. |

### Go-to-definition for Effect Calls

> **LSP navigation for effects:** Go-to-definition on a `Database.query` call (or any effect call)
> navigates to the **effect declaration** (`effect Database { ... }`) to show the API contract.
> A separate "Go to handler" command (available via the command palette or a secondary keymap)
> navigates to the **world handler wiring** to show which concrete implementation handles the call.
> Both destinations are useful; the LSP exposes both.

### Compiling & WebAssembly

The `Database` effect is implemented by a third-party WASM component satisfying the WIT interface.
The compiler generates a WIT import based on the effect declaration.

---

## 9. Standard Handlers Reference

| Handler | Satisfies | Constructor | Environment |
|---|---|---|---|
| `ConsoleLog` | `Log` | `level: :debug \| :info \| :warn \| :error` | Production |
| `CapturedLog` | `Log` | `.new()` | Testing |
| `StdTerminal` | `Terminal` | none | Production |
| `MockTerminal` | `Terminal` | `.new()` | Testing |
| `WasiFileSystem` | `FileSystem` | `root: Path` | Production |
| `MockFileSystem` | `FileSystem` | `.new()` | Testing |
| `WasiHttp` | `Http` | `allowed_origins: List<String>` (optional) | Production |
| `MockHttp` | `Http` | `.new()` | Testing |
| `WasiWebSocket` | `WebSocket` | none | Production |
| `MockWebSocket` | `WebSocket` | `.new()` | Testing |
| `SystemTime` | `Time` | none | Production |
| `SimulatedTime` | `Time` | `.new()` | Testing |
| `WasiRandom` | `Random` | none | Production |
| `SeededRandom` | `Random` | `seed: Int` | Testing |

Database handlers are provided by their respective packages (`bouncelang/postgres`, etc.), not by
the standard library.

---

## 10. Combining Effects

Functions that use multiple effects declare no explicit effect list — the compiler infers the full
transitive effect set and the LSP surfaces it as inlay hints.

```bounce
fn create_user_report(email: String) -> Report {
    // inferred: [Http, Database, Time, Raise<DatabaseError>, Raise<HttpError>]
    let user  = find_user_by_email(email)               // Database
    let stats = Http.get("https://stats.api/{email}")   // Http
    let now   = Time.now()                              // Time
    compile_report(user, stats, now)
}
```

The world must provide handlers for every effect in the transitive set. If a handler is missing:

```
Error: world `production` is missing a handler for effect `Time`.
  Required by: create_user_report → Time.now() (reports.bounce:14)
  Fix: add `handle Time with SystemTime` to world `production`.
```

---

## 11. DST Summary

| Effect | Production handler | Test handler | Deterministic? |
|---|---|---|---|
| `Log` | `ConsoleLog` | `CapturedLog` | ✅ Output captured in memory; entries inspectable |
| `Terminal` | `StdTerminal` | `MockTerminal` | ✅ Stdin fed explicitly; stdout captured |
| `FileSystem` | `WasiFileSystem` | `MockFileSystem` | ✅ Seeded in-memory store |
| `Http` | `WasiHttp` | `MockHttp` | ✅ Configured stub responses |
| `WebSocket` | `WasiWebSocket` | `MockWebSocket` | ✅ Pre-queued messages |
| `Time` | `SystemTime` | `SimulatedTime` | ✅ Virtual clock — freeze and advance on handler |
| `Random` | `WasiRandom` | `SeededRandom` | ✅ Deterministic PRNG from seed |

Every standard effect has a deterministic test double. Code that uses only standard effects is
fully testable via DST.

---

## 12. LSP Summary

All standard effect namespaces appear as first-class namespace tokens in the LSP:

- **Go-to-definition** on any effect call (e.g., `Http.get`) navigates to the `effect Http { ... }`
  declaration in the standard library source.
- **Go-to-handler** (secondary command) navigates to the world handler wiring to show which
  concrete handler satisfies the effect in the current build target.
- **Hover** on any effect call shows the full operation signature and error behaviour.
- **Inlay hints** on functions show the inferred effect set: `// [Http, FileSystem, Raise<HttpError>]`.
- **Diagnostics** report missing world handlers at the `world` declaration, with the full call
  chain showing where each effect originates.

---

## 13. Compiling & WebAssembly Summary

| Effect | WASI Interface | Wasm mechanism |
|---|---|---|
| `Log` | `wasi:cli/stdout` | Fiber suspension per call |
| `Terminal` | `wasi:cli/{stdin,stdout,stderr}` | Fiber suspension per call |
| `FileSystem` | `wasi:filesystem/types` | Fiber suspension per call |
| `Http` | `wasi:http/outgoing-handler` | Fiber suspension per call |
| `WebSocket` | `wasi:http/outgoing-handler` (upgrade) | Fiber suspension per message |
| `Time` | `wasi:clocks/{monotonic,wall}-clock` | Fiber suspension per call |
| `Random` | `wasi:random/random` | WASI import call (no Fiber) |

In DST/test mode, all effect calls are intercepted by the test handler at the world boundary. The
Wasm module does not call WASI — it calls the in-process Bouncelang handler. This makes test
execution fast (no OS I/O) and deterministic.
