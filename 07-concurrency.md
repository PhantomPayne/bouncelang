# Spec v2 layer 7: Concurrency & Sequences

This document defines Bouncelang's concurrency model: scoped task spawning, shared state, channels, sequences as the universal async abstraction, and the `select` function for multiplexing.

---

## 1. Core Principle: Everything is a Sequence

`Sequence<T>` is the universal abstraction for "things that produce values over time." Generators, channels, broadcast subscriptions, tasks, timers, WebSocket streams, and list iterators all satisfy `Sequence<T>`.

Because of this, all existing language tools — `for`, `match`, `|>`, pipeline operators — work on concurrent data with zero new syntax.

| Source | Sequence Type | Finite? | Effectful? |
|---|---|---|---|
| `range(0, 10)` | `Sequence<Int> & :finite` | Yes | No |
| `fibonacci()` | `Sequence<Int> & :infinite` | No | No |
| `[1, 2, 3]` iterator | `Sequence<Int> & :finite` | Yes | No |
| `read_csv(path)` | `Sequence<Row> & :finite` | Yes | Yes (FileSystem) |
| `channel` | `Sequence<T> & :infinite` | No | Yes (Concurrency) |
| `broadcast.subscribe()` | `Sequence<T> & :infinite` | No | Yes (Concurrency) |
| `ws.messages` | `Sequence<Message> & :infinite` | No | Yes (Network) |
| `Time.every(30s)` | `Sequence<Instant> & :infinite` | No | Yes (Time) |
| `Time.after(30s)` | `Sequence<Instant> & :finite` | Yes | Yes (Time) |
| `Task<T>` | `Sequence<T> & :finite` | Yes (1 value) | Yes (Concurrency) |

### Finiteness Tags

Sequences carry `:finite` or `:infinite` as compile-time tags (see `02-data-structures.md` §3 on Intersection Tags).

Terminal operations that consume the entire sequence require `:finite`:

```bounce
fn collect<T>(self: Sequence<T> & :finite) -> List<T>
fn count<T>(self: Sequence<T> & :finite) -> Int
fn sum(self: Sequence<Int> & :finite) -> Int
fn reduce<T>(self: Sequence<T> & :finite, f: fn(T, T) -> T) -> T
```

Calling `collect` on an infinite sequence is a compile error:

```bounce
fibonacci() |> collect
// Compile Error: `collect` requires `Sequence<Int> & :finite`,
//   but `fibonacci()` returns `Sequence<Int> & :infinite`
```

Tag transforms:
- `take(n)` : `Sequence<T>` → `Sequence<T> & :finite`
- `cycle()` : `Sequence<T> & :finite` → `Sequence<T> & :infinite`
- `map`, `filter` : preserve the finiteness of their input

---

## 2. Generators

A generator is a function that uses the `yield` keyword to produce values lazily. It returns a `Sequence<T>`.

### Syntax

```bounce
fn fibonacci() -> Sequence<Int> & :infinite {
    mut a = 0
    mut b = 1
    loop {
        yield a
        (a, b) = (b, a + b)
    }
}

fn range(start: Int, end: Int) -> Sequence<Int> & :finite {
    mut i = start
    while i < end {
        yield i
        i = i + 1
    }
}
```

### Finiteness Inference

The compiler infers finiteness from the control flow:
- `loop { yield }` → `:infinite`
- `while cond { yield }` → `:finite`
- `for x in finite_seq { yield }` → `:finite`
- `for x in infinite_seq { yield }` → `:infinite`

The developer can annotate explicitly if inference is wrong or ambiguous.

### Compilation

**Pure generators** (no effects in body) compile to state machines. The `yield` points become states in an enum. The generator struct implements `next() -> T?`. Zero overhead — no Fiber, no suspension.

**Effectful generators** (use effects like FileSystem, Network) compile to Fiber-based execution. The generator needs Fiber suspension for effects anyway, so `yield` is an additional suspension point on the same Fiber. The consumer interface is identical.

