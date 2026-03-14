# Bouncelang: Core Principles

This document is the **North Star** for Bouncelang's design. Every feature, syntax choice, error message, and library decision must be measured against the principles here. When writing a new spec section and asking "should we add a lint rule for this?", "how can the LSP help here?", "is this good for deterministic simulation testing (DST)?", or "is this the right DX?" — this document gives you a clear, actionable answer.

This document is about **why** decisions are made. The **what** — specific syntax, implemented features, and concrete examples — belongs in the individual spec files.

---

## 1. The One Question: Would a New User Understand This?

Bouncelang leans **practical and familiar over academic and powerful**. C-style over ML-style. This is the primary filter for every decision.

> **Could a competent developer who has never used Bouncelang understand what this does within 30 seconds?**

If the answer is no, the design needs rethinking — regardless of how elegant or powerful it is.

### What This Means in Practice

**Prefer named arguments over positional when order is non-obvious.** Call sites should be self-documenting. A reader should not need to look up a function signature to understand what each argument means.

**Prefer familiar control flow over functional composition chains.** When a developer from any mainstream language can read Bouncelang code and immediately understand the intent, that is a success. When they need to learn a new abstraction before they can read it, that is a cost that must be justified.

**Prefer consistent syntax even if a special case would be slightly more powerful.** Every inconsistency is a new rule to memorize. Bouncelang accepts a small power reduction to avoid a large learning cost. The language should feel like it has one grammar, not many.

**Prefer explicit over implicit at the boundaries that matter.** Inside a function, inference is fine. At a public API boundary, the constraints that callers need to know should be visible without running the compiler.

### The Corollary

A feature that is powerful but that nobody uses because nobody understands it has **negative value**. It adds complexity to the language, creates confusion, and makes error messages harder to write. If only experts understand it, only experts will use it correctly — and everyone else will misuse it or avoid the language.

This principle rules out implicit coercions, surprise inference across module boundaries, magic method names, and "unless you know the trick" syntax.

---

## 2. DX Is a First-Class Feature

**Developer experience** is not polish applied after the language is designed — it is a design constraint from the start. The LSP, formatter, error messages, and tooling are part of the language spec, not afterthoughts.

When specifying any feature, the question is not just "is the semantics correct?" It is: "what does it feel like to use this?"

### 2a. Errors Should Teach, Not Just Reject

A compiler error is a teaching moment. Every error must:

1. Say exactly what went wrong
2. Show the context — what was written and what was expected
3. If possible, trace where the problem originated in the call graph
4. If possible, suggest a concrete fix

An error that says "type mismatch" with no context has failed. An error that shows the expected shape, the actual shape, and why they differ has succeeded. The standard for error quality is: a developer encountering this error for the first time should be able to fix it without searching documentation.

### 2b. The LSP Is Part of the Design

Every significant language feature must have a corresponding **LSP story**. When specifying a new feature, ask:

- What **completions** does this enable?
- What **inlay hints** help users understand what the compiler knows?
- What **diagnostics** catch common mistakes?
- What **code actions** (quick fixes) can we offer?

This is not aspirational. It is a checklist. See §7 for the formal version.

The LSP's job is to make the compiler's knowledge visible to the developer. Inferred types, inferred effects, inferred constraints — all of this should surface as inlay hints so that developers can learn the system by writing code, not by reading documentation.

### 2c. The Formatter Is Part of the Language

One canonical format. No configuration. No style debates. Formatting rules are specified alongside syntax — they are not a separate tool bolted on afterward.

The formatter uses **structural expansion rules**: either a construct fits on one line, or all its elements expand. This makes reformatting predictable and rename-safe — adding one character to a name never cascades into unexpected reformats.

See `formatter-principles.md` for the full specification.

### 2d. `bounce upgrade` — Migrations as a First-Class Citizen

Breaking changes happen. The tooling handles them. Codemods, not apologies.

