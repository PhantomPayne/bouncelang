# Bouncelang Documentation Site — Website Plan

**Status:** Planning document — decisions captured, no code written yet
**Framework:** Astro + Starlight (base) + custom components
**Repo:** `PhantomPayne/bouncelang`, website lives in `website/`

This document is the authoritative architecture reference for the Bouncelang documentation site. It captures every significant decision — framework, content format, component API, URL structure, versioning, diagramming, playground architecture — so that when implementation begins there is a clear, actionable spec. No Astro project is scaffolded yet. No site code exists yet. This document exists so that when we start, we start with intent.

---

## 1. Repository Layout

The website lives in `website/` inside the existing monorepo. Spec files remain at the repo root. The site is a separate deployable unit; it does not need to be in a separate repo.

```
website/
  astro.config.mjs          # Starlight + Astro config, Shiki grammar registration
  package.json
  tsconfig.json
  public/                   # Static assets (favicon, OG images, fonts)
  src/
    components/             # Custom Astro/React components
      BouncePlayground.tsx  # In-browser WASM playground (all three display modes)
      DiagramEmbed.tsx      # Renders .excalidraw files as SVG
      CommitStep.tsx        # Step header for deep dives (step N of M)
      StepDiff.tsx          # Git-diff-style before/after view between steps
      StepNav.tsx           # Previous / next links at the bottom of each step page
    content/
      docs/                 # Starlight content root — all MDX pages
        index.mdx           # Landing page (marketing, not a docs page)
        learn/              # Narrative tutorial sequence
        reference/          # Exhaustive lookup reference
        examples/           # Deep dive project walkthroughs
      diagrams/             # Excalidraw source files (.excalidraw JSON)
      examples/             # .bounce source files used by the playground
    styles/                 # Global CSS, Starlight theme overrides
    layouts/                # Custom page layouts (landing page overrides Starlight default)
    bounce.tmLanguage.json  # Shiki grammar for .bounce syntax highlighting
  playground-worker/        # WASM compiler worker (separate package, built separately)
    Cargo.toml              # (or equivalent — depends on compiler build system)
    src/
      lib.rs                # Entry point: exposes compile(source: String) -> CompileResult
```

### Why examples `.bounce` files are separate from MDX prose

The `.bounce` source files under `src/content/examples/` are the canonical source of truth for every code example shown in the site. They are:

- Real Bouncelang source files that the compiler can parse and type-check.
- Referenced by MDX pages via component props — the prose never duplicates the code inline.
- Runnable in CI via `bounce check src/content/examples/**/*.bounce` once the compiler is stable.
- Independently testable: if a compiler update breaks an example, the CI catches it before the site ships.

Duplicating code between MDX prose and a source file means they drift. One is always wrong. This approach makes the example file the source of truth and makes the MDX prose a consumer.

---

## 2. Content Sections & URL Structure

The site has four main sections with distinct purposes, tones, and formats.

| Section | URL | Purpose | Tone |
|---|---|---|---|
| Landing | `/` | Marketing/pitch | Compelling, fast, visual |
| Learn | `/learn/` | Guided narrative tutorials | Friendly, direct, teaching |
| Reference | `/reference/` | Exhaustive lookup | Precise, structured, terse |
| Examples | `/examples/` | Step-by-step project walkthroughs | Practical, commit-by-commit |
| Playground | `/playground` | Full-screen interactive editor | Tool, no prose |

### `/` — Landing Page

Not a Starlight docs page. Uses a custom layout that overrides Starlight's sidebar/TOC shell.

The landing page must convey five things without the reader having to scroll far:

1. What Bouncelang is (one line)
2. Why it's different (fast, safe, DST built-in, WASM, great DX)
3. What code looks like (a single killer example — idiomatic, shows pattern matching or effects)
4. A run button (playground inline, `readonly={false}` so readers can edit immediately)
5. Two calls to action: **Install** and **Learn**

No feature matrix. No comparison table. No marketing word salad. Show don't tell.

### `/learn/` — Tutorials