```bounce
// Pure generator — compiles to state machine
fn squares(n: Int) -> Sequence<Int> & :finite {
    for i in range(0, n) {
        yield i * i
    }
}

// Effectful generator — compiles to Fiber
fn read_csv_rows(path: String) -> Sequence<Row> & :finite {
    let file = FileSystem.open(path)
    for line in file.lines() {
        yield parse_row(line)
    }
}

// Consumer doesn't know or care which implementation backs it
read_csv_rows("data.csv") |> filter { r => r.age > 18 } |> take(100) |> collect
```

---

## 3. Pipeline Operators on Sequences

All operators are lazy (return a new Sequence) unless marked as terminal.

### Transform
| Operator | Signature | Description |
|---|---|---|
| `map` | `(Sequence<T>, fn(T) -> U) -> Sequence<U>` | 1:1 value transform |
| `flat_map` | `(Sequence<T>, fn(T) -> Sequence<U>) -> Sequence<U>` | Flatten nested sequences, all run concurrently |
| `switch_map` | `(Sequence<T>, fn(T) -> Sequence<U>) -> Sequence<U>` | Cancel previous inner sequence on each new input |
| `concat_map` | `(Sequence<T>, fn(T) -> Sequence<U>) -> Sequence<U>` | Flatten sequentially, preserving order |
| `scan` | `(Sequence<T>, S, fn(S, T) -> S) -> Sequence<S>` | Running accumulator, emits each intermediate state |

### Filter
| Operator | Signature | Description |
|---|---|---|
| `filter` | `(Sequence<T>, fn(T) -> Bool) -> Sequence<T>` | Keep matching values |
| `distinct_until_changed` | `(Sequence<T>) -> Sequence<T>` | Skip consecutive duplicates |
| `take` | `(Sequence<T>, Int) -> Sequence<T> & :finite` | First N values |
| `take_while` | `(Sequence<T>, fn(T) -> Bool) -> Sequence<T> & :finite` | Values until predicate fails |
| `take_until` | `(Sequence<T>, Sequence<U>) -> Sequence<T> & :finite` | Values until another sequence yields |
| `skip` | `(Sequence<T>, Int) -> Sequence<T>` | Drop first N values |
| `skip_while` | `(Sequence<T>, fn(T) -> Bool) -> Sequence<T>` | Drop while predicate holds |

### Time
| Operator | Signature | Description |
|---|---|---|
| `debounce` | `(Sequence<T>, Duration) -> Sequence<T>` | Emit after quiet period |
| `throttle` | `(Sequence<T>, Duration) -> Sequence<T>` | At most one per interval |
| `delay` | `(Sequence<T>, Duration) -> Sequence<T>` | Delay each value |
| `timeout` | `(Sequence<T>, Duration) -> Sequence<T>` | Raise `TimeoutError` if no value within duration |
| `sample` | `(Sequence<T>, Duration) -> Sequence<T>` | Latest value at each interval |

### Combine
| Function | Signature | Description |
|---|---|---|
| `select` | `(name1: Sequence<T>, name2: Sequence<U>, ...) -> Sequence<:name1 { T } \| :name2 { U }>` | Tag + merge heterogeneous sequences |
| `merge` | `(List<Sequence<T>>) -> Sequence<T>` | Interleave same-typed sequences |
| `race` | `(List<Sequence<T>>) -> T` | First value from any source |
| `zip` | `(Sequence<T>, Sequence<U>) -> Sequence<(T, U)>` | Pair values 1:1 by position |
| `combine_latest` | `(Sequence<T>, Sequence<U>) -> Sequence<(T, U)>` | Latest from each when either emits |

### Terminal (require `:finite`)
| Operator | Signature | Description |
|---|---|---|
| `collect` | `(Sequence<T> & :finite) -> List<T>` | Gather all values into a list |
| `count` | `(Sequence<T> & :finite) -> Int` | Count values |
| `sum` | `(Sequence<Int> & :finite) -> Int` | Sum all values |
| `reduce` | `(Sequence<T> & :finite, fn(T, T) -> T) -> T` | Reduce to single value |
| `for_each` | `(Sequence<T> & :finite, fn(T) -> Unit) -> Unit` | Side-effect per value |
| `first` | `(Sequence<T>) -> T?` | First value or none |
| `last` | `(Sequence<T> & :finite) -> T?` | Last value or none |

