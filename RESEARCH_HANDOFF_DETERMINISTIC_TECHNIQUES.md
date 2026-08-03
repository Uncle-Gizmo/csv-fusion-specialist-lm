# Research Handoff: Deterministic Techniques for Relationship Discovery, Schema Matching & Report Interestingness

**Document purpose:** Concrete, mostly non-LLM algorithms and measures that can strengthen CSV Fusion’s deterministic pipeline (especially proposal selectivity) and supply vocabulary / training labels for the future specialist report-planner LM. Suitable starting points for GROK Research deeper dives and for direct codification into Python.  
**Date:** 2026-08-03  
**Status:** Team synthesis (Harper lead + collective). Complements main architecture document and the Datasets/Hard-Cases handoff.  
**Key principle:** These techniques remain under the authority of the deterministic stack; the specialist only improves judgment on top of them.

---

## 1. Inclusion Dependency (IND) & Foreign-Key Discovery

Core of relationship-candidate generation.

### Classic & State-of-the-Art Algorithms

From the HPI experimental evaluation of thirteen algorithms (“Inclusion Dependency Discovery: An Experimental Evaluation of Thirteen Algorithms”, Dürsch et al., CIKM 2019 / HPI):

| Algorithm | Style | Notes |
|-----------|-------|-------|
| **Spider** | Exact unary, disk-backed sort-merge, early termination | Classic, reliable baseline |
| **Binder** | Hash-based, memory-partitioned, n-ary via Apriori-style lattice | Strong practical performer on many real datasets |
| **Faida** | Approximate (unary + n-ary) | Fast & scalable; good first-pass filter for large CSV folders |
| **Sindy** (and family SANDY/ANDY) | Distributed / MapReduce-style | For very large or multi-node settings |
| Others evaluated | DeMarchi, Bell & Brockhausen, S-indd, Many, Mind, ZigZag, Find2, etc. | Varying strengths on unary vs n-ary, memory vs disk |

**Modern open-source implementations worth adopting or benchmarking against:**

- **Desbordante** (C++): Highly optimised parallel Spider + Faida (reported 5–8× faster than classic Metanome Java implementations). Excellent candidate for integration or as external oracle for regression tests.
- **Metanome** (Java research platform): Still the reference collection of many of the algorithms above.
- **HoPF** (HPI): Holistic primary/foreign-key detection. Combines unique-column-combinations (UCCs) + INDs with feature-based ranking and aggressive pruning to produce high-precision key candidates. GitHub: HPI-Information-Systems/hopf.

**Recommendation for CSV Fusion:**  
Benchmark current relationship discovery against Desbordante / Binder / Spider on the existing adversarial fixtures and on RelBench-style multi-table sets. Consider adopting ranking features from HoPF to improve confidence scores and reduce low-value candidates.

---

## 2. Schema Matching (Beyond Pure Value Inclusion)

When column names, encodings or units differ (common in real multi-source CSV folders).

- **Valentine** (Python package, TU Delft): https://github.com/delftdata/valentine  
  Implements COMA, Cupid, Similarity Flooding and related matchers. Explicitly designed for dataset-discovery scenarios (ranking inter-table column correspondences). Can be combined with pure IND signals when value overlap is only partial or names are noisy.

Other classical schema-matching literature (COMA++, Similarity Flooding, etc.) remains relevant; Valentine is currently the most convenient open implementation for Python-centric pipelines.

---

## 3. Interestingness Measures for Ranking & Rejecting Report Proposals

Directly addresses the main residual weakness: too many low-value or execution-hostile proposals.

### Objective / Classic Measures
- Support, confidence, lift, conviction, etc.
- Diversity indices: entropy, Gini, Simpson (from information theory / ecology).
- Survey: Geng & Hamilton, “Interestingness Measures for Data Mining: A Survey” (still foundational).

### Multi-dimensional measures for EDA / OLAP / automatic insights
Recent surveys (e.g. ADBIS 2024) organise interestingness along axes such as:
- Peculiarity
- Novelty
- Relevance (to user goal or prior)
- Surprise (contradicts prior beliefs)
- Diversity
- Presentation quality

These can be computed from the health + relationship JSON artefacts that CSV Fusion already produces.

### Subjective Interestingness (SI)
SI ≈ Information Content / Description Length (MDL-style). Useful for ranking connected multi-table patterns.

### Multi-relational pattern mining
- Constraint-programming approaches for most-interesting connected patterns across tables.
- TGD / Horn-rule miners (e.g. MATILDA / MAHILDA family) that surface cross-table implications with auditable support and confidence. Deterministic and grounded in database semantics.

**Recommendation:**  
Implement a lightweight ranked interestingness scorer (start with lift + diversity + a simple SI approximation) inside the deterministic planner. Use it both as a hard filter and as a soft ranking signal. The same scores become natural training targets / rejection labels for the specialist sidecar.

---

## 4. Multi-Table-Aware Profiling & Health Tools

Useful both for inspiration and for external benchmarking:

- Desbordante (already noted)
- YData-Profiling
- Aggregate Profiler
- WhiteRabbit
- dataset-profiler (PyPI) — includes some cross-table FK hints and correlations

CSV Fusion’s health stage can be compared against these for regression and for discovering additional low-cost signals (e.g. type consistency across candidate join columns).

---

## 5. Practical Recommendations for CSV Fusion & Specialist Design

1. **Relationship discovery**  
   Benchmark current engine against Desbordante / Binder / Spider / HoPF ranking. Consider a two-pass approach: fast approximate (Faida-style) followed by exact validation on high-confidence candidates.

2. **Selectivity of proposals**  
   Add deterministic interestingness scoring (lift, diversity, SI, post-join cardinality estimate, grain-trap detection) *before* any specialist is invoked. This alone may solve a large fraction of the “too many low-value proposals” problem.

3. **Schema matching hybrid**  
   When pure IND confidence is marginal, optionally invoke Valentine-style matchers and blend the scores.

4. **Training signal for specialist**  
   Codify the ranking features from HoPF and the interestingness formulas into the eval harness. The specialist then learns to imitate (or improve upon) these deterministic judgments under the same validator gate.

5. **GROK Research prompts that yield high value**  
   - Exact feature lists and pruning heuristics used by HoPF  
   - Practical parameter settings for Faida / Binder on CSV-folder scale data  
   - Open-source libraries or code snippets for multi-table interestingness scoring  
   - Recent advances in grain / fan-trap / chasm-trap detection for automated reporting  
   - Any new deterministic multi-relational pattern miners beyond the 2019 HPI survey

These techniques are reproducible, local, privacy-safe, and many of their finer ranking details remain under-represented in frontier LLM training data — exactly the kind of material the original research brief asked for.

---

*End of document. See also RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md and RESEARCH_ADVICE_AND_CRITIQUE.md.*
