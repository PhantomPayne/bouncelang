# Spec Recommender Agent — Plan & Prompt

This document is a **self-contained prompt and plan** for running an AI agent whose job is to
translate the Spec Tester's findings into concrete, justified recommendations — and to generate
the new or updated spec content those recommendations require.

The recommender agent's output feeds directly into the **Spec Consistency Agent**
(`spec-consistency-agent.md`), which verifies that the new content is consistent with the existing
spec corpus.

Hand this document to an AI agent (e.g., Claude or GPT-4-class) together with the files listed
in §1.

---

## 1. Inputs

### 1.1. From the Spec Tester

| File | Role |
|---|---|
| `tester-findings.md` | **Primary input** — the full findings log from the tester agent, including all Round findings, round aggregate summaries, and the final summary (§§A–E). |
| `tester-programs.md` | **Secondary input** — the annotated programs, used to understand the context of each finding. Read the specific program sections referenced by findings under review. |

### 1.2. The Existing Spec Corpus

| File | Role |
|---|---|
| `core-principles.md` | **North Star** — the priority order and all design constraints. Every recommendation must be validated against this. |
| `01-primitives.md` | v2 layer 1 — reference for structure and depth of a finished spec. |
| `02-data-structures.md` | v2 layer 2 — structural typing, opaque types, collections. |
| `04-effects-and-handlers.md` | v2 layer 4 — effect system, handlers, test sandboxing. |
| `07-concurrency.md` | v2 layer 7 — `Sequence<T>`, scoped tasks, `State<T>`. |
| `08-serialization-boundaries.md` | v2 layer 8 — `view`, `input`, format selectors. |
| `atoms-and-unions.md` | Atoms, discriminated unions, pattern matching. |
| `error-handling.md` | `error` keyword, `try` blocks, grouped errors. |
| `formatter-principles.md` | Canonical formatter rules. |
| `generics-and-type-system.md` | Three type tiers, Hindley-Milner, interfaces. |
| `methods-and-packages.md` | UFCS, `package.bounce`, companion functions. |
| `modules-and-imports.md` | Import resolution, workspaces, test files. |
| `worlds-and-handlers.md` | `world` declarations, handlers, config block. |

### 1.3. Prior Agent Outputs (if available)

| File | Role |
|---|---|
| `decisions-log.md` | Prior consistency agent decisions — use to avoid re-opening questions already resolved. |

---

## 2. Definitions

| Term | Meaning |
|---|---|
| **Finding** | A single entry from `tester-findings.md` — a Guess, Friction point, Spec gap, Ambiguity, or Contradiction. |
| **Recommendation** | A concrete, actionable answer to a finding: a specific spec change, a new spec section, or a new spec file. |
| **Option** | One possible recommendation for a finding. Options are distinct and evaluated against core principles before one is selected. |
| **New spec file** | A new `.md` file added to the spec corpus, following the v2 numbered spec template. |
| **Spec update** | An addition or change to an existing spec file. Must be the smallest change that fully addresses the finding. |
| **Recommendation ID** | A unique identifier `Rec-N` (e.g., `Rec-7`). Each recommendation addresses one or more related findings. |

---

## 3. The Iteration Loop

The recommender agent works through the tester's findings in a structured loop. Run all four
phases before moving to the next batch of findings.

---

### Phase A — Triage

Read all findings from `tester-findings.md`. Classify each finding into one of four work types:

| Work Type | Description | Output |
|---|---|---|
| **Clarification** | The spec has a rule but it is unclear or requires a concrete example. | Updated section in an existing spec file. |
| **Gap fill** | The spec is silent on something the tester needed. A rule or construct exists logically; it just isn't written down. | New section in an existing spec file, or a new spec file. |
| **New construct** | The tester needed something genuinely absent — a syntax form, a type, an operation — that the spec does not describe anywhere. | New section (with syntax, semantics, DST story, LSP checklist, Wasm story) in the appropriate existing spec file, or a new spec file. |
| **Friction redesign** | The spec covers the feature but the required approach is awkward. The recommendation may change syntax or add a shorthand. Highest risk — requires core-principles justification. | Updated section with revised or supplemented syntax. |