A narrative, linear, opinionated sequence. Assumes the reader knows how to program but is new to Bouncelang.

Ordered sequence (final list TBD, this is the shape):

1. Install
2. First program
3. Types and data
4. Pattern matching
5. Effects and handlers
6. Error handling
7. Worlds and sandboxing
8. Concurrency
9. Testing with DST
10. Packages and imports

Each tutorial page must have:
- **Introduction paragraph** — what the reader will learn and why it matters
- **Inline prose with static code blocks** — explanation-first, code-second
- **One or more interactive `<Playground>` embeds** — reader edits and runs the code
- **"What you learned" summary** — bullet list, past tense ("you defined...", "you saw how...")
- **Link to the relevant Reference pages** — for readers who want exhaustive detail
- **`<StepNav>` links** — previous page, next page

Tone: friendly and direct. Like the Django tutorial. Not academic. Not a spec.

### `/reference/` — Reference Material

Exhaustive. Every keyword, every stdlib function, every CLI flag, every built-in type. Organized alphabetically within categories, not narratively.

Categories:

| Category | Contents |
|---|---|
| `syntax` | Keywords, operators, expression forms, statement forms |
| `types` | Primitive types, record syntax, union syntax, generics, nominal types |
| `effects` | Built-in effects, `with` clause, `pure fn`, handler wiring |
| `stdlib` | All standard library modules and functions |
| `cli` | `bounce` CLI commands and flags |

Each reference entry:
- Is a short, self-contained page
- Has a one-sentence description, the syntax signature, a minimal static code example
- Links to the Learn page that teaches the concept
- Links to at least one Example that uses it in context
- Has structured frontmatter (see Section 3) so it is machine-searchable

Long examples do not belong in Reference. If a concept requires more than ~15 lines to illustrate, it belongs in Learn or Examples. Reference is a lookup resource, not a tutorial.

### `/examples/` — Deep Dive Projects

Step-by-step project walkthroughs. Each example is a complete project built one commit at a time. See Section 6 for the detailed format.

Planned projects (exact list TBD):

| Project | Steps (est.) | Concepts |
|---|---|---|
| CLI tool | ~8 | First real program, effects, error handling |
| HTTP API | ~12 | Network effect, routing, JSON, worlds |
| DST-tested business logic | ~10 | DST, property tests, seed replay |
| Concurrent chat server | ~15 | Concurrency, channels, worlds, supervision |

Each project has an index page (`index.mdx`) and one page per step. The reader can follow entirely in the browser via the embedded playground, or clone and follow locally — the same `.bounce` source files are used both ways.

### `/playground` — Standalone Playground

Full-screen, distraction-free. See Section 4 for the full architecture.

The `/playground` route renders `<BouncePlayground>` in full-screen mode:
- Left panel: editor with syntax highlighting and LSP-lite (inline errors, completions)
- Right panel: output

Shareable URLs, preloaded examples dropdown. Covers people who want to explore without starting a tutorial.

---

## 3. MDX Content Conventions

These conventions are prescriptive. All content authors follow them.

### Frontmatter Schema

Every MDX page must have valid frontmatter. The required and optional fields depend on the section.

**All pages:**

```yaml
---
title: "Pattern Matching"
description: "One sentence for SEO and navigation previews."
---
```

**Learn pages (additionally):**

```yaml
---
section: learn
prev: "types-and-data"   # slug of previous page, omit on first page
next: "effects"          # slug of next page, omit on last page
---
```

**Reference pages (additionally):**

```yaml
---
section: reference
category: syntax | types | effects | stdlib | cli
keywords: [match, pattern, union, exhaustiveness]
---
```

**Example step pages (additionally):**

```yaml
---
section: examples
project: "todo-api"
step: 3
total_steps: 12
commit: "abc1234"   # git commit hash in the examples source directory
---
```

### Code Blocks — Two Kinds

**Static (read-only):** Standard fenced code block with the `bounce` language tag. Starlight renders it with Shiki syntax highlighting. No run button.

````mdx
```bounce
// Static — shown for reading, not running
type Shape =
    | :circle { radius: Float }
    | :rect { width: Float, height: Float }
```
````

