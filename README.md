# CSV Fusion — Specialist LM research notes

Public/research documentation for **CSV Fusion Report Engine** architecture and the plan to marry it with a **small specialist language model** (report planner sidecar).

## Main architecture document

**[CSV_FUSION_SPECIALIST_LM.md](./CSV_FUSION_SPECIALIST_LM.md)** — full write-up for Grok Heavy / mobile follow-on research (design decisions locked; specialist not yet implemented).

## Research handoff & factory specification documents (2026-08-03)

These documents expand the research brief with concrete findings, links, failure-mode catalogues, guidance for the GROK Research tool, and a complete specification of synthetic test fixtures that should be generated.

| Document | Content |
|----------|---------|
| **[RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md](./RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md)** | Curated public multi-table CSV / relational datasets with documented relationships or quality issues; catalogue of hard dataset characteristics (grain traps, temporal misalignment, weak signals, etc.); synthetic factories with controlled fault injection that align with the gold-data strategy. |
| **[RESEARCH_HANDOFF_DETERMINISTIC_TECHNIQUES.md](./RESEARCH_HANDOFF_DETERMINISTIC_TECHNIQUES.md)** | Inclusion-dependency / foreign-key discovery algorithms (Spider, Binder, Faida, HoPF, Desbordante…); schema-matching tools (Valentine); interestingness measures for ranking report proposals; multi-table profiling tools. All deterministic / reproducible. |
| **[RESEARCH_ADVICE_AND_CRITIQUE.md](./RESEARCH_ADVICE_AND_CRITIQUE.md)** | Actionable guidance for briefing GROK Research, critique of design trajectory, recommended local priorities (dataset factory first), and prompt patterns that yield implementable results. |
| **[DATASET_FACTORY_TEST_FIXTURES.md](./DATASET_FACTORY_TEST_FIXTURES.md)** | **Comprehensive specification of synthetic datasets that should be created.** 10 domain bases + 40+ named damage recipes (orphans, grain/fan/chasm traps, temporal misalignment, name collisions, weak INDs, composite keys, sparsity, type drift, selectivity stress, etc.). For every fixture: clean expected results *and* expected results after damage, so `eval_runner` can automatically verify CSV Fusion behaviour. |

## Local full product (not in this repo)

On the operator PC:

`C:\AI\GROK_Build\CSV_Fusion_Reports`

This GitHub repo intentionally holds **docs only** (no large Excel/CSV eval artifacts).

## Resume checklist

See the main document §12. Dataset factory + validated sidecar before any fine-tune. The new research documents and the Test Fixtures specification supply concrete starting points for both.