For each finding, record:

- **Finding ID** (from `tester-findings.md`)
- **Work type**
- **One-sentence description of what needs to change**
- **Target file(s)** — which spec file(s) would change?
- **Is this covered by `decisions-log.md`?** — if yes, the decision stands; skip to the next finding unless the tester found a previously unknown consequence.
- **Priority** — inherit from tester's priority (1–4), or escalate if the finding implies a safety or correctness issue not flagged by the tester.

Group findings that address the same underlying gap into a single **Recommendation group**.
The tester may have encountered the same gap from multiple programs — one recommendation
addresses the whole group.

**Output of Phase A:** A triage table listing every finding, its work type, its target file, and
its recommendation group assignment.

---

### Phase B — Option Generation

For each recommendation group, generate **two to four distinct options**. Each option must:

- Be a complete, concrete proposal — not a direction.
- Show example syntax where the recommendation touches syntax.
- Explicitly state which core principle(s) it advances and which it compromises.
- Be genuinely different from the other options (not minor wording variations).
- Follow all applicable rules:

**Mandatory gates (every option must pass):**

1. **Core-principles gate:** Does this option improve Safety, Understandability, or DX compared to
   the current spec? An option that only adds Power or reduces Performance overhead may be rejected
   if it costs anything on the higher-priority dimensions.

2. **DST gate:** If the option touches time, randomness, concurrency, or I/O, it must answer:
   "Can a user write a deterministic simulation test for code that uses this?"

3. **LSP gate:** If the option introduces a new syntactic construct, it must answer all six items
   in the LSP checklist (`core-principles.md` §7): completions, inlay hints, diagnostics, quick
   fixes, hover/go-to-definition, semantic highlighting.

4. **Formatter gate:** If the option introduces or modifies syntax, it must produce example code
   formatted according to `formatter-principles.md` (structural expansion, leading operators,
   trailing commas, no hard line-length limit).

5. **Consistency gate:** If the option touches a construct that already appears in another spec
   file, it must be consistent with that file's treatment. If it isn't, flag the inconsistency
   and resolve it as part of the recommendation.

**Generation rules:**

- If the finding is a **Clarification**, generate Option A as "add a worked example and a
  clarifying sentence" and Option B as "rewrite the section for clarity." Prefer Option A unless
  the section is fundamentally unclear.
- If the finding is a **Gap fill**, generate Option A following the v2 numbered spec template
  (same structure: what is it / syntax / semantics / DST / LSP / Wasm). Other options may propose
  a more minimal treatment.
- If the finding is a **New construct**, generate Option A as the most conservative addition
  (smallest syntax footprint, most consistent with existing spec), and at least one option with a
  broader or more powerful form.
- If the finding is a **Friction redesign**, generate Option A as adding an ergonomic shorthand
  while keeping the existing long form, and Option B as replacing the long form entirely. Be very
  conservative with Option B — removing existing syntax has high consistency cost.

---

### Phase C — Option Evaluation

For each set of options, evaluate and select one using the same scoring rubric as the consistency
agent:

1. **Score each option** against the five core principles (1 = strong regression, 3 = neutral,
   5 = strong advancement):

   | Option | Safety | Understandability | DX | Power | Performance | Total |
   |---|---|---|---|---|---|---|
   | A | | | | | | |
   | B | | | | | | |

2. **Check for cross-finding conflicts.** Does selecting this option conflict with any other
   finding's recommendation? If so, resolve the conflict before finalizing. Use the core-principles
   priority order to break ties.

3. **Check against `decisions-log.md`.** Has this decision been made before? If the same option
   was previously selected and then reversed, note why and whether the tester's findings constitute
   new evidence that would change the outcome.

4. **Select the winning option.** Prefer the option with the highest score on the highest-priority
   principle where options differ. Document every discarded option and a one-sentence rationale.

