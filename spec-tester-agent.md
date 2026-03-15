# Spec Tester Agent — Plan & Prompt

This document is a **self-contained prompt and plan** for running an AI agent whose job is to
exercise the Bouncelang spec by writing real programs in Bounce syntax, recording every moment of
uncertainty or friction, and producing a structured findings summary that feeds into the
**Spec Recommender Agent** (see `spec-recommender-agent.md`).

Hand this document to an AI agent (e.g., Claude or GPT-4-class) together with the full content of
every spec file listed in §1. The agent follows the loop in §3 for three rounds, then writes the
outputs in §5.

---

## 1. Inputs

Provide the following files verbatim to the agent at the start of the session:

| File | Role |
|---|---|
| `core-principles.md` | **North Star** — the priority order (Safety > Understandability > DX > Power > Performance) and the definitions of what makes a good language decision. |
| `01-primitives.md` | Primitive types, literals, operators, numeric rules. |
| `02-data-structures.md` | Records, opaque types, tags, unions, collections, tuples. |
| `04-effects-and-handlers.md` | Effects system, `Raise`, `Panic`, handlers, test sandboxing. |
| `07-concurrency.md` | `Sequence<T>`, scoped tasks, `State<T>`, channels, `select`. |
| `08-serialization-boundaries.md` | `view`, `input`, format selectors, validators. |
| `atoms-and-unions.md` | Atoms, discriminated unions, exhaustive pattern matching. |
| `error-handling.md` | `error` keyword, `try` blocks, error grouping, pipeline recovery. |
| `formatter-principles.md` | Canonical formatter rules — structural expansion, leading operators, trailing commas. |
| `generics-and-type-system.md` | Three type tiers, Hindley-Milner, interfaces, value semantics. |
| `methods-and-packages.md` | UFCS, `package.bounce`, companion functions, operator overloading. |
| `modules-and-imports.md` | Import resolution, prelude, workspaces, test files. |
| `worlds-and-handlers.md` | `world` declarations, named handlers, config block, entry points. |

**What the agent is NOT doing:**

The tester agent does not implement a Bounce compiler or runtime. It writes programs in Bounce
syntax by reasoning about what the spec says, exactly as a human developer would when writing code
for a language they have only read the docs for — never run. Every moment of "I'm not sure if
this is right" is a finding.

---

## 2. Definitions

| Term | Meaning |
|---|---|
| **Guess** | The agent wrote code in a way it could not fully justify from the spec. The spec was silent or ambiguous on the specific situation. |
| **Friction point** | The spec describes how to do something, but the required syntax or pattern is significantly more verbose, awkward, or surprising than it should be for the conceptual weight of the operation. |
| **Spec gap** | The agent needed to express something that the spec does not cover. The feature may exist conceptually but has no defined syntax. |
| **Ambiguity** | Two or more plausible readings of the spec exist for the same construct. The agent had to pick one but cannot confirm which is correct. |
| **Contradiction** | The agent found two spec files that give conflicting guidance on the same construct. |
| **Round** | One set of programs written by the agent in a single pass. The agent completes three rounds total. Each round increases in program complexity. |
| **Finding** | Any of: Guess, Friction point, Spec gap, Ambiguity, Contradiction. |
| **Finding ID** | A unique identifier `Rn-Fm` (Round n, Finding m). Example: `R1-F3` = Round 1, Finding 3. |

---

## 3. The Round Loop

The agent completes three rounds. Each round adds new programs and produces new findings. Do not
revise or retrospectively update prior rounds — the findings document is an append-only log of
what the agent encountered in real time.

---

### Round Structure

Each round consists of four phases:

**Phase W — Write programs**

Write the programs for this round. Each program must:

- Use only constructs described in the spec (do not invent new syntax).
- Be self-contained enough to illustrate a real use case.
- Represent realistic code a Bouncelang developer would write, not toy examples designed to
  expose gaps.
- Cover a distinct area of the language from the other programs in the same round.

As the agent writes each program, it narrates its reasoning aloud: "I'm reaching for X here
because Y. The spec says Z. I'm writing it as W because..." This narration is how findings
are discovered — it happens naturally during writing, not as a separate review step afterward.

**Phase R — Record findings**

For each finding encountered during Phase W, record it using the finding format in §4. A finding
must be recorded at the moment it occurs, with the specific line of code that triggered it and the
exact question the agent could not answer from the spec.

Do not clean up findings or batch them. Record them in order of occurrence.

