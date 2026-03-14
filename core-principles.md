# Bouncelang: Core Principles & Ergonomics

This document outlines the foundational axioms and design philosophy of Bouncelang. It serves as a North Star for language evolution, standard library development, and ecosystem tooling.

## 1. Data Manipulation: Pure but Pragmatic

Bouncelang relies on persistent data structures and value semantics to guarantee safety, predictability, and fearless concurrency. However, strict immutability must not compromise developer experience (DX).

*   **Deep Spread Syntax over Mutation:** 
    Updating deeply nested records should be expressive and concise. Bouncelang favors deep spread syntax (e.g., `{ ...state, player->stats->hp: new_hp }`) to avoid the boilerplate of manual deep cloning while maintaining value semantics.
*   **A "Lodash-like" Standard Library:**
    Instead of relying on imperative loops and manual accumulation (`let mut list = []; list.push(x)`), Bouncelang provides a highly capable, ergonomic standard library of collection operations (e.g., `fold`, `filter_map`, `chunk`, `group_by`). This eliminates the need for "local mutability" blocks by making functional transformations the path of least resistance.

## 2. Universal Control Flow

Control flow should be expressive, orthogonal, and free of redundant concepts.

*   **Expressions Everywhere:**
    Constructs like `if` / `else` are expressions that return values, enabling declarative bindings: `let x = if cond { a } else { b }`.
*   **The Unified Truthiness Pattern (`:true` / `:false`):**
    Bouncelang minimizes special-case compiler magic. Common types like `Option` and `Result` align structurally with boolean concepts by using `:true` and `:false` variants. 
    *   `Option<T>` is effectively `:true { value: T } | :false`.
    *   `Result<T, E>` is effectively `:true { value: T } | :false { error: E }`.
    This unified shape allows operators like `if (x)` and optional chaining `?` to work uniformly across booleans, optional values, and results.
*   **Pragmatic Early Exits:**
    While expression-oriented, Bouncelang acknowledges that deeply nested `else` blocks damage readability (the "Arrow Anti-Pattern"). An explicit `return` keyword is provided for early exits from functions.

## 3. The Standard Library: The Ecosystem Vocabulary

The standard library is not just a collection of utilities; it defines the common language of the ecosystem to prevent fragmentation.

*   **Defining the Interfaces:**
    Core concepts like `Request`, `Response`, `Url`, `Datetime`, and `Duration` belong in the standard library. By defining these interfaces structurally, Bouncelang ensures that 3rd-party libraries (e.g., different web servers or database drivers) can interoperate seamlessly.
*   **Batteries Included for Common Needs:**
    Fundamental tools like `std/json` and `std/regex` are built-in. Developers should not need to debate or switch between multiple fundamental libraries for basic application needs.

## 4. Worlds: Declarative Execution Environments

Bouncelang's `world` concept extends beyond dependency injection; it defines the complete execution environment, capability sandbox, and interface.

*   **The `cli` World Example:**
    A `cli` world automatically handles runtime concerns like parsing `config` fields from `argv`, generating `--help` text, and enforcing sandbox constraints (e.g., restricting file system access to a specific target directory). The code focuses purely on logic, while the world defines how that code is hosted and constrained.
*   **Containerless Composition:**
    Worlds natively support emerging patterns like WASM Component Linking, enabling sandboxed deployment to CLIs, servers, or serverless runtimes without container overhead.