When Bouncelang or the standard library introduces a breaking change, `bounce upgrade` ships a codemod that mechanically transforms existing code. Users run one command, review the diff, and continue. The alternative — reading migration guides and manually updating hundreds of call sites — is not acceptable.

Versioned stdlib modules allow gradual migration. The old API stays available until explicitly removed, giving users a migration window rather than a hard cutover.

### 2e. Lint Rules and Refactor Targets Are User-Extensible

Teams have domain-specific conventions that generic tooling can't anticipate. Bouncelang's LSP architecture allows users to define **custom lint rules** and **refactor targets** in Bouncelang itself. These are discovered by the LSP automatically and surface through the same machinery as built-in rules — same diagnostic format, same code action UX, same configuration system.

**Domain-specific correctness rules deserve first-class tooling support**, not third-party linter plugins with degraded IDE integration.

The lint rule system needs its own spec file. This principle establishes it as a core design requirement, not an extension mechanism.

---

## 3. Consistency Is Kindness

**Inconsistency is a hidden tax** on every user of the language. When two things look similar, they should behave similarly. When two things behave differently, they should look different.

Consistency is not about aesthetics. It is about reducing the cognitive overhead of learning and using the language. A user who understands one part of the language should be able to predict how another part works.

### One Mechanism per Concept

For any given problem, Bouncelang provides one idiomatic solution. There is one way to call functions, one way to handle errors, one way to define data. Multiple mechanisms for the same concept are not "flexibility" — they are decisions every user must make, every time, with no clear guidance.

When a developer learns one mechanism, that knowledge should transfer directly to related contexts. If it doesn't, that is a design failure.

### One Naming Convention — Always

| Category | Convention |
|---|---|
| Atoms | `:lowercase` |
| Types | `PascalCase` |
| Functions & variables | `snake_case` |
| Effects | `PascalCase` |

The formatter enforces these. There are no exceptions, no configuration options.

### One Format

The formatter decides. Not the team lead. Not the style guide. Not a linter config. If two developers format the same file, they get identical output, always.

### Learning Transfer

The payoff of syntactic consistency is learning speed. When a user understands how pattern matching works on a union, that understanding should apply equally to matching on an `Option`, a `Result`, a `select` result, or any other context that uses the same shape. Each new context is free — the user already knows the pattern.

---

## 4. The Type System Serves the Programmer

Types catch bugs and communicate intent. They are not an obstacle course. The type system works for the developer, not the other way around.

### Inference by Default, Explicit at Boundaries

The compiler should infer what it can. Type annotations are for the programmer's benefit — as documentation and as API contracts — not as tax paid to the compiler.

The exception: at public API boundaries, significant constraints must be explicit. Callers shouldn't need to run the compiler to understand what a function requires and what it can fail with.

### Structural Typing for Data, Nominal for Identity

A record is defined by its shape. If two records have the same fields, they are interchangeable. This makes composition natural and eliminates ceremony — passing a value to a function should not require explicitly declaring compatibility.

Nominal types exist for when two things with the same shape mean semantically different things. A user ID and an order ID may both be integers, but they are not interchangeable. Nominality expresses that distinction at the type level.

### Tags for Refinement, Not Ceremony

Tags communicate facts about values — compile-time evidence that follows a value and enables specific operations — without requiring new wrapper types. The cost of expressing a refinement should match its conceptual weight. A small fact should require a small annotation, not a new type definition.

The unified truthiness model (`Option<T>` as a tagged union using `:true`/`:false`) is the primary instance of this principle. Booleans, optional values, and tagged results all use the same control flow — no special cases, no different syntax per type.

### Errors Are Effects, Not Return Types

Errors propagate structurally through the call graph. Functions that can fail have clean signatures that say what they return on success; the failure modes are tracked by the type system without cluttering the return type. The single handle point is explicit and exhaustive.

### Exhaustiveness Is a Feature

