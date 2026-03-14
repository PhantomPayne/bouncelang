# Modules, Imports, and WASM Linking

**Status:** Draft — evolved from design review discussion
**Related:**
- [methods-and-packages.md](file:///Users/tom/projects/bouncelang/docs/spec/methods-and-packages.md) — UFCS, companions, `package.bounce` exports
- [worlds-and-handlers.md](file:///Users/tom/projects/bouncelang/docs/spec/worlds-and-handlers.md) — worlds, handlers, config

---

## 1. Project Structure

### Library

```bash
bounce init my-lib lib
```

```
my-lib/
  package.bounce
  lib.bounce
  lib.test.bounce
```

### Application

```bash
bounce init my-app app
```

```
my-app/
  package.bounce         # includes world declarations
  app.bounce
  app.test.bounce
```

### Grown Application

```
my-api/
  package.bounce
  app.bounce
  app.test.bounce
  models/
    package.bounce       # sub-module barrel
    user.bounce
    user.test.bounce
    transaction.bounce
    transaction.test.bounce
  services/
    package.bounce
    user_service.bounce
    user_service.test.bounce
  controllers/
    package.bounce
    routes.bounce
    routes.test.bounce
```

---

## 2. `package.bounce` — Unified Format

Every `package.bounce` uses the same format. The root one has a `package {}` block and optional `world` declarations. Sub-module barrels have only exports and optional lint overrides.

### Root `package.bounce`

```bounce
package {
    name: my-api
    version: 0.1.0
    license: MIT

    deps {
        std: "1.2"
        http-client: "^2.0"
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

export opaque type User        with { display, activate }
export opaque type Transaction with { display, total }
export fn handle_request

world server {
    config {
        database_url: String
        log_level: :debug | :info | :warn | :error = :info
    }
    entry handle_request
    handle Network  with WasiHttp
    handle Database with Postgres(config.database_url)
    handle Logger   with StdoutLogger(level: config.log_level)
}

world test {
    entry test_runner
    handle Database with InMemoryDb
    handle Network  with MockNetwork
    handle Logger   with QuietLogger
}
```

### Sub-Module `package.bounce`

```bounce
// models/package.bounce — just exports + optional lint overrides
export opaque type User        with { display, validate }
export opaque type Transaction with { total, line_items }

lints {
    std/boundaries: { deny_from: [controllers, db] }
    std/no-effects: :error
}
```

No `package {}` block = sub-module barrel. The parser is the same; behavior differs based on whether `package {}` is present.

---

## 3. Import Resolution

### Syntax

No quotes. Package and module names are bare identifiers:

```bounce
import { User, Transaction } from models
import { fetch } from http-client
import { sort } from std/collections
import { display as pretty_display } from pretty-print
```

### Resolution Algorithm

Given `import { X } from name`:

1. **Is `name` in `deps` or `dev_deps`?** → external package
2. **Is there a sibling/child directory `name/` with a `package.bounce`?** → internal sub-module
3. **Is `name` the `std` prefix?** → stdlib (always available)
4. **None of the above?** → compile error: *"unknown module `name`"*

```bounce
package {
    deps {
        std: "1.2"
        http-client: "^2.0"          // external
    }
}

// In code:
import { User } from models           // not in deps → internal sub-module
import { fetch } from http-client      // in deps → external package
import { sort } from std/collections   // std prefix → stdlib
```

### Name Collisions

If a local sub-module has the same name as a dependency, the compiler errors:

```
compile error: name collision — local module `http-client`
shadows dependency `http-client`. Rename the local module.
```

### Circular Imports

- **Within a package:** allowed. Sub-modules can reference each other freely.
- **Between packages:** forbidden. The dependency graph must be acyclic. The compiler errors on circular package dependencies.

---

## 4. Standard Library

### Prelude — Always Available

These types and values are in scope in every file without import:

```bounce
// Primitive types
Int, Float, String, Bool, Byte, Never

// Core collections
List, Map, Set

// Core enums
Option    // :some { value: T }, :none

// Unit
()

// Note: there is no Result type in the prelude.
// Errors use the Raise<E> effect — see error-handling.md.
```

### Stdlib — Explicit Import

Everything beyond the prelude requires an explicit import:

```bounce
import { sort, reverse, group_by } from std/collections
import { join_path, extension } from std/fs
import { format, parse } from std/time
import { to_json, from_json } from std/json
```

### Stdlib Is a Versioned Dependency

Stdlib is declared in `deps` with a pinned version:

```bounce
package {
    deps {
        std: "1.2"
    }
}
```

**Why explicit versioning matters:**

1. **Migration support.** `bounce upgrade` handles stdlib the same as any package — detects breaking changes, runs codemods, updates the pin.
2. **No surprise breakage.** Upgrading the compiler doesn't silently change stdlib behavior.
3. **Bundled, not linked.** Stdlib source is compiled and tree-shaken into the output WASM. No separate stdlib component at runtime.
4. **No diamond problem.** Package A on `std: "1.2"` and package B on `std: "1.4"` compose fine — each bundles its own stdlib.

**Compiler compatibility:** The compiler declares a supported stdlib range. Compiler 2.3 supports `std: "1.0"` through `std: "1.5"`. Pinning outside this range is a compile error.

```bash
bounce upgrade
# Output:
# std 1.2 → 1.3
#   - List.sort now requires Ord interface
#   - format_date renamed to format_datetime
#   Running migration... 3 files updated.
# http-client 2.0 → 2.1
#   No breaking changes.
```

---

## 5. Test Files

### Convention

Tests live in sibling `.test.bounce` files:

```
models/
  user.bounce              # implementation
  user.test.bounce         # tests for user
```

### Rules

- `.test.bounce` files can import `dev_deps` — regular files cannot
- `.test.bounce` files can access `test`-tier exports from `package.bounce`
- Functions marked `test fn` are only callable from `.test.bounce` files
- Lint rules can have different severity in test context: `{ default: :error, test: :off }`

```bounce
// user.test.bounce
import { mock_user } from test-utils    // dev_dep — only works in .test.bounce

test fn user_display_shows_name() {
    let user = mock_user(name: "Alice")
    assert_eq(user.display(), "Alice <alice@example.com>")
}
```

---

## 6. Visibility Layers

Three boundaries, from innermost to outermost:

```
File → Sub-Module → Package
```

| Keyword | Visible in same file | Visible in sub-module | Visible to package consumers |
|---|---|---|---|
| *(unmarked)* | ✅ | ❌ | ❌ |
| `internal` | ✅ | ✅ | ❌ |
| `pub` | ✅ | ✅ | ✅ (if listed in root `package.bounce`) |

Sub-module `package.bounce` controls what crosses the sub-module boundary:

```bounce
// models/package.bounce
export opaque type User with { display }      // visible to other sub-modules
// internal helper functions in user.bounce stay hidden
```

Root `package.bounce` controls what crosses the package boundary:

```bounce
// package.bounce
export opaque type User with { display }      // visible to external consumers
```

A `pub` declaration must be listed in the appropriate `package.bounce` to actually be visible. Marking something `pub` without listing it is a lint warning: *"`pub` function `helper` is not listed in any `package.bounce` — did you mean `internal`?"*

---

## 7. Package Namespacing

Package names support optional org namespacing:

```bounce
deps {
    http-client: "^2.0"            // flat — community package
    bouncelang/router: "^1.0"      // namespaced — org-scoped
}
```

```bounce
import { Router } from bouncelang/router
```

The registry enforces uniqueness:
- Flat names: first-come-first-served
- Namespaced names: org ownership verification required

---

## 8. Workspaces

Multiple packages in one repository:

```
my-project/
  workspace.bounce
  packages/
    api/
      package.bounce
    shared/
      package.bounce
    cli/
      package.bounce
```

```bounce
// workspace.bounce
workspace {
    packages {
        api: ./packages/api
        shared: ./packages/shared
        cli: ./packages/cli
    }
}
```

Workspace packages reference each other via the `workspace` keyword:

```bounce
// packages/api/package.bounce
package {
    deps {
        std: "1.2"
        shared: workspace       // sibling package in workspace
        http: "^1.0"
    }
}
```

```bash
bounce build --workspace         # builds all packages
bounce test --workspace          # tests all packages
bounce check --workspace         # type-checks all packages
```

---

## 9. WASM Compilation Model

### Packages = WASM Components

Each package compiles to one WASM Component. Sub-modules are flattened — they're compile-time organization only.

```
Source                           WASM Output
──────                          ────────────
my-api/                      →  my-api.wasm (one component)
  models/                        all internal code flattened
  services/
  controllers/

http-client/ (dep)           →  http-client.wasm (separate component)
pg/ (dep)                    →  pg.wasm (separate component)
std/ (bundled)               →  compiled into my-api.wasm (tree-shaken)
```

### Export Mapping

`package.bounce` exports map to WASM Component exports via WIT:

```bounce
// package.bounce
export fn handle_request
export opaque type User with { display }
```

Generates:
```wit
// my-api.wit (auto-generated)
package my-api:api;

interface api {
    record user { name: string, email: string }
    handle-request: func(req: request) -> response;
    display: func(self: user) -> string;
}
```

### Effect Mapping

Effects map to WIT imports:

```bounce
// code uses Network and Database effects
fn handle_request(req: Request) -> Response {
    let data = Network.get("/upstream")
    let users = Database.query("SELECT ...", [])
    // ...
}
```

Generates:
```wit
// Required imports
interface network {
    get: func(url: string) -> list<u8>;
}

interface database {
    query: func(sql: string, params: list<value>) -> list<row>;
}
```

### World = Composition Instructions

The `world` declaration tells the linker how to wire imports to exports:

```bounce
world server {
    handle Network  with WasiHttp       // import "network" → wasi:http
    handle Database with Postgres(...)  // import "database" → pg component
}
```

The compiler + linker:
1. Compiles `my-api` to a component with `network` and `database` imports
2. Resolves `WasiHttp` → `wasi:http/outgoing-handler` interface
3. Resolves `Postgres` → `pg` component's export
4. Composes the final linked component using `wasm-tools compose`
