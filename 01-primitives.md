# Spec v2 layer 1: Primitives & Values

This document defines the core primitive types in Bouncelang, their literal syntax, their memory semantics, and how they interact with the broader ecosystem (Testing, LSP, Wasm, etc.).

---

## 1. What Are They?

Bouncelang primitives represent the indivisible atoms of data.
Everything in Bouncelang has **strict value semantics**. Primitives are inherently immutable. 

*   `Int`: A 64-bit signed integer. 
*   `Float`: A 64-bit IEEE 754 floating-point number (best for scientific/geometric math).
*   **`Decimal`**: A fixed-point / high-precision fractional number (best for monetary/financial math).
*   `String`: A UTF-8 encoded sequence of text.
*   `Bytes`: A dynamically sized sequence of raw bytes (`u8`).
*   `Bool`: A boolean value `true` or `false` (syntactic sugar for `:true | :false`).
*   **`Datetime`**: A specific point in calendar time. Represented internally as UTC, but is timezone-aware.
*   **`Instant`**: A specific point in strictly monotonic machine time (ignoring timezones, leap seconds, and clock skew).
*   **`Date`**: A calendar date without a time zone (e.g., 2026-03-12).
*   **`Duration`**: A span of time (e.g., 5 seconds, 2 weeks).

*(Note: Collections like `List<T>` and `Map<K, V>` are Data Structures, not primitives, and are covered in `02-data-structures.md`.)*

### The "Null" Question
**There is no `null`, `nil`, or `undefined` in Bouncelang.** Null is not a primitive, a keyword, or a concept. 
Absence of a value is *always* modeled using the `Option<T>` type (which is sugar for the union `(T & :true) | :false` — see `02-data-structures.md` §3). If a function takes an `Int`, it is guaranteed to be an `Int`, never null.

---

## 2. Syntax & Literals

### Integers, Floats, & Decimals
Negative numbers are supported natively via the `-` prefix.
```bounce
let decimal_int = 42
let negative = -1
let hex = 0x2A

// Floats are standard decimals
let pi = 3.14159
let negative_float = -2.5

// Decimals use a 'd' suffix to prevent precision loss during parsing
let price: Decimal = 19.99d
let debt: Decimal = -100.50d
```
*Why `Decimal`?* Floats (`3.14`) suffer from IEEE 754 precision issues (`0.1 + 0.2 != 0.3`). `Decimal` values (`19.99d`) guarantee exact fractional representation, making them the standard choice for all monetary and financial calculations in Bouncelang.
**Infinity & NaN:**
`Float` supports IEEE 754 infinity and Not-a-Number concepts, accessible via the standard library `math.inf()` and `math.nan()`. `Int` does not have representations for NaN or Infinity.

**Parsing from Strings:**
Parsing primitives from strings is a fallible operation and returns an `Option<T>` (`(T & :true) | :false`).
```bounce
let count = Int.parse("42")    // Option<Int>

if count {
    // `count` is narrowed to `Int & :true` — it IS the integer
    IO.println("Parsed: {count}")
}
```

**Limits:**
Integer boundaries are accessible via the `Int` type namespace: `Int.max()` and `Int.min()`.

### Strings & Multi-line
Strings are UTF-8 encoded and support interpolation via `{}`.
```bounce
let name = "World"
let greeting = "Hello, {name}!"
```
**Multi-line Strings:**
Multi-line strings use triple quotes `"""`. 
*Semantics:* The compiler automatically strips the leading indentation of the multi-line string based on the indentation level of the closing `"""`.
```bounce
fn html() -> String {
    let title = "Home"
    // The rendered string will NOT have the 4-space indent
    let page = """
    <html>
        <title>{title}</title>
    </html>
    """
    return page
}
```

### Bytes
Raw byte literals must be prefixed with `b`.
```bounce
let magic_header = b"\x00asm"
```

