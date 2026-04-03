# Agent Skills

Claude Code skills for academic paper review.

| Skill | What it does |
|---|---|
| `writing_editor.md` | Multi-agent writing editor. Clarity, flow, voice, AI-writing detection. |
| `statistical-reviewer.md` | Statistical methods review. Assumptions, reporting, causal inference. Optional named-perspective reviewer (Gelman, Pearl, Rubin, Ioannidis, etc.). |
| `top3-reviewer.md` | Logic, coherence, and contribution review. Argument architecture (Toulmin analysis), framing ("so what?"), and devil's advocate. Pitched at Science/Nature/PNAS standards. |
| `bibliography-checker.md` | Citation and bibliography audit. Verifies citations exist (via web search), checks characterization faithfulness, finds uncited claims, detects near-paraphrase, identifies missing relevant papers. |
| `review-paper-code.md` | 3-agent research code review. Reproducibility & execution feasibility, data integrity & analytical logic (leakage, merges, magic numbers), paper-to-code mapping & results verification (CODECHECK principle). Domain-aware (econ, polisci, ML, biostats). |
| `abstract-checker.md` | 2-agent bidirectional abstract checker. Verifies every abstract claim against the body (abstract → body) and checks that the most important findings are highlighted (body → abstract). Flags numerical mismatches, overstatement, missing findings, emphasis misalignment. |

## Usage

Add a skill to your Claude Code project by copying the `.md` file into your skills directory, or reference it directly with `/skill path/to/skill.md`.
