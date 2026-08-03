# Research Handoff: Public Multi-Table Datasets, Hard Cases & Synthetic Factories

**Document purpose:** Provide concrete starting points, links, and failure-mode catalogues for GROK Research (and human/GB follow-on) supporting the CSV Fusion + specialist report-planner design.  
**Date:** 2026-08-03  
**Status:** Research synthesis from team (Grok + Harper + Benjamin + Lucas). Complements the main architecture document.  
**Key principle (reaffirmed):** Primary gold remains *seeded generators + named damage + control metadata + eval_runner*. Public datasets and research findings are **support** only — seeds, inspiration for damage recipes, validation of relationship discovery, never ground-truth for scoring or training labels.

---

## 1. Priority Public Multi-Table / Multi-CSV Datasets with Documented Relationships or Issues

These are useful as *seeds* for the dataset factory, as realistic targets for relationship-discovery validation, and as sources of documented failure modes that can be turned into named damage recipes.

### 1.1 Classic / Schema-Rich Seeds (clean baselines + known FKs)

| Dataset | Tables / Scale | Why useful | Links / Notes |
|---------|----------------|------------|---------------|
| **AdventureWorks** (Microsoft) | ~60+ tables (OLTP), multiple schemas | Rich PK/FK graph, realistic grain (Sales, Production, HR, Purchasing). CSV exports exist. | GitHub: olafusimichael/AdventureWorksCSV ; original MS samples; schema diagrams widely available. Excellent for “expected relationships” baseline. |
| **Northwind** | ~13 classic tables | Small, well-understood FKs (Customers–Orders–Order Details, etc.). Easy CSV conversion. | Multiple GitHub ports (harryho/db-samples etc.). Good for unit-test scale. |
| **TPC-H / TPC-DS** | 8 (H) / 24+ (DS) tables | Industry-standard multi-table with known relationships, scalable generators. Can export to CSV. Controlled scale factors. | Official TPC tools + open generators (e.g. pg_tpch). Ideal for controlled experiments on join selectivity and fan-out. |

### 1.2 Real Multi-Table with Challenges (RelBench & related)

- **RelBench** (Stanford SNAP) / **4DBInfer**: 7–11 real relational databases (Amazon product reviews 3 tables, H&M 3, Stack Overflow 7, Clinical trials ~15, F1 racing 9, Avito 8, etc.). Full PK/FK schemas, temporal attributes, sparsity, scale.  
  Links: https://relbench.stanford.edu , GitHub snap-stanford/relbench.  
  Value: relationship discovery under real sparsity/temporal misalignment; hard cases for report proposals (many weak signals).

### 1.3 Integration / Schema-Matching Benchmarks with Ground Truth

- **MaDI-Bench** (Uni Mannheim): Heterogeneous multi-source integration tasks (schema matching + entity matching + fusion). Domains include games, products, papers. Human-validated ground truth.  
  GitHub: wbsg-uni-mannheim/MaDI-Bench.  
  Value: tests beyond pure value-inclusion (semantic name mismatches).

- **LakeBench**: Large-scale (16M+ tables from open data lakes). Ground-truth joinable / unionable queries. Excellent scalability stress test for relationship candidates.

- **SchemaPile**: ~221k schemas extracted from GitHub SQL (1.7M tables, ~700k FKs). Metadata-rich for schema-matching research.

### 1.4 Dirty / Quality-Problem Datasets (classic data-cleaning literature)

Commonly used in error-detection papers (often single-table or lightly multi-table, but failure modes transfer):

- HOSP, beers, flights, restaurant, food, rayyan, bridges, cars (from Baran, HoloClean, Renuver, etc. literature).  
  Error types: missing values (MV), typos (T), formatting (F), violated dependencies (VD), category swaps, extraneous values.  
  Value: concrete named faults that can be re-implemented as damage recipes (e.g. “orphan FK after deletion”, “grain blow-up via aggregation mismatch”).

- **TPC-DI**: Explicit documentation of data-quality problems in multi-table flat-file extracts (multi-meaning attributes, inconsistent schemas across files, mixed record types in single files, missing values, format incompatibilities, FK load-order issues). Highly relevant paper(s) from DCU / TPC community.

- Quartz Guide to Bad Data / ProPublica datasets / various open-data portal studies (Canada OGD, NYC, LA) that document denormalization, missing keys, non-key joins, and functional-dependency violations.

### 1.5 Other Useful Repositories

- vincentlaucsb/csv-data (real + fake CSVs for testing suites).  
- CTU Prague Relational Learning Repository.  
- Spider / BIRD (text-to-SQL; multi-table, some CSV + join-path ground truth available via Spider-join-data style resources).  
- CleanML / REIN-style clean+dirty pairs.

**Critique on public sets:** Almost none supply the *full control metadata* required for specialist training (expected high-value report intents, forbidden joins, quality thresholds, post-join cardinality estimates). They are excellent for relationship-discovery validation and for mining realistic damage recipes, but the project’s own generate+damage+metadata remains the only trustworthy primary gold.

---

## 2. Characteristics of Hard / Uncommon Datasets that Resist Useful Automated Reporting

These should become first-class damage categories and test fixtures. They explain many of the residual weaknesses already observed in CSV Fusion (low-value proposals, brittle execution, noisy health).

