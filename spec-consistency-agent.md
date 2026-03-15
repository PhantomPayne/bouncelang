# Spec Consistency Agent — Plan & Prompt

This document is a **self-contained prompt and plan** for running an AI agent whose job is to iterate on the Bouncelang spec files, improve their mutual consistency, and produce a final set of high-quality specs plus a companion decisions log.

Hand this document to an AI agent (e.g., Claude or GPT-4-class) together with the full content of every spec file listed in §1. The agent follows the loop in §3 until the termination criteria in §4 are satisfied, then writes the outputs described in §5.

---

## 1. Inputs

Provide the following files verbatim to the agent at the start of the session:

| File | Role |
|---|---|
| `core-principles.md` | **North Star** — the priority order (Safety > Understandability > DX > Power > Performance) and every "when to add what" table. Every option must be evaluated against this first. |
| `01-primitives.md` | v2 layer 1 spec — the primary reference for primitive types, syntax, and DST/LSP/Wasm stories. Treat this as the **strongest guideline** for what a finished spec looks like. |
| `02-data-structures.md` | v2 layer 2 spec — structural typing, opaque types, collections. |
| `04-effects-and-handlers.md` | v2 layer 4 spec — effect system, `Raise`, `Panic`, world/handler model. |
| `07-concurrency.md` | v2 layer 7 spec — `Sequence<T>`, `Scope`, `State<T>`, virtual time. |
| `08-serialization-boundaries.md` | v2 layer 8 spec — `view`, `input`, format selectors. |
| `atoms-and-unions.md` | Discriminated unions, exhaustive pattern matching. |
| `error-handling.md` | `error` keyword, `try` blocks, grouped errors. |
| `formatter-principles.md` | Canonical formatter rules — structural expansion, leading operators, trailing commas. |
| `generics-and-type-system.md` | Three type tiers, Hindley-Milner, interfaces, auto-derivation. |
| `methods-and-packages.md` | UFCS, `package.bounce`, visibility layers. |
| `modules-and-imports.md` | Import resolution, workspaces, test files. |
| `worlds-and-handlers.md` | `world` files, named handlers, config block, entry points. |

**What is the "v2 spec"?**

The numbered files (`01-`, `02-`, `04-`, `07-`, `08-`) are explicitly labeled as v2 layer specs. They are the most mature documents and contain the most complete DST, LSP, and Wasm stories. Use their structure, terminology, and depth of coverage as the **template** every other spec file should converge toward.

---

## 2. Definitions

| Term | Meaning |
|---|---|
| **Inconsistency** | Any of: conflicting terminology, contradictory rules, a concept present in one spec that contradicts a related concept in another, a section present in v2 specs that is absent (without justification) from another spec, or a naming/syntax choice in one file that conflicts with the convention established in `core-principles.md`. |
| **Open question** | A gap or ambiguity where the spec does not give a complete, unambiguous answer. Expressed as a single, concrete question. |
| **Option** | One possible resolution to an open question. Each option includes its implications and a tradeoff analysis. |
| **Consistency score** | After each iteration, the agent estimates a score from 0–10 for each spec file (10 = fully consistent with v2 and core principles, 0 = major contradictions). |
| **Iteration** | One full pass of phases A–F in §3. |

---

## 3. The Iteration Loop

Run the following phases in order for each iteration. Record every decision in the decisions log (see §5.2) as you go.

---

### Phase A — Audit: Read and Map

On the first iteration, read all spec files in full. On subsequent iterations, read only the files that were modified or that new questions implicate.

For each spec file, produce a structured audit covering:

1. **Terminology check** — Does this file use the same terms for the same concepts as other spec files and `core-principles.md`? Flag every mismatch (e.g., one file calls it `Result`, another calls it `Raise<E>`).

2. **Section completeness check** — The v2 numbered specs (`01-`, `02-`, `04-`, `07-`, `08-`) each contain sections covering: *What is it / Syntax / Semantics / DST story / LSP checklist / Wasm/compilation / Serialization / World interaction*. For each non-v2 spec, check which of these sections is missing, thin, or inconsistent with the v2 equivalents.

