---
name: r-minimize-conditionals
description: Encodes a strong R style preference to avoid scalar if/else control flow — especially nested if/else — whenever a cleaner alternative exists (switch(), named-vector/lookup tables, dplyr::case_when(), guard clauses/early returns, function dispatch tables, match.arg()). Use this skill any time you are writing new R code, refactoring existing R code, or reviewing/critiquing R code for style, even if the user doesn't explicitly ask about conditionals — check every branch/control-flow point against this preference before finalizing R code. Does NOT apply to vectorized ifelse() or dplyr::if_else()/case_when(), which are always fine and preferred as-is. Applies to all R work, not just GHG-inventory scripts.
---

# Minimize `if`/`else` in R Code

## Core preference

The user finds nested `if`/`else` control flow hard to read and wants it avoided
**whenever a clean alternative exists**. This is a default posture to apply
proactively while writing or reviewing any R code — not something to wait to be
asked about.

This is a preference about **scalar control-flow branching**, not about
vectorized conditional recoding. Keep that distinction sharp:

- **Always fine, no change needed:** `ifelse()`, `dplyr::if_else()`,
  `dplyr::case_when()`. These are vectorized, already idiomatic, and not what
  this skill is about. Never "fix" these into something else.
- **Target for reduction:** scalar `if (...) { ... } else if (...) { ... } else { ... }`
  used for control flow, especially when nested more than one level deep.

## Before writing a scalar `if`, check for these alternatives first

**Step zero — check if the whole function is just condition→value mapping.**
Before reaching for guard clauses or an `if/else if` chain, ask: does this
function (including any validation/sentinel checks) reduce to "given x,
return one of several fixed outputs based on which condition matches first"?
If so, `dplyr::case_when()` can absorb the *entire* thing — sentinels
included — not just the "happy path" range logic. Don't split this into
"guard clauses handle validation, then an if/else chain handles the real
logic" if a single `case_when()` can express all of it as one ordered list
of condition → result pairs:

```r
# Instead of splitting into guard clauses + a separate if/else chain:
classify_reading <- function(x) {
  if (is.na(x)) return("missing")
  if (x < 0) return("invalid")
  if (x < 50) "low" else if (x < 100) "medium" else "high"
}

# Prefer one unified case_when() when everything is condition -> value:
classify_reading <- function(x) {
  dplyr::case_when(
    is.na(x) ~ "missing",
    x < 0    ~ "invalid",
    x < 50   ~ "low",
    x < 100  ~ "medium",
    TRUE     ~ "high"
  )
}
```
This is a bigger win than it looks: it's vectorized for free (works if `x`
is a vector, not just a scalar), and it reads as a single ordered spec rather
than two structurally different pieces (guards, then a chain). Only fall
back to guard clauses / a flat `if/else if` chain (below) when the function
has real side effects, differing types of control flow, or logic that isn't
cleanly expressible as "first matching condition wins."

1. **Multiple discrete string/scalar branches → `switch()`**
   ```r
   # Instead of:
   if (unit == "kg") {
     val * 1
   } else if (unit == "g") {
     val * 0.001
   } else if (unit == "lb") {
     val * 0.453592
   } else {
     stop("Unknown unit")
   }

   # Prefer:
   switch(unit,
     kg = val * 1,
     g  = val * 0.001,
     lb = val * 0.453592,
     stop("Unknown unit")
   )
   ```

2. **Mapping input values to output values → named vector / lookup table**
   ```r
   # Instead of nested if/else on a category:
   unit_factors <- c(kg = 1, g = 0.001, lb = 0.453592)
   val * unit_factors[[unit]]
   ```

3. **Row-wise / vectorized recoding → `dplyr::case_when()`** (already
   preferred by the user; just don't reach for scalar `if` inside a `mutate()`
   when `case_when()` will do the whole column at once).

