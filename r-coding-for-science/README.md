# Claude Skills for Scientific Data Work

This repository contains a set of **Claude skills** — structured prompt
templates that guide Claude toward consistent, high-quality outputs for
specific technical tasks. These skills are used in conjunction with Claude's
Projects feature, where a skill file is loaded into the project context to
shape Claude's behavior for a defined workflow.

Each skill lives in its own directory as a `SKILL.md` file. The file contains
a YAML front matter block (used as the skill description) and a detailed
prompt specification that Claude reads before responding.

---

## What Is a Claude Skill?

A Claude skill is a reusable prompt document that encodes:

- **Domain knowledge** — what Claude needs to know about the subject area to
  do the task well
- **Process** — a step-by-step workflow Claude should follow
- **Conventions** — output format, naming rules, style requirements
- **Edge case handling** — explicit guidance for ambiguous or incomplete inputs

Skills are particularly useful for tasks that recur with similar structure but
varying content — for example, parsing a new methodology document each month,
or translating a new R script into a methodology narrative for each inventory
cycle.

---

## Skills in This Repository

### [`ghg-methodology-parser`](./ghg-methodology-parser/SKILL.md)

Converts a narrative methodology description from a greenhouse gas inventory
chapter into a structured, language-agnostic, step-by-step implementation
framework.

**The general pattern:** Scientific inventory documents (GHG inventories,
environmental assessments, national statistics) describe calculation
methodologies in narrative prose. Turning that prose into something
implementable — with explicit inputs, formulas, conditional logic, and
uncertainty steps — requires careful reading and structured extraction. This
skill encodes that extraction process.

**Domain specificity:** This skill is tuned for the U.S. Inventory of
Greenhouse Gas Emissions and Sinks and IPCC-aligned national inventories. It
uses GHG inventory terminology (activity data, emission factors, GWP
weighting, CRF source category codes, IPCC tier levels) and is calibrated to
the structural patterns found in inventory methodology chapters. The underlying
pattern — parse a narrative methodology into a structured implementation
framework — is general and could be adapted to other scientific domains.

**Typical use:** Paste or upload a methodology chapter section; receive a
structured markdown framework with an inputs table, numbered calculation steps,
uncertainty steps, and an implementer notes section.

**Feeds into:** [`r-ghg-coder`](./r-ghg-coder/SKILL.md)

---

### [`r-ghg-coder`](./r-ghg-coder/SKILL.md)

Translates a structured implementation framework (as produced by
`ghg-methodology-parser`) into production-ready R code following a defined set
of pipeline and style conventions.

**The general pattern:** Given a language-agnostic specification of a
calculation workflow, generate idiomatic R code that implements it faithfully.
The skill encodes R conventions — tidyverse style, `{purrr}` for iteration,
`{targets}`-compatible function structure, explicit joining conventions, and a
named constants vector pattern for emission factors and parameters — so that
outputs are consistent and drop into a real codebase without rework.

**Domain specificity:** The R conventions in this skill reflect a specific
GHG inventory codebase, including an internal package (`{syrinx}`), long-format
time series conventions, and GHG-specific column naming patterns. The
`targets`-pipeline-first function structure (`get_` functions returning single
tibbles) and the tidyverse/purrr conventions are general good practice for any
R data pipeline and can be adopted or adapted independently of the GHG context.

**Typical use:** Provide a parsed methodology framework; receive a single R
code block with one `get_` function per calculation phase, a pipeline order
comment block, and a consolidated `# TODO:` section.

**Depends on:** Output from [`ghg-methodology-parser`](./ghg-methodology-parser/SKILL.md)
(or equivalent structured framework)

---

### [`r-methodology-narrator`](./r-methodology-narrator/SKILL.md)

Translates an R script into a plain-language methodology narrative readable by
domain experts who cannot read R code.

**The general pattern:** In scientific computing, R code frequently serves as
the authoritative record of a methodology — but the scientists who need to
review, approve, or replicate that methodology may not be R programmers. This
skill bridges that gap by producing a numbered narrative that describes what
the code does in domain terms, suitable for inclusion in a methods section,
technical report, or peer review package.

**Domain specificity:** This skill is lightly domain-specific. It uses GHG
inventory examples in its pattern table, but the core process — parse R code
into logical phases, describe each phase in plain language, flag assumptions
and ambiguities for reviewer confirmation — applies to any scientific R script.
It is the most transferable of the three skills in this repository.

**Typical use:** Paste or upload an R script; receive a Quarto/Word-ready
markdown document with a preamble, numbered methodology steps, inline reviewer
notes, and a closing summary of open questions.

---

## The Parser → Coder Pipeline

The `ghg-methodology-parser` and `r-ghg-coder` skills are designed to work in
sequence as a **methodology-to-code pipeline**:

```
Inventory chapter (PDF / text)
        │
        ▼
 ghg-methodology-parser
        │
        ▼
 Structured implementation framework (markdown)
        │
        ▼
    r-ghg-coder
        │
        ▼
 Production R code (targets-ready)
```

The parser produces a language-agnostic framework; the coder consumes it and
applies R-specific conventions. Keeping these as separate skills means the
framework step is auditable on its own — a domain scientist can review the
parsed framework before any code is written.

The `r-methodology-narrator` skill closes the loop in the other direction:
given finished R code, it produces a human-readable description that can be
reviewed by the same domain scientists who wrote the original methodology
narrative.

---

## Using These Skills

1. Create a Claude Project (available on Claude Pro, Team, and Enterprise plans)
2. Upload the relevant `SKILL.md` file(s) to the project's knowledge base
3. In your conversation, reference the skill by name or simply begin the task —
   Claude will apply the skill's process and conventions automatically

Skills can be combined in a single project. For example, loading both
`ghg-methodology-parser` and `r-ghg-coder` into one project allows Claude to
move from narrative input to R code output in a single conversation.

---

## Adapting These Skills

These skills are published as working examples of a general pattern. If you
want to adapt them for a different domain or codebase:

- **For the parser:** Replace the GHG inventory context section and the
  "Common Patterns" table with domain-specific equivalents. The five-step
  process (scope → inputs → calculations → uncertainty → gaps) is general.
- **For the coder:** Replace the R conventions section with your own codebase's
  style guide. The function-per-phase structure and the `# TODO:` convention
  are general good practice.
- **For the narrator:** This skill requires minimal adaptation. Update the
  pattern table with any domain- or codebase-specific patterns if needed.

---

## Author

[Joe Wasserman](https://github.com/josephwasserman) — Physical Scientist &
Data Scientist, with a focus on greenhouse gas inventory methodology and
reproducible R data pipelines.