### Time: Instant vs. Datetime
Bouncelang strictly separates human calendar time from monotonic machine time to prevent subtle timezone and clock-skew bugs.

*   **`Datetime` (Calendar Time):** Represents a point in time conceptually tied to a timezone (IANA ID or offset). It is used for human-facing logic (e.g., "when does the sale start?", "next Tuesday"). 
    *   Math on `Datetime` (e.g., `+ 1d`) respects Daylight Savings Time boundaries.
*   **`Instant` (Machine Time):** Represents a point on a strictly monotonic clock. It has no concept of timezone or leap seconds. It is used exclusively for performance profiling, timeouts, and distributed system ordering.
    *   You *cannot* add `1d` to an `Instant`. You can only add fixed `Duration`s (e.g., `+ 24h`).

**Where do `now()` and local timezones come from?**
Because Bouncelang uses strict capability-based side-effects, there is no global `Instant.now()`, `Datetime.now()`, or `Timezone.local()` static method. Fetching the current time or the host machine's timezone requires interacting with the host system (an I/O Effect). To do this, the host World must inject a `Time` capability or handler into the environment:
```bounce
// Acquiring time and host timezone are effectful operations
let start: Instant = Time.now()
let local_tz = Time.local_timezone()
```

Because they are primitives, they also get first-class literal support (useful for testing):

```bounce
// Datetime (Calendar): ISO 8601 literals
let launch_utc: Datetime = @2026-03-12T19:30:15Z
let launch_ny: Datetime = @2026-03-12T15:30:15[America/New_York]

// Instant (Monotonic): Usually acquired from the runtime, but can be explicitly constructed for tests
let start: Instant = @instant(1710271815000000000) 

// Calendar Math understands Timezones
// If `launch_ny` is right before a DST boundary, adding 1 day adjusts the absolute UTC hours correctly.
let next_day = launch_ny + 1d 

let birthday: Date = @2026-03-12

// Duration suffixes
let timeout: Duration = 5s
let delay: Duration = 500ms
let long_wait: Duration = 2h + 30m
```

---

## 3. Math & Division Semantics

Bouncelang enforces strict boundaries between `Int` and `Float` to prevent precision bugs.

### Division
*   **Integer Division (`/`):** When dividing two `Int`s, the result is *always* an `Int` and it **truncates toward zero** (e.g., `5 / 2 == 2`, `-5 / 2 == -2`).
*   **Float Division (`/`):** When dividing two `Float`s, the result is a `Float` (e.g., `5.0 / 2.0 == 2.5`).
*   *You cannot divide an `Int` by a `Float` or `Decimal` directly. You must cast.*

### Decimal Math & Precise Allocation
Because `Decimal` is designed for financial math, Bouncelang includes safeguards against losing pennies (significant figures) during division.
*   **Scale / Significant Digits:** A `Decimal` tracks its own scale. `1.50d` is mathematically equal to `1.5d`, but retains its `2` decimal places of scale during additions or subtractions, preserving the significant figures natively.
*   **Safe Division (`divide_distribute`):** Dividing `$100.00` by `3` yields an infinitely repeating decimal. Regular division `100.00d / 3` will either trap or round based on a context, but the stdlib provides `Decimal.divide_distribute(amount, parts)` which returns a `List<Decimal>`. 
```bounce
// Distributes the "leftover penny" to the first element
let splits = Decimal.divide_distribute(100.00d, 3) 
// splits == [33.34d, 33.33d, 33.33d]
```
*   **Rounding Modes:** `Decimal.round(scale, mode)` allows explicit rounding with deterministic financial modes (e.g., `:half_up`, `:half_even`).

### Type Casting (Rounding vs Truncating)
*   `Float.to_int()`: Trucates the decimal (towards zero). `(3.9).to_int() == 3`.
*   `Float.round()`: Rounds to nearest integer (Returns an `Int`). `(3.9).round() == 4`.
*   `Float.floor()`: Rounds down to nearest integer (Returns an `Int`). `(-3.1).floor() == -4`.
*   `Float.ceil()`: Rounds up to nearest integer (Returns an `Int`). `(3.1).ceil() == 4`.
*   `Int.to_float()`: Converts to `Float`.