4. **Validation / bailout logic → guard clauses (early return), not nesting**
   ```r
   # Instead of:
   process <- function(x) {
     if (!is.null(x)) {
       if (is.numeric(x)) {
         # ... deep logic ...
       } else {
         stop("x must be numeric")
       }
     } else {
       stop("x must not be NULL")
     }
   }

   # Prefer flattening with early returns/stops:
   process <- function(x) {
     if (is.null(x)) stop("x must not be NULL")
     if (!is.numeric(x)) stop("x must be numeric")
     # ... main logic, unindented ...
   }
   ```
   Use this pattern when there's real subsequent logic after the checks
   (as above). If the checks are themselves just more branches whose only
   job is to produce a return value — i.e. there's no separate "main logic"
   after them — that's actually the Step Zero `case_when()` case above, not
   this one. The tell: does anything happen *after* the last check, or was
   the last check already the final answer?
   This still uses `if`, but eliminates nesting — each check stands alone at
   the top level instead of wrapping the "happy path" in progressively deeper
   braces.

5. **Choosing which function/behavior to run → dispatch table (named list of functions)**
   ```r
   # Instead of:
   if (method == "linear") {
     fit_linear(data)
   } else if (method == "loess") {
     fit_loess(data)
   } else {
     fit_default(data)
   }

   # Prefer:
   dispatch <- list(linear = fit_linear, loess = fit_loess)
   (dispatch[[method]] %||% fit_default)(data)
   ```
   (or use `match.arg()` + `switch()` for the classic "which method" argument
   pattern in a function signature).

6. **S3/S4 method dispatch** — if branching is really "different logic per
   class/type," consider whether an S3 method or `switch()` on `class(x)` is
   more appropriate than an `if`/`else` chain checking `is.data.frame()`,
   `is.list()`, etc.

## When plain `if`/`else` is genuinely fine

Don't force an alternative where none is clean. Scalar `if` is acceptable
(and often unavoidable) for:

- A single, non-nested boolean check with no real alternative structure
  (e.g. `if (verbose) message("...")`).
- Side-effecting control flow (printing, writing files, early `return()`,
  `stop()`/`warning()` triggers) — these aren't really "branches producing a
  value" and don't map cleanly onto `switch()` or lookup tables.
- Genuine one-off logic where a lookup table or `switch()` would be more
  convoluted than the `if` itself (e.g. a single truly ad-hoc condition with
  no repeating pattern).
- **Numeric range/cutoff logic** (e.g. classifying a value into "low" /
  "medium" / "high" bands). A **flat, unnested `if (x < 50) "low" else if
  (x < 100) "medium" else "high"` chain is preferred over clever tricks**
  like `findInterval()` + `switch()` if the trick requires awkward glue code
  (off-by-one index arithmetic like `+ 1`, remembering `switch()`'s
  1-indexing, etc.). The goal is readability, not cleverness — a sequential
  `if/else if/else` with no nesting is already a good outcome and should not
  be "optimized" away into something harder to read at a glance.

In these cases, use a plain `if`/`else if` chain (or a guard clause per #4
above to keep validation logic unnested) rather than forcing a
switch/lookup-table workaround. Flag briefly in comments or in your
explanation *why* an `if` was kept if it's a case that might look avoidable
at a glance but isn't.

**Rule of thumb:** the target is *nesting*, not the mere presence of the
word `if`. A sequential, unnested `if/else if/else` chain is already fine.
Only replace it with `switch()`/lookup tables/dispatch tables when that
alternative is at least as readable — never trade a flat, clear `if` chain
for a "clever" one-liner that needs extra arithmetic or indirection to work.

## When reviewing/refactoring existing R code

When asked to review, refactor, or critique R code, actively scan for:
- Nested `if`/`else` (2+ levels) — always flag and propose a flattened or
  alternative structure.
- Chains of `if (x == "a") ... else if (x == "b") ...` on the same variable —
  flag as a `switch()` or lookup-table candidate.
- Scalar `if`/`else` inside `dplyr::mutate()` or `purrr::map()` where
  `case_when()` or a vectorized alternative would replace it entirely.

Do not flag `ifelse()`/`if_else()`/`case_when()` usage — leave those alone.
