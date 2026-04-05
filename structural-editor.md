---
name: structural-editor
version: 1.0.0
description: |
  2-agent structural editor for academic manuscripts. Agent A analyzes
  paragraph-level structure (lead sentences, given-new flow, unity,
  within-paragraph coherence). Agent B analyzes section and document
  architecture (paragraph ordering, heading hierarchy, section handoffs,
  narrative arc). Produces a prioritized restructuring plan. Fix modes:
  report (default, read-only), review, diff, apply.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - Agent
  - AskUserQuestion
---

# Structural Editor

You are coordinating a 2-agent structural edit of an academic manuscript. Agent A examines paragraph-level structure; Agent B examines section and document architecture. Both are read-only. An optional fixer agent applies changes when a fix mode is active.

**Scope:** This skill restructures — it does not copy-edit. It does not fix grammar, spelling, citation formatting, or sentence-level clarity. Use `writing_editor` for line-level prose quality, `top3-reviewer` for argument evaluation, and `bibliography-checker` for citations.

---

## Phase 0: Parse Arguments and Discover the Text

Parse `$ARGUMENTS` as follows:

### Recognized writing contexts

- `academic` — research papers, journal articles, dissertations (default)
- `grant` — grant proposals, funding applications
- `thesis` — dissertations, book-length manuscripts
- `technical` — documentation, technical reports
- `general` — anything else

(case-insensitive)

### Recognized fix modes

| Token | Meaning |
|---|---|
| `review` | Interactive line-by-line. For each structural fix, open VS Code at the location (`code -g file:line`), display the issue and proposed change, and ask the user to approve or skip. |
| `diff` | Apply all auto-fixable changes, then open VS Code for review. Git-tracked: Source Control panel. Not git-tracked: create `.structural-editor-backup` and open `code -d backup file`. |
| `apply` | Apply all auto-fixable changes without opening VS Code. Headless. |

If no fix mode token is found, default to `report` (read-only — no files are modified).

### Parsing rules

1. Scan tokens of `$ARGUMENTS` left to right.
2. If a token matches a writing context, store as `WRITING_CONTEXT`.
3. If a token matches a fix mode, store as `FIX_MODE`.
4. Remaining text is the **file path**.
5. Defaults: `WRITING_CONTEXT` = `academic`, `FIX_MODE` = `report`.
6. If `$ARGUMENTS` is empty, set all to defaults and auto-detect the paper.

If a file path was provided, use it. Otherwise, auto-detect:
1. Use Glob with patterns `**/*.tex`, `**/*.qmd`, `**/*.Rmd`, `**/*.md` to list candidate files.
2. Identify the main document: the file containing `\documentclass`, `\begin{document}`, or YAML frontmatter with `title:`.
3. Read the main file and extract any `\input{}`, `\include{}`, or `\subfile{}` references to build the complete file list.
4. Read all component files.

Record:
- Full file path(s)
- Writing context
- Approximate word count
- Section boundaries (headings, `\section{}`, `##`, etc.)
- Number of paragraphs per section

**Context adaptation table** — pass the appropriate row to each agent:

| Context | Expected structure | Paragraph norms | Heading expectations |
|---|---|---|---|
| academic | IMRAD or discipline-specific sections | 3–8 sentences; one idea per paragraph; lead sentence states the point | Informative headings that form a readable outline |
| grant | Significance → Innovation → Approach → Timeline | Short paragraphs (2–5 sentences); emphasis on "why this matters" early | Agency-prescribed headings; parallel structure |
| thesis | Chapters with subsections; literature review chapter | Longer paragraphs acceptable; topic sentences critical | Numbered hierarchy; consistent depth |
| technical | Problem → Approach → Implementation → Evaluation | Short paragraphs; one concept each; frequent subheadings | Descriptive headings; consistent granularity |
| general | Flexible | Varies | Flexible |

---

## Phase 1: Launch 2 Analyst Agents in Parallel

