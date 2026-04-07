---
name: replication-archive
version: 1.0.0
description: |
  3-agent replication package auditor and README generator. Agent A checks
  archive structure and documentation (README completeness against the SSDE
  template, folder layout, LICENSE, data availability statements). Agent B
  audits code reproducibility (master script, path management, dependency
  pinning, random seeds, batch mode, data pipeline integrity). Agent C
  verifies paper-archive alignment (table-to-script mapping, figure mapping,
  inline number tracing, appendix coverage). Produces a Completeness
  Dashboard and prioritized checklist. Fix modes generate missing
  documentation (README, LICENSE, table mapping, config file) rather than
  just flagging gaps. Supports AEA, AJPS, Nature, and general standards.
user-invocable: true
argument-hint: "[path/to/archive] [path/to/paper.tex] [aea|ajps|nature|general] [report|review|diff|apply]"
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

# Replication Archive

Audit a replication package against journal standards and generate missing documentation. Three read-only agents examine the archive in parallel; an optional fixer agent generates README, LICENSE, table mappings, and config files. Every finding is framed as what a human replicator would experience: "A replicator running this archive would encounter X because Y."

**What this skill does NOT cover** (use companion skills instead):
- Statistical methods and analytical correctness → `review-paper-code` or `statistical-reviewer`
- Writing quality and prose editing → `writing_editor`
- Abstract verification → `abstract-checker`

