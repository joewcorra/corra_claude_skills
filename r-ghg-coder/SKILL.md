---
name: r-ghg-coder
description: >
  Translates a structured GHG inventory implementation framework (as produced
  by the ghg-methodology-parser skill) into production-ready R code following
  the project's established conventions. Use this skill whenever a user provides
  a parsed methodology framework and asks for R code, asks to "code this up",
  "implement this in R", "turn this into a function", or "write the R script for
  this source category". Also trigger when the user provides calculation steps
  with inputs/outputs described and asks for an R implementation, even if the
  ghg-methodology-parser was not explicitly used. Do NOT use for Python, Excel,
  or other environments — this skill is R-only.
---

# R GHG Inventory Coder

Converts a structured implementation framework into production-ready R code
following the project's established conventions. The output is a complete,
targets-pipeline-ready R script composed of `get_` functions — one per logical
calculation phase — that can be dropped directly into the inventory codebase.

---

## Guiding Principles

- **Convention-first**: Every output must follow the established R conventions
  described in this skill. Do not introduce patterns not present in the
  template. When in doubt, default to simplicity and consistency over
  cleverness.
- **Framework-faithful**: Implement exactly what the parsed framework specifies.
  Do not add, remove, or reorganize calculation steps. If a step is ambiguous,
  implement the most defensible interpretation and flag it with a comment.
- **Pipeline-ready**: All functions must be composable in a `targets` pipeline.
  Each function takes named dataframes and/or a named constants vector as
  arguments; each returns a single tibble.
- **Unitless calculations**: All data arrives in canonical units — unit
  conversion is handled upstream at ingestion, never inside these functions.
  The constants vector holds emission factors, GWPs, and dimensionless
  parameters only. Never embed unit conversion factors inside `get_` functions.
- **Explicit over implicit**: Emission factor lookups must always be named and
  traceable via the constants vector. Never embed unexplained magic numbers.
- **Flag gaps**: Where the framework leaves something unresolved — a missing
  data source, an ambiguous join key, an unspecified interpolation method —
  flag it with an inline `# TODO:` comment rather than silently guessing.

---

## R Conventions

These conventions are derived from the project codebase and must be followed
exactly.

### Function Structure

- One function per logical calculation phase, named `get_[descriptive_name]`.
- Function names use snake_case and describe the output, not the process:
  `get_tire_incineration`, not `calculate_tire_incineration`.
- Each function accepts named dataframe arguments plus, where needed, a named
  constants vector (e.g., `incineration_factors`).
- Each function returns a single tibble via an explicit `return()` call.
- No side effects — functions do not write files, print, or modify global state.

```r
get_[output_name] <- function([input_df_1],
                               [input_df_2],
                               [constants_vector]) {
  
  result <- [input_df_1] %>%
    ...
  
  return(result)
  
}
```

### Package Preferences

- Use **tidyverse functions preferentially** throughout.
- Use `{purrr}` (`map()`, `map_dfr()`, `walk()`, etc.) over `apply()` family
  functions for iteration. Use `{purrr}` over `for` loops for iteration.
- Use `pull()` over the `$` operator for extracting columns.
- Use `lubridate::as_factor()` to convert year columns to factors if not
  already in factor form.

### Data Format and Character Conventions

- **Long format** for time series data. Data should arrive in long format;
  do not pivot to wide inside these functions.
- **Year columns** must be factors. Use `lubridate::as_factor(year)` if
  a year column is created inside a function (e.g., when scaffolding a time
  series with `tibble(year = X:Y)`).
- **Character data**: all lowercase, no punctuation except hyphens.
- **Variable names**: snake_case throughout.

### `syrinx` Package

`{syrinx}` is an internal package providing inventory-specific utilities.
Key functions:

- `syrinx::pre_clean()`: standardizes column names, removes empty
  rows/columns, squishes whitespace, lowercases character columns, and
  converts any column matching `"year"` to a factor. Call at the end of a
  function pipeline when output will be passed to downstream functions or
  stored. Typically unnecessary if input data is already clean, but include
  when in doubt.

Do not flag `syrinx::` calls as unknown dependencies — it is a known
internal package. Do flag any other custom/internal package calls with a
`# TODO:` comment.

### Pipe Style

- Use `%>%` (magrittr) throughout. Do not use `|>`.
- Chain operations within a single `%>%` pipeline where the logic flows
  naturally. Break into separate assignments when an intermediate result is
  reused or when a pipeline becomes difficult to read.
- Always `ungroup()` after `group_by()` + `summarize()` chains.

### Joining

