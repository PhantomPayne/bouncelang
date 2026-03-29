# Bouncelang LSP: Quick Fixes, Code Actions & Diagnostics

This document defines the Language Server Protocol (LSP) quick fix and diagnostic system for Bouncelang. It is a first-class design document — not an afterthought — because good tooling is a core part of Bouncelang's DX promise.

---

## Philosophy

A missing quick fix is a DX tax paid on every use of a feature. If views and inputs feel like boilerplate, the problem is the tooling, not the feature. The language can be as well-designed as possible, but if a developer has to hand-write every scaffold the first time they encounter a new construct, the ergonomics suffer.

Quick fixes should generate correct first drafts that the developer tweaks — not perfect final code, but a valid starting point that compiles. The goal is zero-to-working in one keystroke, then refine from there.

The best quick fix teaches the feature as it runs. A developer who has never written a `view` before should be able to trigger "Generate view" and immediately understand what a view looks like from the output. Error messages and scaffolded code are documentation.

Bouncelang's features (views, inputs, exhaustive validators, discriminators) are designed to be mechanically derivable from type information — the compiler already has everything it needs to generate these. The LSP just surfaces that. This is not incidental: the language is designed so that its enforcement rules are always accompanied by machine-actionable fixes. If the compiler can enforce it, the LSP can fix it.

---

## Serialization Boundary Quick Fixes

These are the most important quick fixes because views and inputs are new concepts with real first-draft boilerplate. A developer encountering them for the first time should be able to get to correct code without leaving their editor.

### "Generate view from type"

**Trigger:** Writing `Response.json(user)` or `Logger.info(fields: user)` where `user` is a raw internal type (not a `View<T>`).

**Diagnostic:** `error: Response.json requires a View<_>, got User. Use a view: Response.json(User.public(user))`

**Quick fixes offered:**
1. **Generate view `User.public`** — generates a view scaffold including all fields, with a comment prompting the developer to remove sensitive fields
2. **Generate view `User.log`** — same but named for logging context
3. **Choose existing view** — if `User.public` already exists, offer to wrap the call: `Response.json(User.public(user))`

**Generated output for "Generate view `User.public`":**

```bounce
// Generated — remove or restrict fields that should not be public
view User.public = {
    id
    name
    email
    // password_hash  <-- removed: field name suggests sensitive data
    // session_token  <-- removed: field name suggests sensitive data
    created_at as Datetime.iso8601
}
```

Important details about the generated view:
- All fields are included by default EXCEPT fields whose names match a heuristic list of obviously sensitive names: `password`, `password_hash`, `secret`, `token`, `session_token`, `api_key`, `private_key`, `salt`, `hash`. These are commented out with a note, not silently omitted — the developer sees them and makes a conscious decision.
- Fields with `Datetime` type automatically get `as Datetime.iso8601` as the default format selector (the most common sensible default).
- Fields with atom/union types automatically get `as Atom.string` as a suggested default (commented in if the type is a union).
- If the type has an obvious `id` field it is kept.
- The cursor is placed at the first field after generation so the developer can immediately start editing.

### "Generate input from type"

**Trigger:** Writing `Json.parse<User>(...)` where `User` has no `input` declaration.

**Diagnostic:** `error: Json.parse requires a type with an input declaration. User has no input declaration.  tip: Generate input UserInput -> User`

**Quick fixes offered:**
1. **Generate `input UserInput -> User`** — generates a full input scaffold
2. **Generate `input UserRegistrationInput -> User`** — if the call site is in a function named `handle_register` or similar, infer a more specific name from context

**Generated output:**

```bounce
// Generated — review field mappings, validators, and casing
input UserInput -> User(casing: Casing.to_camel) {
    fields {
        // id              <-- omitted: likely generated, not from input
        name as String.any          // TODO: add validators
        email as String.any         // TODO: consider Email.format
        // password_hash   <-- omitted: likely generated (hash at construction)
        // created_at      <-- omitted: likely generated
    }
}
```

Important details:
- Fields that look generated (`id`, `created_at`, `updated_at`, `inserted_at`, fields with type `UserId` or nominal ID types) are commented out with a note.
- Fields that look like they should be hashed (`password_hash`, `password_digest`) are commented out with a note suggesting the input accept a raw `password: String` instead.
- Every included field gets `as String.any` with a `// TODO` comment encouraging the developer to add real validators. This makes it a compile-time error to leave a field without an explicit validator — `String.any` is the explicit acknowledgement that "any value is fine here", not a silent default.
- `casing: Casing.to_camel` is the default (most common for REST APIs) but is clearly visible so the developer can change it.
- The cursor lands on the first `as String.any` so the developer can immediately start replacing validators.

