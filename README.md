# CSV Fusion — Specialist LM research notes

Public/research documentation for **CSV Fusion Report Engine** architecture and the plan to marry it with a **small specialist language model** (report planner sidecar).

## Main architecture document

**[CSV_FUSION_SPECIALIST_LM.md](./CSV_FUSION_SPECIALIST_LM.md)** — full write-up for Grok Heavy / mobile follow-on research (design decisions locked; specialist not yet implemented).

## New research handoff documents (2026-08-03)

These documents expand the research brief with concrete findings, links, failure-mode catalogues, and guidance for the GROK Research tool and future implementation sessions.

| Document | Content |
|----------|---------|
| **[RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md](./RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md)** | Curated public multi-table CSV / relational datasets with documented relationships or quality issues; catalogue of hard dataset characteristics (grain traps, temporal misalignment, weak signals, etc.); synthetic factories with controlled fault injection that align with the gold-data strategy. |
| **[RESEARCH_HANDOFF_DETERMINISTIC_TECHNIQUES.md](./RESEARCH_HANDOFF_DETERMINISTIC_TECHNIQUES.md)** | Inclusion-dependency / foreign-key discovery algorithms (Spider, Binder, Faida, HoPF, Desbordante…); schema-matching tools (Valentine); interestingness measures for ranking report proposals; multi-table profiling tools. All deterministic / reproducible. |
| **[RESEARCH_ADVICE_AND_CRITIQUE.md](./RESEARCH_ADVICE_AND_CRITIQUE.md)** | Actionable guidance for briefing GROK Research, critique of design trajectory, recommended local priorities (dataset factory first), and prompt patterns that yield implementable results. |

## Local full product (not in this repo)

On the operator PC:

`C:\AI\GROK_Build\CSV_Fusion_Reports`

This GitHub repo intentionally holds **docs only** (no large Excel/CSV eval artifacts).

## Resume checklist

See the main document §12. Dataset factory + validated sidecar before any fine-tune. The new research documents supply concrete starting points for both.