---

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` left-to-right:

### Recognized journal standards

| Token | Standard | Key characteristics |
|---|---|---|
| `aea` | AEA Data Editor | Strictest. Code must run without manual intervention. Per-dataset Data Availability Statements with formal citations. Folder structure: `data/raw/`, `data/analysis/`, `code/`, `results/`. Clean-environment test required. LICENSE required (CC-BY 4.0 default). Full SSDE template README. |
| `ajps` | AJPS / APSR | Dataverse deposit. SSDE-compatible README. Code verification by journal staff. |
| `nature` | Nature / Springer Nature | Code availability statement in the paper. Code DOI required (Zenodo/Code Ocean, not just GitHub). Open-source license required. Data availability statement. |
| `general` | SSDE template baseline (default) | 7 required README sections. No journal-specific extras. |

(case-insensitive)

### Recognized fix modes

| Token | Meaning |
|---|---|
| `review` | Interactive item-by-item. For each generated file or fixable finding, display the content and ask the user to approve or skip. |
| `diff` | Generate all documentation and apply all fixes, then open VS Code for review. Git-tracked: Source Control. Not git-tracked: create `.replication-backup` copies and open `code -d backup file`. |
| `apply` | Generate all documentation and apply all fixes without opening VS Code. Headless. |

If no fix mode token is found, default to `report` (read-only — no files created or modified).

### Parsing rules

1. If a token matches a journal standard, store as `STANDARD`.
2. If a token matches a fix mode, store as `FIX_MODE`.
3. If a token looks like a directory path, store as `ARCHIVE_DIR`.
4. If a token looks like a `.tex`, `.qmd`, `.Rmd`, or `.md` file path, store as `PAPER_FILE`.
5. Defaults: `STANDARD` = `general`, `FIX_MODE` = `report`.
6. If no `ARCHIVE_DIR` is provided, use the current working directory.
7. If no `PAPER_FILE` is provided, auto-detect (see Phase 1).

Store: `ARCHIVE_DIR`, `PAPER_FILE`, `STANDARD`, `FIX_MODE`.

---

## Phase 1: Discover the Archive and Paper

### 1. Find the paper

If `PAPER_FILE` was provided, use it. Otherwise, auto-detect:

1. Use Glob for `**/*.tex`, `**/*.qmd`, `**/*.Rmd`, excluding `_minted-*`, `build/`, `output/`, `.git/`, `node_modules/`, `__pycache__/`, `.quarto/`, `_site/`.
2. Identify the main document: `.tex` containing `\documentclass` or `\begin{document}`; `.qmd`/`.Rmd` containing YAML frontmatter with `title:`.
3. If multiple candidates, prefer `Writing/`, `Paper/`, `Draft/`, or root; then the file with the most `\input{}`/`\include{}`.
4. Read the main file and all referenced files.

### 2. Map the archive folder structure

Run `ls -R` or use Glob to build a complete tree of `ARCHIVE_DIR`. Categorize each directory:

| Expected directory | Purpose | Required by |
|---|---|---|
| `data/raw/` or `data/source/` | Original unmodified data | AEA (required), others (recommended) |
| `data/analysis/` or `data/derived/` | Processed analysis-ready data | AEA (required), others (recommended) |
| `code/` or `programs/` or `scripts/` | All code files | All |
| `results/` or `output/` or `tables/` + `figures/` | Generated outputs | All |

Record as `ARCHIVE_TREE`.

### 3. Find documentation files

Look for:
- README: `README.md`, `README.txt`, `README.pdf`, `README` (any case)
- LICENSE: `LICENSE`, `LICENSE.md`, `LICENSE.txt`
- Dependency files: `requirements.txt`, `environment.yml`, `pyproject.toml`, `setup.py`, `renv.lock`, `DESCRIPTION`, `Manifest.toml`, `Project.toml`
- Config files: `config.do`, `config.R`, `config.py`, `profile.do`, `paths.do`, `setup.R`
- Build files: `Makefile`, `Dockerfile`, `docker-compose.yml`
- Citation: `CITATION.cff`, `CITATION.bib`

Record as `DOC_FILES`.

### 4. Find code files

Use Glob with patterns: `**/*.do`, `**/*.R`, `**/*.r`, `**/*.py`, `**/*.jl`, `**/*.m`, `**/*.sas`, `**/*.ipynb`, `**/*.Rmd`, `**/*.qmd`, `**/*.sh`, `**/*.bash`.

Exclude: `.git/`, `node_modules/`, `__pycache__/`, `renv/`, `.venv/`, `venv/`, `env/`.

Record as `CODE_FILES` with a count by language.

### 5. Find data files

Use Glob: `**/*.dta`, `**/*.csv`, `**/*.xlsx`, `**/*.xls`, `**/*.parquet`, `**/*.arrow`, `**/*.rds`, `**/*.rda`, `**/*.sav`, `**/*.sas7bdat`, `**/*.feather`, `**/*.json`, `**/*.tsv`.

Categorize by directory location:
- `DATA_RAW`: files in `data/raw/`, `data/source/`, `original-data/`, or similar
- `DATA_ANALYSIS`: files in `data/analysis/`, `data/derived/`, `analysis-data/`, or similar
- `DATA_OTHER`: files elsewhere

### 6. Find output artifacts

- Tables: `**/*.tex`, `**/*.csv`, `**/*.html`, `**/*.rtf`, `**/*.xlsx` in `results/`, `output/`, `tables/`, or similar
- Figures: `**/*.pdf`, `**/*.png`, `**/*.eps`, `**/*.svg`, `**/*.jpg` in `results/`, `output/`, `figures/`, or similar
- Logs: `**/*.log`, `**/*.smcl`

Record as `OUTPUT_FILES`.

### 7. Report discovery to user

Before proceeding, tell the user:
- Archive directory and paper file identified
- Folder structure summary (matches expected layout?)
- README: found? format?
- LICENSE: found?
- Code files: count by language
- Data files: count (raw vs. analysis vs. other)
- Output artifacts: count (tables, figures, logs)
- Dependency files found
- Journal standard being applied
- Fix mode

---

## Phase 2: Read the Paper and README

### Read the paper

Read `PAPER_FILE` and all included files. Extract:

- **Paper title and authors**
- **Every table**: number, caption, data source (`\input{}` path if LaTeX), location in text
- **Every figure**: number, caption, file reference, location in text
- **Appendix tables and figures**: same as above
- **Inline numbers**: statistics, sample sizes, coefficients, p-values, counts cited in running prose (not inside `\input{}` of a table file). Use grep for patterns like `$N = `, `\$[0-9]`, `[0-9]+\%`, `$p [<>=]`, `$\beta`, coefficient values.
- **Dataset names**: survey names, administrative data descriptions, any named data sources
- **Sample sizes**: every mention of N, sample size, or observation count
- **Robustness claims**: "as a robustness check," "results are robust to," "sensitivity analysis," appendix references to alternative specifications

Record as `PAPER_INVENTORY`:
- `TABLES`: list of (number, caption, input path if any)
- `FIGURES`: list of (number, caption, file reference if any)
- `INLINE_NUMBERS`: list of (value, context sentence, location)
- `DATASETS`: list of (name, description, location in text)
- `ROBUSTNESS_CLAIMS`: list of (claim, location)

### Read the README (if it exists)

Read the full README. Parse it against the SSDE template's 7 required sections:

1. **Overview**: Look for runtime estimate, output count, general description
2. **Data Availability and Provenance Statements**: Look for per-dataset entries with source, URL/DOI, access conditions, redistribution rights
3. **Computational Requirements**: Look for software names + versions, packages, hardware, OS, runtime
4. **Description of Programs/Code**: Look for a file listing with descriptions
5. **Instructions to Replicators**: Look for numbered steps, master script reference
6. **List of Tables and Programs**: Look for a table→script mapping
7. **References**: Look for data source citations

For each section, record: PRESENT (complete) / PARTIAL (exists but incomplete — note what's missing) / ABSENT.

Record as `README_ANALYSIS`.

---

## Phase 3: Launch 3 Agents in Parallel

In a **single message**, launch all 3 agents using the Agent tool with `subagent_type: "general-purpose"`. Each agent reads source files independently. **No agent writes any file.**

Pass to each agent: all file paths discovered, `PAPER_INVENTORY`, `README_ANALYSIS`, `ARCHIVE_TREE`, `STANDARD`, and the relevant checklist.

---

### AGENT A — Archive Structure and Documentation

You are auditing a replication package's structure and documentation against the Social Science Data Editors (SSDE) template README standard. Read the archive files and assess completeness. **Do not write any files.**

Frame every finding as what a human replicator would experience. Not "Section 2 is incomplete" but "A replicator would not know where to obtain the Census data because the Data Availability Statement does not include access instructions."

Journal standard: `STANDARD`
Archive directory: `ARCHIVE_DIR`
Paper inventory: [insert `PAPER_INVENTORY`]
README analysis: [insert `README_ANALYSIS`]
Documentation files found: [insert `DOC_FILES`]
Archive tree: [insert `ARCHIVE_TREE`]

#### Checklist

**A1. README Existence and Format**
- Does a README exist? (CRITICAL if absent)
- Is it machine-readable (Markdown or plain text, not PDF-only)?
- Is it in the archive root?

**A2. SSDE §1 — Overview**
- States paper title and authors?
- States total expected runtime?
- States number of tables and figures the archive produces?
- Mentions the master script or entry point?

**A3. SSDE §2 — Data Availability and Provenance Statements**

For EVERY dataset used by the code or mentioned in the paper:
- Is there a Data Availability Statement?
- Does it specify: source name, URL or DOI, access conditions (public / restricted / purchased / author-collected), whether the data is included in the archive, redistribution rights?
- For restricted data: are there instructions for how a replicator can obtain access?
- For included data: is there confirmation the authors have redistribution rights?
- Is the dataset formally cited in §7 (References)?

Cross-reference the `DATASETS` list from the paper inventory. Flag any dataset mentioned in the paper but missing from the DAS.

**A4. SSDE §3 — Computational Requirements**
- Software names with exact version numbers? (e.g., "Stata 17 MP", "R 4.3.2", "Python 3.11.4")
- Required packages/libraries with versions?
- Hardware requirements if non-trivial (memory, storage, CPU/GPU)?
- Operating system(s) tested on?
- Total expected runtime (broken down by step if > 1 hour)?
- Any commercial software required?

Cross-reference against dependency files found (`requirements.txt`, `renv.lock`, etc.). Flag discrepancies.

**A5. SSDE §4 — Description of Programs/Code**
- Is there a listing of every code file with a one-line description of what it does?
- Is the listing organized by function (data cleaning → analysis → output generation)?
- Are dependencies between scripts documented (which must run before which)?

Cross-reference against `CODE_FILES`. Flag code files present in the archive but missing from the listing.

**A6. SSDE §5 — Instructions to Replicators**
- Is there a numbered, step-by-step sequence?
- Are there 5 or fewer steps (or a single master script)?
- Is the only manual step "set the root directory path in the config file"?
- Is it clear which script to run first?
- Are prerequisites stated (install software, obtain restricted data)?

**A7. SSDE §6 — List of Tables and Programs**
- Is there a mapping from each table/figure in the paper to: generating script, line number(s), output filename?
- Does the mapping cover ALL tables and figures including appendix?
- Does it cover inline numbers cited in the text?

Cross-reference against `TABLES` and `FIGURES` from the paper inventory. Flag paper items missing from the mapping.

**A8. SSDE §7 — References**
- Are all data sources formally cited with standard bibliographic references?

**A9. LICENSE**
- Does a LICENSE file exist?
- Is it an appropriate type for the standard? (AEA: CC-BY 4.0 default. Nature: open-source required.)
- Does it cover both code and data, or are they licensed separately?

**A10. Folder Structure**
- Does the archive follow a clear directory convention?
- Is there separation between raw data, processed data, code, and results?
- AEA-specific: are `data/raw/`, `data/analysis/`, `code/`, `results/` present or equivalent?
- Are there data files mixed into code directories or vice versa?

**A11. Standard-Specific Extras**
- AEA: Is there evidence of a clean-environment test? openICPSR deposit preparation?
- AJPS: Dataverse deposit metadata? Verification-ready format?
- Nature: Code availability statement in the paper text? DOI for code deposit (not just a GitHub URL)? Reporting checklist?

#### Output format

```
## Archive Structure & Documentation