### "Generate input for discriminated union"

**Trigger:** Writing `Json.parse<Shape>(...)` where `Shape` is a union type with no `input` declaration.

**Generated output:**

```bounce
// Generated — configure discriminator field and add validators per variant
input ShapeInput -> Shape(casing: Casing.to_camel) {
    discriminate {
        field: type_field           // TODO: replace with actual discriminator field name
        casing: Casing.to_snake
        // default: :unknown        // uncomment if external spec may add new variants
    }
    fields {
        :circle => {
            radius as Any           // TODO: add validators
        }
        :rectangle => {
            width as Any
            height as Any
        }
    }
}
```

Every variant is scaffolded. `Any` is used as a placeholder validator throughout — distinct from `String.any`, which is a typed validator for known string fields. In a discriminated union scaffold the LSP does not know the concrete type of each field, so `Any` serves as the most generic signal that a real validator must be chosen. The discriminator field is named `type_field` with a TODO comment — the LSP cannot know the actual discriminator field name from the type alone, and guessing silently would be worse than making the developer fill it in deliberately.

### "Add missing validator"

**Trigger:** A field in an `input` `fields` block has no `as Validator` clause.

**Diagnostic:** `error: field 'nickname' has no validator. Add 'as String.any' to explicitly accept any value, or add a validator.`

**Quick fix:** Inserts `as String.any` with a `// TODO` comment inline. Single keystroke to acknowledge the field.

This is the key enforcement: every field on an `input` type must have an explicit validator so a forgotten field is a compile-time error, not a runtime surprise. `String.any` is the escape hatch when you genuinely want no constraints, but it must be written explicitly.

### "Add missing variant to input"

**Trigger:** A union type gains a new variant but the `input` declaration's `fields` block does not cover it.

**Diagnostic:** `error: input ShapeInput does not cover variant :triangle of Shape. Add a fields entry for :triangle.`

**Quick fix:** Scaffolds the missing variant inline at the end of the `fields` block:

```bounce
:triangle => {
    base as Any         // TODO: add validators
    height as Any
}
```

### "Add missing variant to view"

**Trigger:** A union type gains a new variant but a view does not cover it.

**Diagnostic:** `error: view Shape.public does not cover variant :triangle. Add a field entry for :triangle.`

**Quick fix:** Scaffolds the missing variant:

```bounce
:triangle => {
    base
    height
}
```

---

## Pattern Match Exhaustiveness Quick Fixes

These are analogous to the serialization quick fixes but for `match` expressions. The same principle applies: if the compiler can detect the gap, the LSP can fill it.

### "Add missing match arm"

**Trigger:** A `match` expression does not cover all variants of a union.

**Diagnostic:** `error: match is not exhaustive. Missing variants: :triangle, :pentagon`

**Quick fixes:**
1. **Add missing arms** — scaffolds each missing arm with a `todo!()` body
2. **Add catch-all arm** — inserts `_ => todo!()` (offered but not recommended as default — the diagnostic message should note that a catch-all silences future exhaustiveness errors when new variants are added)

**Generated:**

```bounce
:triangle { base, height } => todo!()
:pentagon { sides } => todo!()
```

Fields are destructured in the scaffold so the developer can see what's available immediately. The `todo!()` body ensures the code compiles while making it obvious that the arm is not yet implemented.

---

## Type Construction Quick Fixes

### "Fill missing fields"

**Trigger:** Constructing a record type without all required fields.

**Diagnostic:** `error: User missing required fields: id, password_hash, created_at`

**Quick fix:** Inserts placeholders for all missing fields:

```bounce
User {
    name: input.name
    email: input.email
    id: todo!()             // inserted
    password_hash: todo!()  // inserted
    created_at: todo!()     // inserted
}
```

Cursor lands on the first `todo!()`.

### "Wrap in view application"

**Trigger:** Passing a raw type where a `View<T>` is expected (e.g. `Response.json(user)`).

**Quick fix options:**
1. If one view exists for this type: **Wrap with `User.public`** → `Response.json(User.public(user))`
2. If multiple views exist: **Choose view...** → picker showing all views for `User`
3. If no views exist: **Generate view `User.public`** → creates the view and wraps the call

---

## Spread / Composition Quick Fixes

### "Resolve duplicate key from spread"