The compiler checks pattern coverage, not the developer. Adding a new variant to a union immediately surfaces every place in the codebase that needs updating, as a compile error. This is what makes refactoring trustworthy.

---

## 5. Safe by Default, Powerful When Needed

Bouncelang's defaults eliminate entire categories of bugs. Users opt into complexity when they need it, not by default.

### 5a. Value Semantics by Default

There is no shared mutable state by accident. Records are values. Copying is safe and cheap. Mutation is explicit — either local rebinding (which does not affect other references) or scoped shared state primitives for concurrent code.

Updating nested data should be ergonomic without mutation. The language provides deep update syntax so that "change one field deep in a structure" does not require imperative step-by-step reassignment.

A functional standard library — `map`, `filter`, `fold`, `group_by`, and similar operations — makes transformations the path of least resistance, so developers don't reach for mutation because it's easier.

### 5b. Effect System as Capability Control

Functions declare what they do. The execution environment decides what is allowed. Sandboxing is **structural** — the type system enforces it, not runtime checks.

A function that requires a capability cannot be called in an environment that doesn't provide it. This is checked at compile time. There is no "accidentally calling a network function in a sandboxed context" class of bug.

### 5c. Panics for Bugs, Raise for Expected Failures

Two distinct failure modes with a hard boundary:

| Mechanism | Meaning | Catchable? |
|---|---|---|
| Expected failure | The caller should anticipate and handle this | Yes |
| Programming bug | This should never happen in correct code | No |

The language makes this distinction structural and statically visible. Callers know which category a failure belongs to from the type system alone. Users never confuse the two, because the language does not let them look the same.

### 5d. No Implicit Coercions

No implicit numeric widening. No implicit string conversion. No implicit null. Every conversion is explicit and visible in the source code.

This makes code **predictable and searchable**. A developer can search for conversion calls and find every place one happens. Implicit coercions are invisible in the source and create surprising behavior — Bouncelang eliminates that category of bug entirely.

---

## 6. Deterministic Simulation Testing (DST) Is a Design Constraint

DST is not a testing library bolted on. It is a **design constraint that shapes language features**. When specifying any feature that involves time, randomness, concurrency, or I/O, ask:

> **"Can a user write a deterministic simulation test for this?"**

If the answer is "they can't" or "it would require mock objects and prayer", the design needs rethinking.

### What Makes DST Possible

All effects — time, network, filesystem, randomness, concurrency — are replaceable via the world/handler system. A test can substitute deterministic implementations for all of them. This is not a testing convenience; it is a direct consequence of the effect system design. DST is possible because effects are first-class and swappable by design.

### Virtual Time

Real time makes tests slow, flaky, and non-deterministic. The time effect must be replaceable with a virtual clock that can be advanced programmatically. Code that relies on real wall-clock time is untestable — the language makes real time an explicit capability so it can be replaced.

### Deterministic Scheduling

Given the same seed, concurrent code must interleave the same way every time. Concurrent bugs must become **reproducible findings**, not "it happens sometimes." A flaky test is a bug with an unknown seed; the tooling should make any failure replayable.

### Fault Injection

The test runner must be able to inject failures at effect boundaries — fail a network request on the nth call, close a channel after a certain number of sends, make a filesystem write fail. This tests error handling paths that are difficult or impossible to exercise with real infrastructure.

### The DST Test

When designing a feature, ask: "if a user wrote a deterministic simulation test for the code that uses this feature, what would that look like?" Apply this to any feature involving time, concurrency, I/O, or randomness.

If the test would require mocking frameworks, monkeypatching, or "just don't test that part", the feature design is resisting testability. Redesign so the world/handler system can intercept it.

---

## 7. The LSP Checklist

Every language feature, when added to the spec, must answer all of these questions. This is a **formal checklist**, not a suggestion. A spec section that does not address these is incomplete.