### Utility
| Operator | Signature | Description |
|---|---|---|
| `tap` | `(Sequence<T>, fn(T) -> Unit) -> Sequence<T>` | Side-effect without altering stream |
| `enumerate` | `(Sequence<T>) -> Sequence<(Int, T)>` | Attach index |
| `chunk` | `(Sequence<T>, Int) -> Sequence<List<T>>` | Group into fixed-size batches |
| `buffer` | `(Sequence<T>, Duration) -> Sequence<List<T>>` | Group by time window |

---

## 4. Concurrency Scope

`Concurrency` is an effect that maps to WASI 0.3 Component Model Async (futures, streams, green threads). All concurrent operations happen within a **scope** — a bounded region that guarantees no orphan tasks.

### Syntax

```bounce
// Scope is an expression — the block's last expression is the return value
let result = Concurrency.scope { s =>
    // spawn tasks, create state, do work
    // ...
    final_value
}
// When the block body completes, the scope waits for ALL children
// before returning. No task escapes the scope.
```

### Scope Handle

The scope handle `s` is the gateway to all concurrency primitives. This ties every primitive's lifetime to the scope — when the scope ends, everything it created is cleaned up.

```bounce
Concurrency.scope { s =>
    // === Spawning ===
    let task = s.spawn { expr }      // linked task — failure cancels scope
    let task = s.detach { expr }     // unlinked task — failure is isolated

    // === Shared Primitives ===
    let counter = s.state(0)         // shared mutable state
    let ch = s.channel<T>(100)       // bounded single-consumer queue
    let bc = s.broadcast<T>(100)     // multi-subscriber fan-out
    let reg = s.registry<K, V>(f)    // named entity lookup

    // === Scope Control ===
    s.cancel()                       // cancel all children, end scope
}
```

### Spawning: `spawn` vs `detach`

**`s.spawn { expr }`** — Creates a linked task. If the task fails (raises an unhandled error), the scope cancels all other children and propagates the error. Use for tasks whose failure means the overall operation failed.

**`s.detach { expr }`** — Creates an unlinked task. If the task fails, only that task dies. Other children and the scope continue. Use for independent work like connection handlers.

Both return a `Task<T>` handle.

```bounce
// Parallel computation — linked (one failure cancels everything)
let (users, posts) = Concurrency.scope { s =>
    let a = s.spawn { fetch_users() }
    let b = s.spawn { fetch_posts() }
    (a.join(), b.join())
}

// Server — detached (one connection dying doesn't affect others)
Concurrency.scope { s =>
    for conn in listener.accept() {
        s.detach {
            try { handle_connection(conn) }
            catch { err => IO.eprintln("connection error: {err}") }
        }
    }
}
```

### Scope Nesting

Scopes can nest. An inner scope is a child task of the outer scope. Cancellation propagates downward — cancelling the outer scope cancels all inner scopes.

```bounce
Concurrency.scope { outer =>
    outer.spawn {
        Concurrency.scope { inner =>
            // inner scope is a child of the outer task
            // if outer is cancelled, inner is cancelled too
        }
    }
}
```

### Scope Completion Rules

1. The scope body runs to completion.
2. The scope waits for ALL children (spawned and detached) to finish.
3. If a linked child fails, all siblings are cancelled and the error propagates.
4. The scope returns the body's last expression value.
5. A detached child's failure is isolated — it does not affect the scope.

If a child never terminates (e.g., an infinite loop), the scope hangs. Use `s.cancel()` or a shutdown mechanism.

---

## 5. Task Handle

`Task<T>` represents a spawned concurrent computation. It satisfies `Sequence<T> & :finite` (yields one value, then completes), so tasks work with `select`, `for`, and all sequence operations.

```bounce
let task = s.spawn { compute_something() }

task.join() -> T         // Block until complete. Returns result or raises the task's error.
task.cancel()            // Request cooperative cancellation.
task.is_done -> Bool     // Non-blocking check.
```

### Join Semantics