**Interactive (runnable):** Use the `<Playground>` component, referencing a `.bounce` source file. Never inline source code directly into MDX for interactive examples.

```mdx
<Playground src="examples/pattern-match-shape.bounce" />
```

The `src` prop is relative to `src/content/`. The component loads the file contents, passes them to the WASM worker, and renders the editor + output panel.

**Rule:** If the reader should be able to run and edit it, use `<Playground>`. If it is for illustration only, use a fenced block. Never use `<Playground>` for code the reader should not run (e.g., pseudocode, intentionally broken examples shown to explain an error).

### Full Tutorial Page Example

```mdx
---
title: "Pattern Matching"
section: learn
description: "How to handle union types with match expressions in Bouncelang."
prev: "types-and-data"
next: "effects"
---

Pattern matching is how you handle union types in Bounce.
Every `match` expression must be exhaustive — the compiler rejects it if any variant is unhandled.

```bounce
// Static — shows the syntax without a run button
type Shape =
    | :circle { radius: Float }
    | :rect { width: Float, height: Float }

fn area(shape: Shape) -> Float {
    match shape {
        :circle { radius } -> Float.pi * radius * radius,
        :rect { width, height } -> width * height,
    }
}
```

Here's a live version you can edit:

<Playground src="examples/pattern-match-shape.bounce" />

Notice that removing one arm from the `match` causes a compile error. Try deleting the `:rect` arm.

<Diagram
  src="diagrams/pattern-match-exhaustiveness.excalidraw"
  alt="Exhaustiveness checking flow"
  caption="Figure 1: The compiler verifies all variants are covered before accepting a match expression"
/>

## What you learned

- Union types group multiple variants under one type name
- `match` expressions must handle every variant — the compiler enforces this
- Destructuring syntax inside match arms binds fields directly into scope
- [Pattern Matching Reference](/reference/syntax/match) — exhaustive syntax documentation

<StepNav prev="types-and-data" next="effects" />
```

### Diagrams

Diagrams are embedded with the `<Diagram>` component:

```mdx
<Diagram
  src="diagrams/effect-handler-stack.excalidraw"
  alt="Effect handler stack"
  caption="Figure 1: A world wires effects declared in code to named handler implementations"
/>
```

The `src` prop is relative to `src/content/`. The `alt` text is required (accessibility). `caption` is optional.

See Section 5 for the full diagramming system.

---

## 4. Playground Architecture — Tier 3: In-Browser WASM

The playground is the highest-value feature of the site. It is also the most complex. It is built after the core content is live, but the architecture is planned from the start so the content conventions (`<Playground src="..." />`) work without changes when the implementation arrives.

### Concept

The Bouncelang compiler itself compiles to WASM (`bounce-compiler.wasm`). It runs in a Web Worker so compilation does not block the UI thread. The playground sends source text to the worker via `postMessage`, receives either compiled WASM bytes or structured error messages, then executes the compiled WASM in a sandboxed runtime inside the browser.

```
[Editor UI]
    |
    | postMessage({ type: "compile", source: "..." })
    v
[Web Worker — bounce-compiler.wasm]
    |
    | postMessage({ type: "compiled", wasm: ArrayBuffer })
    | or postMessage({ type: "error", errors: [...] })
    v
[Output Panel]
    | — runs compiled WASM in sandboxed iframe or in-page WASM runtime
    | — captures stdout, stderr, panic messages
    | — displays output or error list
```

No server required. No API calls. The whole pipeline is client-side.

### `<BouncePlayground>` Component API

| Prop | Type | Default | Description |
|---|---|---|---|
| `src` | `string` | required | Path to a `.bounce` file in `src/content/examples/` |
| `id` | `string` | — | Stable identifier for shareable URL links |
| `height` | `string \| number` | `auto` | Height of the component |
| `readonly` | `boolean` | `false` | Disables editing — use for demo-only embeds |
| `showTests` | `boolean` | `false` | Enables test output panel (see below) |