### README Completeness

| # | SSDE Section | Status | What a replicator would experience |
|---|---|---|---|
| 1 | Overview | PRESENT / PARTIAL / ABSENT | [specific gap or "Complete"] |
| 2 | Data Availability | ... | ... |
| 3 | Computational Requirements | ... | ... |
| 4 | Description of Programs | ... | ... |
| 5 | Instructions to Replicators | ... | ... |
| 6 | List of Tables and Programs | ... | ... |
| 7 | References | ... | ... |

Sections complete: X of 7

### Data Availability Statements

| Dataset | DAS Present? | Source Cited? | Access Conditions? | Redistribution? | In Archive? | Status |
|---|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... | PRESENT / PARTIAL / ABSENT |

Datasets fully documented: X of Y

### LICENSE
[Status, type, compliance with STANDARD]

### Folder Structure
[Tree summary, compliance assessment]

### Standard-Specific Requirements ([STANDARD])
[Items specific to selected standard]

### Findings

[Numbered list, severity-ordered. Format:]
A1. [CRITICAL/MAJOR/MINOR] **[Short title]** — [What a replicator would experience] — Fix: [what to add/change]
A2. ...
```

---

### AGENT B — Code Reproducibility

You are auditing whether a replicator could clone this archive and reproduce all results without manual intervention (beyond setting a root directory path). Read all code files and assess execution feasibility. **Do not write any files.**

Frame every finding as what a human replicator would experience. Not "Line 47 has a hardcoded path" but "A replicator on any machine other than the author's would get a 'file not found' error at line 47 of clean_data.do because the path `/Users/smith/Desktop/project/data/` is hardcoded."

Journal standard: `STANDARD`
Code files: [insert `CODE_FILES`]
Data files: [insert `DATA_FILES` with raw/analysis/other categorization]
Output files: [insert `OUTPUT_FILES`]
Documentation files: [insert `DOC_FILES`]
Archive tree: [insert `ARCHIVE_TREE`]
Paper inventory: [insert `PAPER_INVENTORY`]

#### Checklist

**B1. Master Script / Entry Point**
- Does a master script exist? Look for: `main.do`, `master.do`, `run_all.R`, `main.R`, `main.py`, `run_all.py`, `Makefile`, `_targets.R`, or a script that calls all others.
- Does it call all other scripts in the correct dependency order?
- Does it run end-to-end without manual intervention?
- If no master script: can the run order be inferred from file numbering (e.g., `01_clean.do`, `02_analyze.do`)? Is this documented?

**B2. Path Management**
- Are all file paths relative to a single root variable or config file?
- List every hardcoded absolute path found. For each, note the file, line, and the path.
- Is there a config file where the replicator sets one root path? (e.g., `config.do`, `paths.R`, `config.py`)
- Do scripts create output directories before writing to them, or would they fail with "directory not found"?

**B3. Dependency Management**
- For each language used, is there a dependency specification file?
  - Stata: `_ado/` directory or install commands in master script
  - R: `renv.lock`, `DESCRIPTION`, or `install.packages()` calls
  - Python: `requirements.txt`, `environment.yml`, `pyproject.toml`, `setup.py`
  - Julia: `Project.toml`, `Manifest.toml`
- Are versions pinned (exact version numbers, not just package names)?
- Cross-reference: for every `library()`, `import`, `ssc install`, etc. in code, is the package declared?
- Flag packages used but not declared, and packages declared but not used.
- Flag packages from non-standard repositories (GitHub installs, custom CRAN mirrors).

**B4. Random Seeds**
- For every stochastic procedure (sampling, bootstrapping, simulation, train/test split, MCMC, permutation test, jittering in plots, random forest, cross-validation), is a seed set BEFORE the call?
- Is there a global seed at the top of the master script?
- Flag: `set seed` / `set.seed()` / `np.random.seed()` / `random.seed()` / `torch.manual_seed()` calls and whether they precede the stochastic code.

**B5. Batch Mode / Non-Interactive Execution**
- Stata: any `browse`, `edit`, `pause`, `graph` without `nodraw`, `more` without `set more off`?
- R: any `readline()`, `menu()`, `locator()`, `X11()`, `quartz()`, `windows()` without file-device alternative? Any `View()` calls?
- Python: any `input()`, `plt.show()` without `plt.savefig()`, `breakpoint()`, `pdb.set_trace()`?
- Jupyter notebooks: any cells that depend on interactive execution order?
- General: any script that waits for user input or opens a GUI?

**B6. Cross-Platform Compatibility**
- Windows-style path separators (`\`) in code?
- OS-specific shell commands in scripts?
- File names that differ only by case (would collide on case-insensitive filesystems)?
- Non-UTF-8 file encodings or BOM markers?

**B7. Data Pipeline Integrity**
- For every data file that a script reads, is it either: (a) raw data included in the archive, (b) produced by a prior script, or (c) documented as externally obtained with access instructions?
- Are there circular dependencies (script A reads output of script B which reads output of script A)?
- Map the full pipeline: which script reads what and produces what. Identify any breaks.

**B8. Output Generation Completeness**
- Cross-reference `PAPER_INVENTORY` tables and figures against what the code would produce.
- For each table/figure: which script generates it? What is the output filename? Would it be written to the expected location?
- Flag tables/figures in the paper that have no generating code.
- Flag inline numbers that are not generated by any script (manually transcribed).

**B9. Error Handling and Logging**
- Stata: `set more off` at top? `capture noisily` or `assert` after merges?
- R: no global `tryCatch` that swallows errors silently? `stopifnot()` or assertions for critical data checks?
- Python: no bare `except:` or `except Exception: pass`?
- Are merge/join results checked (expected rows, many-to-one vs. many-to-many)?
- Is there logging of runtime, dataset dimensions, or key statistics?

**B10. Clean Environment Test**
- Is there evidence the archive was tested in a clean environment (fresh machine, empty library, no pre-existing files)?
- Is this documented in the README?
- Is there a Docker/container specification?
- AEA-specific: is a clean-environment test result documented?

#### Output format

```
## Code Reproducibility Audit