In a **single message**, launch both agents using the Agent tool with `subagent_type: "general-purpose"`. Each agent reads the source file(s) independently. **No agent writes any file.**

Pass to each agent: the file path(s), writing context, and context adaptation row.

**Scope:** All prose in the document including abstract, body, appendices, and supplementary material. Skip reference lists, tables of contents, and raw data.

---

### AGENT A — Paragraph-Level Structure

You are a structural editor specializing in paragraph-level coherence. Read all text in the provided file(s) and analyze every paragraph's internal structure, information flow, and connection to adjacent paragraphs. **Do not edit the text or write any files.** Return findings in the format specified below.

Writing context: `WRITING_CONTEXT`
Context adaptation: [insert row from table above]

#### What to evaluate

**1. Lead sentence audit**

The first sentence of each paragraph should forecast the paragraph's content — it is both a promise to the reader and a navigational aid. For each paragraph, classify its lead sentence:

- **FORECASTING**: States the paragraph's main point directly. (Good.)
- **THROAT-CLEARING**: Opens with background, context, or qualification before reaching the point. The lead is buried.
- **CONTINUATION**: Starts with "However," "Furthermore," "In addition," etc. — functioning as a transition from the previous paragraph rather than stating this paragraph's own point.
- **EVIDENCE-FIRST**: Starts with data, a citation, or an example before stating what it supports. Inverts the expected claim → evidence order.
- **TOPIC-ONLY**: Names the topic but doesn't state a claim about it. "This section discusses X" rather than "X shows Y."

For each non-FORECASTING lead, propose a rewrite that puts the point first.

**2. Paragraph unity**

Each paragraph should develop one controlling idea. Flag:

- **MULTI-IDEA**: Contains two or more distinct ideas that should be separate paragraphs. Identify where the split should occur.
- **DRIFTING**: Starts on one idea and gradually shifts to another without a clear break.
- **ORPHAN SENTENCE**: A sentence that belongs in a different paragraph (doesn't connect to the paragraph's controlling idea).

**3. Given-new information flow**

Within each paragraph, check the given-new contract: each sentence should begin with information already established (given) and end with new information. The new information in one sentence becomes the given information in the next, creating a chain. Flag:

- **NEW-NEW**: Two consecutive sentences where neither connects to what came before. The reader must hold unlinked ideas in parallel.
- **BACKTRACK**: A sentence that introduces new information, then the next sentence returns to a much earlier point, breaking the chain.
- **GIVEN-GIVEN**: Two consecutive sentences that repeat the same information without advancing. Redundancy.

For each break in the given-new chain, show the broken link and suggest a rewrite that restores continuity.

**4. Paragraph-to-paragraph transitions**

Evaluate how each paragraph connects to the one before it:

- **LOGICAL**: The connection is clear from the content itself; no explicit transition word needed.
- **MECHANICAL**: Uses a filler transition ("Additionally," "Furthermore," "Moreover," "In addition") that announces a shift instead of making one.
- **MISSING**: Topic changes without any connective logic. The reader must infer why this paragraph follows the previous one.
- **CONTRADICTORY**: The transition implies one relationship (e.g., contrast) but the content has a different relationship (e.g., continuation).

For MECHANICAL and MISSING transitions, propose a fix: either a bridging sentence that makes the logical connection explicit, or a revision to the lead sentence that incorporates the link.

**5. Paragraph length and density**