3. **Cross-reference check** — Does this file reference constructs defined in another spec? If yes, does it use the same definition? Flag contradictions.

4. **Core-principles alignment check** — For every claim, rule, or design choice in the file, identify whether it aligns with the core principles priority order (Safety > Understandability > DX > Power > Performance). Flag anything that appears to violate this order without explicit justification.

5. **Formatter consistency check** — Does any syntax example in this file violate the rules in `formatter-principles.md`? Flag every violation.

Output of Phase A: A numbered list of **open questions**, one per inconsistency or gap found.

---

### Phase B — Question Prioritization

Order the open questions from Phase A by impact using this priority:

1. **Contradictions** — Two specs say opposite things about the same construct. Must be resolved before anything else.
2. **Missing required sections** — A spec lacks a DST, LSP, or Wasm story that other v2 specs have. Blocks completeness.
3. **Terminology drift** — The same concept uses different names across files. Causes reader confusion.
4. **Thin coverage** — A section exists but is shallower than the v2 equivalent. Lower risk, but reduces quality.
5. **Style / formatter violations** — Code examples that don't match `formatter-principles.md`.

Work the top five questions (or all questions if fewer than five remain) in the current iteration.

---

### Phase C — Option Generation

For each question selected in Phase B, generate **two to four distinct options**. Each option must:

- Be a complete, concrete answer (not a vague direction).
- Show example syntax or text if the answer affects syntax.
- Explicitly state which core principle(s) it advances and which it compromises.
- Be genuinely different from the other options (not minor wording variations).

**Generation rules:**

- Start from `core-principles.md` §9 (Practical Tradeoff Guide). The priority order is Safety > Understandability > DX > Power > Performance. Options that maximize higher-priority principles score better.
- If the question touches the effect system, ensure every option preserves the DST constraint from `core-principles.md` §6: "Can a user write a deterministic simulation test for this?"
- If the question touches public APIs or visibility, ensure every option preserves the rule that public API boundaries are explicit.
- If the question touches syntax, ensure every option is consistent with `formatter-principles.md` and the naming conventions in `core-principles.md` §3.
- If the question has an analogue in a v2 numbered spec, treat the v2 approach as the default option (Option A). Other options must make the case for diverging.

---

### Phase D — Option Evaluation

For each set of options generated in Phase C:

1. **Score each option** against the five core principles (1 = strong regression, 3 = neutral, 5 = strong advancement):

   | Option | Safety | Understandability | DX | Power | Performance |
   |---|---|---|---|---|---|
   | A | | | | | |
   | B | | | | | |

2. **Cross-check against all other open questions.** Does selecting this option create or resolve any other open questions? Update the question list accordingly.

3. **Check for internal contradiction.** If two questions are in the same phase and their preferred options conflict, flag it and resolve the conflict before proceeding. Use the core-principles priority order to break ties.

4. **Select the winning option.** Prefer the option with the highest score on the highest-priority principle where options differ. If options are tied on all five dimensions, prefer the one most consistent with the v2 numbered spec for that topic area.

5. **Record the discarded options** and a one-sentence rationale for each in the decisions log.

---

### Phase E — Spec Update

Apply the selected options to the relevant spec files. Each edit must:

- Be the **smallest change** that fully resolves the question. Do not refactor unrelated sections.
- Preserve the existing document structure unless the question specifically concerns structure.
- Update cross-references in other spec files if the edit changes a term, construct name, or rule that other specs rely on.
- Add or update the relevant section (DST story, LSP checklist item, etc.) if that is what was missing.
- Format all code examples according to `formatter-principles.md`.

After applying edits, re-read the changed sections and verify that:
- The inconsistency that motivated the question is gone.
- No new inconsistency was introduced in the edited text.
- The edit is consistent with all other files that reference the changed construct.

---

### Phase F — Consistency Scoring

After all edits in the current iteration are applied, produce a consistency score for each spec file:

| File | Score (0–10) | Remaining issues |
|---|---|---|
| `01-primitives.md` | | |
| `02-data-structures.md` | | |
| … | | |

