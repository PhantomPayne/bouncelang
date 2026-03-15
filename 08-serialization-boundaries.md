# Spec v2 layer 8: Serialization Boundaries — Views, Inputs, and Formats

This document defines Bouncelang's three-concept model for serialization boundaries: `type` (internal domain shape), `view` (output boundary), and `input` (input boundary). It covers the complete syntax for field selection, aliasing, flattening, format selectors, casing policies, validators, cross-field validation, discriminated union parsing, and nested declarations — alongside the rationale for each design decision.

---

## 1. Overview and Philosophy

Bouncelang has three distinct boundary concepts, each doing exactly one job:

| Concept | Direction | Concerns |
|---|---|---|
| `type` | Internal | Shape, domain meaning |
| `view T.name` | Outbound | Field selection, formatting, casing, nesting |
| `input X -> T` | Inbound | Parsing, validation, field mapping, discriminators |

**Why separate output from input?** Output is pure — projecting an internal value to an external shape cannot fail and requires no effects. Input is not — parsing external data can fail (`ParseError`), and field validators may have effects (e.g., hashing a password). Conflating them into one concept (as some ORMs and validation libraries do) creates a mechanism that is effectful in one direction and pure in the other: harder to reason about, harder to teach, and harder to check statically. Separation keeps each concept simple and each mechanism doing exactly one job.

**Why keep `type` free of serialization concerns?** A developer writing an internal `type` is modeling a domain concept. The question "do I need a discriminate block on my union?" should never arise at that level. `type` declarations carry no parsing or formatting configuration — those concerns live exclusively on `view` and `input` declarations, which are created only when and if a type crosses a serialization boundary.

A `type` can have zero or many views. A `type` can have zero or more `input` declarations. Internal types with no serialization needs have no views or inputs — the type stays clean.

---

## 2. Views

### What a view is

A `view` is a **pure, named, compiler-recognized projection function**. It takes a value of an internal type and produces a narrower typed record containing only the declared fields. Because views are pure functions with no effects, they compose freely, work safely in concurrent contexts, and are transparent to the effect system.

Views return a nominal `View<T>` wrapper — not a plain record — so that serialization APIs (`Response.json`, `Logger.info`) can enforce at compile time that only view-projected values are passed. Passing a raw internal type to `Response.json` is a compile error.

### Basic field selection

```bounce
type User = {
    id: UserId
    name: String
    email: Email
    password_hash: String
    session_token: String
    created_at: Datetime
}

view User.public = {
    id
    name
}
```

Whitelist-only. Every field in the view body is an explicit inclusion. There is no blacklist or exclusion syntax — omitting a field means it is not included. This is safer by default: adding a field to `User` never silently changes the output of `User.public`. Sensitive fields like `password_hash` and `session_token` are excluded unless explicitly listed.

### Alias

```bounce
view User.public = {
    userId: id          // output key is "userId", source field is "id"
    displayName: name
}
```

Output key on the left, source field on the right. Reads as "output `userId` comes from field `id`." Per-field aliases take precedence over any global casing policy declared on the view.

### Flatten (path expressions)

```bounce
view User.public = {
    id
    city: address.city       // flatten nested field to top level
    country: address.country
}
```

Path expressions select a field from a nested record and promote it to the top level of the output. The `.` operator is the read-path operator (distinct from `->`, which is the write/map operator used in `input` field entries). Paths must be statically resolvable — arbitrary expressions are not allowed. Views select and rearrange existing data; they do not compute new data.

### Nested selection (inline)

```bounce
view User.public = {
    id
    address {
        city
        country
    }
}
```

Produces a nested output object `{ id, address: { city, country } }`. Inline nested selection desugars to an anonymous nested view. Use named subviews when the nested shape is reused across multiple views.

### Nested named subviews

```bounce
view Address.public = { city, country }

view User.public = {
    id
    name
    address: Address.public          // named subview for a nested record
    roles: List<Role.public>         // named subview applied to each list element
}
```

Named subviews are reusable. When applied to a `List<T>`, the view is mapped over each element — equivalent to `roles |> map(Role.public)`. This is the primary mechanism for nested reuse: explicit and named, with no hidden magic.

### Composition (spread)

```bounce
view User.core = {
    id
    name
}

view User.public = {
    ...User.core                     // expand User.core fields inline
    roles: List<Role.public>
}

view User.admin = {
    ...User.core
    email
    roles: List<Role.admin>
}
```