---

### Phase D — Spec Content Generation

For each selected recommendation, generate the actual spec content.

**For a Clarification:**

Write the specific sentence(s) or paragraph(s) to add to the existing section. Specify the exact
location: file name, section heading, and whether to insert before, after, or replace existing
content.

**For a Gap fill or New construct:**

Write the full new section. Follow the v2 numbered spec section template:

```markdown
## N. <Section Title>

### What it is

<One paragraph: what this is, why it exists, and what problem it solves.>

### Syntax

<Formal description or BNF if applicable, followed by a worked example in Bounce syntax,
formatted according to formatter-principles.md.>

### Semantics

<What the compiler guarantees. What happens at runtime. Edge cases and what happens in each.>

### DST / Testing

<How this construct interacts with the test world. Can it be sandboxed? What does a
deterministic test look like? If it cannot be sandboxed, explain why and what the mitigation is.>

### LSP / DX

<The six-item LSP checklist table (see core-principles.md §7). Every item must be answered.>

### Compiling & WebAssembly

<How this construct compiles. What Wasm instruction or component model type it maps to.
Is there any overhead? How does it appear in the WIT interface (if applicable)?> 

### Serialization (if applicable)

<How this construct interacts with the view/input boundary. Can it appear in a view or input?
What does it look like in JSON or another serialization format?>
```

**For a Friction redesign:**

Write the specific syntax addition or change, including:
- The old form (retained or deprecated).
- The new form.
- A desugar rule if the new form is sugar for the old form.
- Updated examples using the new form.
- Formatter rules for the new construct.

**All generated content must:**

- Be formatted according to `formatter-principles.md`.
- Use only terminology that is consistent with the existing spec corpus.
- Include at least one complete `bounce` code example.
- Not introduce any new concept that is not already implied by the existing spec.

---

### Phase E — Recommendation Record

After generating all spec content, record each recommendation in the recommendations log (§5.2).
The log must be complete enough that a future agent running the consistency pass can understand
why each new section exists and what finding motivated it.

---

## 4. Guidelines for the Agent

### On scope

The recommender agent's job is to fill gaps and remove friction, not to redesign the language. Do
not use a finding as an excuse to propose major redesigns or to add features the tester did not
specifically need. One finding → one focused recommendation.

If a finding reveals a genuine structural problem with the spec (e.g., two incompatible models for
the same construct), the recommendation should be to resolve the inconsistency using the
consistency agent's methodology — not to invent a third model.

### On new spec files

Create a new spec file only when:

1. The gap is large enough that adding a section to an existing file would be structurally awkward.
2. The new content covers a distinct area not currently addressed by any existing file.
3. The new file would follow the v2 numbered spec template completely.

New spec files must be listed in the consistency agent's §1 input table (in
`spec-consistency-agent.md`) and in their own related-links section. File names follow the
pattern `NN-area-name.md` for numbered spec files or `area-name.md` for unnumbered.

Do not split an existing spec file to create a new one unless the tester's findings provide
strong evidence that the existing file is genuinely confusing as a result of its scope.

### On conservatism

When in doubt, prefer a smaller change. A well-placed example or a clarifying sentence often
resolves an ambiguity more effectively than a new syntax form. New syntax has a permanent cost —
it adds to the cognitive overhead of learning the language. A clarifying sentence has near-zero
cost.

When a friction finding could be addressed by either adding a shorthand or improving documentation,
try the documentation approach first. If the spec section clearly described what the tester was
trying to do, the tester's friction was a learning-curve issue, not a design issue. Document it.
If the spec section was clear and the approach was still awkward, that is a genuine friction point
that warrants a syntax recommendation.

### On the consistency agent

The recommender agent does not run the consistency pass. It generates spec content that is
internally consistent and consistent with the existing corpus as far as the agent can tell. The
consistency agent (`spec-consistency-agent.md`) then makes the final validation pass.