### Extended Math (Stdlib)
The `std/math` library provides constants and trigonometric functions on `Float`s:
```bounce
import math

let circumference = 2.0 * math.pi * radius
let y = math.sin(angle)
```

---

## 4. String Specifics & Allocations

### Length is Explicit
Because UTF-8 strings have variable byte lengths and complex grapheme clusters, a generic `.len()` is dangerous. Bouncelang requires you to be explicit about what you are measuring:
*   `str.byte_length()`: Returns the number of utf-8 bytes (fast, O(1)).
*   `str.glyph_length()`: Returns the number of human-readable grapheme clusters (slower, O(n)).

### String Slicing & Extraction
Because `String` is UTF-8 encoded, indexed access is non-trivial. 
*   **Slicing by Bytes:** `str.slice_bytes(start, end)` is an O(1) operation but returns a `Result<String, Utf8Error>` because the byte indices might slice a grapheme cluster in half.
*   **Slicing by Glyphs:** `str.slice_glyphs(start, end)` is an O(N) operation that guarantees a valid `String` return.
*   **Regex Extraction:** The `std/regex` module is the idiomatic way to extract substrings based on patterns.

### Byte Encoding & Manipulation
`Bytes` represents a raw `List<u8>`. Modifying raw bytes is the only way to do binary protocol work.
*   `bytes.slice(start, end)`: O(1) subsetting (returns a view).
*   `Bytes.from_string(str)`: O(N) UTF-8 encoding.
*   `bytes.to_string()`: Returns `Result<String, Utf8Error>`.

### The `Alloc` Effect
Because `String` and `Bytes` are dynamically sized, under the hood they allocate memory on the heap. 
**Is Allocation an Effect?**
No. Treating basic memory allocation as a tracked Effect (e.g., `try append() { AllocError => ... }`) would make the language completely unusable. Memory allocation is treated as a runtime engine concern. If the Wasm container hits an Out-Of-Memory limit, the guest module traps (panics) and the Host catches it. 
*Note: In DST (Deterministic Simulation Testing) mode, the runner can artificially inject OOM panics to test host resilience, but Bouncelang guest code does not "handle" allocations as first-class effects.*

---

## 5. Testing / DST (Deterministic Simulation)

Primitives behave predictably under Deterministic Simulation Testing (DST). 

### Generators
When writing property-based tests, the standard library provides generators for all primitive types:
```bounce
test fn time_math_works(start: Datetime, d: Duration) with [
    (gen(Datetime), gen(Duration(0s..1h)))
] { 
    assert(start + d > start) 
}
```
### Determinism Considerations
*   `Float` math is strictly defined by IEEE 754 to ensure deterministic execution across all CPU architectures.
*   *Warning:* When hashing `String` or `Bytes` values (e.g., using them as keys in a `Map`), the resulting iteration order across that Map must remain deterministic if the random seed is fixed. The runtime must use a deterministic hasher for DST mode.

---

## 6. LSP / DX (Developer Experience)

The Language Server Protocol (LSP) handles primitives as foundational semantic tokens to guarantee a rich editing experience.

*   **Semantic Tokens (Highlighting):**
    *   `Decimal`, `Int`, `Float`: Rendered using the `number` semantic token type. Modifiers should be applied for numeric suffixes (`d`).
    *   `Datetime`, `Date`, `Duration`: Rendered as `number` but ideally with a custom modifier (e.g. `number.datetime`) to allow themes to differentiate time values from pure math.
    *   `"""` Multi-line Strings: The LSP should visually dim the stripped indentation of `"""` strings to show exactly what whitespace will be included in the compiled binary.
    *   Interpolation: The `{}` brackets inside a string switch the lexer context back to expression mode. The LSP must highlight the interior expression properly, not as part of the string literal.
