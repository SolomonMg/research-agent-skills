---
name: abstract-checker
version: 1.0.0
description: |
  2-agent bidirectional abstract checker. Agent A verifies every claim in the
  abstract against the manuscript body (abstract → body). Agent B identifies
  the manuscript's most important findings and checks whether the abstract
  highlights them clearly (body → abstract). Synthesis flags unsupported
  claims, numerical mismatches, missing findings, and framing gaps. Supports
  review/diff/apply fix modes to rewrite the abstract directly.
user-invocable: true
argument-hint: "[path/to/paper.tex] [review|diff|apply]"
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

# Abstract Checker

Bidirectional verification of an academic paper's abstract. Direction 1: does every claim in the abstract have support in the manuscript? Direction 2: does the abstract highlight the manuscript's most important findings? Be concrete, cite specific locations, and treat gaps as items to verify rather than accusations.

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` left-to-right:

### Recognized fix modes

| Token | Meaning |
|---|---|
| `review` | Interactive sentence-by-sentence. For each proposed abstract revision, open VS Code at the location (`code -g file:line`), display the current sentence and proposed replacement, and ask the user to approve or skip. Only approved edits are applied. |
| `diff` | Rewrite the abstract with all recommended changes, then open VS Code for review. If git-tracked, tell user to use Source Control (Cmd+Shift+G) for per-hunk revert. If not, create `.abstract-backup` and open `code -d backup file`. |
| `apply` | Rewrite the abstract with all recommended changes without opening VS Code. For headless or non-VS-Code environments. |

If no fix mode is found, default to `report` (read-only — no files are modified).

### Parsing rules

1. If a token matches a fix mode (`review`, `diff`, `apply`), store it as `FIX_MODE`.
2. Remaining text is the **file path**.
3. If no fix mode is found, set `FIX_MODE` to `report`.
4. If `$ARGUMENTS` is empty, auto-detect the paper and set `FIX_MODE` to `report`.

Store: `FIX_MODE`, file path.

## Phase 1: Discover and Read the Paper

### Find the paper

If a file path was provided, use it. Otherwise, auto-detect:

1. Use Glob for `**/*.tex`, `**/*.qmd`, `**/*.Rmd`, excluding `_minted-*`, `build/`, `output/`, `.git/`, `node_modules/`, `__pycache__/`, `.quarto/`, `_site/`.
2. Identify the main document: `.tex` containing `\documentclass` or `\begin{document}`; `.qmd`/`.Rmd` containing YAML frontmatter with `title:`.
3. If multiple candidates, prefer a file in `Writing/`, `Paper/`, `Draft/`, or root; then the file with the most `\input{}`/`\include{}` references.

Record as `PAPER_FILE`.

### Read the paper

Read `PAPER_FILE` and all files it references via `\input{}`, `\include{}`, `\subfile{}`, or Quarto `{{< include >}}`.

### Extract the abstract

Locate the abstract. Common patterns:
- `\begin{abstract}` ... `\end{abstract}`
- `\abstract{` ... `}`
- YAML field `abstract:` in `.qmd`/`.Rmd`
- A section titled "Abstract" near the top of the document

Extract the full abstract text verbatim. Record as `ABSTRACT_TEXT` and note its file location and line numbers as `ABSTRACT_LOCATION`.

If no abstract is found, use AskUserQuestion to ask the user where it is or whether they want to paste it.

### Extract structured content from the body

Build a working summary of the manuscript body (everything outside the abstract):

**Claims inventory from the abstract:**
Parse the abstract into individual claim-bearing sentences. For each sentence, extract:
- The sentence text (verbatim)
- The type of claim: empirical finding, methodological claim, data description, scope/framing, contribution/novelty, or implication/policy
- Any specific numbers, statistics, or magnitudes mentioned
- The direction of the effect (positive/negative/null/comparative) if applicable

Record as `ABSTRACT_CLAIMS` (ordered list).

**Body findings inventory:**
From the results, discussion, and conclusion sections, extract:
- Each distinct empirical finding (with location: section, paragraph, table/figure reference)
- The supporting evidence (coefficient, p-value, confidence interval, effect size, descriptive statistic)
- The author's own emphasis signals: findings mentioned in both results and discussion/conclusion; findings given their own subsection; findings called "main," "key," "central," "primary," or "important"
- Robustness or supplementary status: is the finding presented as the main result or as a secondary/robustness check?

Rank the findings by importance using these signals:
1. Mentioned in both results and conclusion → high importance
2. Has its own subsection or extended discussion → high importance
3. Appears in the first or most prominent table/figure → high importance
4. Described with emphasis language ("main finding," "key result") → high importance
5. Appears only in robustness checks or appendix → low importance
6. Mentioned briefly without elaboration → medium importance

Record as `BODY_FINDINGS` (ranked by importance).

**Paper metadata:**
- Title, authors
- Number of tables and figures
- Main estimation method
- Main data source and sample

Record as `PAPER_METADATA`.

Before proceeding, tell the user:
- The paper file found
- The abstract location (file and line numbers)
- Number of claim-bearing sentences in the abstract
- Number of findings extracted from the body
- Fix mode

## Phase 2: Launch 2 Agents in Parallel

In a single message, launch both agents using the Agent tool with `subagent_type: "general-purpose"`. Both agents are read-only.

---

### AGENT A: Abstract → Body Verification

Store output as `VERIFICATION_SUMMARY`.

Prompt:

> You are verifying whether every claim in an academic paper's abstract is supported by the manuscript body. Your direction is abstract → body: for each abstract claim, find the supporting evidence.
>
> **Inputs:**
> - Abstract text: [insert `ABSTRACT_TEXT`]
> - Abstract claims (parsed): [insert `ABSTRACT_CLAIMS`]
> - Paper files: [insert all .tex/.qmd/.Rmd file paths]
> - Paper metadata: [insert `PAPER_METADATA`]
>
> Read the paper body. For each claim in the abstract (in order), check:
>
> **1. Evidentiary support**
> - Can you find the specific evidence in the body that supports this claim?
> - Is the evidence in a results section, a table, a figure, or only in the discussion?
> - How direct is the support? (directly stated with evidence → indirect/implied → not found)
>
> **2. Numerical accuracy**
> - If the abstract states a specific number (percentage, coefficient, sample size, p-value, effect size), does it exactly match the body?
> - Check against: the relevant table or figure, the in-text discussion of that result, and any footnotes
> - Flag any discrepancy, even small ones (e.g., abstract says "15%" but table shows "14.8%")
>
> **3. Directional accuracy**
> - If the abstract states a direction (increases, decreases, positive, negative, higher, lower), does the body confirm this direction?
> - If the abstract states a null result ("no effect," "no difference"), does the body support this?
>
> **4. Strength calibration**
> - Does the abstract overstate the finding? (e.g., body says "suggestive evidence" but abstract says "we show")
> - Does the abstract understate the finding? (e.g., body has a strong, robust result but abstract hedges excessively)
> - Does the abstract use causal language for correlational findings?
>
> **5. Scope accuracy**
> - Does the abstract's scope match the body's scope? (e.g., abstract claims generality but the study is limited to one country/time period/population)
> - Does the abstract's framing of the contribution match what the body actually delivers?
>
> **6. Novelty and contribution claims**
> - "First to show X," "novel approach," "new evidence" — are these defensible based on the literature review in the body?
> - Does the body's literature review support the gap the abstract claims to fill?
>
> **Use these labels:**
> - **SUPPORTED**: claim is directly supported by evidence in the body
> - **PARTIALLY SUPPORTED**: claim is supported but with caveats the abstract omits, or the match is imprecise
> - **OVERSTATED**: claim goes beyond what the evidence supports
> - **UNDERSTATED**: claim undersells a strong finding
> - **NUMERICALLY INCONSISTENT**: specific number doesn't match
> - **NOT FOUND**: no clear support in the body
>
> **Output exactly these sections:**
>
> ## Claim-by-Claim Verification
>
> For each abstract claim (numbered to match `ABSTRACT_CLAIMS`):
>
> | # | Abstract Claim (abbreviated) | Label | Body Evidence Location | Notes |
> |---|---|---|---|---|
> | 1 | "..." | SUPPORTED / etc. | Section X, Table Y | ... |
>
> ## Numerical Mismatches
> List every instance where a number in the abstract doesn't match the body. Format:
> - Abstract says [X] (line N) — Body says [Y] (section/table/line) — Discrepancy: [description]
>
> ## Overstatement Flags
> Claims where the abstract's language exceeds the evidence. For each:
> - Abstract sentence (verbatim)
> - Body evidence (location and what it actually says)
> - Suggested recalibration
>
> ## Understatement Flags
> Claims where the abstract undersells a strong result. Same format as above.
>
> ## Unsupported Claims
> Claims with no clear match in the body. For each:
> - Abstract sentence (verbatim)
> - What was searched for and where
> - Possible explanations (e.g., "may be in an appendix not reviewed")
>
> ## Summary
> - Total claims: N
> - Supported: N | Partially supported: N | Overstated: N | Understated: N | Numerically inconsistent: N | Not found: N

---

### AGENT B: Body → Abstract Coverage

Store output as `COVERAGE_SUMMARY`.

Prompt:

> You are checking whether an academic paper's abstract highlights its most important findings. Your direction is body → abstract: identify what matters most in the manuscript and check whether the abstract covers it.
>
> **Inputs:**
> - Abstract text: [insert `ABSTRACT_TEXT`]
> - Body findings (ranked by importance): [insert `BODY_FINDINGS`]
> - Paper files: [insert all .tex/.qmd/.Rmd file paths]
> - Paper metadata: [insert `PAPER_METADATA`]
>
> Read the paper body. Then evaluate:
>
> **1. Coverage of top findings**
> For each finding in `BODY_FINDINGS` (in ranked importance order), check:
> - Is it mentioned in the abstract? (yes / partially / no)
> - If yes, is it given appropriate weight? (prominent / mentioned in passing / buried)
> - If no, should it be? (essential — its absence is misleading / important — would strengthen the abstract / optional — fine to omit)
>
> **2. Emphasis alignment**
> - Does the abstract's emphasis match the manuscript's emphasis? (i.e., does the abstract spend the most words on the manuscript's most important findings?)
> - Are secondary findings given more abstract space than primary findings?
> - Is the ordering in the abstract logical given the manuscript's structure?
>
> **3. Missing findings**
> Identify findings that are important in the body but absent from the abstract. For each:
> - The finding and its evidence
> - Where it appears in the body
> - Why it's important enough for the abstract
> - A suggested sentence to add (concrete, publication-ready)
>
> **4. Unnecessary content**
> Identify abstract content that could be cut to make room for more important findings:
> - Methodological detail that is standard and doesn't need abstract space
> - Redundant statements (two sentences saying the same thing differently)
> - Context-setting that is too long relative to findings
> - Literature positioning that belongs in the introduction, not the abstract
>
> **5. Structural assessment**
> Evaluate the abstract's structure:
> - Does it follow a logical flow? (context → gap → approach → findings → implications)
> - Is the research question stated clearly?
> - Is there a clear "so what" — why should the reader care?
> - Is the abstract the right length for the target venue? (most journals: 150-250 words)
> - Word count of current abstract
>
> **6. Opening and closing sentences**
> - Does the opening sentence hook the reader with the broader question or significance?
> - Does the closing sentence state the main takeaway or implication?
> - Are either of these weak, generic, or missing?
>
> **Output exactly these sections:**
>
> ## Finding Coverage Table
>
> | Rank | Finding (abbreviated) | Body Location | In Abstract? | Weight | Assessment |
> |---|---|---|---|---|---|
> | 1 | ... | Section X, Table Y | Yes/Partial/No | Prominent/Passing/Absent | Essential/Important/Optional |
>
> ## Missing from Abstract
> Findings that should be in the abstract but aren't. For each:
> - **Finding**: [description]
> - **Body location**: [section, table/figure]
> - **Importance**: Essential / Important
> - **Suggested sentence**: [concrete, publication-ready text]
>
> ## Could Be Cut or Shortened
> Abstract content that could be trimmed. For each:
> - **Current text**: [verbatim]
> - **Why it can be cut**: [reason]
> - **Suggested replacement** (if shortening, not cutting): [text]
>
> ## Emphasis Misalignment
> Cases where the abstract gives too much or too little weight to a finding relative to its importance in the body.
>
> ## Structural Assessment
> - Flow: [logical / partially logical / disorganized]
> - Research question clarity: [clear / implied / unclear]
> - "So what" factor: [strong / present but weak / absent]
> - Word count: [N words] (target: [venue-appropriate range])
> - Opening sentence quality: [strong / adequate / weak / generic]
> - Closing sentence quality: [strong / adequate / weak / generic]
>
> ## Summary
> - Top findings covered in abstract: N of M
> - Essential findings missing: N
> - Sentences that could be cut or shortened: N
> - Overall abstract quality: [Strong | Good | Needs work | Major revision needed]

---

## Phase 3: Synthesize

After both agents return, synthesize the results yourself.

### Synthesis steps

1. **Cross-reference.** Where Agent A flags a claim as NOT FOUND and Agent B identifies that finding as absent from the body entirely, unify these into a single "unsupported claim" finding. Where Agent A flags OVERSTATED and Agent B flags emphasis misalignment on the same finding, combine them.

2. **Build the revision plan.** For each abstract sentence, determine its disposition:

   - **Keep as-is**: SUPPORTED by Agent A, adequate weight per Agent B
   - **Revise**: PARTIALLY SUPPORTED, OVERSTATED, UNDERSTATED, or NUMERICALLY INCONSISTENT — needs rewording
   - **Cut**: Agent B identifies it as unnecessary or low-priority
   - **Add**: Agent B identifies a missing essential finding — draft a new sentence

3. **Draft the revised abstract.** If `FIX_MODE` is not `report`, draft a complete revised abstract that:
   - Fixes every numerical mismatch
   - Recalibrates overstated or understated claims
   - Adds sentences for essential missing findings
   - Cuts or shortens unnecessary content
   - Preserves the author's voice and style
   - Maintains appropriate length for the venue
   - Follows a logical flow (context → gap → approach → findings → implications)

   Record as `REVISED_ABSTRACT`.

4. **Compile findings:**
   - `OVERALL_ASSESSMENT`: 3-5 sentences on abstract quality
   - `CRITICAL_ISSUES`: unsupported claims, numerical mismatches, missing essential findings
   - `MAJOR_ISSUES`: overstated claims, emphasis misalignment, structural problems
   - `MINOR_ISSUES`: minor wording, unnecessary detail, understatement

## Phase 3b: Fix Mode Gate

*Skip this phase entirely if `FIX_MODE` is `report` (the default). The skill remains read-only and proceeds directly to Phase 4.*

### If `FIX_MODE` is `review` (interactive sentence-by-sentence):

For each sentence in the abstract that has a disposition of **Revise**, **Cut**, or **Add** (from the revision plan):

1. Run `code -g ABSTRACT_FILE:LINE` via Bash to open VS Code at the sentence.
2. Display: the disposition (Revise/Cut/Add), the current sentence, the proposed replacement or deletion, the reason (which agent findings support this change), and the severity.
3. Use AskUserQuestion: "Apply this change? (yes / skip / stop)"
   - **yes** → Apply immediately with Edit tool.
   - **skip** → Next item.
   - **stop** → End; skip remaining.
4. Track applied vs. skipped.

For **Add** dispositions (new sentences), show the proposed sentence and its suggested position in the abstract.

### If `FIX_MODE` is `diff` or `apply`:

Locate the abstract in the source file. Replace the entire abstract text with `REVISED_ABSTRACT` using the Edit tool.

- If `FIX_MODE` is `diff` and git-tracked: run `code -g ABSTRACT_FILE:ABSTRACT_LINE` and tell the user: "Open Source Control (Cmd+Shift+G) and click the file to see the abstract diff. Use the revert buttons to reject any changes."
- If `FIX_MODE` is `diff` and not git-tracked: create a backup (`cp FILE FILE.abstract-backup`), apply the revision, then run `code -d FILE.abstract-backup FILE`.
- If `FIX_MODE` is `apply`: apply the revision and report the changes summary.

Record results as `FIX_RESULTS`.

## Phase 4: Write the Report

Write the report to the current working directory as:

`abstract_check_[YYYY-MM-DD].md`

where `[YYYY-MM-DD]` is today's date.

Use this structure:

```markdown
# Abstract Check Report: [Paper Title]

*Reviewed: [today's date] | Paper: [PAPER_FILE] | Abstract: [line range] | Word count: [N]*

---

## Overall Assessment

[3-5 sentences. How well does the abstract represent the paper? Lead with strengths. Then the most important gap or mismatch. End with overall quality characterization.]

**Abstract quality**: [Strong | Good | Needs work | Major revision needed]

---

## Claim Verification (Abstract → Body)

### Claim-by-Claim Results

| # | Abstract Claim | Verdict | Body Evidence | Notes |
|---|---|---|---|---|
| 1 | ... | SUPPORTED | ... | ... |

### Numerical Mismatches

[List, or "None found"]

### Overstatement Flags

[List, or "None found"]

### Unsupported Claims

[List, or "None found"]

**Summary**: N of M claims fully supported, N partially supported, N overstated, N numerically inconsistent, N not found.

---

## Finding Coverage (Body → Abstract)

### Coverage Table

| Rank | Finding | In Abstract? | Weight | Should Include? |
|---|---|---|---|---|
| 1 | ... | Yes/No | ... | Essential/Important/Optional |

### Missing from Abstract

[Findings that should be there, with suggested sentences]

### Could Be Cut or Shortened

[Abstract content that could be trimmed]

### Emphasis Misalignment

[Where weight doesn't match importance]

**Summary**: N of M top findings covered, N essential findings missing, N sentences could be cut.

---

## Structural Assessment

- Flow: [assessment]
- Research question clarity: [assessment]
- "So what" factor: [assessment]
- Opening sentence: [assessment]
- Closing sentence: [assessment]
- Word count: [N] (target: [range])

---

## Fixes Applied

*Include this section only if `FIX_MODE` is `review`, `diff`, or `apply`. Omit if `FIX_MODE` is `report`.*

| # | Change Type | Current Text | Revised Text | Reason | Status |
|---|---|---|---|---|---|
| 1 | Revise / Cut / Add | ... | ... | ... | Applied / Skipped |

**Summary:** X changes applied, Y skipped.

---

## Revised Abstract

*Include only if `FIX_MODE` is not `report` and changes were applied. Show the final abstract as it stands after edits.*

> [Revised abstract text]

---

## Recommended Abstract

*Include only if `FIX_MODE` is `report`. This is the skill's recommended revision for the author to consider.*

> [Recommended revised abstract]

**Changes from original:**
1. [Change 1 and why]
2. [Change 2 and why]
3. ...

---

## Priority Action Items

**CRITICAL** (abstract misrepresents the paper):
1. ...

**MAJOR** (abstract omits important content or overstates):
2. ...

**MINOR** (polish):
3. ...

---

## Appendix: Agent Outputs

### A. Verification (Abstract → Body)
[Paste `VERIFICATION_SUMMARY`]

### B. Coverage (Body → Abstract)
[Paste `COVERAGE_SUMMARY`]
```

## Final User Message

After writing the report, tell the user:
- That the abstract check is complete
- The path to the saved report
- The overall assessment (3-5 sentences)
- The claim verification summary (N of M supported, etc.)
- The coverage summary (N of M top findings covered, N essential missing)
- If fixes were applied: how many applied and skipped
- The top 3 priority action items
- If `FIX_MODE` is `report`: note that a recommended revised abstract is in the report
- If `FIX_MODE` is `diff`: remind the user to review changes in VS Code Source Control