Spread expands another view's fields at the composition site. Duplicate output keys are a compile error — there is no "last one wins" or override mechanism. If you need to override a field from a spread view, do not spread that view; list the fields explicitly instead.

### Format selectors

Views can declare how a field is formatted for output using `as`:

```bounce
view User.public = {
    id
    name
    created_at as Datetime.iso8601    // field name unchanged, format applied
    role as Atom.string               // :admin -> "admin"
}

view Order.public = {
    id
    total as Money.display
    placed_at as Datetime.date
}
```

`as` reads as "include this field *as* this format." The stdlib ships common formats:

```bounce
// Stdlib formats (illustrative)
format Datetime.iso8601   // "2026-03-15T10:30:00Z"
format Datetime.date      // "2026-03-15"
format Datetime.unix      // 1742000000 (Int)
format Atom.string        // :admin_user -> "admin_user"
format Atom.camel         // :admin_user -> "adminUser"
format Float.fixed(n)     // 3.14159 -> "3.14"
```

User-defined formats for their own types:

```bounce
format Money.display = fn(m: Money) -> String {
    "{m.amount / 100}.{m.amount % 100 |> String.pad_left(2, "0")} {m.currency}"
}
```

Format functions must be pure and infallible — no `with` effects, always return a value. This is enforced by the compiler. Formats used in parsing (input direction) can fail; see the Formats section.

**Why format selectors instead of computed fields?** Computed fields (arbitrary expressions in view bodies) would require the view system to become a general-purpose programming language with its own effect checking, error handling, and determinism guarantees. Format selectors cover the common cases (date formatting, enum stringification, numeric precision) without that complexity. True computation — combining fields, aggregating lists — belongs on the source type or in a function before the view is applied.

### Casing policy

A view can declare a global output casing policy as a function reference:

```bounce
view User.public(casing: Casing.to_camel) = {
    id                               // -> "id"
    display_name                     // -> "displayName"
    created_at as Datetime.iso8601   // -> "createdAt", formatted
}
```

The stdlib ships casing functions:

```bounce
Casing.to_snake    // displayName -> display_name
Casing.to_camel    // display_name -> displayName
Casing.to_pascal   // display_name -> DisplayName
Casing.to_kebab    // display_name -> display-name
```

Casing is a `fn(String) -> String` — any pure function works. Per-field aliases (`userId: id`) take precedence over the global casing policy and are used when the output name genuinely differs from any casing transformation of the source name.

**Why a function reference instead of an atom like `:camel`?** Atoms create a closed set — adding a new casing convention requires a language change. Function references are open: any pure `fn(String) -> String` works. The stdlib ships the common cases; users can provide their own without waiting for language updates. This follows the "ecosystem vocabulary" principle: define the interfaces, not an exhaustive list of implementations.

### View field order

View field order in the output matches declaration order. Composition (`...OtherView`) expands fields inline at the spread site. This makes JSON output stable and predictable — important for API contract stability, snapshot testing, and API diff tooling.

### Views as functions

Views are callable as functions:

```bounce
let projected = User.public(user)              // single value
let projected_list = users |> map(User.public) // list — views work naturally with map
```

This is the primary ergonomic payoff of "views are functions": list conversion requires no special syntax.

### Serialization APIs require views

```bounce
Response.json(User.public(user))                                  // ✅
Logger.info(message: "login", fields: User.log(user))            // ✅
Response.json(user)                                               // ❌ compile error: Response.json requires a View<_>
```

The `View<T>` nominal wrapper is what makes this enforceable. A plain record and a view-projected record have the same structural shape but different nominal types — the compiler distinguishes them. This is the only mechanism that prevents accidental serialization of raw internal types.

**Why a nominal wrapper instead of a plain record?** If `User.public(user)` returned a plain `{ id, name }` record, `Response.json` would have to accept any record — you could pass raw `user` and it would compile. The nominal wrapper is invisible in normal use but provides a hard compile-time boundary at serialization sites.

### Union views

Views over union types must cover all variants — missing a variant is a compile error, matching the exhaustiveness philosophy of `match`:

```bounce
type Shape =
    | :circle { radius: Float }
    | :rectangle { width: Float, height: Float }

view Shape.public(discriminate: ShapeInput) = {
    :circle    => { radius }
    :rectangle => { width, height }
}
```

`discriminate: ShapeInput` tells the serializer to include the discriminator field defined in the corresponding `input` declaration when serializing. This avoids duplicating the discriminator definition across `view` and `input`.

### LSP and tooling