### Reproducibility Checklist

| # | Check | Status | What a replicator would experience |
|---|---|---|---|
| 1 | Master script | PASS / FAIL / MISSING | ... |
| 2 | Path management | PASS / FAIL / PARTIAL | ... |
| 3 | Dependencies pinned | PASS / FAIL / PARTIAL | ... |
| 4 | Random seeds | PASS / FAIL / PARTIAL | ... |
| 5 | Batch mode | PASS / FAIL / PARTIAL | ... |
| 6 | Cross-platform | PASS / WARN | ... |
| 7 | Data pipeline | PASS / FAIL / PARTIAL | ... |
| 8 | Output completeness | PASS / FAIL / PARTIAL | ... |
| 9 | Error handling | PASS / WARN | ... |
| 10 | Clean-env test | PASS / MISSING | ... |

Checks passing: X of 10

### Execution Walkthrough

[Step-by-step narration of what happens when a replicator clones this archive and
tries to run it. Start from "The replicator opens the README..." and walk through
every step. Identify exactly where the pipeline would break and why. Be specific
about file names, line numbers, and error messages the replicator would see.

This is the most important section of your output. A replicator should be able to
read this walkthrough and know exactly what to expect.]

### Data Pipeline Map

[For each script, list: what it reads (inputs) → what it produces (outputs).
Mark any broken links where an input file is neither in the archive nor produced
by a prior script.]