### Three Display Modes

**1. Inline** — used in Learn pages and Example steps

Appears within the prose flow. Editor on left (or top on narrow screens), output on right (or bottom). Fixed height, scrollable editor. Reader can edit and re-run without leaving the page.

**2. Full-screen** — the `/playground` route

Maximum editor space. The full viewport. Top bar contains: example dropdown, share button, run button. No sidebar, no TOC, no Starlight chrome.

**3. Step diff** — used in Example deep dives

Shows only the *new* code added in the current step highlighted in green, with previous-step code grayed out. The reader sees exactly what changed without reading the entire file. Previous code is visible but not editable. New code is editable and runnable.

### `showTests` Mode

When `showTests={true}`:

- The playground automatically runs all `test fn` functions in the source on load and after each edit.
- Test results appear inline next to each function name: ✅ passed / ❌ failed with message.
- DST (deterministic simulation testing) test runs show the seed that was used, with a "replay" button that re-runs with the same seed.
- Assertion failures display the expected vs. actual values.

This mode is the key feature for the testing tutorial and for DST reference pages. It turns the playground from a REPL into a miniature test runner, making DST tangible without the reader needing a local install.

### Shareable URLs

| Scenario | URL format |
|---|---|
| Named example from dropdown | `/playground?example=pattern-match-shape` |
| User-edited short program | `/playground#<base64url-encoded-source>` |
| User-edited long program | Future: gist-backed URL shortener (post-launch) |

Selecting an example from the dropdown updates the URL. Editing the source updates the hash. Sharing the URL restores the editor state exactly.

### Example Source Files

`src/content/examples/*.bounce` (and `src/content/examples/<project>/source/step-NN/*.bounce` for deep dives) are:

- The canonical source — the MDX never contains duplicate code.
- Run by the WASM worker at page load to pre-populate the output panel.
- Checked by CI via `bounce check` once the compiler is stable enough.
- Version-controlled alongside the docs — no external "examples repo."

---

## 5. Diagramming System

### Format

Excalidraw's native `.excalidraw` JSON format. Chosen because:

- Open format — files are human-readable JSON, diff cleanly in pull requests.
- Editable in VS Code with the Excalidraw extension — zero friction for contributors.
- Editable at excalidraw.com and exported back to the repo.
- Can be converted to SVG programmatically at build time without the Excalidraw app.
- Supports dark mode natively.

### Storage

All diagram source files live at `src/content/diagrams/*.excalidraw`.

Naming convention: `<concept>-<detail>.excalidraw`. Examples:
- `effect-handler-stack.excalidraw`
- `world-handler-wiring.excalidraw`
- `pattern-match-exhaustiveness.excalidraw`
- `dst-simulation-loop.excalidraw`

### Rendering

Build-time SVG generation is preferred over runtime rendering. The `DiagramEmbed` component (or the Astro build pipeline) uses `excalidraw-to-svg` (npm) to convert each `.excalidraw` file to an SVG at build time. The SVG is shipped as a static asset. No runtime JS required to display diagrams.