Because views are explicit declarations with a known source type, the LSP can provide:

- Autocomplete of valid field names inside view bodies, scoped to the source type
- Hover on a view application showing the resulting structural type
- "Find all views of User" as a navigation action
- Diagnostics: unknown field, type-incompatible format selector, unused view
- Quick-fix: "Create view `User.public`" when `Response.json(user)` is written without a view

---

## 3. Inputs

### What an input is

An `input` declaration is a **named parsing boundary** — the input analogue of a view. It defines how external data (JSON, query params, etc.) maps to an internal type, including field renaming, path unflattening, per-field validation, cross-field validation, and discriminator configuration for union types.

`input` declarations are separate from `type` declarations. This keeps internal types clean — a developer writing an internal `type` never encounters parsing or validation concerns unless they explicitly create an `input` for that type.

### Basic structure

```bounce
input RegistrationInput -> User(casing: Casing.to_camel) {
    fields {
        display_name -> name as String.any
        email as Email.format & String.max_length(255)
        password as String.min_length(8) & String.max_length(100) & String.no_whitespace
    }
    validate_all [
        fn(i) => if i.email |> String.contains(i.name)
            { raise ValidationError(field: "email", "email cannot contain username") }
            else { i }
    ]
}
```

`input RegistrationInput -> User` means:

- `RegistrationInput` is the name of the resulting parsed type — `Json.parse<RegistrationInput>` returns a `RegistrationInput`
- `-> User` is a compile-time contract: every mapped field must exist on `User` with a compatible type. If `User.name` is renamed, the mapping `display_name -> name` breaks at the `input` declaration site immediately, not silently somewhere in a function body.

`-> User` does **not** require that every field on `User` appears in the `input`. Fields like `id`, `password_hash`, and `created_at` are generated at construction time — the type system enforces their presence when you construct `User { ... }`, which is the right place for that check. Adding a redundant "generated fields" block to `input` would be two mechanisms enforcing the same thing — one mechanism per concern.

### Field entries

Each field entry in the `fields` block follows the pattern:

```
input_field -> internal_path as Validator & Validator ...
```

Where:
- `input_field` is the field name as it arrives in the external data (after casing is applied)
- `-> internal_path` maps to a field or nested path on the target type (optional when names match)
- `as Validator & Validator` declares one or more validators with AND semantics

All fields in the `fields` block must have at least one validator (or `String.any` / `Any` as an explicit "no validation needed" acknowledgment). The compiler enforces that no field silently passes through without a declared validation policy. This mirrors the exhaustiveness of `match` over unions: leaving a field without a validator is a compile error, not a runtime gap.

**Why exhaustive validators?** A missing validator is a missing security decision. Requiring `String.any` forces the author to consciously acknowledge "I have decided this field needs no validation" rather than forgetting. The audit trail is in the code.

### Field entry forms

```bounce
fields {
    // Same name, validate
    email as Email.format

    // Rename: JSON "displayName" (after casing) -> internal "name"
    display_name -> name as String.any

    // Unflatten: top-level input field -> nested path on internal type
    city -> address.city as String.any
    country -> address.country as String.any

    // Multiple validators (AND semantics — all must pass)
    password as String.min_length(8) & String.max_length(100) & String.no_whitespace

    // Optional field — `?` suffix marks the field as not required in incoming data.
    // If the field is absent, the mapped target field receives `:false` (Option<T> absent value).
    // If the field is present, it is validated normally. The target type must have `field: T?`.
    nickname as String.max_length(50)?
}
```

`->` is the "maps to" operator — consistent with its use elsewhere in the language for field path mapping. `:` is not used in `input` field entries because `:` means "alias" in views (output direction); `->` signals input direction mapping.

Unflattening (`city -> address.city`) is the inverse of view flattening (`city: address.city`). Together they let a flat external representation map to a nested internal type and back, without requiring the internal type to mirror the external shape.

### Validator composition

Validators compose with `&` (AND — all must pass):

```bounce
email as Email.format & String.max_length(255)
```

`&` was chosen over `|>` (pipeline) because pipeline implies sequential transformation of a value, while `&` implies multiple independent conditions asserted on the same value. The semantic distinction matters: pipeline transforms, `&` constrains.

`|` (OR) is reserved for future use (e.g., `Phone.format | Email.format` for a contact field that accepts either).

### Stdlib validators

