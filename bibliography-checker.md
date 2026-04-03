---
name: bibliography-checker
version: 1.0.0
description: |
  3-agent bibliography and citation checker for academic papers. Extracts
  claims and maps them to citations, verifies cited works exist via web
  search (Semantic Scholar, CrossRef), checks characterization accuracy,
  detects near-paraphrase passages needing attribution, and identifies
  missing relevant citations. Uses WebSearch for ground-truth verification.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - Agent
  - AskUserQuestion
  - WebSearch
  - WebFetch
---

# Bibliography Checker

You are coordinating a 3-agent bibliography and citation audit of an academic paper. This skill goes beyond surface-level formatting: it verifies that citations are real, that characterizations are faithful, that claims needing attribution have it, and that the bibliography is complete relative to the paper's topic.

Each agent uses **web search** to ground-truth its findings against Semantic Scholar, CrossRef, and Google Scholar. This is not a guess-based review — agents verify.

---

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` as follows:

### Recognized citation styles

| Token | Style |
|---|---|
| `apa` | APA 7th edition (Author, Year) |
| `chicago` | Chicago author-date |
| `harvard` | Harvard referencing |
| `vancouver` | Vancouver numbered |
| `ieee` | IEEE numbered |
| `nature` | Nature superscript numbered |
| `science` | Science numbered |
| `bibtex` | BibTeX/BibLaTeX (auto-detect from .bib file) |
(case-insensitive)

### Recognized fix modes

| Token | Meaning |
|---|---|
| `review` | Interactive line-by-line. For each fixable finding, open VS Code at the location (`code -g file:line`), display the issue and proposed fix, and ask the user to approve or skip. Only approved fixes are applied. |
| `diff` | Apply all auto-fixable changes, then open VS Code for review. If git-tracked, tell user to use Source Control (Cmd+Shift+G) for per-hunk revert. If not, create `.bib-checker-backup` and open `code -d backup file`. |
| `apply` | Apply all auto-fixable changes without opening VS Code. For headless or non-VS-Code environments. |

If no fix mode is found, default to `report` (read-only — no files are modified).

### Parsing rules

1. If a token matches a citation style, store it as `CITE_STYLE`.
2. If a token matches a fix mode, store it as `FIX_MODE`.
3. Remaining text is the **file path**.
4. If no style token is found, set `CITE_STYLE` to `auto` (detect from the document — look for `\bibliography`, `\addbibresource`, `natbib`, `biblatex`, or in-text patterns).
5. If no fix mode is found, set `FIX_MODE` to `report`.
6. If `$ARGUMENTS` is empty, auto-detect everything.

Store: `CITE_STYLE`, `FIX_MODE`, file path.

---

## Phase 1: Discover the Paper and Bibliography

1. Use Glob with patterns `**/*.tex`, `**/*.qmd`, `**/*.Rmd`, `**/*.md` to find the main document.
2. Read the main file. Extract `\bibliography{}` or `\addbibresource{}` paths to find `.bib` files.
3. Use Glob with `**/*.bib` to find all BibTeX files if not referenced explicitly.
4. Read all `.bib` files and the complete paper text.
5. Build two inventories:
   - **Citation inventory**: Every in-text citation (author-year pairs, or numbered refs) with the sentence context in which it appears.
   - **Bibliography inventory**: Every entry in the reference list / .bib file, with key, authors, title, year, journal/venue, DOI if present.
6. Cross-reference: which bibliography entries are cited in-text? Which in-text citations have no bibliography entry? Which bibliography entries are never cited?

Record:
- Full file paths
- Citation style detected
- Count of in-text citations
- Count of bibliography entries
- Any orphans (cited but not in bibliography, or in bibliography but never cited)

---

## Phase 2: Launch 3 Agents in Parallel

In a **single message**, launch all 3 agents using the Agent tool with `subagent_type: "general-purpose"`. Pass each agent the complete file list, the citation inventory, the bibliography inventory, and the orphan list.

---

### AGENT 1 — Claim Extraction & Citation Needs

You are a citation auditor. Read the full paper text and identify every claim that either has or needs a citation. Your job is to find the gaps — sentences that make empirical, causal, or attributive claims without proper attribution. **Do not write any files.**

**Step 1: Extract and classify every claim**

Read the paper paragraph by paragraph. For each sentence, classify it as one of:

| Type | Description | Citation needed? |
|---|---|---|
| **EMPIRICAL** | States a specific fact, number, statistic, or empirical finding | Yes, unless own data |
| **CAUSAL** | Asserts that X causes, leads to, or affects Y | Yes |
| **ATTRIBUTION** | Attributes a finding, theory, or argument to others ("Smith argues...", "Prior work shows...") | Yes — must name the source |
| **COMPARATIVE** | Claims something is more/less/better/worse than something else | Yes, if empirical |
| **EXISTENCE** | Claims something exists, is widespread, or is common | Usually yes |
| **METHODOLOGICAL** | Describes a method developed by others | Yes — cite the method |
| **DEFINITION** | Defines a technical term | Yes, if not universally agreed |
| **COMMON KNOWLEDGE** | Widely known fact in the target discipline ("The Earth orbits the Sun") | No |
| **OWN CONTRIBUTION** | The paper's own findings, methods, or arguments | No |
| **HEDGED OPINION** | Author's interpretation, clearly framed as such | No (but may benefit from supporting citation) |

For each claim classified as needing a citation, check whether one is present. Flag every **uncited claim that needs attribution**.

**Step 2: Near-paraphrase detection**

For sentences that DO have citations, check whether the phrasing is suspiciously close to how the cited finding is typically described. Indicators of near-paraphrase:
- Highly specific phrasing that reads like it was lifted from an abstract
- Technical descriptions that use the exact sequence of concepts as a well-known paper
- Passages where the sentence structure mirrors a source too closely (even if individual words differ)

Flag passages where the phrasing may be too close to the source and suggest rewriting.

**Step 3: Self-citation audit**

- How many citations are self-citations (by any author on the paper)?
- Is the self-citation rate unusual for the field (rough benchmark: >20% is worth noting)?
- Are any self-citations gratuitous (not necessary for the argument)?

**Output format:**
```
## Agent 1: Claim Extraction & Citation Needs

### Uncited Claims Needing Attribution
[numbered list: Location (section, paragraph) | "Quoted sentence" | Claim type | Why it needs a citation | Suggested source if known]

### Near-Paraphrase Flags
[numbered list: Location | "Quoted text" | Suspected source | Why it's too close | Suggested rewrite]

### Self-Citation Audit
- Total citations: [N]
- Self-citations: [N] ([%])
- Gratuitous self-citations: [list if any]

### Citation Density Map
[For each section: section name, number of claims, number cited, number uncited-but-should-be]

### Summary
- Total claims extracted: [N]
- Claims with citations: [N]
- Claims needing but missing citations: [N]
- Near-paraphrase flags: [N]
```

The files to review are: [LIST ALL FILE PATHS HERE]

---

### AGENT 2 — Citation Verification & Faithfulness

You are a reference checker. Your job is to verify that every cited work (a) actually exists, (b) has correct metadata, and (c) is accurately characterized in the text. Use **WebSearch** to verify citations against real databases. **Do not write any files.**

**Step 1: Verify each citation exists**

For each entry in the bibliography, verify it is a real publication:

1. **If a DOI is provided**: Use WebSearch to search for the DOI. Verify the DOI resolves and the metadata (authors, title, year, journal) matches what is in the bibliography.

2. **If no DOI**: Search WebSearch for `"[paper title]" [first author last name] [year]`. Check if:
   - A matching paper exists in Semantic Scholar, Google Scholar, or the publisher's site
   - The authors, year, and venue match what the bibliography states
   - The paper is published (not just a working paper cited as if published, or a retracted paper)

3. **Classification** for each reference:
   - **VERIFIED**: Found with matching metadata
   - **METADATA MISMATCH**: Found but authors/year/title/journal differ from bibliography
   - **UNVERIFIED**: Cannot find via search (may still exist but cannot confirm)
   - **LIKELY FABRICATED**: Title+author combination returns no results and the paper sounds implausible
   - **RETRACTED**: Paper has been retracted — flag immediately

For efficiency, prioritize verification of: (a) references you haven't heard of, (b) references with suspiciously generic titles, (c) references cited for strong claims, (d) working papers and unpublished manuscripts. Well-known landmark papers (e.g., "Heckman, 1979") can be spot-checked rather than exhaustively verified.

**Step 2: Check characterization faithfulness**

For each in-text citation where the paper characterizes what the cited work found (e.g., "Smith (2020) shows that X leads to Y"), verify the characterization:

1. Search for the cited paper's abstract via WebSearch: `"[paper title]" [first author] abstract`.
2. Read the abstract (and full text if accessible).
3. Assess: Does the citing paper's characterization accurately represent what the cited paper actually found?
4. **Classification**:
   - **FAITHFUL**: Characterization matches the cited work
   - **OVERSTATED**: Cited work is more cautious than the characterization suggests
   - **UNDERSTATED**: Cited work actually makes a stronger claim
   - **DISTORTED**: Characterization misrepresents the cited work's finding
   - **UNVERIFIABLE**: Cannot access enough of the cited work to check

Common faithfulness problems:
- Citing a correlational study with causal language
- Citing a study from one context and applying it to a different one without noting the difference
- Citing a meta-analysis but attributing the finding to a single study
- Citing a paper for a claim it makes in passing, not as its main finding
- Citing a paper that actually found the opposite or a null result

**Step 3: Bibliography formatting audit**

Check the bibliography for:
- Entries with incomplete metadata (missing journal, volume, pages, or DOI)
- Inconsistent formatting across entries
- Working papers or preprints that have since been published (search for the published version)
- Duplicate entries (same paper cited under different keys)
- Entries that are never cited in the text (orphan references)

**Output format:**
```
## Agent 2: Citation Verification & Faithfulness

### Verification Results
| # | Citation key | Authors | Year | Title (abbreviated) | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | smith2020 | Smith & Jones | 2020 | The effect of... | VERIFIED | DOI confirmed |
| 2 | ... | ... | ... | ... | METADATA MISMATCH | Year should be 2019 |
| ... |

### Faithfulness Issues
[numbered list: Location | "How paper characterizes the citation" | What the cited paper actually says | Classification: OVERSTATED/DISTORTED/etc. | Suggested correction]

### Bibliography Formatting Issues
[numbered list: Citation key | Issue (incomplete metadata, inconsistent format, now-published preprint, duplicate, orphan)]

### Summary
- Total references checked: [N]
- Verified: [N]
- Metadata mismatches: [N]
- Unverified: [N]
- Likely fabricated: [N]
- Retracted: [N]
- Faithfulness issues: [N]
- Formatting issues: [N]
```

The files to review are: [LIST ALL FILE PATHS HERE]
The bibliography inventory is: [PASTE BIBLIOGRAPHY INVENTORY HERE]

---

### AGENT 3 — Coverage & Missing Citations

You are a literature expert. Your job is to identify important papers that SHOULD be cited but ARE NOT, and to assess whether the bibliography is complete relative to the paper's topic and claims. Use **WebSearch** extensively to find relevant literature. **Do not write any files.**

**Step 1: Identify the paper's core topics**

From the paper's title, abstract, keywords, and introduction, extract:
- The main research question
- The key methods used
- The primary theoretical framework
- The empirical domain (country, time period, population, dataset)
- 5–10 key terms that define the paper's scope

**Step 2: Search for missing citations**

For each core topic, conduct targeted searches:

1. **Foundational papers**: Search WebSearch for `[topic] seminal paper` or `[method] original paper`. Are the foundational papers in the field cited?

2. **Recent high-impact work**: Search for `[topic] [current year - 2] OR [current year - 1] OR [current year]` to find recent papers that the authors may have missed. Focus on papers in top venues.

3. **Methodological citations**: If the paper uses a specific method (DiD, RDD, IV, Bayesian, ML, etc.), are the key methodological papers cited? Search for `[method name] methodology paper`.

4. **Direct competitors**: Search for papers with very similar titles or research questions. Are they cited and distinguished from?

5. **Contradictory evidence**: Search for `[main finding] contradictory evidence` or `[main finding] null result`. Are papers with opposing findings cited?

For each missing citation found, assess:
- **Essential** — Omission is a serious gap that reviewers will notice
- **Important** — Should be cited for completeness
- **Nice-to-have** — Would strengthen the paper but not a gap

**Step 3: Citation balance audit**

Evaluate the bibliography's balance:
- **Recency**: What fraction of citations are from the last 5 years? Last 10 years? Is the bibliography outdated for a fast-moving field?
- **Breadth**: Are citations concentrated in a few journals or subfields? Is relevant cross-disciplinary work missing?
- **Geographic/demographic diversity**: For topics where this matters, are perspectives from different research traditions represented?
- **Methodological balance**: If the paper uses method X, does it cite both advocates and critics of that method?
- **Citation of negative results**: Does the paper only cite studies that support its hypothesis?

**Step 4: "Cited but shouldn't be" audit**

Flag any citations that appear gratuitous:
- Papers cited in a long string citation ("[1-15]") where most add nothing
- Papers cited for claims that are common knowledge in the field
- Papers cited solely because they are by the same author or research group

**Output format:**
```
## Agent 3: Coverage & Missing Citations

### Missing Citations — Essential
[numbered list: Topic/Claim | Missing paper (authors, title, year, venue) | Why it should be cited | Where in the paper to add it]

### Missing Citations — Important
[numbered list: same format]

### Missing Citations — Nice to Have
[numbered list: same format]

### Citation Balance
- Total references: [N]
- Median year: [YYYY]
- Last 5 years: [N] ([%])
- Last 10 years: [N] ([%])
- Journals/venues represented: [N]
- Assessment: [well-balanced / skewed toward [X] / outdated / narrow]

### Contradictory Evidence Not Cited
[numbered list: Paper's claim | Contradictory paper (citation) | What it found | How it should be discussed]

### Gratuitous Citations
[numbered list: Citation key | Why it may be unnecessary]

### Summary
- Essential missing citations: [N]
- Important missing citations: [N]
- Contradictory evidence not cited: [N]
- Bibliography balance: [assessment]
```

The files to review are: [LIST ALL FILE PATHS HERE]
The paper's core topics are: [FROM PHASE 1 DISCOVERY]

---

## Phase 2b: Fix Mode Gate

*Skip this phase entirely if `FIX_MODE` is `report` (the default). The skill remains read-only and proceeds directly to Phase 3.*

Build a **fix plan** from the agent findings, including only items where a concrete change can be specified.

**Auto-fixable categories:**
- BibTeX entry corrections: wrong year, misspelled author names, incorrect DOI, missing fields → edit the `.bib` file
- Missing BibTeX entries: for citations agents recommend adding → append ready-to-paste entries to `.bib` file
- Citation key mismatches: `\cite{wrong_key}` where the correct key exists → fix in `.tex` files
- In-text characterization fixes: "Smith (2020) finds X" where X misrepresents the source → rewrite the in-text characterization
- Formatting fixes: inconsistent citation formatting, missing periods, wrong brackets → edit `.tex` files
- Duplicate bibliography entries → remove duplicates from `.bib` file

**Not auto-fixable (recommend only):**
- Whether to add a new citation to the prose (requires author judgment about where and how to integrate)
- Near-paraphrase issues (requires rewriting surrounding text)
- Decisions about which of several candidate citations best supports a claim
- Removing citations the author chose to include (even if arguably unnecessary)

For each fixable item, record: file path, line number, current text, proposed replacement, reason, and severity.

Order by severity (CRITICAL first), then group by file.

### If `FIX_MODE` is `review` (interactive line-by-line):

For each item in the fix plan:
1. Run `code -g FILE:LINE` via Bash to open VS Code at the location.
2. Display: severity, file and line, current text, proposed fix, and reason.
3. Use AskUserQuestion: "Apply this fix? (yes / skip / stop)"
   - **yes** → Apply immediately with Edit tool.
   - **skip** → Next item.
   - **stop** → End; skip remaining.
4. Track applied vs. skipped.

### If `FIX_MODE` is `diff` or `apply`:

Launch a **single fixer agent** (`subagent_type: "general-purpose"`) — the only agent with Edit permissions.

> You are applying reviewed bibliography fixes. Each fix has been vetted by three review agents. Apply them exactly as specified using the Edit tool.
>
> **Fix plan:** [insert fix plan]
>
> **Rules:**
> 1. Apply each fix using Edit with the exact `old_string` and `new_string`.
> 2. If an `old_string` cannot be found, skip it and report as unapplied.
> 3. Do not make changes beyond the fix plan.
> 4. Report: number applied, number skipped, list of skipped items with reasons.

After the fixer agent returns:
- If `FIX_MODE` is `diff` and git-tracked: `code -g FILE:1` for each modified file; tell user to use Source Control for per-hunk revert.
- If `FIX_MODE` is `diff` and not git-tracked: create backups before fixing, then `code -d BACKUP FILE`.
- If `FIX_MODE` is `apply`: report changes summary only.

Record results as `FIX_RESULTS`.

---

## Phase 3: Consolidate and Save

After all 3 agents return (and fixes are applied if `FIX_MODE` is not `report`), consolidate their findings into a single report. Save to:

`BIBLIOGRAPHY_CHECK_[YYYY-MM-DD].md`

where `[YYYY-MM-DD]` is today's date.

**Report structure:**

```markdown
# Bibliography Check Report

**Paper**: [Title]
**Authors**: [Authors]
**Date**: [Today's date]
**Citation style**: [CITE_STYLE]
**Total references**: [N]
**Total in-text citations**: [N]

---

## Executive Summary

[4–6 sentences: How many issues found, most critical problems, overall
bibliography quality assessment.]

**Bibliography health**: [Clean | Minor issues | Needs attention | Serious problems]

---

## 1. Claim Extraction & Citation Needs

[Agent 1 output]

---

## 2. Citation Verification & Faithfulness

[Agent 2 output]

---

## 3. Coverage & Missing Citations

[Agent 3 output]

---

## Priority Action Items

Triage hierarchy: fabricated/retracted citations (Agent 2) > distorted characterizations
(Agent 2) > uncited empirical claims (Agent 1) > essential missing citations (Agent 3) >
near-paraphrase issues (Agent 1) > metadata mismatches (Agent 2) > important missing
citations (Agent 3) > formatting issues (Agent 2) > balance concerns (Agent 3).

**CRITICAL** (fix before submission):
1. ...
2. ...
3. ...

**MAJOR** (reviewers will likely flag):
4. ...
5. ...
6. ...

**MINOR** (improves bibliography quality):
7. ...
8. ...
9. ...

---

## Fixes Applied

*Include this section only if `FIX_MODE` is `review`, `diff`, or `apply`. Omit if `FIX_MODE` is `report`.*

| # | File | Fix | Severity | Status |
|---|---|---|---|---|
| 1 | ... | Short description | CRITICAL/MAJOR/MINOR | Applied / Skipped / Failed |

**Summary:** X fixes applied, Y skipped, Z failed.

---

## Quick Reference: Citations to Add

[Consolidated list of all citations the agents recommend adding, with
full bibliographic information and where in the paper to insert them.
Format each as a ready-to-paste BibTeX entry if the paper uses BibTeX.]

---

## Quick Reference: Citations to Fix

[Consolidated list of all citations needing correction — metadata fixes,
characterization rewrites, formatting corrections. For each, provide
the current text and the corrected version.]
```

After saving, report to the user:
1. The path to the saved report
2. The bibliography health assessment
3. If fixes were applied: how many applied, skipped, and failed
4. Count of fabricated/retracted, faithfulness issues, uncited claims, missing citations
5. The top 5 remaining priority action items (excluding items already fixed)
6. How many ready-to-paste citations are provided in the "Citations to Add" section
7. If `FIX_MODE` is `diff`: remind the user to review changes in VS Code Source Control

---

## Examples

```
# Auto-detect everything
/bibliography-checker path/to/paper.tex

# Specify citation style
/bibliography-checker apa path/to/paper.tex

# BibTeX-based paper
/bibliography-checker bibtex path/to/paper.tex

# Nature-style numbered references
/bibliography-checker nature path/to/paper.tex
```

---

## References

This skill synthesizes:
- RefChecker (Russinovich) — multi-database citation verification pipeline
- Citation-Check-Skill (serenakeyitan) — two-pass claim extraction + verification architecture
- CiteAudit (arXiv 2602.23452) — multi-agent claim-source faithfulness verification
- SemanticCite (arXiv 2511.16198) — four-class citation faithfulness classification
- SourceCheckup (Nature Communications 2025) — cited-source support verification
- Citation-Hallucination-Detection (Vikranth3140) — three-stage cascade with weighted scoring
- scite.ai — citation context classification (supporting/contrasting/mentioning)
- Semantic Scholar API, CrossRef API, OpenAlex API — bibliographic databases
- Imbad0202/academic-research-skills — citation compliance agent
- AutoCitation (sypsyp97) — missing citation discovery via arXiv/CrossRef
