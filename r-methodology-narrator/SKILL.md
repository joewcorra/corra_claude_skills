---
name: r-methodology-narrator
description: >
  Translates R scripts into plain-language methodology narratives readable by
  domain experts who are not R programmers. Use this skill whenever a user
  pastes or uploads an R script and asks for a methodology description, plain
  language summary, expert review document, or non-technical explanation of what
  the code does. Also trigger when a user says things like "explain this script
  for a reviewer", "write up what this code does", "translate this R code into
  steps", or "my colleague needs to review this but can't read R". The output
  is a numbered methodology narrative formatted for use in Quarto or Word
  documents.
---

# R Methodology Narrator

Produces a plain-language, numbered methodology narrative from an R script,
suitable for review by a scientific domain expert who cannot read R code.

---

## Guiding Principles

- **Audience**: A domain expert (scientist, analyst, or subject matter expert)
  who understands the field but not R syntax. Never explain R functions by
  name. Instead, describe *what* they accomplish in plain language.
- **Tone**: Clear, precise, and professional — appropriate for a methods
  section of a technical report or scientific document.
- **Granularity**: Capture meaningful methodological steps, not individual
  lines of code. Group related lines into logical operations. Omit purely
  mechanical steps (loading libraries, setting file paths) unless they have
  methodological significance (e.g., a specific version dependency or a
  deliberate file structure choice).
- **Fidelity**: Do not infer intent beyond what the code supports. If
  something is ambiguous, note it explicitly as a point for the reviewer
  to confirm.

---

## Process

### Step 1 — Parse the Script

Read the full script before writing anything. Identify:

- **Inputs**: What data sources, files, or parameters does the script consume?
- **Outputs**: What does the script produce — tables, files, model objects,
  plots?
- **Major phases**: Mentally segment the script into logical blocks (e.g.,
  data ingestion, cleaning/validation, calculation, aggregation, output).
  These will become the top-level numbered steps.
- **Key assumptions**: Hard-coded values, filter criteria, thresholds, or
  lookup tables embedded in the code that a reviewer should be aware of.
- **Ambiguities**: Places where intent is unclear or where multiple
  interpretations are possible.

### Step 2 — Draft the Narrative

Write a numbered list of methodology steps. Follow these rules:

1. **Number sequentially** from 1. Use sub-numbers (e.g., 2.1, 2.2) only
   when a major phase has distinct sub-steps worth calling out separately.
2. **Lead each step with a plain-language verb**: "Load", "Filter", "Calculate",
   "Join", "Apply", "Aggregate", "Export" — not R function names.
3. **Name the objects being acted on** in domain terms, not R variable names.
   E.g., "the equipment inventory table" not "`df_equip`".
4. **Call out parameters and assumptions explicitly**. If a filter threshold,
   emission factor, or reference year is hard-coded, state the value and flag
   it for reviewer confirmation.
5. **Note any iterative or recursive logic** in plain terms (e.g., "This step
   is repeated for each year in the study period").
6. **Flag ambiguities** inline with a note: *[Reviewer note: please confirm
   whether X is intended here]*.

### Step 3 — Add a Preamble and Closing Summary

**Preamble** (before the numbered list): 2–4 sentences describing the overall
purpose of the script, its primary input(s), and its primary output(s).

**Closing summary** (after the numbered list): 2–3 sentences summarizing any
assumptions, limitations, or open questions the reviewer should pay particular
attention to.

### Step 4 — Format for Quarto / Word

Structure the output as clean markdown suitable for pasting into a `.qmd` file
or converting to Word via `pandoc` / `knitr`. Use:

- `##` for the document title (e.g., `## Methodology: [Script Name or Purpose]`)
- `###` for major phase headings if the script has clearly distinct phases
- Plain numbered list for steps (no markdown nesting beyond sub-numbers)
- `> Reviewer note:` blockquote style for flagged ambiguities or assumptions
  needing confirmation
- A horizontal rule (`---`) before the closing summary

Do **not** include R code blocks in the output. This document is for
non-coders.

---

## Output Template

```
## Methodology: [Brief description of script purpose]

[Preamble: 2–4 sentences on overall purpose, inputs, and outputs.]

### [Phase 1 Name, if applicable]

1. [Step description.]
2. [Step description.]
   2.1. [Sub-step if needed.]
   2.2. [Sub-step if needed.]

### [Phase 2 Name, if applicable]

3. [Step description.]

> **Reviewer note:** [Flag any assumption or ambiguity here.]

4. [Step description.]

---

**Summary and open questions:** [2–3 sentences on key assumptions,
limitations, or items requiring reviewer confirmation.]
```

---

## Notes on Common R Patterns

Translate these patterns into plain language without mentioning R:

| R pattern | Plain language |
|---|---|
| `left_join` / `merge` | "matched to" / "combined with [table] using [key]" |
| `filter` / `subset` | "restricted to records where [condition]" |
| `group_by` + `summarise` | "aggregated by [group variable], calculating [metric]" |
| `mutate` | "a new field was calculated as [description]" |
| `purrr::map` / `lapply` | "this process was repeated for each [element]" |
| `accumulate` | "applied cumulatively across [sequence], carrying forward prior values" |
| `pivot_wider` / `pivot_longer` | "restructured from [wide/long] to [long/wide] format" |
| `readxl::read_excel` / `read_csv` | "imported from [file type]" |
| `write_csv` / `openxlsx::write.xlsx` | "exported as [file type]" |
| `targets` pipeline | "each calculation step was run in sequence, with results cached for reproducibility" |