For dark mode: generate both a light and a dark SVG variant at build time. The `DiagramEmbed` component uses a CSS `prefers-color-scheme` media query (or Starlight's theme class) to switch between them.

### Usage in MDX

```mdx
<Diagram
  src="diagrams/world-handler-wiring.excalidraw"
  alt="How worlds wire effects to handlers"
  caption="Figure 1: A world wires effects declared in code to named handler implementations"
/>
```

Props:

| Prop | Required | Description |
|---|---|---|
| `src` | yes | Path to `.excalidraw` file, relative to `src/content/` |
| `alt` | yes | Alt text for screen readers and broken-image fallback |
| `caption` | no | Figure caption rendered below the diagram |

---

## 6. Deep Dive Format — Commit-by-Commit Walkthroughs

The Examples section follows a formal content format. Every project follows this structure exactly.

### Project Directory Layout

```
src/content/docs/examples/todo-api/
  index.mdx                    # Overview: what we're building, prerequisites, final result link
  step-01-setup.mdx
  step-02-data-types.mdx
  step-03-effects.mdx
  ...
  step-12-dst-testing.mdx

src/content/examples/todo-api/source/
  step-01/
    package.bounce
    app.bounce
  step-02/
    package.bounce
    app.bounce
    models/
      user.bounce
  step-03/
    package.bounce
    app.bounce
    models/
      user.bounce
      todo.bounce
  ...
```

**Each step directory is the complete project state at that step — not a diff.** Every step is independently runnable in the playground. The `<StepDiff>` component computes the diff between `step-N-1` and `step-N` at render time.

This approach trades disk space (files are duplicated across steps) for simplicity and reliability. There is no complex tooling to reconstruct a step from cumulative patches. Any step can be opened standalone.

### Step Page Template

```mdx
---
title: "Step 3: Defining our data types"
section: examples
project: todo-api
step: 3
total_steps: 12
commit: "abc1234"
---

## What we're building in this step

In this step we define the core data types for our API: `Todo`, `User`, and `TodoList`.
By the end of this step the package compiles and passes all type checks.
No runtime behavior yet — just the data model.

## The code

<StepDiff project="todo-api" from={2} to={3} />

The `Todo` type is a record with four fields. The `id` field is a nominal `TodoId` rather than
a plain `Int` — this prevents us from accidentally passing a `UserId` where a `TodoId` is
expected, even though they are both integers at runtime.

<Playground
  src="examples/todo-api/source/step-03/models/todo.bounce"
  showTests={true}
/>

## Why we did it this way

We made `TodoId` a nominal type rather than a plain `Int` because...
We used a record rather than a class because...

## What you learned

- How to define records in Bounce
- The difference between structural and nominal types
- How `opaque type` controls what outside code can see
- [Nominal Types Reference](/reference/types/nominal)
- [Opaque Types Reference](/reference/types/opaque)

<StepNav prev="step-02-setup" next="step-04-effects" />
```

### Required Sections on Every Step Page

| Section | Purpose |
|---|---|
| **What we're building in this step** | Sets expectations. One paragraph. What exists at the end of this step that didn't exist at the start. |
| **The code** | `<StepDiff>` first, then prose explaining each changed piece. `<Playground>` for the main runnable file. |
| **Why we did it this way** | Design rationale. This is what separates a walkthrough from a code dump. |
| **What you learned** | Bullet list of concepts, each linking to the relevant Reference page. |
| `<StepNav>` | Previous and next links. Always last. |

### The `<StepDiff>` Component

Renders a git-diff-style side-by-side (or unified) view of what changed between two steps:

- Lines added: green background
- Lines removed: red background (with strikethrough)
- Unchanged lines: shown with reduced opacity to keep context visible
- File headers: show which file is being diffed

The diff is computed at build time from the two step source directories. The component ships as static HTML — no client-side JS required.

### The `<StepNav>` Component

Renders at the bottom of every step page:

```
← Step 2: Setup                    Step 4: Adding effects →
```

Includes the project name and a progress indicator ("Step 3 of 12").

---

## 7. Navigation & Discoverability

### Left Sidebar

Configured in `astro.config.mjs` using Starlight's sidebar API.

**Learn section** — ordered sequence, numbered:

```
Learn
  1. Install
  2. First program
  3. Types and data
  4. Pattern matching
  5. Effects and handlers
  6. Error handling
  7. Worlds and sandboxing
  8. Concurrency
  9. Testing with DST
  10. Packages and imports
```

**Reference section** — alphabetical within categories:

```
Reference
  Syntax
    Expressions
    Functions
    Match
    Types
    ...
  Types
    Bool
    Float
    Int
    List
    Map
    ...
  Effects
    FileSystem
    Network
    Random
    Time
    ...
  Stdlib
    std/json
    std/regex
    std/collections
    ...
  CLI
    bounce build
    bounce check
    bounce fmt
    bounce test
    ...
```

**Examples section** — listed by project:

```
Examples
  CLI Tool
    Overview
    Step 1: Setup
    ...
  HTTP API
    Overview
    Step 1: Setup
    ...
  ...
```

### Top Navigation

```
[Logo]  Learn  Reference  Examples  Playground  GitHub        [v0.1 ▾]
```

- The version selector (top right) lists available documentation versions.
- All links except Playground and GitHub are Starlight-managed docs links.
- Playground opens `/playground` — full-screen, outside the docs layout.
- GitHub opens `https://github.com/PhantomPayne/bouncelang`.

### On-Page TOC

Starlight's default right sidebar TOC is enabled on all Learn, Reference, and Example pages. Disabled on the Landing page (custom layout).

### Search

Starlight's built-in search via **Pagefind** is used. All pages are indexed. Reference pages receive higher search weighting via Pagefind's weighting API — when a reader searches for `match`, the Reference page for `match` should appear above a Learn page that mentions `match` in passing.

Additional search optimization:
- Reference page frontmatter `keywords` field is injected into the page as a hidden `<meta name="pagefind:filter">` element so Pagefind indexes them even if they don't appear in prose.
- Every page has a populated `description` frontmatter field (used as the search result snippet).

### Cross-Linking Conventions

These are mandatory, not optional:

| Context | Required links |
|---|---|
| Every Learn page that introduces a concept | Link to the relevant Reference page at the bottom of the page |
| Every Reference page | Link to at least one Learn page and one Example that uses it |
| Every Example step | Link to Reference pages for every syntax or stdlib feature introduced in that step |

Cross-links belong in the **"What you learned"** section of Learn pages and step pages. Reference pages have a dedicated **"See also"** footer.

---

## 8. Syntax Highlighting

Starlight uses **Shiki** for syntax highlighting. Shiki uses TextMate grammar files (`.tmLanguage.json`).

### Grammar File

Location: `src/bounce.tmLanguage.json`

Registered in `astro.config.mjs`:

```js
import { defineConfig } from 'astro/config';
import starlight from '@astrojs/starlight';

export default defineConfig({
  integrations: [
    starlight({
      // ...
      expressiveCode: {
        shikiConfig: {
          langs: [
            { path: './src/bounce.tmLanguage.json' }
          ],
        },
      },
    }),
  ],
});
```

The grammar language ID must be `bounce`. Code fences use ` ```bounce `.

### Minimum Token Coverage

The grammar must highlight at minimum:

| Token type | Examples | Scope |
|---|---|---|
| Keywords | `fn`, `let`, `mut`, `type`, `effect`, `handler`, `world`, `package`, `import`, `export`, `match`, `if`, `else`, `return`, `try`, `with`, `handle`, `spawn`, `pub`, `internal`, `opaque`, `nominal`, `test` | `keyword.control.bounce` |
| Atoms | `:some`, `:error`, `:true`, `:false`, `:ok`, and any `:identifier` | `constant.language.atom.bounce` |
| Type identifiers | `PascalCase` names (`User`, `TodoId`, `Float`, `List`) | `entity.name.type.bounce` |
| String literals | `"hello"` | `string.quoted.double.bounce` |
| String interpolation | `{expr}` inside strings | `meta.interpolation.bounce` |
| Comments | `// line comment` | `comment.line.double-slash.bounce` |
| Function definitions | name following `fn` keyword | `entity.name.function.bounce` |
| Effects in `with` clause | identifiers after `with` | `storage.type.effect.bounce` |
| Numbers | `42`, `3.14` | `constant.numeric.bounce` |

### Reference for Token List

The spec files at the repo root are the authoritative source for Bouncelang syntax. When writing the grammar, consult:

- `01-primitives.md` — literal syntax, operators
- `02-data-structures.md` — record and list syntax
- `04-effects-and-handlers.md` — `effect`, `handler`, `with`, `handle` syntax
- `atoms-and-unions.md` — atom syntax, union type syntax
- `generics-and-type-system.md` — type parameter syntax

---

## 9. Versioning Strategy

Docs must be versioned from day one in the architecture so that older language versions remain accessible when Bouncelang v0.2, v0.3, etc. ship.

### Recommended Approach: Versioned Path Prefix

Content lives under versioned paths:

```
/docs/v0.1/learn/...
/docs/v0.1/reference/...
/docs/latest/learn/...   →  redirects to current stable version
```

In the file system:

```
src/content/docs/
  v0.1/
    learn/
    reference/
    examples/
  v0.2/
    learn/
    reference/
    examples/
```

A top-right version dropdown in the nav lets readers switch between documented versions. Old versions display a banner: **"You are viewing documentation for Bouncelang v0.1. The current version is v0.2. [View latest →]"**

Old version pages disable the "Edit this page" link (read-only). Old versions are not re-indexed for search by default — Pagefind excludes them so search defaults to the latest version.

### Tradeoffs

| Approach | Pros | Cons |
|---|---|---|
| **Versioned directories** (recommended) | Simple, fully static, works natively with Starlight, each version is independently deployable | Content is duplicated on disk per version (acceptable for small docs) |
| **Git-tag-based generation** | No content duplication, history is the truth | Complex build pipeline, requires custom Starlight integration, slower cold builds |

**Decision:** Start with versioned directories. Revisit when there are 3+ live versions and disk duplication becomes a maintenance burden.

### Version Selector

Implemented as a Starlight custom component that replaces or augments the default header component. The selector reads the current URL path, extracts the version segment, and renders a `<select>` dropdown that navigates to the same page in the selected version (falling back to the section root if that specific page doesn't exist in the target version).

---

## 10. Inspirations & Design References

These sites influenced the decisions in this document. Future contributors should understand the intent behind each reference.

| Reference | What to steal |
|---|---|
| **djangoproject.com** | Tutorial structure: linear, opinionated, step-by-step, commit-by-commit. The tutorial builds a real thing. The reader ends with something that works. |
| **Rust Book (`doc.rust-lang.org/book`)** | Narrative progression. "By the end of this chapter you will..." sets expectation. Teaches the mental model, not just the syntax. |
| **Rust by Example (`doc.rust-lang.org/rust-by-example`)** | Code-first. Every concept has a live example. The example is the explanation. |
| **Elm Guide (`guide.elm-lang.org`)** | Opinionated narrative. Teaches the philosophy, not just the mechanics. Tells you what to think, not just what the syntax is. Small, tight, no filler. |
| **Storybook** | Examples/stories are the docs. `showTests` mode — test results inline next to the code. Play buttons inline. The component IS the documentation. |
| **excalidraw.com** | Diagram style: informal, clear, hand-drawn feel. Diagrams should communicate, not impress. |
| **Astro Starlight** | Foundation: sidebar navigation, Pagefind search, MDX, dark mode, Shiki highlighting. Use it, extend it, don't fight it. |
| **TypeScript Playground / Rust Playground** | Shareable URLs. Preloaded examples dropdown. State encoded in the URL hash. No account required to share. |

---

## Open Questions (to resolve before implementation begins)

These decisions are deferred — captured here so they are not forgotten.

| Question | Notes |
|---|---|
| Which WASM runtime runs compiled `.bounce` WASM in the browser? | Options: Wasmtime-compiled-to-JS, a custom thin runtime, or direct `WebAssembly` API. Depends on what the compiler's WASM output looks like. |
| LSP-lite implementation for the playground editor | Full LSP over a worker, or a simplified parser that provides inline errors only? Completions are a stretch goal. |
| Exact example projects list | Confirmed: CLI tool, HTTP API, DST testing, concurrent server. Exact steps TBD when compiler is stable enough to run them. |
| `excalidraw-to-svg` vs. runtime rendering | Prefer build-time SVG generation. Verify `excalidraw-to-svg` produces acceptable output for the diagram styles we use. |
| Version directory layout when v0.1 ships | Content may live flat (no version prefix) until v0.2 is needed. A redirect layer handles the transition. |
| Editor component choice for playground | CodeMirror 6 or Monaco. CodeMirror is lighter and more customizable for syntax grammars. Monaco has better out-of-the-box TypeScript support but is heavy. Lean toward CodeMirror. |