```bounce
// String validators
String.any                     // accepts any string — explicit "no validation"
String.min_length(n)
String.max_length(n)
String.no_whitespace
String.matches(pattern)        // regex

// Numeric validators
Int.min(n)
Int.max(n)
Int.range(min, max)
Int.positive                   // > 0
Float.positive

// Domain validators
Email.format
Url.format
Phone.format

// Universal wildcard
Any                            // like _ in match — accepts any value of any type
```

### Cross-field validation

`validate_all` declares cross-field validators that run after all individual field validators pass. Multiple cross-field validators are listed in order — all run, and errors are collected:

```bounce
validate_all [
    fn(i) => if i.password != i.confirm_password
        { raise ValidationError(field: "confirm_password", "must match password") }
        else { i }

    fn(i) => if i.birth_year > 2006 && i.promo_code?
        { raise ValidationError(field: "promo_code", "promo not available for users under 18") }
        else { i }
]
```

**Execution order:**
1. Individual field validators run for all fields — errors collected
2. If all field validators pass → cross-field validators run in order — errors collected
3. All collected errors raised together as `ValidationError`

This gives the caller all errors at once rather than fail-fast, which is important for user-facing APIs.

### Casing

Casing is declared as a function reference on the `input` declaration:

```bounce
input RegistrationInput -> User(casing: Casing.to_camel) { ... }
```

The casing function converts incoming JSON field names to snake_case before matching against the `fields` block. So `"displayName"` becomes `"display_name"` before the `display_name -> name` entry is matched. Casing is a `fn(String) -> String` — any pure function works.

### Parsing

```bounce
fn handle_register(req: Request) -> Response
    with Database, Raise<ParseError>, Raise<ValidationError> {

    let input: RegistrationInput = req.body
        |> Json.parse<RegistrationInput>

    let user = Database.insert(User {
        id: UserId.generate()
        name: input.name
        email: Email(input.email)
        address: Address {
            city: input.city
            country: input.country
            street: input.street
        }
        password_hash: Password.hash(input.password)
        created_at: Datetime.now()
    })

    Response.json(User.public(user))
}
```

`Json.parse<RegistrationInput>` returns a `RegistrationInput` — a real named type with known fields. `Json.parse` requires an `input` declaration; passing a plain `type` is a compile error. This mirrors `Response.json` requiring a `View<T>` — both serialization boundaries are enforced by the compiler, not by convention.

### Discriminated union inputs

For union types parsed from external data, the `input` declaration carries discriminator configuration:

```bounce
type WebhookEvent =
    | :order_placed { order_id: String, total_cents: Int }
    | :payment_failed { order_id: String, reason: String }
    | :item_shipped { order_id: String, tracking_number: String }

input WebhookEventInput -> WebhookEvent(casing: Casing.to_snake) {
    discriminate {
        field: event_type           // identifier, not a string literal — "event_type" would be a parse error
        casing: Casing.to_snake     // "order_placed" -> :order_placed
        default: :unknown           // catch-all variant — see note below
    }
    fields {
        :order_placed => {
            order_id as String.any
            total_cents as Int.positive
        } validate_all [
            fn(i) => if i.total_cents < 0
                { raise ValidationError(field: "total_cents", "must be non-negative") }
                else { i }
        ]

        :payment_failed => {
            order_id as String.any
            reason as String.any
        }

        :item_shipped => {
            order_id as String.any
            tracking_number as String.any
        }
    }
}
```

**Why discriminator config on `input`, not `type`?** An internal union type like `type AppState = | :loading | :ready { data: T } | :error { message: String }` never needs a discriminator — it is never parsed from external JSON. Putting discriminator config on `type` would make every developer writing an internal union wonder "do I need a discriminate block?" Keeping it on `input` makes it clear: this is a parsing concern, not a type concern.

#### Discriminator strategies

- **Tagged** (most common): `field: event_type` — a named field in the JSON payload determines the variant
- **Structural**: `strategy: :structural` — variant determined by which fields are present. The compiler verifies all variants are mutually unambiguous; ambiguous structural variants are a compile error.
- **External tag**: not supported. For input, the wrapping-key pattern is uncommon enough that manual construction is preferred over a first-class language feature.

#### The `default` arm

```bounce
default: :unknown
```

Declares a catch-all variant for unknown discriminator values. When the incoming JSON has a `event_type` value that does not match any known variant, the value is parsed into `:unknown` (which must be a variant on the union type) rather than raising `ParseError`.

