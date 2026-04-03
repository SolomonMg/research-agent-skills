---
name: review-paper-code
version: 1.0.0
description: |
  3-agent review of research code for reproducibility, data integrity,
  analytical correctness, and paper-code alignment. Agent A checks
  reproducibility and execution feasibility. Agent B reviews data
  processing logic, statistical methods, and common pitfalls (leakage,
  missing data, magic numbers). Agent C maps the paper's empirical
  claims to code and verifies that outputs exist and match. Produces
  an impact-tiered markdown report. Supports Stata, R, Python, Julia,
  MATLAB, SAS, notebooks, and shell scripts.
user-invocable: true
argument-hint: "[path/to/main.tex] [path/to/code_dir] [main|full] [econ|polisci|ml|biostats|general]"
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

# Review Paper Code

Review a research project's paper and code for reproducibility, data integrity, analytical correctness, and paper-code alignment. Be constructive, concrete, and calibrated. Treat gaps as items to verify, not accusations.

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` left-to-right:

### Recognized tokens

**Review depth** (case-insensitive):

| Token | Meaning |
|---|---|
| `main` | Prioritize main scripts and core outputs (default) |
| `full` | Inspect all detected code files |

**Domain** (case-insensitive):

| Token | Analytical focus |
|---|---|
| `econ` | Economics / finance: IV, RDD, DID, panel FE, clustering, Stata conventions |
| `polisci` | Political science: survey experiments, observational causal inference, text-as-data |
| `ml` | Machine learning: train/test splits, leakage taxonomy, cross-validation, metric selection |
| `biostats` | Biostatistics / epidemiology: survival analysis, propensity scores, multiple testing, E-values |
| `general` | General social science (default) |

**File path and directory:**
- If a token looks like a `.tex`, `.qmd`, `.Rmd`, or `.md` path, store it as `PAPER_FILE`.
- If a token looks like a directory path, store it as `CODE_DIR`.

**Defaults:** If no depth token is found, default to `main`. If no domain token is found, default to `general`.

Store resolved values as `PAPER_FILE`, `CODE_DIR`, `REVIEW_DEPTH`, and `DOMAIN`.

## Phase 1: Discover the Project

### 1. Find the paper

Use Glob to search for candidate paper files, excluding build and cache folders (`_minted-*`, `build/`, `output/`, `.git/`, `node_modules/`, `__pycache__/`, `.quarto/`, `_site/`):

1. `**/*.tex`
2. `**/*.qmd`
3. `**/*.Rmd`

Identify the main paper file:
- For `.tex`: contains `\documentclass` or `\begin{document}`
- For `.qmd` / `.Rmd`: contains YAML frontmatter with `title:`

If multiple candidates, prefer:
1. A path explicitly provided in `$ARGUMENTS`
2. A file in `Writing/`, `writing/`, `Paper/`, `paper/`, `Draft/`, `draft/`, or the repo root
3. The file that includes the most component files via `\input{}` / `\include{}` / `\subfile{}`

Record as `PAPER_FILE`.

### 2. Find the code directory

If `CODE_DIR` was not provided, look for likely code roots in order:
- `Code/`, `code/`, `Analysis/`, `analysis/`, `scripts/`, `Scripts/`
- `src/`, `programs/`, `Programs/`, `replication/`, `Replication/`
- `do/`, `R/`, `python/`, `stata/`

If no single directory is clearly best, use the repo root.

Record as `CODE_DIR`.

### 3. Find code files

Within `CODE_DIR` and subdirectories, find all files matching:

| Pattern | Language |
|---|---|
| `**/*.do` | Stata |
| `**/*.R`, `**/*.r` | R |
| `**/*.py` | Python |
| `**/*.jl` | Julia |
| `**/*.m` | MATLAB |
| `**/*.sas` | SAS |
| `**/*.ipynb` | Jupyter notebook |
| `**/*.Rmd`, `**/*.qmd` | RMarkdown / Quarto (analysis notebooks) |
| `**/*.sh`, `**/*.bash` | Shell (pipeline orchestration) |

Exclude: `__pycache__/`, `.ipynb_checkpoints/`, `renv/`, `venv/`, `.venv/`, `env/`, `node_modules/`, `.git/`.

If `REVIEW_DEPTH = main`, prioritize:
- Master scripts: `main.*`, `master.*`, `run_all.*`, `run.*`, `Makefile`, `makefile`, `_targets.R`
- Files referenced or called by master scripts
- Files whose names suggest they produce tables or figures (`table_*.R`, `fig_*.py`, `*_results.*`)
- Cap the initial review set at ~15 files; note the remainder as out-of-scope

If `REVIEW_DEPTH = full`, include all detected code files.

Record:
- `CODE_FILES_ALL` (every code file found)
- `CODE_FILES_REVIEWED` (files selected for review)
- `LANGUAGES` (languages present)

### 4. Find output artifacts

Search for generated outputs:
- `**/*.csv`, `**/*.dta`, `**/*.rds`, `**/*.parquet`, `**/*.arrow` in data/output directories
- Figures: `**/*.pdf`, `**/*.png`, `**/*.eps`, `**/*.svg` in `figures/`, `Figures/`, `output/`, `results/`, or root
- Tables: `**/*.tex`, `**/*.csv`, `**/*.html` in `tables/`, `Tables/`, `output/`, `results/`
- Logs: `**/*.log`, `**/*.txt` in `log/`, `logs/`, `output/`

Record as `OUTPUT_FILES`.

### 5. Find supporting documentation

Look for:
- `README.md`, `README.txt`, `readme.md`, `README`
- `requirements.txt`, `environment.yml`, `pyproject.toml`, `setup.py`, `setup.cfg`
- `renv.lock`, `DESCRIPTION`, `pkg.lock`
- `Makefile`, `Dockerfile`, `docker-compose.yml`
- `.env.example`, `config.yml`, `config.json`

Record as `SUPPORT_FILES`.

### 6. Handle ambiguity

If you find a paper and at least some code, continue even if discovery is imperfect.

Only stop if you cannot find either:
- a main paper file, or
- any relevant code files

If you stop, tell the user what was missing and what paths they can pass explicitly.

Before proceeding, tell the user:
- The paper file chosen
- The code directory chosen
- Number of code files detected vs. selected for review
- Languages found
- Output artifacts found (count by type)
- Review depth and domain
- Any ambiguity worth noting

## Phase 2: Read the Paper

Read `PAPER_FILE` and all files it references via `\input{}`, `\include{}`, `\subfile{}`, or Quarto `{{< include >}}`.

Extract a working summary organized for cross-checking. This must go deeper than headline claims.

### PAPER_SUMMARY structure

**Metadata:**
- Paper title, authors, abstract

**Research design:**
- Research question
- Main data sources (name, time period, level of observation)
- Sample restrictions and filters (stated N, exclusion criteria)
- Main dependent variables (name and definition)
- Main explanatory variables or treatments (name and definition)
- Main estimation methods (OLS, IV, DID, RDD, ML model, etc.)
- Fixed effects and clustering (if applicable)
- Control variables (if enumerated)

**Quantitative claims (extract up to 20):**
For each main table and figure referenced in the results section, extract:
- Table/figure number and brief description
- Key coefficients or statistics cited in the prose (with magnitude, direction, significance)
- Sample size stated for that specification
- Any footnote caveats (sample restrictions, variable definitions, SE type)

**Robustness and supplementary claims:**
- Every explicit robustness claim ("results robust to X", "alternative specification Y")
- Appendix analyses referenced in the main text
- Subgroup analyses mentioned in the discussion
- Placebo or falsification tests described

**Hardcoded numbers scan:**
- Grep the paper source for numbers that appear in prose but are not generated by `\input{}` of a table file. These are candidates for manual transcription from code output and are high-priority verification targets.

Store as `PAPER_SUMMARY`.

## Phase 3: Launch 3 Agents in Parallel

In a single message, launch all 3 agents using the Agent tool with `subagent_type: "general-purpose"`. Each agent is read-only. Pass complete file lists and the paper summary to each agent. Substitute the resolved `DOMAIN` value into Agent B's prompt.

---

### AGENT A: Reproducibility and Execution Feasibility

Store output as `REPRODUCIBILITY_SUMMARY`.

Prompt:

> You are reviewing research code for reproducibility and execution feasibility. Your job is to determine: could a new researcher clone this repo and reproduce the results?
>
> **Project context:**
> - Code files in scope: [insert `CODE_FILES_REVIEWED`]
> - All code files found: [insert `CODE_FILES_ALL`]
> - Output artifacts found: [insert `OUTPUT_FILES`]
> - Supporting documentation: [insert `SUPPORT_FILES`]
> - Languages: [insert `LANGUAGES`]
>
> Read the code files in scope and produce a compact, high-signal report.
>
> **Check each of the following:**
>
> **1. Execution feasibility (can this code plausibly run?)**
> - Missing imports, undefined variables, or references to packages not in any requirements file
> - Syntax errors detectable by reading (unmatched brackets, Python 2/3 issues, deprecated function calls)
> - References to data files that do not exist in the repo
> - References to output directories that do not exist and are not created by the code
> - Missing entry point: is there a clear way to run the pipeline from start to finish?
>
> **2. Path and environment assumptions**
> - Hardcoded absolute paths or machine-specific assumptions (usernames, drive letters, specific mount points)
> - Implicit working directory assumptions (does the code assume it is run from a specific directory?)
> - OS-specific commands or path separators that limit portability
>
> **3. Reproducibility infrastructure**
> - Random seeds: are all stochastic procedures seeded? Check for random number generation, sampling, bootstrapping, train/test splits, simulation draws. Note whether seeds are set locally, globally, or not at all.
> - Dependency management: are software versions pinned? Is there a requirements file, lock file, or environment specification?
> - Run order: is there a master script, Makefile, or documented sequence? If not, can the order be inferred from file naming or comments?
> - Documentation: does the README explain how to run the code? Are there inline comments explaining non-obvious steps?
>
> **4. Cross-language pipeline integrity** (if multiple languages are present)
> - Do intermediate files written by one language get read by another? Are file formats consistent?
> - Do column names, variable names, or data encodings survive the handoff?
> - Are there implicit ordering dependencies between scripts in different languages?
>
> **5. Output traceability**
> - Can each output file (table, figure, dataset) be traced back to a specific script?
> - Are there output files in the repo that no script appears to generate (stale artifacts)?
> - Are there scripts that write outputs not found in the repo (missing artifacts)?
>
> **Use these labels:**
> - **PASS**: looks solid
> - **NOTE**: minor improvement opportunity
> - **VERIFY**: worth human confirmation — may be fine, may be a problem
> - **MISSING**: expected file, documentation, or infrastructure element is absent
> - **FAIL**: definite problem that would block execution or reproduction
>
> **Output exactly these sections:**
>
> ## Overall
> 3-6 bullets on the overall reproducibility posture.
>
> ## Top Findings
> Up to 12 items, ordered by severity (FAIL first, then VERIFY, then NOTE).
> Format: `- [LABEL] **Short title** — file(s): line(s) if available — what the issue is — why it matters — what to check or fix`
>
> ## Execution Feasibility
> Assess whether the code could plausibly run on a clean machine with the stated dependencies. List specific blockers if any.
>
> ## Reproducibility Checklist
> One line each, using format: `- Check name: LABEL — brief note`
> - Entry point / master script
> - Run order documented
> - Relative file paths
> - Random seed practice
> - Dependency management (versions pinned)
> - Output files traceable to scripts
> - Cross-language handoffs (if applicable)
> - README / documentation
>
> ## Strengths
> 3-8 bullets with genuine positives. Be specific.

---

### AGENT B: Data Integrity and Analytical Logic

Store output as `ANALYSIS_SUMMARY`.

Prompt:

> You are a methodologist reviewing research code for data integrity and analytical correctness. You are not checking infrastructure (paths, seeds, dependencies) — a separate agent handles that. Your focus is whether the analytical logic in the code is sound and whether common pitfalls are present.
>
> **Project context:**
> - Code files in scope: [insert `CODE_FILES_REVIEWED`]
> - Paper summary: [insert `PAPER_SUMMARY`]
> - Domain: [insert `DOMAIN`]
>
> Read the code files and produce a focused report on analytical issues.
>
> **Check each of the following:**
>
> **1. Data processing integrity**
> - Merge operations: are merge types explicit (1:1, m:1, 1:m, m:m)? Are unmatched observations handled and documented? Are merge diagnostics checked?
> - Filters and sample restrictions: do the filters in code match what the paper describes? Are observations dropped silently without logging the count?
> - Missing data: how are missing values handled? Listwise deletion, imputation, or ignored? Is the approach documented and consistent with the paper?
> - Duplicate observations: is there a check for duplicates? Could duplicates affect results?
> - Variable construction: are constructed variables (indices, ratios, dummies, interactions) built correctly? Do definitions match the paper?
>
> **2. Statistical and methodological issues**
> - Wrong test for the data type: OLS on count data, t-tests on heavily skewed distributions, linear probability models without discussion
> - Clustering: is the clustering level appropriate for the data structure? Does it match what the paper claims?
> - Standard errors: are heteroskedasticity-robust or clustered SEs used where appropriate? Does the code match the paper's claims about inference?
> - Multiple comparisons: how many specifications or tests are run? Is there any correction? Are many tests run but few reported?
> - Fixed effects: do the fixed effects in the code match the paper's claims? Are they correctly absorbed?
>
> **3. ML pipeline integrity** (if applicable, especially when DOMAIN = ml)
> - Data leakage: is preprocessing (scaling, imputation, encoding, feature selection) applied before or after the train/test split?
> - Feature leakage: are any features constructed using information from the outcome or from future time periods?
> - Split integrity: are there duplicate or near-duplicate observations across train/test splits? Is there temporal leakage?
> - Cross-validation: is it implemented correctly? Are there dependencies between folds (e.g., same subject in multiple folds)?
> - Metric selection: is the evaluation metric appropriate for the task and class balance?
>
> **4. Sensitivity and robustness**
> - Magic numbers: hardcoded thresholds, cutoffs, bin counts, or bandwidths without justification or sensitivity analysis
> - Outlier treatment: winsorization, trimming, or exclusion that could drive results — is it documented? Is it tested?
> - Specification choices: are there degrees of freedom in the analytical pipeline that could produce different results? (variable transformations, sample windows, model selections)
> - Do claimed robustness checks in the paper actually appear in the code?
>
> **5. Domain-specific checks for `DOMAIN`**
>
> If DOMAIN = `econ`:
> - IV: is the first stage reported? Is the instrument plausibly exogenous? Weak instrument diagnostics?
> - DID: parallel trends test? Staggered treatment timing handled?
> - RDD: bandwidth selection method? Density test at cutoff? Sensitivity to bandwidth?
>
> If DOMAIN = `ml`:
> - Full Kapoor & Narayanan (2023) leakage taxonomy: preprocessing, feature, temporal, non-independence, train/test contamination, duplicates, label, and illegitimate features
>
> If DOMAIN = `biostats`:
> - Survival analysis: proportional hazards assumption tested?
> - Propensity scores: overlap assessed? Balance checked?
> - Multiple endpoints: family-wise error rate or FDR controlled?
>
> If DOMAIN = `polisci`:
> - Survey experiments: randomization checks? Attrition analysis? ITT vs LATE distinction?
> - Text-as-data: preprocessing choices documented? Validation of measures?
>
> If DOMAIN = `general`, apply the checks from all domains that appear relevant based on the code.
>
> **Use these labels:**
> - **SOUND**: logic is correct and well-implemented
> - **CONCERN**: plausible analytical issue that could affect results
> - **VERIFY**: ambiguous — may be fine depending on context not visible in the code
> - **ABSENT**: expected analytical step (diagnostic, robustness check, validation) is missing
>
> **Output exactly these sections:**
>
> ## Overall
> 3-6 bullets summarizing the analytical quality of the code.
>
> ## Top Findings
> Up to 12 items, ordered by potential impact on the paper's main claims (CONCERN first, then ABSENT, then VERIFY).
> Format: `- [LABEL] **Short title** — file(s): line(s) if available — what the issue is — why it matters for the paper's claims — what to check or fix`
>
> ## Data Processing Assessment
> Compact summary of merge, filter, missing data, and variable construction quality.
>
> ## Statistical Methods Assessment
> Compact summary of estimation, inference, and testing quality.
>
> ## Robustness and Sensitivity
> Which claimed robustness checks are present in code? Which are absent? Are there untested specification choices that could matter?
>
> ## Strengths
> 3-8 bullets with genuine analytical positives.

---

### AGENT C: Paper-to-Code Mapping and Results Verification

Store output as `MAPPING_SUMMARY`.

Prompt:

> You are mapping a research paper's empirical claims to its code implementation and verifying that the code could produce the claimed results. You are not checking code quality or analytical methods — separate agents handle those. Your focus is alignment between paper and code.
>
> **Inputs:**
> - Paper summary: [insert `PAPER_SUMMARY`]
> - Code files in scope: [insert `CODE_FILES_REVIEWED`]
> - Code directory: [insert `CODE_DIR`]
> - Output artifacts found: [insert `OUTPUT_FILES`]
> - Hardcoded numbers from paper scan: [insert hardcoded numbers list from PAPER_SUMMARY]
>
> Read the code files and check alignment with the paper.
>
> **1. Core design mapping**
> For each main element of the research design, find the corresponding code:
> - Main dependent variables: where are they constructed or loaded?
> - Main explanatory variables / treatments: where are they defined?
> - Sample restrictions: do the filters in code match the paper's stated sample?
> - Estimation method: does the estimation command match the paper's description?
> - Fixed effects and clustering: do they match?
> - Control variables: are the controls in the regression consistent with the paper?
>
> **2. Results verification (the CODECHECK principle)**
> For each main table and figure referenced in the paper:
> - Does the code contain a script that appears to generate it?
> - Does an output file exist in the repo that corresponds to it? (Check filenames against table/figure numbers.)
> - Do variable names in the code match variable names in the table headers?
> - If the paper cites specific coefficients or statistics in prose, can you trace them to a specific regression or computation in the code?
>
> **3. Hardcoded number verification**
> For each number identified as potentially hardcoded in the paper (not generated by `\input{}` of a table file):
> - Can you find the computation that produces this number in the code?
> - Is there a log file or output file that contains this number?
> - If not, flag it as a manual transcription risk.
>
> **4. Robustness claim verification**
> For each robustness claim in the paper summary:
> - Does corresponding code exist?
> - If the paper says "results robust to X," is X actually implemented and run?
> - Are there robustness checks in the code that the paper does not mention?
>
> **5. Sample size consistency**
> - Do sample sizes (N) stated in the paper match what the code's filters and data loading would produce?
> - If multiple specifications use different samples, is this reflected in both paper and code?
>
> **Use these confidence labels:**
> - **HIGH**: clear and specific match between paper and code
> - **MEDIUM**: plausible match but naming, structure, or specification details are ambiguous
> - **LOW**: weak or indirect match — something related exists but the mapping is uncertain
> - **NOT FOUND**: no plausible match in reviewed files
>
> **Output exactly these sections:**
>
> ## Verified Matches
> Up to 15 items. Format: `- **Paper element** → Code evidence (file:line if possible) → HIGH / MEDIUM — brief note`
>
> ## Items to Verify
> Up to 15 items. Format: `- **Paper element** → Code evidence or absence → LOW / NOT FOUND / MEDIUM — why this deserves a check — suggested next step`
>
> ## Likely Discrepancies
> Only items where paper and code appear to point in different directions. Up to 10 items.
> Format: `- **Paper says** [X] — **Code does** [Y] — file(s) — why this matters`
>
> ## Results Verification
> For each main table/figure:
> - Table/Figure number → generating script (if found) → output file (if found) → status
>
> ## Hardcoded Numbers at Risk
> Numbers in the paper prose that could not be traced to code output. Flag each with location in paper and suggested verification step.
>
> ## Robustness Claim Status
> For each robustness claim: Claimed in paper → Found in code? → file if yes → status
>
> ## Coverage Notes
> 3-6 bullets on what was easy to match, what was ambiguous, and what may sit outside the reviewed files.

---

## Phase 4: Synthesize

After all 3 agents return, synthesize the results yourself. Do not launch another agent unless the repo is unusually complex and a second pass is truly necessary.

### Synthesis steps

1. **Cross-reference agent outputs.** Where Agent A flags a missing seed and Agent B flags a stochastic procedure, combine them into a single finding. Where Agent C reports NOT FOUND and Agent A shows the script exists but is outside the review scope, note this.

2. **Resolve tensions.** If Agent B calls a statistical method SOUND but Agent C shows the paper describes a different method, prioritize the discrepancy. If Agent A reports FAIL on execution and Agent C reports HIGH match on design, note that the design may be correct but the code may not run.

3. **Assign impact tiers.** Classify each unique finding into one of three tiers:

   **CRITICAL** — blocks reproducibility or may invalidate main claims:
   - Data integrity failures (Agent B: CONCERN on merges, filters, leakage affecting main results)
   - Execution blockers (Agent A: FAIL items)
   - Paper-code discrepancies on main specifications (Agent C: Likely Discrepancies on core design)
   - Missing code for main tables/figures (Agent C: NOT FOUND on key outputs)

   **MAJOR** — should be addressed, likely to be raised in review:
   - Reproducibility gaps (Agent A: VERIFY or MISSING on seeds, dependencies, run order)
   - Analytical concerns on secondary specifications (Agent B: CONCERN items not affecting main results)
   - Hardcoded numbers at risk (Agent C)
   - Missing robustness checks claimed in paper (Agent C)
   - Ambiguous paper-code matches on important elements (Agent C: LOW confidence on key items)

   **MINOR** — improves quality:
   - Code style and structure issues (Agent A: NOTE items)
   - Documentation gaps (Agent A: MISSING README details)
   - Minor analytical notes (Agent B: VERIFY items)
   - Robustness checks in code but not in paper (Agent C)

4. **Compile final lists:**
   - `TOP_ACTIONS`: 5-10 concrete next steps, ordered by impact tier then specificity
   - `MATCHED_ITEMS`: high-confidence paper-code matches
   - `VERIFY_ITEMS`: gaps or ambiguous matches
   - `DISCREPANCY_ITEMS`: paper-code conflicts
   - `OVERALL_ASSESSMENT`: 3-5 sentences, lead with strengths

## Phase 5: Write the Report

Write the report to the current working directory as:

`code_review_report_[YYYY-MM-DD].md`

where `[YYYY-MM-DD]` is today's date.

Use this structure:

```markdown
# Code Review Report: [Paper Title]

*Reviewed: [today's date] | Languages: [LANGUAGES] | Depth: [REVIEW_DEPTH] | Domain: [DOMAIN] | Paper: [PAPER_FILE filename] | Code: [CODE_DIR]*

---

## Overall Assessment

[3-5 sentences. Lead with strengths. Then the most important reproducibility, analytical, or alignment issue. End with an overall characterization: e.g., "The codebase is largely reproducible with a few verification items" or "Several issues require attention before the code can be considered a reliable replication package."]

## What's Working Well

- [Specific positive from Agent A]
- [Specific positive from Agent B]
- [Specific positive from Agent C]
- [Additional positives]

---

## Reproducibility and Execution Feasibility

### Reproducibility Checklist

| Check | Status | Details |
|---|---|---|
| Entry point / master script | LABEL | ... |
| Run order documented | LABEL | ... |
| Relative file paths | LABEL | ... |
| Random seed practice | LABEL | ... |
| Dependency management | LABEL | ... |
| Output files traceable | LABEL | ... |
| Cross-language handoffs | LABEL | ... |
| README / documentation | LABEL | ... |

### Execution Assessment

[Can this code plausibly run on a clean machine? Specific blockers if any.]

### Key Findings

[Top findings from Agent A, condensed]

---

## Data Integrity and Analytical Logic

### Data Processing

[Summary of merge, filter, missing data, variable construction quality from Agent B]

### Statistical Methods

[Summary of estimation, inference, testing quality from Agent B]

### Sensitivity and Robustness

[Which claimed robustness checks exist in code? Which are absent? Untested specification choices?]

### Key Findings

[Top findings from Agent B, condensed]

---

## Paper-Code Alignment

### Design Match

[Summary: does the code implement the research design described in the paper?]

### Results Verification

| Table / Figure | Generating Script | Output File | Status |
|---|---|---|---|
| Table 1 | ... | ... | ... |
| Figure 1 | ... | ... | ... |

### Hardcoded Numbers at Risk

[Numbers in paper prose not traceable to code output]

### Robustness Claim Status

| Claim in Paper | Found in Code? | File | Status |
|---|---|---|---|
| ... | ... | ... | ... |

### Likely Discrepancies

[Items where paper and code point in different directions]

### Items to Verify

[Ambiguous matches worth checking]

---

## Priority Action Items

Triage hierarchy: data integrity failures > execution blockers > paper-code discrepancies on main results > reproducibility gaps > analytical concerns on secondary results > hardcoded number risks > missing robustness code > code quality.

**CRITICAL** (must address — blocks reproducibility or may invalidate main claims):
1. ...

**MAJOR** (should address — likely raised in review or replication):
2. ...

**MINOR** (improves quality):
3. ...

---

## Appendix: Agent Outputs

### A. Reproducibility and Execution Feasibility
[Paste `REPRODUCIBILITY_SUMMARY`]

### B. Data Integrity and Analytical Logic
[Paste `ANALYSIS_SUMMARY`]

### C. Paper-to-Code Mapping and Results Verification
[Paste `MAPPING_SUMMARY`]

### Paper Summary
[Paste `PAPER_SUMMARY`]
```

## Final User Message

After writing the report, tell the user:
- That the code review is complete
- The path to the saved report
- The `Overall Assessment` (3-5 sentences)
- The top 5 priority action items
- Counts: how many CRITICAL, MAJOR, and MINOR items were found
- Any caveats about review coverage (files not reviewed, languages not fully covered, etc.)