**Scoring rubric:**

| Score | Meaning |
|---|---|
| 10 | No open questions. All sections complete relative to v2 template. No contradictions with any other file or `core-principles.md`. All code examples match formatter rules. |
| 8–9 | One or two thin sections. No contradictions. Minor terminology drift. |
| 6–7 | Missing one required section (DST, LSP, or Wasm story). One terminology inconsistency. |
| 4–5 | Multiple missing sections. One contradiction with another file. |
| 2–3 | Major section gaps. Multiple contradictions. |
| 0–1 | Structurally incomplete. Contradicts core principles on fundamental points. |

---

### Termination Check

After Phase F, evaluate:

- **Hard stop:** All files score ≥ 8 **and** there are zero remaining questions of priority 1 (Contradiction) or priority 2 (Missing required section).
- **Soft stop:** All files score ≥ 6 and the remaining questions are all priority 4 (Thin coverage) or priority 5 (Style violations) **and** at least three full iterations have been completed.
- **Maximum iterations:** 10. If the termination criteria are not met after 10 iterations, stop and document the remaining questions as "known open issues" in the decisions log.

If neither stop condition is met, return to Phase A for the next iteration.

---

## 4. Guidelines for the Agent

### On asking questions

Open questions must be **specific and answerable**. Bad: "Is the error handling consistent?" Good: "The `error-handling.md` spec uses the term `Result` in §2 but `04-effects-and-handlers.md` §3 says errors are the `Raise<E>` effect, not a return type. Which term and model is canonical, and should `error-handling.md` be updated to use `Raise<E>` throughout?"

### On generating options

Do not generate options that violate a core principle unless the violation is explicitly acknowledged and justified by a higher-priority principle. Do not generate options that are functionally identical — if two options differ only in surface syntax but have identical semantics and tradeoffs, merge them.

### On selecting options

When two options are genuinely equivalent under the core principles, prefer the option that requires fewer changes across the full spec corpus. Minimizing diff size reduces the chance of introducing new inconsistencies.

### On editing specs

Never delete content without replacing it with something equivalent or recording the deletion in the decisions log. Do not introduce new syntax or semantics that are not already present in some form in the existing spec corpus — this agent's job is to **harmonize**, not to **invent**.

### On the v2 spec as a guideline

The v2 numbered specs are a strong guideline, not an absolute law. If a non-v2 spec makes a choice that is genuinely better under the core principles priority order, the v2 spec should be updated to match — not the other way around. The v2 spec is the reference for *structure and completeness*, but `core-principles.md` is the reference for *correctness of decisions*.

---

## 5. Outputs

When the termination criteria are met, produce the following:

### 5.1. Updated Spec Files

A full replacement for each spec file that was modified. Each file must:

- Carry an `Updated: <iteration number> iterations` note at the top.
- Retain all existing content that was not changed.
- Use consistent terminology throughout the full spec corpus.
- Have all mandatory sections (What is it, Syntax, DST story, LSP checklist, Wasm/compilation, Serialization, World interaction) present at least at a summary level.

### 5.2. Decisions Log

A new file named `decisions-log.md` in the repository root, following the template in `decisions-log-template.md`. The log must document every iteration, every question raised, every option generated, and every discarded option with its one-sentence rationale.

The log is a permanent record of **why** the specs look the way they do. A future contributor reading only `decisions-log.md` should be able to understand every significant choice made during the consistency pass.

---

## 6. Quick Reference: Core Principles Summary

The agent should keep this table in mind when evaluating every option:

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

**DST gate:** Any option that touches time, randomness, concurrency, or I/O must answer: "Can a user write a deterministic simulation test for code that uses this?"

**LSP gate:** Any option that introduces a new syntactic construct must answer all six items in the LSP checklist (`core-principles.md` §7): completions, inlay hints, diagnostics, quick fixes, hover/go-to-definition, semantic highlighting.

**Formatter gate:** Any option that introduces or modifies syntax must produce examples formatted according to `formatter-principles.md`: structural expansion (one line or fully expanded), leading operators, trailing commas, no hard line-length limit.