### Hardcoded Paths

| File | Line | Path | Impact |
|---|---|---|---|
| ... | ... | ... | Would fail on any machine other than author's |

### Dependency Inventory

| Package | Language | Declared? | Version Pinned? | Used in File(s) |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |

Declared and pinned: X of Y packages

### Findings

[Numbered list, severity-ordered. Format:]
B1. [CRITICAL/MAJOR/MINOR] **[Short title]** — [What a replicator would experience] — File: [path:line] — Fix: [what to change]
B2. ...
```

---

### AGENT C — Paper-Archive Alignment

You are verifying that every empirical claim in the paper can be traced to a specific script and output file in the archive. This is the human replicator's roadmap — if this mapping is complete, a replicator knows exactly which script produces which result. **Do not write any files.**

Frame every finding as what a replicator would experience. Not "Table 3 is unmapped" but "A replicator looking for the code that produces Table 3 (regression results) would have no way to find it — no script in the archive appears to generate this output."

Journal standard: `STANDARD`
Paper inventory: [insert `PAPER_INVENTORY`]
Code files: [insert `CODE_FILES`]
Output files: [insert `OUTPUT_FILES`]
Archive tree: [insert `ARCHIVE_TREE`]
README analysis: [insert `README_ANALYSIS`]

#### Checklist

**C1. Table Mapping**

For EVERY table in the paper (main body AND appendix):
- Which script generates it? Search for: output file references, `esttab`, `outreg`, `stargazer`, `modelsummary`, `write.csv`, `to_latex`, `savefig`, table-writing commands.
- What line(s) in that script produce the output?
- What is the output filename?
- Does the output file exist in the archive?
- If the output file exists, do the column headers / variable names match the table in the paper?

**C2. Figure Mapping**

For EVERY figure in the paper (main body AND appendix):
- Which script generates it?
- What is the output filename?
- Does the output file exist in the archive?
- Does the figure file match what appears in the paper (same variables, same axes)?

**C3. Inline Number Tracing**

For numbers that appear in the paper prose (not inside `\input{}` of a table):
- Sample sizes ("N = 15,234"), percentages, summary statistics, key coefficients mentioned in text
- Can each be traced to a specific computation in the code?
- Is there a log file or saved scalar/variable that records it?
- Flag numbers that appear to be manually transcribed from output (high risk of transcription error)

**C4. Appendix Coverage**
- Are ALL appendix tables and figures generated by code in the archive?
- Are there appendix items with no corresponding code?
- Are there code outputs not mentioned in the paper or appendix? (Not a problem, but worth noting.)

**C5. Sample Size Consistency**
- Do N values stated in the paper match what the code's data loading and filtering steps would produce?
- If multiple samples are used (e.g., different subsets for different analyses), are they all documented?
- Flag discrepancies between paper-stated N and code-implied N.

**C6. Robustness Check Coverage**

For every robustness check or sensitivity analysis mentioned in the paper:
- Does corresponding code exist in the archive?
- Does the code match the description (same specification, same sample, same variables)?

**C7. Data Source Tracing**

For every dataset mentioned in the paper:
- Is it either: (a) included in the archive, or (b) documented in a Data Availability Statement with access instructions?
- Do variable names in the code match the data source descriptions in the paper?

#### Output format

```
## Paper-Archive Alignment

### Table Mapping

| Table | Caption (first 10 words) | Script | Line(s) | Output File | Exists? | Status |
|---|---|---|---|---|---|---|
| Table 1 | ... | ... | ... | ... | Y/N | MAPPED / PARTIAL / UNMAPPED |
| Table 2 | ... | ... | ... | ... | ... | ... |
| Table A1 | ... | ... | ... | ... | ... | ... |

Tables fully mapped: X of Y (main), X of Y (appendix)

### Figure Mapping

| Figure | Caption (first 10 words) | Script | Line(s) | Output File | Exists? | Status |
|---|---|---|---|---|---|---|
| Figure 1 | ... | ... | ... | ... | Y/N | MAPPED / PARTIAL / UNMAPPED |

Figures fully mapped: X of Y (main), X of Y (appendix)

### Inline Numbers at Risk

| Number | Context (paper location) | Traced to Code? | Code Location | Status |
|---|---|---|---|---|
| N = 15,234 | Section 3, para 2 | Yes / No | clean.do:47 | MAPPED / UNMAPPED |

Inline numbers traced: X of Y

### Appendix Coverage
[List of appendix items with status]

### Sample Size Consistency
[List of N values in paper vs. what code produces. Flag discrepancies.]

### Robustness Check Coverage

| Claim in Paper | Paper Location | Code Exists? | Script | Status |
|---|---|---|---|---|
| ... | ... | Yes / No | ... | MAPPED / UNMAPPED |

Robustness checks mapped: X of Y

### Data Source Tracing

| Dataset | Mentioned in Paper | In Archive? | DAS in README? | Access Instructions? | Status |
|---|---|---|---|---|---|
| ... | ... | Y/N | Y/N | Y/N | MAPPED / PARTIAL / UNMAPPED |