**Phase S — Self-test**

After writing all programs for the round, do one more pass: read each program back as if reviewing
it. Ask: "Does this compile under the spec's rules? Is this the idiomatic way to do this in
Bouncelang?" Record any additional findings discovered during this pass with the suffix `-review`
(e.g., `R2-F4-review`).

**Phase A — Round aggregate**

After all programs and findings for the round are recorded, produce a brief aggregate summary:

- How many findings in this round?
- Which spec areas generated the most friction?
- Were any spec areas particularly clear and easy to use?

---

### Round 1 — Core Language (Small Programs)

Write **three programs**, each 15–40 lines of Bounce code. Focus on the fundamental language
building blocks. Use at most one or two effects per program.

**Required coverage:** Each of the three programs must be drawn from a different area:

| Program | Must use |
|---|---|
| 1a | Records, unions, pattern matching, `try`/`error` |
| 1b | Functions with UFCS, generics or constraints, pipelines |
| 1c | Basic effects (`FileSystem` or `IO`), `world` + `handler` wiring |

---

### Round 2 — Composition (Medium Programs)

Write **three programs**, each 40–100 lines of Bounce code. Focus on how language features compose
together. Each program should use at least two separate spec areas that must work together.

**Required coverage:**

| Program | Must use |
|---|---|
| 2a | Opaque types with companion functions + serialization (`view` / `input`) |
| 2b | Generics + interfaces + error grouping across a multi-step pipeline |
| 2c | Concurrency (`Concurrency.scope`, `State<T>`, `spawn`/`detach`) + effects |

---

### Round 3 — Realistic End-to-End (Full Programs)

Write **two programs**, each 80–200 lines of Bounce code. These must represent realistic
production-quality Bouncelang applications, not demos. Pick from the following two targets:

| Program | Description |
|---|---|
| 3a | A simple HTTP API handler: receives a request, parses an input, validates it, queries a database effect, applies a view, and returns a JSON response. Must include a `world` declaration, at least one custom `error` type, and a `test fn`. |
| 3b | A background worker or CLI tool: reads from a queue or filesystem, processes records in a pipeline with error recovery, writes results, and shuts down cleanly. Must include a `world` declaration, at least one `Sequence<T>` usage with a terminal operator, and a `test fn`. |

The agent must write both 3a and 3b.

---

## 4. Finding Format

Each finding is recorded in this exact structure:

```
### Finding R<n>-F<m> — <Category>

**Program:** <program ID, e.g. 1a, 2c, 3a>
**Location:** <brief description of where in the program, e.g. "defining the User opaque type's companion export">
**The code I wrote:**
<code block — just the specific lines that triggered the finding>

**What I was trying to do:** <one sentence>

**What the spec says:** <exact quote or section reference, or "spec is silent">

**The gap / ambiguity / friction:**
<clear description of what is unclear, missing, or awkward>

**My best guess (if a Guess or Ambiguity):** <what I wrote and why>

**Impact on reader:**
- [ ] Compile error if wrong
- [ ] Incorrect runtime behavior if wrong  
- [ ] Style/readability only
- [ ] Unknown — could not determine from spec

**Recommendation type hint:**
- [ ] Spec clarification needed (the rule exists but is unclear)
- [ ] New syntax or construct needed (concept exists, no syntax)
- [ ] New spec section needed (entire area uncovered)
- [ ] Formatter rule clarification
- [ ] Example needed (rule is clear; a worked example would help)
```

---

## 5. Outputs

When all three rounds are complete, produce the following:

### 5.1. Programs File

A file named `tester-programs.md` containing all programs written in all three rounds, formatted
and annotated with finding IDs inline at the point where each finding occurred.

Inline annotations use this format: `// [R1-F3: ambiguous — see findings]`

### 5.2. Findings Log

A file named `tester-findings.md` following the template in `tester-findings-template.md`. The
log must contain:

- Every finding from all three rounds, in order.
- The round aggregate summary after each round's findings.
- The final summary at the end (§5.3).

### 5.3. Final Summary

After all three rounds, produce a final summary at the bottom of `tester-findings.md` with these
sections:

#### A. Spec Areas by Coverage Quality

Rate each major spec area on two dimensions: **clarity** (how easy was it to know what to write?)
and **completeness** (were all needed constructs present?). Use a three-level scale: ✅ good,
⚠️ needs clarification, ❌ missing or broken.