| Checklist Item | Question to Answer |
|---|---|
| **Completions** | What does the LSP suggest in the context of this feature? Be specific about what completions are available and what information is shown with each. |
| **Inlay hints** | What types, effects, or constraints does the compiler know that the user didn't write? Where should those be surfaced? |
| **Diagnostics** | What mistakes do users make with this feature? What can the compiler detect statically and report as a warning or error? |
| **Quick fixes / code actions** | What mechanical transformations can the LSP offer when a diagnostic fires or the user requests a refactor? |
| **Hover / go-to-definition** | What information is most useful on hover for this construct? What should go-to-definition navigate to? |
| **Semantic highlighting** | Does this construct benefit from distinct visual treatment to help users identify it at a glance? |

---

## 8. The Lint Rule Question

When a pattern is problematic but not a type error, it should be a **lint rule**. When specifying a feature, ask:

> **"What incorrect-but-legal things can a user do with this that we should warn about?"**

Built-in lint rules are part of the language. User-defined lint rules are first-class. Both surface through the same LSP machinery — same diagnostic format, same severity levels, same quick-fix integration. The lint rule catalogue belongs in its own spec file; this document establishes the decision framework for when to add one.

### When to Add What

| Mechanism | Threshold |
|---|---|
| **Compiler error** | Statically provably wrong — no legitimate use exists |
| **Lint error** (default on, blocks build) | Almost certainly a mistake; legitimate uses are extremely rare |
| **Lint warning** (default on, advisory) | Often a mistake, but legitimate uses exist |
| **Lint off by default** | Style preference; teams may want it, but it's not universal |

### User-Defined Lint Rules

Teams define custom rules in Bouncelang itself. The LSP discovers them automatically. A custom rule has the same format, severity levels, and quick-fix capability as a built-in rule.

**Domain-specific correctness rules deserve first-class tooling support**, not third-party linter plugins with degraded IDE integration. If a team wants to enforce an architectural boundary or a naming convention, they should be able to express that in the same language they write their application in, and have it enforced with the same quality of tooling.

---

## 9. Practical Tradeoff Guide

Design decisions often involve genuine tradeoffs. When two good principles conflict, use this priority order:

### Priority Order

| Priority | Principle | Meaning |
|---|---|---|
| 1 | **Safety** | Does this prevent a class of bugs? A feature that makes unsafe code impossible is worth almost any other cost. |
| 2 | **Understandability** | Can a new user read this and understand it within 30 seconds? Complexity that doesn't pay for itself in safety must be justified by understandability. |
| 3 | **DX / Ergonomics** | Is this pleasant to write? Boilerplate that serves no safety or clarity purpose should be eliminated. |
| 4 | **Power / Expressiveness** | Can advanced users do more with this? Power that conflicts with safety or understandability requires a strong case. |
| 5 | **Performance** | Is this fast? Performance is never ignored, but it is the last thing to sacrifice other principles for. The compiler and runtime handle performance; the language spec does not compromise safety for it. |

### Anti-Priorities

These are explicitly **not** goals. When a proposal is justified by one of these, it should be rejected or redesigned:

| Anti-priority | Why it's not a goal |
|---|---|
| **Minimizing keystrokes** | Saving 3 keystrokes at the cost of clarity costs every reader of that code. Readers outnumber writers. |
| **Academic purity** | The type system serves the programmer, not the other way around. A theoretically elegant system that is confusing in practice is a failure. |
| **Feature parity** | We add features because they solve real problems, not to check boxes relative to other languages. |
| **Configuration options** | Every configuration option is a decision the user must make. We make decisions for them when we can. Fewer options = fewer disagreements = faster teams. |

### How to Apply the Priority Order

When evaluating a proposed feature or design decision, work through the priorities in order. A proposal that significantly improves safety can justify a small ergonomics cost. A proposal that only improves expressiveness cannot justify a safety regression or a meaningful understandability cost.

If two proposals score equally on the highest relevant priority, move to the next. The goal is not to maximize every dimension simultaneously — it is to make principled tradeoffs with a clear rationale that can be communicated to the team.