The default arm exists for forward compatibility with external specs (webhooks, third-party APIs) where new event types may be added by the external party without notice. It is **not recommended** when you control both sides of the interface — a fully discriminated union with no default arm is always preferable for code you own, because the compiler's exhaustiveness checking on `match` then catches unhandled variants at compile time.

#### Variant-level `validate_all`

Cross-field validators are declared per variant because each variant has its own field set. A validator for `:order_placed` has nothing to say about `:payment_failed` fields. Execution order matches the record case: field validators first, then `validate_all` for the matched variant.

### Nesting

```bounce
input OrderInput -> Order(casing: Casing.to_camel) {
    fields {
        id as String.any
        total_cents as Int.positive
        line_items: List<LineItemInput> as Any   // nested input applied to list
    }
}

input LineItemInput -> LineItem(casing: Casing.to_camel) {
    fields {
        product_id as String.any
        quantity as Int.positive
        unit_price_cents as Int.positive
    }
}
```

Nested `input` declarations applied to list fields work the same way as nested view declarations — the named input is applied to each element.

---

## 4. Formats (for parsing)

Output formats are pure and infallible. Input formats — used when parsing external data into internal types — can fail:

```bounce
format Datetime.from_uk_date = fn(s: String) -> Datetime with Raise<ParseError> {
    let [day, month, year] = s |> String.split("/")?
    Datetime.from_parts(day: Int.parse(day)?, month: Int.parse(month)?, year: Int.parse(year)?)?
}
```

Per-field format overrides can be passed to `Json.parse` at the call site:

```bounce
fn fetch_member(id: String) -> ExternalMember with Http, Raise<ParseError> {
    Http.get("/members/{id}")
        |> Json.parse<ExternalMember>(
            casing: Casing.to_pascal
            formats: { member_since: Datetime.from_uk_date }
        )
}
```

This handles the common case of a third-party API with a non-standard date format without requiring a nominal type wrapper. The format is a parsing concern local to that call site, not a fact about `Datetime` itself.

**Why allow consumer-defined formats instead of requiring nominal types?** A nominal type for "Barclays-style date string" is absurd overhead. The format is a fact about *that API's contract*, not about the `Datetime` type. Formats as pure function references keep the concern at the right level.

---

## 5. The Three-Concept Model

| Concept | Direction | Concerns | Declared on |
|---|---|---|---|
| `type` | Internal | Shape, domain meaning | Type definition |
| `view T.name` | Outbound | Field selection, formatting, casing, nesting | Separate `view` declaration |
| `input X -> T` | Inbound | Parsing, validation, field mapping, discriminators | Separate `input` declaration |

A `type` can have zero or many views. A `type` can have zero or more `input` declarations — multiple input shapes for the same internal type are handled by creating multiple `input` declarations targeting the same type. Internal types with no serialization or parsing needs have no views or inputs; the type stays clean.

---

## 6. Symmetry Reference

| Feature | View (output) | Input (inbound) |
|---|---|---|
| Field policy keyword | `field as Format` | `field as Validator & Validator` |
| Rename | `outputName: field` | `inputField -> internalField` |
| Flatten / Unflatten | `city: address.city` | `city -> address.city` |
| Casing | `casing: Casing.to_camel` fn ref | `casing: Casing.to_snake` fn ref |
| Nesting | `roles: List<Role.public>` | `roles: List<RoleInput>` |
| Composition | `...User.core` | — (not applicable) |
| Cross-field | — | `validate_all [ ... ]` |
| Returns | `View<T>` nominal wrapper | Named `InputType` record |
| Compiler enforcement | `Response.json` requires `View<T>` | `Json.parse` requires `input` declaration |

---

## 7. Future / Deferred

The following are explicitly deferred and should not be implemented in v1:

- **Computed fields in views** (e.g., `fullName: first_name + " " + last_name`) — most cases are covered by format selectors. True computed fields require purity verification and error handling in view bodies. Revisit when real usage patterns emerge.
- **OR validator composition** (`Phone.format | Email.format`) — syntax is reserved (`|`) but semantics are deferred.
- **Validator/matcher unification** — validators (`String.min_length(8)`) and test matchers share conceptual overlap. A unified `Predicate<T>` type usable in both contexts is a strong future direction but out of scope for v1.
- **External tag discriminator strategy for output** — rarely needed; manual construction is the recommended workaround.
- **World/WASM export integration** — views should eventually be usable in world export declarations (`export get_user: fn(UserId) -> User.public`) so WIT interfaces are generated from view shapes, not internal types. Deferred until the worlds spec is more complete.