`join()` blocks the calling task until the target task completes.
- If the task succeeded, `join()` returns its value.
- If the task raised an error, `join()` re-raises that error in the caller.
- If the task was cancelled, `join()` raises `Cancelled`.

```bounce
let task = s.spawn { might_fail() }

try {
    let result = task.join()
} catch {
    SomeError => handle_error()
    Cancelled => handle_cancellation()
}
```

### Tasks as Sequences

Because `Task<T>` satisfies `Sequence<T>`, tasks work with `select`:

```bounce
let a = s.spawn { strategy_a() }
let b = s.spawn { strategy_b() }

// Wait for whichever finishes first
for event in select(a: a, b: b) {
    match event {
        :a { result } => IO.print("A won: {result}")
        :b { result } => IO.print("B won: {result}")
    }
}
```

---

## 6. Shared State

`State<T>` is a concurrency-safe mutable value. Internally, it spawns a coordinator task within the scope that owns the value and processes requests sequentially. No locks, no mutex — atomicity comes from single-owner sequential processing.

### Creation

```bounce
let counter = s.state(0)
let rooms = s.state(Map<String, List<String>>{})
```

### Reading

```bounce
// Simple read — returns the whole value
let count = counter.value

// Extracting — closure runs in coordinator, only result crosses the channel
let member_count = rooms.read { r => r["general"]?.len() ?? 0 }
```

`value` copies the entire state to the caller. For large state, prefer `read { }` to extract only what you need.

### Updating

```bounce
// Atomic read-modify-write
counter.update { n => n + 1 }

// Returns a value from the update
let old = counter.update { n =>
    (n + 1, n)  // (new_state, return_value)
}
```

### Pure Closure Requirement

**Closures passed to `read { }` and `update { }` must be `pure` — no effects allowed.**

This is enforced by the compiler. The closure signatures are:

```bounce
fn read<R>(self: State<T>, f: pure fn(T) -> R) -> R
fn update(self: State<T>, f: pure fn(T) -> T) -> Unit
fn update<R>(self: State<T>, f: pure fn(T) -> (T, R)) -> R
```

The `pure` annotation prevents:
- **Blocking the coordinator** — an effect (Network, FileSystem) inside a closure would block all other reads/updates.
- **Deadlocks** — accessing another `State` inside a closure creates a circular dependency between coordinators.

```bounce
// COMPILE ERROR: closure is not pure
counter.update { n =>
    let data = Network.get(url)     // effect inside pure closure!
    n + data.value
}

// CORRECT: do effects outside, update with the result
let data = Network.fetch(url)
counter.update { n => n + data.value }
```

### TOCTOU Prevention

Never `read` then `update` separately when the update depends on the current value:

```bounce
// WRONG — race condition
let count = counter.value
counter.update { _ => count + 1 }   // another task may have updated between read and write

// CORRECT — atomic read-modify-write
counter.update { n => n + 1 }
```

### Coordinator Panic Resilience

If a user's closure panics, the coordinator catches the panic, sends the error back to the caller, and retains its previous state. The `State<T>` remains usable.

### Lifecycle

`State<T>` lives as long as its scope. When the scope ends, the coordinator task ends. Any pending `read`/`update` calls raise `Cancelled`.

---

## 7. Channels

A `Channel<T>` is a bounded, single-consumer queue. It satisfies `Sequence<T>` on the receiving side.

### Creation

```bounce
let ch = s.channel<Message>(capacity: 100)
```

### Sending

```bounce
ch.send(msg)
```

`send` **blocks** if the channel is full (backpressure). The calling task suspends until space is available. This provides natural flow control — fast producers are slowed to match consumer speed.

### Receiving

```bounce
// As a Sequence — use in for loops and pipelines
for msg in ch {
    process(msg)
}

// Or in select
for event in select(msgs: ch, cmds: command_channel) {
    match event {
        :msgs { m } => handle(m)
        :cmds { c } => execute(c)
    }
}
```

### Closing

When the scope ends, all channels are closed. Senders receive `Cancelled`. Receivers see the channel's sequence end (the `for` loop exits naturally).

A channel can also be explicitly closed:

```bounce
ch.close()
// Subsequent sends raise ChannelClosed
// Receivers drain remaining buffered values, then the sequence ends
```