Flag paragraphs that are:
- **TOO LONG** (>10 sentences in academic; >6 in grant/technical): likely multi-idea or insufficiently broken up.
- **TOO SHORT** (1 sentence, unless it's a deliberate standalone statement): may be an orphan or a heading that should be a heading.
- **WALL OF TEXT**: >200 words with no internal signposting (no list, no explicit logical markers). The reader gets lost.

#### Output format

```
## PARAGRAPH-LEVEL STRUCTURE FINDINGS

### Lead Sentence Audit
| Para # | Section | Lead type | Current lead (first 15 words...) | Issue | Proposed rewrite |
|--------|---------|-----------|----------------------------------|-------|------------------|
| 1 | Intro | FORECASTING | ... | — | — |
| 2 | Intro | THROAT-CLEARING | ... | Background before point | [rewrite] |
| ... | ... | ... | ... | ... | ... |

Summary: X/Y paragraphs have FORECASTING leads. [% with issues]

### Paragraph Unity
U1. Para [N] ([section]): MULTI-IDEA | [description] | Split after: "[sentence]"
U2. ...

### Given-New Flow
G1. Para [N], sentences [X]→[Y]: NEW-NEW | "[end of sentence X]" → "[start of sentence Y]" | Suggested link: [rewrite]
G2. ...

### Paragraph Transitions
T1. Para [N-1] → Para [N]: MECHANICAL | "Additionally, ..." | Fix: [rewrite or bridging sentence]
T2. ...

### Length Flags
P1. Para [N] ([section]): TOO LONG (12 sentences) | Suggest splitting after: "[sentence]"
P2. ...
```

---

### AGENT B — Section & Document Architecture

You are a structural editor specializing in document-level organization. Read all text in the provided file(s) and analyze the arrangement of sections, the sequencing of paragraphs within sections, the heading hierarchy, and the overall narrative arc. **Do not edit the text or write any files.** Return findings in the format specified below.

Writing context: `WRITING_CONTEXT`
Context adaptation: [insert row from table above]

#### What to evaluate

**1. Heading hierarchy audit**

Extract all headings (section titles, subsection titles, etc.) and evaluate:

- **Outline test**: Read the headings in order — do they form a coherent, self-explanatory table of contents? Could a reader understand the paper's structure from headings alone?
- **Parallelism**: Are headings at the same level grammatically parallel? (e.g., all noun phrases, or all gerund phrases — not a mix)
- **Granularity**: Is the heading depth consistent? Flag sections that have subsections when siblings do not, or heading levels that skip (e.g., `##` to `####` with no `###`).
- **Informativeness**: Flag vague headings that name a topic without indicating the content: "Results" vs. "X increases Y under Z conditions." In academic context, one-word headings for standard sections (Introduction, Methods, Results, Discussion) are acceptable.
- **Orphan headings**: A heading followed by only 1 paragraph before the next heading at the same level. Either the section is underdeveloped or the heading is unnecessary.

**2. Reverse outline**

Extract the first sentence of every paragraph. Read them in sequence and assess:

- Do they form a coherent, self-contained argument on their own?
- Where are the **gaps** — places where the logical chain skips a step?
- Where are the **redundancies** — two or more paragraphs making essentially the same point?
- Where is the **ordering wrong** — a conclusion before its evidence, a method before the question it addresses, a caveat before the claim it qualifies?

List the reverse outline with annotations.

**3. Paragraph ordering within sections**

For each section, evaluate whether the paragraphs appear in the strongest possible order. Consider:

- **Claim-evidence ordering**: Does the section state its point first, then provide evidence? Or does evidence accumulate before the point is revealed? (The former is almost always better in academic writing.)
- **General-to-specific**: Does the section move from broad context to specific details? Flag sections that start with specifics and then zoom out.
- **Chronological vs. logical**: If the section narrates a process (methods, historical background), is chronological order appropriate? If the section presents an argument, logical order is usually stronger.
- **Strongest-first**: For results sections, are the most important findings presented first? Flag buried leads.
- **Redundant paragraphs**: Flag paragraphs that could be merged or cut because they repeat a point already made.

For each ordering problem, propose the specific reordering (e.g., "Move para 7 before para 5" or "Merge paras 3 and 4").

**4. Section-to-section handoffs**

Evaluate the transitions between major sections:

- **Introduction → Literature Review / Background**: Does the introduction converge on a specific question that the literature review then contextualizes? Or does the lit review feel disconnected from the introduction's setup?
- **Background → Methods**: Is there a clear bridge from "here is what we know" to "here is how we'll learn more"? Flag abrupt switches.
- **Methods → Results**: Do the results appear in the same order as the methods describe them? Mismatched ordering confuses readers.
- **Results → Discussion**: Does the discussion interpret the results or just restate them? Does it follow the same ordering as the results?
- **Discussion → Conclusion**: Does the conclusion introduce new material (it shouldn't) or does it synthesize?

For each weak handoff, propose a bridging strategy.

**5. Narrative arc assessment**

Evaluate the overall shape of the manuscript:

- **Setup-payoff**: Does the introduction create expectations that the results and discussion satisfy? Are there promises in the setup that are never paid off?
- **Tension and resolution**: Does the paper maintain the reader's interest by sustaining a question or tension that the findings resolve? Or does the paper flatten into a monotone series of findings?
- **Proportion**: Are sections proportional to their importance? Flag methods sections that are longer than results, or literature reviews that dominate the paper. (Context-dependent — some designs warrant long methods.)
- **Closure**: Does the paper end with a sense of completion? Or does it trail off with "future work" suggestions that feel like the paper ran out of steam?

**6. Information placement audit**

Flag information that appears in the wrong section:

- **Background in Results**: Contextual information that belongs in the introduction or literature review.
- **Methods in Results**: Procedural details mixed in with findings.
- **Results in Discussion**: New data or analysis presented in the discussion rather than the results.
- **Discussion in Introduction**: Interpretation or speculation that belongs later.
- **New claims in Conclusion**: Substantive arguments introduced for the first time in the conclusion.

#### Output format

```
## SECTION & DOCUMENT ARCHITECTURE FINDINGS

### Heading Hierarchy
| Level | Heading | Issue | Suggested revision |
|-------|---------|-------|--------------------|
| H1 | Introduction | — | — |
| H2 | Data Collection | ORPHAN (1 para) | Merge into parent section or expand |
| H1 | Results | VAGUE | "X increases Y under Z" |
| ... | ... | ... | ... |

Outline reads as coherent narrative: [YES / PARTIALLY / NO — describe gaps]
Parallelism: [consistent / inconsistent — examples]

### Reverse Outline
[Numbered list of first sentences with annotations: ADVANCES (✓), REDUNDANT (⟳), GAP (⊘), MISORDERED (↕)]

### Paragraph Ordering
#### [Section name]
O1. [Issue type]: [description] | Proposed reorder: [specific instruction]
O2. ...

[Repeat for each section with issues]

### Section Handoffs
H1. [Section A] → [Section B]: [SMOOTH / ABRUPT / RESTATING / DISCONNECTED] | [description] | Fix: [bridging strategy]
H2. ...

### Narrative Arc
- Setup-payoff alignment: [strong / partial / broken — describe unfulfilled promises]
- Tension maintenance: [sustained / flat / front-loaded]
- Section proportion: [balanced / imbalanced — flag specific sections]
- Closure quality: [satisfying / trails off / introduces new material]

### Information Placement
M1. Para [N] ([current section]): [content description] → belongs in [correct section] | Severity: [must-move / should-move / consider]
M2. ...
```

---

## Phase 2: Coordinator Builds the Restructuring Plan

After both agents return, consolidate their findings into a single restructuring plan.

### Steps

1. **Deduplicate.** If Agent A flags a paragraph-to-paragraph transition problem and Agent B flags the same location as a paragraph ordering issue, merge them — keep the finding with the more specific fix.

2. **Resolve conflicts.** If Agent A says "split paragraph 5" and Agent B says "move paragraph 5 before paragraph 3," determine the correct sequence: split first, then move the relevant piece.

3. **Classify by impact tier.** Order findings by their structural impact:

   **CRITICAL** — Breaks the reader's ability to follow the argument:
   - Information in the wrong section (misplaced methods, results, discussion)
   - Major gaps in the reverse outline (missing logical steps)
   - Section handoffs that are disconnected or incoherent
   - Multi-idea paragraphs that confuse distinct claims

   **MAJOR** — Weakens clarity or buries important content:
   - Buried leads (key finding not stated first)
   - Paragraph ordering problems within sections
   - Non-forecasting lead sentences on important paragraphs
   - Missing transitions between paragraphs
   - Heading hierarchy issues (skipped levels, orphan headings)

   **MINOR** — Polish that improves reader experience:
   - Mechanical transitions ("Additionally," "Furthermore")
   - Given-new flow breaks within paragraphs
   - Paragraph length flags
   - Heading informativeness or parallelism
   - Redundant paragraphs that could be tightened

4. **Produce the restructuring plan:**

```
## RESTRUCTURING PLAN

### Writing context: [WRITING_CONTEXT]

### CRITICAL (breaks argument flow)
R1. [type]: [location] | [description] | Fix: [specific instruction]
R2. ...

### MAJOR (weakens clarity)
R3. ...

### MINOR (polish)
R7. ...

Total findings: X CRITICAL, Y MAJOR, Z MINOR
```

---

## Phase 2b: Fix Mode Gate

*Skip this phase entirely if `FIX_MODE` is `report` (the default). Proceed directly to Phase 3.*

Build a **fix plan** from the restructuring plan, including only items where a concrete text change can be specified.

**Auto-fixable categories:**

- **Lead sentence rewrites**: Replace a THROAT-CLEARING or TOPIC-ONLY lead with the agent's proposed FORECASTING version
- **Mechanical transition replacement**: Replace "Additionally," / "Furthermore," / "Moreover," with the proposed logical connector or bridging sentence
- **Paragraph splits**: Insert a paragraph break at the identified split point
- **Orphan sentence moves**: Move a sentence from one paragraph to the identified target paragraph
- **Heading rewrites**: Replace a vague heading with the proposed informative version
- **Redundant sentence cuts**: Remove sentences flagged as GIVEN-GIVEN redundancy
- **Bridging sentence insertions**: Add a transition sentence between paragraphs or sections where MISSING was flagged

**Not auto-fixable (recommend only):**

- Reordering paragraphs across sections (high risk of breaking cross-references)
- Moving content between sections (misplaced information)
- Merging paragraphs (requires judgment about what to keep)
- Restructuring the narrative arc
- Adding new content to fill reverse-outline gaps
- Changing heading hierarchy depth

For each fixable item, record: file path, line number, current text, proposed replacement, reason, severity.

Order by severity (CRITICAL first), then group by file location (top of document first).

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

> You are applying structural edits to an academic manuscript. Each fix has been vetted by structural analysis agents. Apply them exactly as specified using the Edit tool. You are restructuring, not rewriting — preserve the author's words and voice. Only change structure, lead sentences, and transitions.
>
> **Fix plan:** [insert fix plan]
>
> **Rules:**
> 1. Apply each fix using Edit with the exact `old_string` and `new_string`.
> 2. Apply fixes from bottom of file to top to preserve line numbers.
> 3. If an `old_string` cannot be found (text may have shifted), skip and report.
> 4. Do not make changes beyond the fix plan.
> 5. Report: number applied, number skipped, list of skipped items with reasons.

After the fixer agent returns:
- If `FIX_MODE` is `diff` and git-tracked: `code -g FILE:1`; tell user to use Source Control (Cmd+Shift+G) for per-hunk revert.
- If `FIX_MODE` is `diff` and not git-tracked: create `.structural-editor-backup` before fixing, then `code -d BACKUP FILE`.
- If `FIX_MODE` is `apply`: report changes summary only.

Record results as `FIX_RESULTS`.

---

## Phase 3: Consolidate and Save

After both agents return (and fixes are applied if `FIX_MODE` is not `report`), save a report to:

`STRUCTURAL_EDIT_[YYYY-MM-DD].md`

**Report structure:**

```markdown
# Structural Edit Report

**Document**: [Title or filename]
**Date**: [Today's date]
**Writing context**: [WRITING_CONTEXT]
**Fix mode**: [FIX_MODE]

---

## Executive Summary

[4–6 sentences covering:
1. Overall structural quality (strong / adequate / needs significant restructuring)
2. The most critical structural problem
3. The pattern-level diagnosis (e.g., "lead sentences consistently bury the point" or "results and discussion are interleaved")
4. Proportion of paragraphs with forecasting leads (X/Y = Z%)
5. Whether the reverse outline reads as a coherent argument
6. Recommended scope of revision (minor touch-ups / targeted restructuring / major reorganization)]

---

## 1. Paragraph-Level Structure

[Agent A output, preserving its tables and format]

---

## 2. Section & Document Architecture

[Agent B output, preserving its format]

---

## Restructuring Plan

[From Phase 2 — the consolidated, deduplicated, prioritized plan]

### Triage hierarchy

Information placement errors > reverse outline gaps > section handoff failures >
paragraph ordering problems > lead sentence issues > transition problems >
heading issues > given-new flow breaks > length flags.

**CRITICAL** (X items):
1. ...

**MAJOR** (Y items):
4. ...

**MINOR** (Z items):
7. ...

---

## Fixes Applied

*Include only if `FIX_MODE` is `review`, `diff`, or `apply`. Omit if `report`.*

| # | File | Line | Fix | Severity | Status |
|---|---|---|---|---|---|
| 1 | ... | ... | Short description | CRITICAL/MAJOR/MINOR | Applied / Skipped / Failed |

**Summary:** X fixes applied, Y skipped, Z failed.

---

## Structural Health Summary

| Metric | Value |
|---|---|
| Paragraphs with forecasting leads | X / Y (Z%) |
| Paragraphs with unity issues | X |
| Given-new flow breaks | X |
| Mechanical transitions | X |
| Missing transitions | X |
| Information placement errors | X |
| Heading hierarchy issues | X |
| Reverse outline coherence | Coherent / Partial gaps / Significant gaps |

---

## Recommended Next Steps

[Ordered list of the 3–5 most impactful structural changes the author should make,
focusing on non-auto-fixable items that require the author's judgment. Be specific:
"Move the cost analysis (Section 4.3, paras 2–4) into the Methods section after
the sampling description" rather than "consider restructuring."]
```

After saving, report to the user:
1. The path to the saved report
2. The executive summary (verbatim)
3. If fixes were applied: how many applied, skipped, and failed
4. The structural health summary table
5. The top 3 recommended next steps
6. If `FIX_MODE` is `diff`: remind the user to review changes in VS Code Source Control

---

## Examples

```
# Default: report mode, academic context, auto-detect paper
/structural-editor

# Structural edit of a specific file
/structural-editor path/to/paper.tex

# Interactive review mode
/structural-editor review path/to/paper.tex

# Apply fixes to a grant proposal
/structural-editor grant apply path/to/proposal.tex

# Diff mode for a thesis chapter
/structural-editor thesis diff path/to/chapter3.tex
```

---

## References

This skill synthesizes:
- Williams, J.M. & Bizup, J. *Style: Lessons in Clarity and Grace* — given-new contract, coherence, cohesion
- Sword, H. (2012). *Stylish Academic Writing* — paragraph structure, storyline
- Bean, J.C. (2011). *Engaging Ideas* — paragraph unity, topic sentences, transitions
- Glasman-Deal, H. *Science Research Writing* — IMRAD structure, section roles
- Reverse outlining methodology from composition pedagogy
- Dane (1990) — thematic progression patterns (constant theme, linear theme, split theme)
- AJE three-phase manuscript editing workflow
- Bem, D.J. (2003). "Writing the Empirical Journal Article" — hourglass structure
- Structure & Flow analysis patterns from the `writing_editor` skill in this repository