Datasets fully traceable: X of Y

### Coverage Summary

- Tables mapped: X of Y main + X of Y appendix
- Figures mapped: X of Y main + X of Y appendix
- Inline numbers traced: X of Y
- Robustness checks mapped: X of Y
- Datasets documented: X of Y

### Findings

[Numbered list, severity-ordered. UNMAPPED main tables/figures first. Format:]
C1. [CRITICAL/MAJOR/MINOR] **[Short title]** — [What a replicator would experience] — Fix: [what to add]
C2. ...
```

---

## Phase 4: Synthesize

After all 3 agents return, consolidate their findings.

### 4.1 Cross-reference and deduplicate

Combine findings that flag the same underlying issue from different angles:
- Agent A "Data Availability Statement ABSENT for dataset X" + Agent C "Dataset X UNMAPPED" → single finding
- Agent B "Script Y reads data file not in archive" + Agent C "Table 3 UNMAPPED" → single finding
- Agent A "Instructions section ABSENT" + Agent B "No master script and run order unclear" → single finding

### 4.2 Assign severity tiers

Severity depends on the journal standard. The table below shows the **base** severity; when `STANDARD = aea`, items marked ★ are elevated one tier.

**CRITICAL** — Archive will be rejected or results cannot be reproduced:
- README absent or missing 3+ of 7 SSDE sections
- No master script AND no documented run order
- Main table or figure UNMAPPED (no generating script found)
- Hardcoded absolute paths throughout (archive won't run on another machine)
- Data files referenced by code but neither included nor documented as restricted
- Paper-archive numerical mismatch on a main result
- LICENSE absent (when required by standard: AEA, Nature)

**MAJOR** — Likely revision request:
- README missing key sections (runtime, computational requirements, table mapping)
- Dependencies not pinned (versions not specified) ★
- Random seeds missing for stochastic procedures ★
- Appendix table/figure UNMAPPED
- Interactive commands blocking batch execution ★
- Incomplete Data Availability Statements (some datasets undocumented)
- No evidence of clean-environment test ★
- Inline numbers not traceable to code

**MINOR** — Improves replicator experience:
- Folder structure doesn't follow convention (but archive still navigable)
- Minor documentation gaps (no hardware requirements when runtime < 10 min)
- Cross-platform warnings
- Missing logging/diagnostics
- Redundant output files
- Code style issues that don't affect execution

### 4.3 Triage hierarchy

missing code for main results > archive won't run > README absent/incomplete > data undocumented > dependencies unmanaged > appendix gaps > seeds missing > inline numbers untraced > style improvements.

### 4.4 Compile synthesis

- `ARCHIVE_HEALTH`: one of "Ready for submission" / "Minor revisions needed" / "Major revisions needed" / "Not submittable"
- `TOP_ACTIONS`: 5–10 concrete next steps, ordered by severity
- `COMPLETENESS_METRICS`: from agent outputs

---

## Phase 4b: Fix Mode Gate

*Skip this phase entirely if `FIX_MODE` is `report` (the default). Proceed directly to Phase 5.*

This skill's fixes are primarily **generative** — creating missing documentation rather than editing existing code. Build a fix plan from the synthesis.

### Auto-fixable (generative) categories

**1. Generate README**

If the README is absent or missing 3+ sections, generate a complete SSDE template README. Populate from discovered metadata:

```markdown
# Data and Code for: "[Paper Title]"

## Overview

**Authors:** [from paper]
**Date:** [current date]
**README last updated:** [current date]

This archive contains the data and code to replicate all tables, figures, and
in-text numbers in "[Paper Title]."

- **Runtime:** TO BE FILLED BY AUTHOR
- **Tables produced:** [N, from paper inventory]
- **Figures produced:** [N, from paper inventory]

## Data Availability and Provenance Statements

[One subsection per dataset detected by agents:]

### [Dataset Name]

- **Source:** TO BE FILLED BY AUTHOR
- **URL/DOI:** TO BE FILLED BY AUTHOR
- **Access conditions:** [Public / Restricted / Purchased — infer if possible, else TO BE FILLED]
- **Included in this archive:** [Yes / No — check if data file exists]
- **Redistribution rights:** TO BE FILLED BY AUTHOR
- **Citation:** TO BE FILLED BY AUTHOR

## Computational Requirements

### Software

[For each language detected:]
- [Language] [version — TO BE FILLED BY AUTHOR]
  - Packages: [list from dependency analysis, versions TO BE FILLED if not pinned]

### Hardware

- TO BE FILLED BY AUTHOR

### Runtime

- TO BE FILLED BY AUTHOR

## Description of Programs

| File | Description |
|---|---|
| [master script] | Master script. Runs all other scripts in sequence. |
| [each code file] | [auto-inferred from file name and comments, or TO BE FILLED] |

## Instructions to Replicators

1. Set the root directory path in `[config file]` to the location of this archive.
2. Install required packages: [language-specific instructions from dependency files].
3. Run `[master script name]`.
4. Outputs will be saved to `[output directory]`.

## List of Tables and Programs