- Always use `left_join(..., by = "[key]")` with an explicit `by` argument.
- Trim joining dataframes to only the columns needed before joining:
  `left_join(df %>% select(year, value), by = "year")`.
- The primary join key is typically `year`; use additional keys when the
  framework specifies disaggregation dimensions.

### Grouping and Summarizing

- Always pair `group_by()` with `ungroup()` after the summarize step.
- Pattern:
  ```r
  df %>%
    group_by(year, [dimension]) %>%
    summarize([output] = [expression], .groups = "drop") %>%
    ungroup()
  ```

### Conditional Logic

- Use `case_when()` for multi-branch conditionals, `if_else()` for binary.
- `case_when()` must always include a `.default` clause.
- When a `case_when()` branch depends on a value computed in an earlier
  branch of the same `mutate()` call, split into separate `mutate()` calls
  (see interpolation pattern below).

### Interpolation / Gap-Filling

Use `lag` and `lead` inside `mutate()` for linear interpolation of missing
years. Split into multiple `mutate()` calls when later years depend on
earlier imputed values:

```r
mutate(value = case_when(
  year == [gap_year] & is.na(value) ~ lag(value, 1) + ((lead(value, N) - lag(value, 1)) / N),
  is.na(value) ~ lag(value, 1) + ((lead(value, 1) - lag(value, 1)) / 2),
  .default = value)) %>%
mutate(value = if_else(
  year == [dependent_year] & is.na(value),
  lag(value, 1) + ((lead(value, N) - lag(value, 1)) / N),
  value))
```

### Constants and Emission Factors

- All emission factors, GWPs, and dimensionless parameters are passed in as a
  named numeric vector (e.g., `combustion_factors`, `incineration_factors`).
- Reference values by name: `constants["c_content_rubber"]`, never as bare
  numbers.
- **No unit conversion factors** belong in this vector — all data arrives in
  canonical units. If a framework step describes a unit conversion, flag it
  with a `# TODO:` noting that the conversion should be handled at ingestion.

### Time Series Scaffolding

When the framework requires a full time series, scaffold with a year-factor
column:

```r
tibble(year = lubridate::as_factor(seq([start_year], [end_year]))) %>%
  left_join(..., by = "year")
```

### Pre/Post Period Methodology Splits

Use `case_when()` with year thresholds as the primary pattern:

```r
mutate(value = case_when(
  year <= [threshold_year] ~ [method_a_expression],
  .default = [method_b_expression]))
```

### Source/Subsource Grouping with `lst()`

Within an inventory section, sources and subsources are grouped using
`lst()`. The section-level function returns a named list where each element
is a source tibble; subsources are nested elements within that list.

```r
get_[section] <- function(...) {
  
  source_a <- get_[source_a](...)
  source_b <- get_[source_b](...)
  
  lst(source_a, source_b)
  
}
```

Use this pattern whenever the framework describes multiple sources being
compiled into a section. Do not `return()` explicitly when using `lst()` as
the final expression — `lst()` returns the named list implicitly.

### Helper Functions

- Helper functions used across both national- and state-level compilation
  live in a shared script (not in the national or state function scripts).
- Helper functions specific to one level live in the corresponding script.
- Name helper functions descriptively in snake_case; prefix with `h_` only
  if needed to distinguish from `get_` pipeline functions.

### Output Column Naming

- **No unit suffixes in column names.** Do not append `_kt`, `_mt`, `_mmt`,
  or any other unit indicator to column names. Unit context belongs in the
  data dictionary, not in variable names. Write `waste_composted`, not
  `waste_composted_kt`; `ch4_emissions`, not `ch4_emissions_kt`.
- Emission columns: `[gas]_emissions` (e.g., `co2_emissions`, `ch4_emissions`,
  `n2o_emissions`). CO₂-equivalent columns: `[gas]_co2e`.
- Activity data columns: descriptive snake_case matching the framework's
  variable names (e.g., `tires_incinerated`, `waste_composted`).
- Intermediate calculation columns may be retained in the returned tibble
  for traceability; do not prematurely drop them unless the framework
  specifies a clean output.
- Do not embed unit commentary inline (e.g., `# result is in kt`). If a
  unit assumption needs flagging, put it in the TODOs block instead.

### Iteration

- Use `{purrr}` functions (`map()`, `map_dfr()`, `map2()`, `walk()`, etc.)
  for all iteration over lists or vectors.
- Do not use `for` loops or `apply()` family functions.
- When iterating to produce a combined tibble, prefer `map_dfr()` or
  `map()` followed by `bind_rows()`.

---

## Process

### Step 1 — Parse the Framework

