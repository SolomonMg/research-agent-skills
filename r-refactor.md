---
name: r-refactor
version: 1.0.0
description: |
  2-agent R code refactoring skill for research replication code and data
  visualization. Agent A modernizes tidyverse idioms, fixes anti-patterns,
  simplifies pipelines, and checks reproducibility. Agent B reviews ggplot2
  quality, theme consistency, color accessibility, table generation, and
  publication readiness. Optionally runs lintr/styler for automated
  detection. Supports review/diff/apply fix modes.
user-invocable: true
argument-hint: "[path/to/file.R or directory] [review|diff|apply] [strict|moderate|light]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Agent
  - AskUserQuestion
---

# R Refactor

Refactor and simplify R code for research replication packages and publication-quality visualization. Modernize tidyverse idioms, eliminate anti-patterns, simplify pipelines, and improve ggplot2 output. Designed for the kind of R code that accompanies empirical social science papers.

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` left-to-right:

### Recognized fix modes

| Token | Meaning |
|---|---|
| `review` | Interactive. For each proposed change, open VS Code at the location, display the current code and proposed replacement, ask user to approve or skip. |
| `diff` | Apply all changes, then open VS Code for review. If git-tracked, use Source Control. If not, create `.r-refactor-backup` and open `code -d backup file`. |
| `apply` | Apply all changes without opening VS Code. |

Default: `report` (read-only — produces a report with suggested changes but modifies nothing).

### Recognized strictness levels

| Token | Meaning |
|---|---|
| `strict` | Flag everything: style, modernization, performance, anti-patterns, visualization. Enforce modern dplyr 1.1+ idioms throughout. |
| `moderate` | Flag anti-patterns, performance issues, and visualization problems. Suggest but don't require modernization of working idioms. **(Default.)** |
| `light` | Only flag clear anti-patterns and bugs. Leave working code alone. |

### File path / directory

- If a token looks like an `.R`, `.r`, `.Rmd`, or `.qmd` file path, store as `TARGET_FILE`.
- If a token looks like a directory, store as `TARGET_DIR`.
- If neither is provided, use the current working directory.

Store: `FIX_MODE`, `STRICTNESS`, `TARGET_FILE` or `TARGET_DIR`.

## Phase 1: Discover R Files

### If `TARGET_FILE` is set:
Use that single file. Also scan the same directory for shared utilities, theme files, or helper scripts that the target file may source.

### If `TARGET_DIR` is set (or defaulting to cwd):
Use Glob to find:
- `**/*.R`, `**/*.r`
- `**/*.Rmd`, `**/*.qmd`

Exclude: `renv/`, `packrat/`, `.Rproj.user/`, `_site/`, `.quarto/`, `node_modules/`.

### Categorize files

For each R file, determine its role by scanning the first 50 lines and filename:
- **Data processing**: reads data, merges, filters, transforms, writes intermediate files
- **Analysis**: runs regressions, statistical tests, models
- **Visualization**: creates plots (contains `ggplot`, `plot(`, `ggsave`, `pdf(`)
- **Table generation**: creates formatted tables (contains `modelsummary`, `gtsummary`, `kable`, `stargazer`, `xtable`, `huxtable`)
- **Utility/helper**: contains function definitions sourced by other scripts
- **Master script**: orchestrates other scripts (`source(`)

Record: `R_FILES` (all files), `FILE_ROLES` (categorization), `LANGUAGES` (if non-R files are present).

### Run lintr (if available)

Check whether `lintr` is installed:
```bash
Rscript -e 'cat(requireNamespace("lintr", quietly=TRUE))'
```

If available, run on each file in scope:
```bash
Rscript -e 'cat(lintr::lint("FILE") |> as.data.frame() |> jsonlite::toJSON())'
```

Record lint results as `LINT_RESULTS`. If lintr is not installed, note this and proceed without it.

Before proceeding, tell the user:
- Files found and their roles
- Whether lintr was available and how many lint warnings were found
- Fix mode and strictness level

## Phase 2: Launch 2 Agents in Parallel

In a single message, launch both agents using the Agent tool with `subagent_type: "general-purpose"`. Both are read-only.

---

### AGENT A: Code Modernization and Simplification

Store output as `CODE_SUMMARY`.

Prompt:

> You are an expert R developer refactoring research code. Your goal: make the code simpler, more modern, more readable, and more reproducible — without changing what it does.
>
> **Project context:**
> - R files in scope: [insert `R_FILES` with roles]
> - Strictness: [insert `STRICTNESS`]
> - Lint results: [insert `LINT_RESULTS` or "lintr not available"]
>
> Read all files in scope. For each file, check the following categories. Only flag items appropriate for the strictness level.
>
> **1. Modernize tidyverse idioms**
>
> | Old pattern | Modern replacement | When to flag |
> |---|---|---|
> | `%>%` | `\|>` (native pipe) | strict only — working code is fine |
> | `group_by() + summarise() + ungroup()` | `summarise(..., .by = var)` | strict and moderate |
> | `left_join(a, b, by = "key")` | `left_join(a, b, by = join_by(key))` | strict only |
> | `left_join()` without `unmatched` or `multiple` args | Add `unmatched = "error"` or `multiple = "error"` where appropriate for data integrity | moderate and strict |
> | `spread()` / `gather()` | `pivot_wider()` / `pivot_longer()` | all levels |
> | `map_dfr()` / `map_dfc()` | `map() \|> list_rbind()` / `list_cbind()` | strict and moderate |
> | `do()` | `reframe()` or `summarise()` | all levels |
> | `funs()` in `mutate_at/summarise_at` | `across()` with lambda | all levels |
> | `mutate_at()` / `mutate_if()` / `mutate_all()` | `mutate(across(...))` | all levels |
> | Copy-pasting the same transform for N columns | `mutate(across(c(col1, col2, ...), transform))` | all levels |
>
> **2. Anti-patterns (flag at all strictness levels)**
>
> - `rowwise()` when vectorized alternatives exist (`rowSums()`, `rowMeans()`, `pmax()`, `pmin()`, `purrr::pmap()`)
> - `x == NA` instead of `is.na(x)`
> - `1:length(x)` or `1:nrow(df)` instead of `seq_along(x)` or `seq_len(nrow(df))`
> - `T` / `F` instead of `TRUE` / `FALSE`
> - `sapply()` (type-unstable) instead of `vapply()` or `map_*()`
> - `rbind()` or `c()` growing objects in a loop instead of pre-allocating a list
> - `rm(list = ls())` at the top of a script (breaks `source()` composability)
> - `setwd()` calls (hardcodes the working directory)
> - `library()` calls scattered throughout the file instead of grouped at the top
> - Assignment with `=` instead of `<-` (flag at strict; note at moderate)
> - `&` / `|` in `if()` / `while()` conditions instead of `&&` / `||`
> - Positional column indexing (`df[,3]`) instead of named columns
>
> **3. Pipeline simplification**
>
> - Multiple sequential `mutate()` calls that could be one `mutate(a = ..., b = ..., c = ...)`
> - Pipelines longer than ~10 steps — suggest breaking into named intermediates with clear intent
> - `filter()` + `filter()` that could be one `filter(cond1, cond2)`
> - `select()` immediately followed by `rename()` — combine into one `select(new = old)`
> - Repeated identical `read_csv()` calls — read once and reuse
>
> **4. Reproducibility checks**
>
> - Missing `set.seed()` before stochastic operations (sampling, bootstrapping, simulation, train/test splits, jittering)
> - Hardcoded absolute paths (`"C:/Users/..."`, `"/home/user/..."`, `"~/specific_project/..."`)
> - Missing `library()` declarations for packages used
> - No `sessionInfo()` or `renv` snapshot anywhere in the project
> - Undocumented magic numbers (thresholds, cutoffs, filter values) without comments
>
> **5. Statistical code patterns**
>
> - Manual coefficient extraction (`summary(model)$coefficients[2,1]`) → suggest `broom::tidy(model)`
> - Copy-pasted regression specifications that differ in one variable → suggest `purrr::map()` over a formula list
> - Manual p-value formatting → suggest `scales::pvalue()` or `gtsummary`
> - `summary()` on a model object printed but not captured → suggest `broom::tidy()` + `broom::glance()`
>
> **6. Code structure**
>
> - Functions longer than ~50 lines that do multiple things → suggest decomposition
> - Duplicated code blocks (3+ lines repeated) → suggest extracting a helper function
> - `source()` chains without a clear master script → note
>
> **Use these labels:**
> - **FIX**: clear anti-pattern or bug — should be changed
> - **MODERNIZE**: working code that uses deprecated or outdated idioms — improvement opportunity
> - **SIMPLIFY**: code that works but is unnecessarily complex — can be shortened
> - **REPRODUCE**: reproducibility gap — not a code quality issue but matters for replication
> - **STYLE**: formatting or naming convention issue
>
> **Output exactly these sections:**
>
> ## Summary
> 3-6 bullets on overall code quality and modernization state.
>
> ## Top Findings
> Up to 15 items, ordered by impact (FIX first, then SIMPLIFY, MODERNIZE, REPRODUCE, STYLE).
> Format: `- [LABEL] **Short title** — file:line(s) — current pattern — suggested replacement — why`
>
> ## File-by-File Notes
> For each file with findings, provide:
> - File name and role
> - Bullet list of specific changes (with line numbers and concrete before/after code snippets)
>
> ## Strengths
> 3-6 bullets on what's done well.

---

### AGENT B: Visualization and Output Quality

Store output as `VIZ_SUMMARY`.

Prompt:

> You are reviewing R code that produces figures and tables for an academic publication. Your goal: make plots publication-ready, accessible, consistent, and simple.
>
> **Project context:**
> - R files in scope: [insert `R_FILES` with roles — focus on visualization and table files]
> - Strictness: [insert `STRICTNESS`]
>
> Read files that produce plots or tables. Check:
>
> **1. ggplot2 simplification**
>
> | Pattern | Simplification |
> |---|---|
> | `xlab()` + `ylab()` + `ggtitle()` on same plot | `labs(x = ..., y = ..., title = ...)` |
> | `+ theme_minimal()` on every plot | `theme_set(theme_minimal(base_size = 12))` once at top of script |
> | Repeated identical `theme()` calls across plots | Extract a custom theme function: `theme_pub <- function() theme_minimal(base_size = 12) + theme(...)` |
> | Manual axis label formatting | `scale_x_continuous(labels = scales::comma)` or `scales::percent` |
> | `scale_color_manual(values = c("red", "blue", ...))` with >3 colors | Colorblind-safe palette: `scale_color_viridis_d()` or `scale_color_brewer(palette = "Set2")` |
> | Manual legend positioning with coordinates | `theme(legend.position = "bottom")` or `"inside"` with `legend.position.inside` |
> | `ggsave()` with `png` or low-dpi output | `ggsave("fig.pdf", width = 6, height = 4)` for publication |
> | `png()` / `dev.off()` device sandwich | `ggsave()` is simpler and auto-detects format |
> | Repeated `geom_*()` calls that could use `aes(group = ...)` or `facet_wrap()` | Suggest faceting or group aesthetic |
>
> **2. Color accessibility**
>
> - Flag any use of red/green color pairs (indistinguishable to ~8% of males)
> - Flag rainbow or heat color palettes
> - Flag default ggplot2 discrete colors when >4 categories (hard to distinguish)
> - Suggest: `viridis`, `ColorBrewer Set2/Dark2`, or `ggpubfigs` palettes
> - Check that color is not the only visual channel (use shape, linetype, or faceting as redundant encoding)
>
> **3. Publication readiness**
>
> - Are axis labels human-readable (not raw variable names like `gdp_pc_ppp`)?
> - Are units included in axis labels where needed?
> - Are confidence intervals shown for point estimates?
> - Is text large enough for the likely print size? (base_size ≥ 11 for single-column, ≥ 12 for full-page)
> - Are figure dimensions specified in `ggsave()` and appropriate for a journal column (~3.5 in single, ~7 in double)?
> - Output format: PDF for print publications, PNG only when raster elements require it
>
> **4. Theme consistency across figures**
>
> - Do all figures in the project use the same base theme?
> - Are font sizes consistent across figures?
> - Are color palettes consistent (same variable → same color throughout the paper)?
> - Are axis formatting conventions consistent (same decimal places, same label style)?
>
> **5. Table generation**
>
> - Manual table assembly (`paste()`, `sprintf()`, hand-formatted LaTeX) → suggest `modelsummary`, `gtsummary`, `kableExtra`, or `gt`
> - `stargazer` usage → note that `modelsummary` is more flexible and actively maintained
> - `broom::tidy()` results formatted manually → suggest `modelsummary(model_list)` for regression tables
> - Hard-coded significance stars → suggest `modelsummary` stars argument
> - Tables saved as `.csv` that will become LaTeX tables → suggest direct LaTeX output from `modelsummary` or `kableExtra`
>
> **6. Faceting and multi-panel figures**
>
> - Multiple separate plots manually combined → suggest `facet_wrap()` / `facet_grid()` if the data supports it, or `patchwork` / `cowplot::plot_grid()` for heterogeneous panels
> - `par(mfrow = ...)` from base R mixed with ggplot2 → these don't interact; use patchwork
> - Inconsistent axis ranges across related panels → suggest `facet_*` with shared scales, or manual `coord_cartesian(ylim = ...)` across plots
>
> **Use these labels:**
> - **FIX**: accessibility issue, wrong output format, or misleading visual encoding
> - **SIMPLIFY**: code that works but is more complex than needed
> - **CONSISTENCY**: inconsistency across figures/tables in the project
> - **PUBLISH**: not yet publication-ready (missing labels, low resolution, raw variable names)
>
> **Output exactly these sections:**
>
> ## Summary
> 3-6 bullets on visualization quality and publication readiness.
>
> ## Top Findings
> Up to 12 items, ordered by impact.
> Format: `- [LABEL] **Short title** — file:line(s) — current pattern — suggested replacement — why`
>
> ## Figure-by-Figure Notes
> For each visualization file, list specific changes with line numbers and before/after code.
>
> ## Table Assessment
> Summary of table generation quality and suggestions.
>
> ## Color Palette Assessment
> Are the palettes accessible and consistent? Specific flag for any red-green pair or rainbow palette.
>
> ## Strengths
> 3-6 bullets on what's done well in the visualizations.

---

## Phase 3: Synthesize

After both agents return, synthesize the results.

1. **Deduplicate.** Where Agent A flags a pipeline issue and Agent B flags the same code from a visualization angle, unify into one finding with both perspectives.

2. **Build the fix plan.** For each finding where `FIX_MODE` is not `report`, create a concrete edit entry:
   - File path and line number(s)
   - Current code (exact `old_string`)
   - Proposed replacement (`new_string`)
   - Label (FIX / SIMPLIFY / MODERNIZE / etc.)
   - Reason (one sentence)

   Order: FIX first, then SIMPLIFY, MODERNIZE, PUBLISH, CONSISTENCY, REPRODUCE, STYLE.

3. **Compile:**
   - `OVERALL_ASSESSMENT`: 3-5 sentences on code quality
   - `FIX_PLAN`: ordered list of concrete changes
   - `STRENGTHS`: merged from both agents

## Phase 3b: Fix Mode Gate

*Skip entirely if `FIX_MODE` is `report` (default).*

### Optional: Run styler first

If `FIX_MODE` is `diff` or `apply`, and `styler` is installed:
```bash
Rscript -e 'styler::style_file("FILE")'
```
Run on each file in scope before applying semantic changes. This handles formatting (spacing, indentation, braces) so the agent can focus on semantic refactoring.

### If `FIX_MODE` is `review`:

For each item in the fix plan:
1. Run `code -g FILE:LINE` via Bash.
2. Display: label, file:line, current code, proposed replacement, reason.
3. Use AskUserQuestion: "Apply this change? (yes / skip / stop)"
   - **yes** → Apply with Edit tool.
   - **skip** → Next.
   - **stop** → End.
4. Track applied vs. skipped.

### If `FIX_MODE` is `diff` or `apply`:

Launch a **single fixer agent** (`subagent_type: "general-purpose"`) with Edit permissions.

> You are applying reviewed refactoring changes to R code. Each change has been vetted by two review agents. Apply them exactly as specified. Do not change code logic — only modernize patterns, simplify pipelines, and fix anti-patterns as described.
>
> **Fix plan:** [insert fix plan]
>
> **Rules:**
> 1. Apply each fix using Edit with the exact `old_string` and `new_string`.
> 2. If `old_string` cannot be found (code may have shifted from styler), try to find the equivalent code and apply the semantic change. If you cannot, skip and report.
> 3. Do not make changes beyond the fix plan.
> 4. Report: number applied, number skipped, list of skipped items.

After the fixer agent returns:
- `diff`: open VS Code (`code -g FILE:1` for each modified file); tell user to use Source Control for per-hunk revert. If not git-tracked, create `.r-refactor-backup` first and `code -d backup file`.
- `apply`: report summary only.

Record as `FIX_RESULTS`.

## Phase 4: Write the Report

Write to:

`r_refactor_report_[YYYY-MM-DD].md`

```markdown
# R Refactor Report

*Reviewed: [date] | Files: [N] | Strictness: [STRICTNESS] | Fix mode: [FIX_MODE]*

---

## Overall Assessment

[3-5 sentences. Lead with strengths. Then main opportunities for improvement.]

**Code quality**: [Modern & clean | Good with minor updates | Needs modernization | Significant refactoring needed]

## What's Working Well

- [Strength 1]
- [Strength 2]
- ...

---

## Code Modernization & Simplification

[Top findings from Agent A, organized by file or by theme]

### Modernization Opportunities

[Table or bullet list of old → new pattern replacements]

### Anti-Patterns

[Specific issues with file:line and fix]

### Reproducibility Notes

[Seeds, paths, dependencies]

---

## Visualization & Output Quality

[Top findings from Agent B]

### Figure Assessment

[Per-figure notes]

### Color Accessibility

[Palette assessment]

### Table Generation

[Table code assessment]

### Publication Readiness Checklist

| Check | Status | Notes |
|---|---|---|
| PDF output format | ✓/✗ | ... |
| Colorblind-safe palettes | ✓/✗ | ... |
| Axis labels human-readable | ✓/✗ | ... |
| Confidence intervals shown | ✓/✗ | ... |
| Theme consistency across figures | ✓/✗ | ... |
| Text size appropriate | ✓/✗ | ... |
| Figure dimensions specified | ✓/✗ | ... |

---

## Fixes Applied

*Include only if `FIX_MODE` is not `report`.*

| # | File | Line | Change | Label | Status |
|---|---|---|---|---|---|
| 1 | ... | ... | ... | FIX/SIMPLIFY/... | Applied/Skipped |

**Summary:** X applied, Y skipped.

---

## All Suggested Changes

*Complete list for reference, regardless of fix mode.*

| # | File | Line | Current | Suggested | Label | Reason |
|---|---|---|---|---|---|---|
| 1 | ... | ... | `old code` | `new code` | ... | ... |

---

## Appendix: Agent Outputs

### A. Code Modernization & Simplification
[Paste `CODE_SUMMARY`]

### B. Visualization & Output Quality
[Paste `VIZ_SUMMARY`]

### Lint Results
[Paste `LINT_RESULTS` summary or "lintr not available"]
```

## Final User Message

After writing the report, tell the user:
- That the refactoring review is complete
- The path to the saved report
- The overall assessment
- Total findings by label (FIX / SIMPLIFY / MODERNIZE / PUBLISH / etc.)
- If fixes were applied: how many applied, skipped
- The top 5 suggested changes
- Whether lintr/styler were available and used
- If `FIX_MODE` is `diff`: remind user to review in VS Code Source Control