| Spec Area | Clarity | Completeness | Top Finding IDs |
|---|---|---|---|
| Primitives | | | |
| Records & structural typing | | | |
| Opaque types | | | |
| Atoms & unions | | | |
| Pattern matching | | | |
| Generics & interfaces | | | |
| Error handling (`try`/`error`) | | | |
| Effects system | | | |
| Handlers & worlds | | | |
| Concurrency & `Sequence<T>` | | | |
| Serialization (`view`/`input`) | | | |
| Modules & imports | | | |
| Formatter / style | | | |

#### B. Prioritized Finding List

Order all findings from all three rounds by this priority:

1. **Compiler-observable** — Getting this wrong causes a compile error or incorrect runtime behavior
2. **API surface** — Gets this wrong means a public API is misleading or incomplete
3. **Friction** — Makes Bouncelang feel harder than it should be
4. **Clarity only** — Would help new developers; not a semantic issue

For each finding, record: Finding ID, category, one-line description, priority.

#### C. Gaps Requiring New Spec Content

List every finding that requires entirely new spec content — either a new spec file or a new major
section in an existing file. For each, describe the area, the gap, and the minimum required content.

#### D. Clarifications Needed in Existing Spec Files

List every finding that requires a clarification or addition to an existing spec section. For each,
name the target file and section and describe the specific clarification needed.

#### E. Patterns That Worked Well

List any language patterns or areas where writing the programs felt natural, clear, and correct.
These are equally important for the recommender agent — they should not be changed.

---

## 6. Guidelines for the Agent

### On writing programs

Write programs that would actually be useful, not programs designed to expose spec gaps. The
findings emerge naturally from writing real code. Programs written specifically to find gaps tend
to find artificial ones.

If the spec does not cover something you need, do one of:

1. **Skip the feature** — and record a finding that the program could not be completed without it.
2. **Use a workaround** — write the closest thing the spec does support, and record a finding that
   it required a workaround.
3. **Make a guess** — write what seems most consistent with the rest of the spec, annotate it with
   a guess finding, and continue.

Never invent new syntax without recording it as a finding of category **Spec gap** or **Guess**.

### On the narration

The narration ("I'm reaching for X here because Y") should be written in the programs file as
comments, then stripped out when the clean program version is written. Think of it as a
rubber-duck debugging session — talking through what you're doing is how you find the gaps.

### On scope

Do not try to write a complete language implementation, standard library, or test suite. Write the
kind of code a developer would write on day one of learning Bouncelang. The spec should guide you
all the way to working code. If it doesn't, that's a finding.

### On being fair to the spec

When you encounter friction, ask: "Is this the spec's problem, or is this inherent complexity?" A
pattern that is genuinely hard to express because the domain is hard is not a spec gap. A pattern
that is hard to express because the syntax is awkward for common cases is a friction point. The
distinction matters for the recommender agent.

### On annotations in programs

Use the inline annotation format `// [Rn-Fm: category — brief description]` at the exact line
where the finding occurred. Keep annotations brief — the detail is in the findings log. The
programs file should remain readable as programs, not as a list of complaints.

---

## 7. Inputs to the Next Agent

The **Spec Recommender Agent** (`spec-recommender-agent.md`) receives these outputs as its inputs:

- `tester-programs.md` — the annotated programs
- `tester-findings.md` — the full findings log including the final summary

The recommender agent uses these to generate concrete options and spec updates. The tester agent's
job is to produce a complete and honest picture of the spec's current state from a user's
perspective — not to propose solutions.

---

## 8. Quick Reference: Finding Categories

| Category | When to use |
|---|---|
| **Guess** | Wrote code that seemed right but cannot be justified from the spec. |
| **Friction point** | Spec covers it, but the required approach is disproportionately complex for the conceptual weight. |
| **Spec gap** | Needed to express something the spec has no syntax or rule for. |
| **Ambiguity** | Two or more plausible readings of the spec exist; had to pick one. |
| **Contradiction** | Two spec files give conflicting guidance on the same construct. |

---

## 9. Quick Reference: Program Complexity Targets

| Round | Lines per program | Effects | Type system depth | Concurrency |
|---|---|---|---|---|
| 1 (small) | 15–40 | ≤ 2 | Basic records, unions, simple generics | None |
| 2 (medium) | 40–100 | 2–3 | Opaque types, interfaces, error groups | Optional |
| 3 (realistic) | 80–200 | 3+ | Full type system, views, inputs | Required in 3b |