*   **Hover:** Hovering over a primitive literal should show its inferred type (`Int`, `Duration`, `Datetime`). 
*   **Completions:** 
    *   Typing `.` after a primitive triggers UFCS completions.
    *   `42.` -> suggests `to_float()`.
    *   `3.14.` -> suggests `round()`, `floor()`, `ceil()`.
    *   `"hello".` -> suggests `byte_length()`, `glyph_length()`, `starts_with()`, `split()`.
*   **Formatting:** The formatter should standardise numeric separators (e.g., rewriting `1000000` to `1_000_000` for `Int` literals above a certain threshold) and align decimal points in column-oriented record fields if requested.

---

## 7. Compiling & WebAssembly (Wasm)

Bouncelang primitives map exceptionally well to the WebAssembly Component Model (WITI):

*   **Int** -> Wasm `s64`.
*   **Float** -> Wasm `float64`.
*   **Decimal** -> Passed as a packed byte-array or struct (e.g. `tuple<s64, s8>` representing value and scale) to ensure exact precision across the Wasm boundary.
*   **Bool** -> Wasm `bool`.
*   **String** -> Wasm Component Model `string` (UTF-8).
*   **Bytes** -> Wasm Component Model `list<u8>`.
*   **Datetime / Date / Duration** -> These are compiled down to `s64` (usually representing nanoseconds or seconds since Unix epoch). For `Datetime`, the internal representation might be a packed struct containing the UTC nanoseconds and an index into a global timezone dictionary to maintain the embedded offset strictly by-value.

---

## 8. World / Sandboxing

Primitives themselves do not interact directly with Capability Sandboxes (Worlds). 
However, Worlds frequently accept primitives (including `Datetime` and `Duration`) as `config` parameters.
```bounce
world cli {
    config {
        timeout: Duration = 30s
        host: String = "localhost"
    }
}
```
The runtime injection mechanism guarantees that incoming host configuration values are safely parsed and validated into these primitive types before entering the Bouncelang execution environment.

---

## 9. Serialization, Formatting, & Hashing

Because these are the foundational building blocks of all logic, Bouncelang enforces strict, zero-configuration guarantees for serialization, display, and memory layout (hashing).

### JSON Serialization
Primitives serialize cleanly into standard JSON via `std/json`.
*   **`Int` / `Float` / `Decimal`** -> JSON Number. *(Note: `Decimal` can be configured globally or per-serialize to output as a JSON String to avoid frontend JS parsing precision loss, e.g., `"19.99"` vs `19.99`)*.
*   **`String`** -> JSON String.
*   **`Bool`** -> JSON Boolean.
*   **`Datetime` / `Date`** -> JSON String (ISO 8601).
*   **`Duration`** -> JSON String (e.g., `"5s"`).
*   **`Bytes`** -> JSON String (Base64 encoded).

### String Formatting (Display vs. Debug)
Bouncelang distinguishes between human-readable display and developer debug output. String interpolation `{x}` always calls the display trait.
*   `"{19.99d}"` -> `"19.99"` (Display)
*   **Debug Output:** There is a standard compiler macro `dbg!(x)` (or similar syntax) that prints the exact type semantics to stderr: `Decimal(19.99)`.

### Hashing & Map Keys
To serve as a key in a `Map<K, V>`, a type must be structurally hashable. All primitives in Bouncelang are guaranteed to be hashable and fulfill the `Hash` trait automatically.
*   **Floats & Decimals:** `NaN` != `NaN` in standard IEEE math, but for hash maps, if a `Map` allows a `Float` key, all `NaN` values hash to the exact same bucket.
*   **Strings & Bytes:** Hash functions are structurally stable (as mentioned in the DST section), meaning `Map` iteration order remains deterministic across runs if the RNG seed is fixed.