### Oneshot Channel

For single-value request-response patterns:

```bounce
let reply = s.channel<Int>(capacity: 1)
request_channel.send(:count { reply })
let count = reply |> first   // blocks until the single reply arrives
```

This is just a channel with capacity 1, consumed once. No separate type needed.

---

## 8. Broadcast

`Broadcast<T>` is a multi-subscriber fan-out channel. Each subscriber gets its own independent buffer.

### Creation

```bounce
let announcements = s.broadcast<String>(capacity: 100)
```

The `capacity` is **per subscriber** — each subscription has its own buffer of this size.

### Publishing

```bounce
announcements.publish("server shutting down")
```

`publish` is **non-blocking**. The message is enqueued into each subscriber's buffer independently. If a specific subscriber's buffer is full, the overflow policy applies to that subscriber only.

### Subscribing

```bounce
let sub: Sequence<String> = announcements.subscribe()

// Each call to subscribe() creates a new independent subscription
// Use in for loops, pipelines, select — it's a Sequence
for msg in sub {
    IO.print(msg)
}
```

### Overflow Policy

```bounce
let bc = s.broadcast<String>(
    capacity: 100,
    overflow: :drop_oldest,     // default: drop oldest buffered value
    // alternatives: :drop_newest, :block, :error
)
```

- `:drop_oldest` — discard the oldest buffered value to make room (default).
- `:drop_newest` — discard the incoming value if buffer is full.
- `:block` — block the publisher until this subscriber catches up (dangerous — one slow subscriber blocks all publishing).
- `:error` — raise `BufferOverflow` to the subscriber on next read.

### Unsubscribe

Subscriptions are automatically cleaned up when the consuming task ends. When a task's `for sub in ...` loop terminates (task cancelled, break, scope ends), the subscription is removed from the broadcast's subscriber list. No manual unsubscribe needed.

---

## 9. Registry

`Registry<K, V>` manages a collection of named `State<V>` instances with automatic creation on first access.

### Creation

```bounce
let rooms = s.registry<String, RoomState>(
    init: { room_id => { members: [], history: [] } },
)
```

### Access

```bounce
// Returns State<RoomState> — creates it on first access using init function
let room = rooms.get("general")

room.update { r => { ...r, members: r.members |> append(user) } }
```

### Listing

```bounce
let room_ids: List<String> = rooms.keys()
let active_count = rooms.count()
```

### Removal

```bounce
rooms.remove("old-room")
// The State coordinator task is cancelled and cleaned up
```

---

## 10. The `select` Function

`select` merges multiple heterogeneous sequences into a single tagged sequence. It is a regular function — no new syntax.

### How It Works

```bounce
select(tag1: sequence1, tag2: sequence2, ...)
```

Each named argument is a `Sequence<T>`. `select` returns a `Sequence` of a tagged union where each tag corresponds to an argument name. It yields from whichever source has a value ready first.

### Type Generation

The compiler generates the return type from the named arguments:

```bounce
select(msg: ws.messages, tick: Time.every(30s))
// Return type: Sequence<:msg { Message } | :tick { Instant }>
```

The match exhaustiveness checker works normally on the generated union.

### Usage

```bounce
// Typical: in a for loop with match
for event in select(msg: ws.messages, tick: Time.every(30s), quit: shutdown) {
    match event {
        :msg { m } => handle(m)
        :tick => ws.send(:ping)
        :quit => break
    }
}

// Single-shot: consume one event (e.g., timeout pattern)
match select(result: task, timeout: Time.after(30s)) |> first {
    :result { r } => r
    :timeout => raise TimeoutError
}
```

### Fairness

When multiple sources have values ready simultaneously, `select` uses round-robin ordering to prevent starvation. No source is permanently prioritized over another.

In DST mode, the ordering is deterministic based on the test seed.

---

## 11. Convenience Functions

These are stdlib helpers built on scopes and sequences — no special language support.

### `Concurrency.timeout`

Cancel work after a deadline:

```bounce
let result = Concurrency.timeout(30s) {
    slow_operation()
}
// Returns: T & :true | :false (Option — :false means timeout)
```