If the recommender agent generates a new spec section that is inconsistent with an existing spec,
the consistency agent will catch and fix it. The recommender agent should make a good-faith effort
at consistency but should not let the pursuit of perfect consistency block it from generating
necessary new content.

---

## 5. Outputs

### 5.1. Updated or New Spec Files

All spec files modified or created by the recommender agent. Each modified file must:

- Carry an `Updated: Recommender pass (findings from tester-findings.md)` note in its status header.
- Retain all existing content that was not changed.
- Use consistent terminology throughout.
- Have all mandatory sections (§3 Phase D template) present.

New spec files must follow the v2 numbered spec template exactly and be named consistently with the
existing corpus.

### 5.2. Recommendations Log

A file named `recommender-recommendations.md` in the repository root. It must contain:

#### Header

```markdown
# Spec Recommender — Recommendations Log

**Date:** <date>
**Input:** tester-findings.md (<N> findings across <N> rounds)
**Recommendations generated:** <N>
**New spec files created:** <N>
**Existing spec files updated:** <N>
```

#### Triage Table

The full Phase A triage table: every finding, its work type, its target file, and its
recommendation group.

#### Recommendation Entries

One entry per recommendation group, in priority order (Priority 1 first):

```markdown
### Rec-N — <Short title>

**Addresses findings:** R?-F?, R?-F?, …
**Work type:** Clarification / Gap fill / New construct / Friction redesign
**Target file(s):** …
**Priority:** 1 / 2 / 3 / 4

**Selected option:** Option ? — <one-sentence description>

**Options evaluated:**

| Option | Safety | Understandability | DX | Power | Performance | Total |
|---|---|---|---|---|---|---|
| A | | | | | | |
| B | | | | | | |

**Discarded options:**

| Option | Rationale |
|---|---|
| A | *One sentence.* |

**Spec content generated:** <file name and section heading where the content was added>

**Summary of change:** <two to three sentences describing what was added or changed and why>
```

### 5.3. Handoff Summary

The final section of `recommender-recommendations.md` must be a **Handoff Summary** for the
Spec Consistency Agent. It must list:

1. Every new spec file created — name and brief description.
2. Every existing spec file modified — name and a one-sentence description of each change.
3. Any known inconsistencies or open questions the recommender agent could not resolve.
4. The recommended order in which the consistency agent should review changes (highest-risk first).

---

## 6. Quick Reference: The Recommendation Pipeline

```
spec-tester-agent.md
        │
        │  produces
        ▼
tester-programs.md      ←── annotated Bounce programs (3 rounds)
tester-findings.md      ←── findings log + final summary (§§A–E)
        │
        │  feeds into
        ▼
spec-recommender-agent.md
        │
        │  produces
        ▼
recommender-recommendations.md   ←── triage, options, decisions
<new or updated spec files>      ←── actual spec content
        │
        │  feeds into
        ▼
spec-consistency-agent.md
        │
        │  produces
        ▼
decisions-log.md                 ←── final verified spec corpus
<updated spec files>
```

---

## 7. Quick Reference: Core Principles Summary

| Priority | Principle | Key test |
|---|---|---|
| 1 | **Safety** | Does this prevent a class of bugs? Is it enforced at compile time? |
| 2 | **Understandability** | Would a new user understand this within 30 seconds? |
| 3 | **DX / Ergonomics** | Does the LSP checklist pass? Does the formatter handle it? |
| 4 | **Power / Expressiveness** | Does it solve a real problem that couldn't be solved otherwise? |
| 5 | **Performance** | Is there a compile-time or runtime cost? (Last priority.) |

**Anti-priorities** (reject any option justified primarily by these):

- Minimizing keystrokes
- Academic purity
- Feature parity with other languages
- Adding configuration options

**DST gate:** Any option that touches time, randomness, concurrency, or I/O must answer: "Can a
user write a deterministic simulation test for code that uses this?"

**LSP gate:** Any option that introduces a new syntactic construct must answer all six items in the
LSP checklist (`core-principles.md` §7).

**Formatter gate:** All generated code examples must follow `formatter-principles.md`.