| Table/Figure | Script | Line(s) | Output File |
|---|---|---|---|
[from Agent C's mapping]

## References

TO BE FILLED BY AUTHOR — cite all data sources used.
```

Mark every "TO BE FILLED BY AUTHOR" field clearly. These require author judgment and cannot be inferred.

**2. Generate LICENSE file**

If absent, generate based on standard:
- AEA: CC-BY 4.0 with author names and year
- Nature: MIT License (or CC-BY 4.0 if data included)
- General: MIT License

**3. Generate table/figure mapping**

If README §6 is absent, generate the mapping table from Agent C's analysis and insert it into the README (or create a standalone `TABLE_MAPPING.md`).

**4. Fix hardcoded absolute paths**

For each hardcoded path found by Agent B, replace with a relative path using a config variable. If no config file exists, create one first (see #6).

**5. Add missing random seeds**

Insert seed-setting calls before stochastic procedures:
- Stata: `set seed 12345` before `bsample`, `bootstrap`, `simulate`, `permute`
- R: `set.seed(12345)` before `sample()`, `boot()`, `replicate()`, `randomForest()`
- Python: `np.random.seed(12345)` / `random.seed(12345)` / `torch.manual_seed(12345)`

Use seed value `12345` as a placeholder. Add a comment: `// TO BE SET BY AUTHOR — choose a meaningful seed`.

**6. Create config file**

If no config file exists and paths are scattered across scripts, generate:
- Stata: `config.do` with `global root "TO BE SET BY AUTHOR"`
- R: `config.R` with `root <- "TO BE SET BY AUTHOR"`
- Python: `config.py` with `ROOT = "TO BE SET BY AUTHOR"`

**7. Add output directory creation**

Insert directory-creation guards before file writes:
- Stata: `cap mkdir "[dir]"`
- R: `dir.create("[dir]", recursive = TRUE, showWarnings = FALSE)`
- Python: `os.makedirs("[dir]", exist_ok=True)`

**8. Add Stata housekeeping**

If Stata scripts lack standard preamble, insert at top of master script:
```stata
clear all
set more off
set maxvar 10000
```

### Not auto-fixable (recommend only)

- Writing Data Availability Statements for restricted data (requires author knowledge of access conditions)
- Filling in runtime estimates (requires actually running the code)
- Adding missing code for tables/figures (requires domain expertise)
- Restructuring folder hierarchy (too destructive)
- Clean-environment testing (requires actual execution)
- Verifying numerical accuracy (requires running the code)
- Describing data sources the agents cannot identify

### Fix plan structure

For each fixable item, record:
- **Type**: `GENERATE` (new file) or `EDIT` (modify existing file)
- **File path** (for GENERATE: the new file path; for EDIT: existing file path)
- **Content** (for GENERATE: full file content; for EDIT: `old_string` → `new_string`)
- **Reason**: what problem this fixes, in replicator-frame language
- **Severity**: CRITICAL / MAJOR / MINOR

Order: GENERATE items first (README, LICENSE, config), then EDIT items by severity.

### If `FIX_MODE` is `review` (interactive):

For each item in the fix plan:

1. Display: type, severity, file path, content preview (first 20 lines for GENERATE; full text for EDIT), and reason.
2. For GENERATE items: use AskUserQuestion: "Create this file? (yes / skip / stop)"
   - **yes** → Write the file using the Write tool.
3. For EDIT items: run `code -g FILE:LINE` to open VS Code, display the change, use AskUserQuestion: "Apply this fix? (yes / skip / stop)"
   - **yes** → Apply using Edit tool.
4. Track applied vs. skipped.

### If `FIX_MODE` is `diff` or `apply`:

Launch a **single fixer agent** (`subagent_type: "general-purpose"`) — the only agent with Write and Edit permissions.

> You are generating documentation and applying infrastructure fixes to a replication archive. Each item has been vetted by three audit agents. For new files, use the Write tool. For edits to existing files, use the Edit tool. Apply exactly what the fix plan specifies.
>
> **Fix plan:** [insert fix plan]
>
> **Rules:**
> 1. For GENERATE items: Write the complete file using the Write tool.
> 2. For EDIT items: Apply using the Edit tool with exact `old_string` and `new_string`.
> 3. If an `old_string` cannot be found, skip and report.
> 4. Do not modify analysis code logic. Only change infrastructure (paths, seeds, directory creation, housekeeping).
> 5. Preserve all "TO BE FILLED BY AUTHOR" placeholders — do not guess at values.
> 6. Report: files created, edits applied, items skipped with reasons.

After the fixer agent returns:
- If `FIX_MODE` is `diff` and git-tracked: `code -g FILE:1` for each modified/created file; tell user to use Source Control (Cmd+Shift+G) for per-hunk revert.
- If `FIX_MODE` is `diff` and not git-tracked: for EDIT items, create `.replication-backup` copies before fixing, then `code -d backup file`. For GENERATE items, just open the new file.
- If `FIX_MODE` is `apply`: report changes summary only.

Record results as `FIX_RESULTS`.

---

## Phase 5: Write the Report

Save to: `replication_archive_[YYYY-MM-DD].md`

```markdown
# Replication Archive Audit: [Paper Title]

*Audited: [date] | Standard: [STANDARD] | Languages: [LANGUAGES] | Paper: [PAPER_FILE] | Archive: [ARCHIVE_DIR]*

---

## Archive Health: [Ready for submission / Minor revisions needed / Major revisions needed / Not submittable]

[3–5 sentences. Frame as: "A human replicator attempting to reproduce these results
would..." Lead with the strongest aspect of the archive. Then the most critical gap.
End with overall characterization.]

---

## Completeness Dashboard

| Metric | Status | Details |
|---|---|---|
| README (7 SSDE sections) | X of 7 present | [list missing sections] |
| LICENSE | Present / Absent | [type if present] |
| Master script | Present / Absent | [name if present] |
| Tables mapped to scripts | X of Y (main) + X of Y (appendix) | [list unmapped] |
| Figures mapped to scripts | X of Y (main) + X of Y (appendix) | [list unmapped] |
| Inline numbers traced | X of Y | [list untraced] |
| Datasets documented (DAS) | X of Y | [list undocumented] |
| Dependencies declared & pinned | X of Y packages | [list undeclared] |
| Random seeds set | X of Y stochastic calls | [list unseeded] |
| Clean environment test | Documented / Not documented | |

---

## 1. Archive Structure & Documentation

[Agent A output, condensed. Preserve README completeness table and Data Availability
table. Omit raw output that is redundant with the Dashboard.]

---

## 2. Code Reproducibility

[Agent B output, condensed. Preserve Reproducibility Checklist table, Execution
Walkthrough (in full — this is the highest-value section), and Hardcoded Paths table.]

---

## 3. Paper-Archive Alignment

[Agent C output, condensed. Preserve Table Mapping, Figure Mapping, Inline Numbers
tables, and Coverage Summary.]

---

## Fixes Applied

*Include only if `FIX_MODE` is `review`, `diff`, or `apply`. Omit if `report`.*

| # | Type | File | Description | Severity | Status |
|---|---|---|---|---|---|
| 1 | GENERATE | README.md | Complete SSDE template README | CRITICAL | Applied / Skipped |
| 2 | GENERATE | LICENSE | CC-BY 4.0 | MAJOR | Applied / Skipped |
| 3 | EDIT | clean.do:47 | Replace hardcoded path with config variable | CRITICAL | Applied / Skipped / Failed |
| ... | ... | ... | ... | ... | ... |

**Summary:** X files generated, Y edits applied, Z skipped.

---

## Priority Action Items

Triage hierarchy: missing code for main results > archive won't run > README
absent/incomplete > data undocumented > dependencies unmanaged > appendix gaps >
seeds missing > inline numbers untraced > style improvements.

**CRITICAL** (must fix before submission):
1. ...

**MAJOR** (likely to trigger revision request):
4. ...

**MINOR** (improves replicator experience):
7. ...

---

## Recommended README

*Include in report mode as a reference the author can adopt. In fix modes where
the README was generated, note the path to the generated file instead.*

[If `FIX_MODE` is `report`: include the complete generated README text here so the
author can copy it. If a README was generated in fix mode: "See generated file at
[path]. Search for 'TO BE FILLED BY AUTHOR' to find fields requiring your input."]

---

## Appendix: Full Agent Outputs

### A. Archive Structure & Documentation
[Full Agent A output]

### B. Code Reproducibility
[Full Agent B output]

### C. Paper-Archive Alignment
[Full Agent C output]
```

---

## Phase 6: Final User Message

After saving the report, tell the user:

1. **Report path** — where the report was saved
2. **Archive health** — one sentence
3. **Completeness dashboard** — the key metrics table
4. **Fixes** (if fix mode active): what was generated (README, LICENSE, config), how many edits applied. Remind to search for "TO BE FILLED BY AUTHOR" in generated files.
5. **Top 5 priority action items** — from the CRITICAL and MAJOR tiers
6. **Counts**: X CRITICAL, Y MAJOR, Z MINOR findings
7. **Standard-specific note**: e.g., "AEA requires a clean-environment test — run your archive on a fresh machine before submitting" or "Nature requires a DOI for the code deposit — create a Zenodo release from the GitHub repo"
8. If `FIX_MODE` is `diff`: remind user to review changes in VS Code Source Control
9. If `FIX_MODE` is `report`: note that a recommended README is in the report appendix

---

## Examples

```
# Audit against general SSDE standards (default)
/replication-archive path/to/archive path/to/paper.tex

# Audit against AEA standards
/replication-archive aea path/to/archive path/to/paper.tex

# Audit and generate missing documentation
/replication-archive aea apply path/to/archive path/to/paper.tex

# Interactive review of generated documentation
/replication-archive ajps review path/to/archive

# Audit for Nature submission
/replication-archive nature path/to/archive path/to/paper.tex
```

---

## References

This skill implements standards and checklists from:
- Social Science Data Editors template README (https://social-science-data-editors.github.io/template_README/)
- AEA Data Editor guidance (https://aeadataeditor.github.io/aea-de-guidance/)
- Vilhuber, L. (2024). Self-Checking Reproducibility Guide (https://larsvilhuber.github.io/self-checking-reproducibility/)
- TIER Protocol v3.0 (https://www.projecttier.org/tier-protocol/)
- AJPS Verification Policy (https://ajps.org/ajps-verification-policy/)
- Nature reporting and code availability requirements
- reifjulian/my-project — AEA-compliant replication package template
- REPKIT/IEDOREP — World Bank DIME reproducibility checker (Stata)