**Trigger:** A view uses `...User.core` and also declares a field that exists in `User.core`, creating a duplicate key.

**Diagnostic:** `error: duplicate output key 'id' — defined in ...User.core and explicitly here`

**Quick fixes:**
1. **Remove explicit `id` field** (keep the spread version)
2. **Remove `...User.core` and list fields explicitly** (expand the spread inline, then remove the duplicate)

### "Expand spread inline"

**Trigger:** Developer wants to see/edit what a spread contributes.

**Code action (not an error):** "Expand `...User.core` inline" — replaces the spread with the explicit field list it contributes. Useful when you want to fork a composed view without affecting the original.

---

## Casing / Rename Quick Fixes

### "Add alias for casing mismatch"

**Trigger:** A view has `casing: Casing.to_camel` but one field has a name that doesn't transform correctly, or the developer wants to override a specific field's output name.

**Code action:** "Add alias for `display_name`" → inserts `displayName: display_name` replacing the bare `display_name` entry, so the developer can edit the output name.

---

## Import / Module Quick Fixes

### "Import missing type"

Standard — if `Email` is used but not imported, offer to add the import. Not unique to Bouncelang but must be present.

### "Import missing view/input"

**Trigger:** `User.public` is used but `User.public` is defined in another module and not imported.

**Quick fix:** Adds the import for the module containing `User.public`. The LSP knows the view's home module from the `view User.public = ...` declaration.

---

## Diagnostic Severity Guidelines

These should be consistent across the LSP so developers learn to distinguish them and build intuition for what requires immediate action versus what is advisory.

| Severity | When to use | Examples |
|---|---|---|
| **Error** | Code cannot be correct as written | Missing validator, wrong type passed to `Response.json`, non-exhaustive view over union |
| **Warning** | Code is valid but likely wrong | Unused view, `default` arm in discriminator where developer controls both sides |
| **Info** | Code is fine, but there's a better way | `as String.any` on a field that looks like an email (suggest `Email.format`) |
| **Hint** | Purely stylistic | `TODO` comments left in generated code |

---

## Error Message Quality Guidelines

Every diagnostic that involves a view or input should follow this template:

```
error: <what went wrong>
  found: <what the developer wrote>
  expected: <what was needed>
  tip: <one concrete next step>
  quick fix: <name of available quick fix> [Y/n]
```

Example:

```
error: Response.json requires a View<User>, got User
  found: Response.json(user)
  expected: Response.json(someView(user))
  tip: Define a view first — view User.public = { id, name, ... }
  quick fix: Generate view User.public [Y/n]
```

**Why this format matters:** A developer encountering `view` for the first time via an error message should understand the concept from the error alone, without reading documentation. The error message is the tutorial for the 90% case. If the error only says "type mismatch", it fails to teach. If it says what was found, what was expected, and what to do next, it succeeds.

---

## Completeness Principle

Every compile-time enforcement added to the language should have a corresponding quick fix that resolves it in one action. This is a design constraint, not a nice-to-have:

- Exhaustive validators required → "Add missing validator" quick fix
- Exhaustive union variant coverage in views → "Add missing variant to view" quick fix
- Exhaustive union variant coverage in inputs → "Add missing variant to input" quick fix
- `View<T>` required at serialization sites → "Wrap in view" / "Generate view" quick fix
- `input` declaration required for `Json.parse` → "Generate input" quick fix
- Non-exhaustive `match` → "Add missing arms" quick fix

If a compile error does not have a quick fix, the enforcement is incomplete from a DX perspective. New compiler errors should be shipped with their quick fix at the same time. This is the standard: the error and the fix are a unit.

---

## Future / Deferred

- **"Generate OpenAPI schema from views"** — a code action on a `view` declaration that generates a JSON Schema / OpenAPI fragment. Deferred until the view system stabilizes.
- **"Generate test for input"** — scaffolds a test that exercises the happy path and common validation failure cases for a given `input` declaration. Relates to the validator/matcher unification idea (validators and matchers for testing have significant overlap — `String.any` being one example — worth exploring as a unified concept later).
- **"Rename field across type + views + inputs"** — a refactor action that renames a field on a `type` and updates all `view` and `input` declarations that reference it. Requires the LSP to track the `-> Type` relationship in input declarations. High value, non-trivial to implement.
- **"Extract inline nested selection to named view"** — when a view has `roles { name }` inline, offer to extract it to `view Role.public = { name }` and replace inline with `roles: List<Role.public>`.