Before writing any code, read the full implementation framework and extract:

1. **Function boundaries**: Identify the distinct calculation phases. Each
   phase that produces a reusable intermediate result becomes its own
   `get_` function.
2. **Inputs per function**: For each function, list the dataframes and
   constants it consumes. Dataframe column names may need to be inferred
   from the framework's variable descriptions — use snake_case versions of
   the plain-English names.
3. **Output per function**: Identify the tibble each function returns and
   its key columns.
4. **Dependency order**: Establish which functions feed into which. This
   determines argument structure and pipeline order.
5. **Constants**: List all emission factors, GWPs, and conversion factors
   referenced. These will be arguments to the relevant functions.
6. **Gaps and ambiguities**: Note any steps where the framework is silent
   on join keys, column names, interpolation method, or unit conversions.
   These become `# TODO:` comments.

### Step 2 — Write the Functions

Write one `get_` function per calculation phase, in dependency order
(upstream functions first). Follow all conventions in the R Conventions
section above.

For each function:
- Open with a brief comment block (2–4 lines) describing what the function
  computes, its primary inputs, and its output.
- Implement each framework step as a pipeline stage or `mutate()` operation,
  in the same sequence as the framework.
- Add inline comments for non-obvious steps: methodology breaks, special
  cases, interpolation logic, and unit conversions.
- Close with `return([result_tibble])`.

### Step 3 — Add a Pipeline Comment Block

After all functions, add a comment block showing the intended `targets`
pipeline call order:

```r
# --- targets pipeline order ---
# 1. [constants_object]  ← define named vector externally
# 2. get_[phase_1]([inputs])
# 3. get_[phase_2]([phase_1_output], [other_inputs], [constants])
# ...
```

This is a comment only — do not write the `tar_target()` calls, as pipeline
wiring is handled in the main `_targets.R` file.

### Step 4 — Flag All TODOs

After the pipeline comment block, add a consolidated `# --- TODOs ---`
section listing every unresolved gap, ambiguous join, missing column name,
or external dependency that needs verification. Mirror the "Implementer Notes
and Gaps" section of the parsed framework.

---

## Output Format

Produce a single R code block. Do not split across multiple blocks. Structure:

```r
# =============================================================================
# [Sector] — [CRF Code] [Source Category Name]
# [GHG(s) covered]
# =============================================================================


# --- [Phase 1 description] ---------------------------------------------------

get_[phase_1] <- function(...) {
  ...
  return(...)
}


# --- [Phase 2 description] ---------------------------------------------------

get_[phase_2] <- function(...) {
  ...
  return(...)
}


# --- targets pipeline order --------------------------------------------------
# 1. ...
# 2. ...


# --- TODOs -------------------------------------------------------------------
# - ...
# - ...
```

---

## Handling Incomplete Frameworks

If the provided framework is missing sections (e.g., no uncertainty steps, no
named data sources), implement only what is present. Note omissions in the
TODOs block:

```r
# TODO: Uncertainty steps were not included in the parsed framework.
#       Add Approach 1 error propagation once uncertainty inputs are defined.
```

Do not fabricate steps for sections not provided.

---

## Common GHG Inventory Patterns → R Translations

| Framework description | R implementation pattern |
|---|---|
| "For each year in the time series [X]–[Y]" | `tibble(year = lubridate::as_factor(seq(X, Y))) %>% left_join(...)` |
| "Multiply activity data by emission factor" | `mutate(emissions = activity * constants["ef_name"])` |
| "Apply GWP weighting" | `mutate(co2e = emissions * constants["gwp_[gas]"])` |
| "Linear interpolation for missing years" | `lag`/`lead` pattern in sequential `mutate()` calls |
| "Pre-[year] method A; post-[year] method B" | `case_when(year <= threshold ~ ..., .default = ...)` |
| "Sum across [dimension] within year" | `group_by(year) %>% summarize(total = sum(value)) %>% ungroup()` |
| "Join [table A] to [table B] using year" | `left_join(table_b %>% select(year, col), by = "year")` |
| "Retain only records where [condition]" | `filter([condition])` |
| "Average of available years used for gap years" | `mutate(value = if_else(is.na(value), mean(value, na.rm = TRUE), value))` |
| "GHGRP data available from [year] onward; prior years use average" | `bind_rows()` of observed and pre-period tibbles with mean fill |
| "Repeat for each source / fuel type / category" | `purrr::map_dfr()` or `purrr::map()` + `bind_rows()` |
| "Compile sources within section" | `lst(source_a, source_b)` as final expression |
| "Extract single value from column" | `pull([column])` not `$[column]` |