Internally: creates a scope, spawns the work, uses select with `Time.after`, cancels on timeout.

### `race`

First result from multiple same-typed sequences:

```bounce
let fastest = race([
    s.spawn { try_cdn_a(file) },
    s.spawn { try_cdn_b(file) },
])
```

Returns the first value from any source. Remaining sources continue (they are not automatically cancelled — use a scope for that):

```bounce
let fastest = Concurrency.scope { s =>
    race([
        s.spawn { try_cdn_a(file) },
        s.spawn { try_cdn_b(file) },
    ])
    // scope ends → losing task cancelled
}
```

### `concurrent_map`

Bounded parallel map over a sequence:

```bounce
urls
    |> concurrent_map(max: 10) { url => Network.get(url) }
    |> collect
```

Processes items concurrently with at most `max` tasks running simultaneously. Preserves input order in the output.

### `retry`

Retry with configurable backoff:

```bounce
let result = retry(max: 5, backoff: :exponential) {
    Network.get(flaky_url)
}
```

### `supervise`

Catch errors, log, and continue (for use inside `detach`):

```bounce
s.detach {
    supervise { handle_connection(conn) }
}
// If handle_connection raises, the error is logged and the task ends cleanly
```

---

## 12. Cancellation

Cancellation is **cooperative**. A cancelled task doesn't die instantly — it raises `Cancelled` at its next effect yield point.

### How Cancellation Propagates

1. A task is cancelled (via `task.cancel()`, `s.cancel()`, or scope shutdown).
2. At the task's next effect yield (any Concurrency, Network, FileSystem, Time, or IO operation), `Cancelled` is raised.
3. `defer` blocks run. `using` resources are disposed.
4. The task terminates.

### Cancellation Points

Any effect operation is a cancellation point. For pure CPU work without effects, use explicit checkpoints:

```bounce
for item in huge_list {
    Concurrency.checkpoint()    // explicit cancellation point
    heavy_compute(item)
}
```

### Shielding Critical Work

Some cleanup must complete even during cancellation (e.g., sending a final message, flushing a buffer):

```bounce
s.detach {
    try {
        do_work()
    } catch {
        Cancelled => {
            Concurrency.unshielded {
                ws.send(:text { data: "goodbye" })    // won't be cancelled
            }
        }
    }
}
```

`Concurrency.unshielded { }` prevents cancellation from interrupting the block. Use sparingly.

### Graceful Shutdown

```bounce
Concurrency.scope(shutdown_timeout: 30s) { s =>
    s.detach { server_loop() }

    shutdown_signal.join()
    s.cancel()
    // 1. All children receive Cancelled at their next yield
    // 2. defer/using cleanup runs
    // 3. Wait up to 30s for cleanup to complete
    // 4. Force-terminate any remaining tasks
}
```

---

## 13. Error Handling in Concurrent Code

### Linked Tasks (`spawn`)

An unhandled error in a linked task cancels all siblings and propagates to the scope:

```bounce
let result = Concurrency.scope { s =>
    let a = s.spawn { fetch_users() }       // raises NetworkError
    let b = s.spawn { fetch_posts() }       // cancelled when a fails
    (a.join(), b.join())
}
// NetworkError propagates out of the scope
```

### Detached Tasks (`detach`)

An unhandled error in a detached task kills only that task. The scope and siblings continue:

```bounce
Concurrency.scope { s =>
    s.detach { might_fail() }    // if this fails, only this task dies
    s.detach { other_work() }    // continues running
}
```

### Errors Through Join

`join()` re-raises the task's error in the caller:

```bounce
let task = s.spawn { might_fail() }

try {
    let result = task.join()
} catch {
    NetworkError => fallback()
    Cancelled => IO.eprintln("task was cancelled")
}
```

### Errors in Select

If a sequence inside `select` raises an error, the error propagates through the `for` loop:

```bounce
for event in select(msg: ws.messages, data: flaky_stream) {
    // If flaky_stream raises NetworkError, it propagates here
    match event { ... }
}
```

Handle per-source errors by wrapping the source:

```bounce
let safe_data = flaky_stream
    |> map { d => :ok { d } }
    |> on_error { err => yield :error { err } }

for event in select(msg: ws.messages, data: safe_data) {
    match event {
        :data { :ok { d } } => process(d)
        :data { :error { err } } => IO.log("stream error: {err}")
        :msg { m } => handle(m)
    }
}
```

---

## 14. Testing / DST (Deterministic Simulation)

Concurrency is where DST provides the most value. A single seed controls task scheduling, timer resolution, and fault injection.

### Deterministic Scheduling

In DST mode, the WASI runtime uses a **deterministic scheduler**. Given the same seed, tasks are interleaved in the exact same order. This makes concurrent bugs reproducible.

```bounce
test fn no_lost_updates(seed: Int) with [sim: seed(seed)] {
    let counter = Concurrency.scope { s =>
        let c = s.state(0)
        let tasks = range(0, 100) |> map { _ =>
            s.spawn { c.update { n => n + 1 } }
        } |> collect

        tasks |> for_each { t => t.join() }
        c.value
    }

    assert(counter == 100)
}
```

Running with `bounce test --seed 42` reproduces the exact same interleaving.

### Virtual Time

In DST mode, `Time.after(30s)` and `Time.every(5s)` use virtual time. The test runner advances time instantly — no actual waiting.

```bounce
test fn timeout_works() with [sim: seed(1)] {
    let result = Concurrency.timeout(5s) {
        Time.sleep(10s)    // virtual — advances clock instantly
    }
    assert(result == :false)    // timed out
}
```

### Fault Injection

The DST runner can inject faults into concurrent primitives:

```bounce
test fn handles_channel_close(seed: Int) with [sim: seed(seed)] {
    simulate {
        inject_fault(:channel_close, after: 50)    // close channel after 50 sends
        run_producer_consumer()
    }
}
```

### Select Ordering

In DST mode, `select` resolves ties deterministically based on the seed. This ensures that "two sources ready at the same time" is reproducible.

---

## 15. LSP / DX (Developer Experience)

### Semantic Highlighting

- `spawn` and `detach` calls are highlighted distinctly (linked vs unlinked).
- `pure` closure boundaries inside `state.read` and `state.update` are visually marked.
- Cancellation points (effect yields) are subtly indicated in the margin.

### Diagnostics

- **Non-pure closure error**: If a `read`/`update` closure contains an effect, the LSP shows an inline error with a suggested fix (move the effect outside the closure).
- **TOCTOU warning**: If the compiler detects a `value`/`read` followed by `update` on the same state handle without an intervening scope boundary, it warns about a potential race condition.
- **Infinite scope warning**: If all children in a scope are detached and none have a termination condition, the LSP warns that the scope will never complete.
- **Unused task warning**: If a `spawn` result is not joined or otherwise consumed, the LSP warns that the task's error will be silently ignored.

### Completions

- After `s.`, suggest `spawn`, `detach`, `state`, `channel`, `broadcast`, `registry`, `cancel`.
- After `task.`, suggest `join`, `cancel`, `is_done`.
- After `state.`, suggest `value`, `read`, `update`.
- Inside `select()`, named argument completions suggest available sequences in scope.
- Inside `match` on a `select` result, auto-generate the atom tags from the `select` argument names.

### Inlay Hints

- Show the generated union type for `select` calls.
- Show `:finite` / `:infinite` tags on sequence-producing expressions.
- Show `linked` / `detached` next to spawn calls.

---

## 16. Compiling & WebAssembly

### WASI 0.3 Mapping

Bouncelang's concurrency maps to WASI 0.3 Component Model Async:

| Bouncelang | WASI 0.3 |
|---|---|
| `Concurrency.scope` | Component-level task spawning |
| `s.spawn` / `s.detach` | `task.spawn` |
| `Task<T>` | `future<T>` |
| `Channel<T>` | Internal `stream<T>` with backpressure |
| `Broadcast<T>` | Multiple `stream<T>` with fan-out |
| `select` | `task.poll` over multiple futures/streams |
| `Cancelled` | `task.cancel` signal |
| Sequence (effectful) | `stream<T>` |
| Sequence (pure) | Compiled to state machine, no WASI primitive |

