# Agent Skills

Claude Code skills for academic paper review.

| Skill | What it does |
|---|---|
| `writing_editor.md` | Multi-agent writing editor. Clarity, flow, voice. |
| `structural-editor.md` | 2-agent structural editor. Paragraph-level structure (lead sentences, given-new flow, unity, transitions) and section/document architecture (paragraph ordering, heading hierarchy, section handoffs, narrative arc). |
| `abstract-checker.md` | 2-agent bidirectional abstract checker. Verifies every abstract claim against the body (abstract → body) and checks that the most important findings are highlighted (body → abstract). Flags numerical mismatches, overstatement, missing findings, emphasis misalignment. |
| `statistical-reviewer.md` | Statistical methods review. Assumptions, reporting, causal inference. Optional named-perspective reviewer (Gelman, Pearl, Rubin, Ioannidis, etc.). |
| `top3-reviewer.md` | Logic, coherence, and contribution review. Argument architecture (Toulmin analysis), framing ("so what?"), and devil's advocate. Pitched at Science/Nature/PNAS standards. |
| `bibliography-checker.md` | Citation and bibliography audit. Verifies citations exist (via web search), checks characterization faithfulness, finds uncited claims, detects near-paraphrase, identifies missing relevant papers. |
| `review-paper-code.md` | 3-agent research code review. Reproducibility & execution feasibility, data integrity & analytical logic (leakage, merges, magic numbers), paper-to-code mapping & results verification (CODECHECK principle). Domain-aware (econ, polisci, ML, biostats). |
| `replication-archive.md` | 3-agent replication package auditor and README generator. Audits archive structure against SSDE template (7 sections), code reproducibility (master script, paths, seeds, dependencies, batch mode), and paper-archive alignment (table/figure/inline-number mapping). Generates README, LICENSE, config files. Supports AEA, AJPS, Nature standards. |
| `r-refactor.md` | 2-agent R code refactoring for research replication code and visualization. Modernizes tidyverse idioms (dplyr 1.1+), fixes anti-patterns, simplifies pipelines, and reviews ggplot2 for publication readiness, color accessibility, and theme consistency. Optionally runs lintr/styler. |

## Usage

Add a skill to your Claude Code project by copying the `.md` file into your skills directory, or reference it directly with `/skill path/to/skill.md`.