| Category | Concrete failure modes | Why reports break | Suggested damage recipe names |
|----------|------------------------|-------------------|-------------------------------|
| Grain / cardinality | Fan-out explosion, size imbalance (1:N with high N), many-to-many without bridge, chasm traps / fan traps | Aggregation inflation or vanishing rows; post-join estimates wrong | `grain_mismatch`, `fanout_explosion`, `chasm_trap`, `size_imbalance` |
| Referential integrity | Orphan FKs, missing parents, cyclic dependencies, composite/overlapping keys | Invalid joins proposed or silent data loss | `orphan_fk`, `missing_parent`, `composite_key_partial` |
| Temporal | Date misalignment, predated children, non-overlapping windows, slowly-changing dimensions mixed with snapshots | Time-based reports produce empty or nonsensical results | `temporal_misalign`, `predated_txn`, `scd_snapshot_mix` |
| Schema / naming | Column name collisions across tables, abbreviations, multilingual labels, semantic synonymy without value overlap | False-positive or false-negative relationship candidates | `name_collision`, `semantic_ambiguity`, `multilingual_headers` |
| Signal strength | Weak inclusion (partial IND only), high nulls on candidate keys, extreme skew | Confidence scores noisy; many low-value proposals | `weak_ind`, `high_null_key`, `cardinality_skew` |
| Structural | Denormalized pivots, multi-meaning attributes, mixed record types in one file, incomplete metadata | Health checks noisy; planner invents impossible columns | `denormalized_pivot`, `multi_meaning_attr`, `mixed_record_file` |
| Scale / sparsity | Very large #tables/columns, extreme sparsity, rare patterns | Timeouts or combinatorial explosion of proposals | `high_table_count`, `extreme_sparsity` |

**BI-specific traps (important for report proposals):** Fan traps and chasm traps produce mathematically correct but business-wrong aggregates. The specialist (and deterministic planner) must learn to detect and reject or caveat them.

---

## 3. Synthetic Dataset Factories with Controlled Fault Injection (Primary Gold Path)

These align directly with the locked design law: “If we did not generate or damage it under a seed and document expected outcomes, it is not gold.”

### Recommended generators / libraries (deterministic preference)

| Tool / Approach | Strengths | Notes for CSV Fusion |
|-----------------|-----------|----------------------|
| **Project’s own generate + damage registry** (to be built) | Full control of seed, expected edges, forbidden joins, report quality criteria | Highest priority (see main doc §7 Step A) |
| **SYDAG** (Synthetic Dataset Generator for Data Integration) | From real seed → horizontal/vertical splits, schema/data errors, normalization changes, obfuscation, noise on overlapping attributes. Highly configurable. | Paper: CEUR-WS Vol-4118. Excellent model for multi-CSV integration scenarios. |
| **MessyData** (sodadata/messydata) | YAML schema → realistic messy DataFrame. Anomalies: missing, duplicates, invalid categories, bad dates, outliers. | Good single-table damage; can be extended to multi-table with FK control. |
| **Custom Python (Faker + numpy/scipy + networkx)** | Full control of parent–child cardinality distributions (log-normal, power-law), temporal realism, rare patterns (orphans, zero-children, predated events), referential integrity flags. | Recommended core of the local dataset factory. |
| **TPC generators + post-processing** | Industry schemas + scale factors; then apply named damage scripts. | Clean baseline + controlled degradation. |
| Others of interest | IRG-1 / Betterdata (complex relational with composite/overlapping FKs), Gretel Relational, YData multi-table (with YAML relations). | Use only if seeded and metadata-documented. |

**Explicitly lower priority / caution:** LLM-based error generators (e.g. TableEG). Acceptable for brainstorming damage-recipe *ideas*, but never as control truth (matches main document §6.3).

### Suggested damage-library categories (to implement first)

1. `orphan_fk` / `missing_parent`  
2. `grain_mismatch` / `fanout_explosion` / `chasm_trap`  
3. `name_collision` / `semantic_ambiguity`  
4. `temporal_misalign` / `predated_child`  
5. `weak_ind` / `partial_inclusion`  
6. `high_null_on_key` / `cardinality_skew`  
7. `denormalized_pivot` / `multi_meaning_column`  
8. `size_imbalance` (one table 100–1000× larger)  
9. `column_type_drift` / `format_inconsistency`  
10. `forbidden_join_candidate` (value overlap but semantically wrong)

Each recipe must emit control metadata: expected relationships that *should* still be found, relationships that *must not* be proposed, minimum acceptable proposal quality, and post-join cardinality bounds.

---

## 4. Evaluation Patterns Worth Adopting

- Control metadata JSON per fixture (expected edges, forbidden, quality criteria).  
- `eval_runner` already exists locally — extend it to score specialist outputs against the same metadata.  
- Benchmark relationship discovery against Desbordante / Metanome implementations on the same fixtures.  
- CleanML / REIN style clean+dirty pairs for health-check regression.  
- Spider-join-data style CSV + ground-truth join paths for relationship evaluation.

---

## 5. Immediate Recommendations for GROK Research & Local Implementation

1. **Highest value searches for GROK Research**  
   - “TPC-DI data quality problems” + related papers  
   - “SYDAG synthetic dataset generator” + similar controlled-fault tools  
   - “fan trap chasm trap BI reporting”  
   - Specific failure modes from RelBench / open-data lake studies  
   - “controlled error injection multi-table relational synthetic data”

2. **Local next steps (dataset factory)**  
   - Implement 5–8 named damage recipes first (start with orphan_fk, grain_mismatch, name_collision, temporal_misalign, size_imbalance).  
   - Re-encode one existing adversarial set (e.g. Himalayan or a synthetic HR set) as a proof of the factory.  
   - Define the metadata schema early (expected / forbidden / quality).

3. **Critique**  
   Public datasets are valuable but incomplete for the specialist’s training objective. Relying on them alone would re-introduce the very problem the design laws forbid (uncontrolled or hallucinated ground truth). The generate+damage path is both safer and more powerful.

---

*End of document. See also RESEARCH_HANDOFF_DETERMINISTIC_TECHNIQUES.md and RESEARCH_ADVICE_AND_CRITIQUE.md.*