### Green Threads

Each spawned task runs as a WASI green thread. The host runtime (wasmtime, Spin, etc.) schedules these cooperatively. Bouncelang code writes straight-line sequential code; the runtime handles multiplexing via epoll/io_uring/kqueue internally.

### State<T> Implementation

`State<T>` compiles to a coordinator task + two WASI streams (request/response). The coordinator is a green thread that owns the value and processes requests from its input stream. No shared linear memory between tasks — all communication is through WASI streams.

### Pure Generators

Pure generators (no effects) compile entirely within the WASM component as state machine structs. They do not use any WASI async primitives. The `next()` method is a regular synchronous function call.

---

## 17. World / Sandboxing

`Concurrency` is an effect. Worlds can deny it.

```bounce
// Game engine sandbox — no concurrency allowed
export world type game_script_sandbox {
    sandbox {
        deny [Concurrency]
    }
}
```

Code running in a world without `Concurrency` cannot use `scope`, `spawn`, `select`, or any concurrency primitives. Generators still work (they compile to state machines, no effect needed). Sequences from lists and pure generators work. Only concurrent/async sequences are blocked.

### Resource Limits

Worlds can constrain concurrency resources:

```bounce
world api_server {
    config {
        max_tasks: Int = 1000
        max_channels: Int = 100
    }
    handle Concurrency with WasiAsync(
        max_tasks: config.max_tasks,
        max_channels: config.max_channels,
    )
}
```

Exceeding limits raises `ResourceExhausted`.

---

## 18. Summary: Complete API Reference

### Effect Entry Point
```bounce
Concurrency.scope { s => expr } -> T       // scoped concurrency boundary
Concurrency.timeout(duration) { expr } -> T?  // cancel after deadline
Concurrency.checkpoint()                    // explicit cancellation point
Concurrency.unshielded { expr }             // shield from cancellation
```

### Scope Handle
```bounce
s.spawn { expr } -> Task<T>                // linked task
s.detach { expr } -> Task<T>               // unlinked task
s.state(initial) -> State<T>               // shared mutable state
s.channel<T>(capacity) -> Channel<T>        // bounded queue
s.broadcast<T>(capacity) -> Broadcast<T>    // multi-subscriber fan-out
s.registry<K, V>(init: fn) -> Registry<K,V> // named entity lookup
s.cancel()                                  // cancel all children
```

### Task Handle
```bounce
task.join() -> T                            // block until done
task.cancel()                               // request cancellation
task.is_done -> Bool                        // non-blocking check
// Task<T> satisfies Sequence<T> & :finite
```

### State Handle
```bounce
state.value -> T                            // read entire value
state.read { s => expr } -> R              // extract with pure closure
state.update { s => new_s } -> Unit         // atomic modify with pure closure
state.update { s => (new_s, result) } -> R  // atomic modify + return
```

### Channel
```bounce
ch.send(value)                              // blocks if full (backpressure)
ch.close()                                  // close the channel
// Channel<T> satisfies Sequence<T> & :infinite
```

### Broadcast
```bounce
bc.publish(value)                           // non-blocking, sends to all subscribers
bc.subscribe() -> Sequence<T> & :infinite   // independent subscription
```

### Registry
```bounce
reg.get(key) -> State<V>                    // get or create
reg.keys() -> List<K>                       // list all keys
reg.count() -> Int                          // number of entries
reg.remove(key)                             // remove and clean up
```

### Standalone Functions
```bounce
select(tag: seq, ...) -> Sequence<union>    // merge heterogeneous sequences
merge([seq, ...]) -> Sequence<T>            // interleave same-typed sequences
race([seq, ...]) -> T                       // first value from any source
zip(seq_a, seq_b) -> Sequence<(A, B)>       // pair 1:1 by position
combine_latest(seq_a, seq_b) -> Sequence<(A, B)>  // latest from each

concurrent_map(seq, max: N) { fn } -> Sequence<U>  // bounded parallel map
retry(max: N, backoff: strategy) { expr } -> T     // retry with backoff
supervise { expr }                                  // catch + log errors
```
