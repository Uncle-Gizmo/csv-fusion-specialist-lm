# Dataset Factory: Comprehensive Test Fixtures Specification for CSV Fusion

**Document purpose:** Provide GROK Build (and the local dataset factory) with a complete, actionable catalogue of synthetic multi-CSV fixtures that must be generated. Each fixture has a clean baseline + named damage variants, control metadata describing *expected results*, and the specific capabilities / failure modes of CSV Fusion that the fixture is designed to exercise.  

**Date:** 2026-08-03  
**Status:** Specification for implementation. Aligns with the locked design law: *generate + named damage + control metadata = primary gold*.  

**How to use this document:**  
1. Implement a seeded generator for each base domain.  
2. Implement the listed named damage recipes (they should be composable).  
3. Emit for every generated run a `metadata.json` that contains the “Expected Results” sections below.  
4. `eval_runner` compares actual Fusion artefacts (`health_report.json`, `relationship_candidates.json`, proposals, advisory notes) against that metadata.  

---

## Guiding Principles for Every Fixture

- **Clean version** must produce high-confidence correct relationships, clean health, and a small set of high-value, executable report proposals.  
- **Damaged version(s)** must expose one (or a controlled combination of) failure modes. Expected results must change in a precise, documentable way (e.g. “this relationship confidence drops below 0.6”, “this proposal is now forbidden”, “health warning X appears”).  
- Metadata must include:  
  - `expected_relationships` (list of {left, right, type, min_confidence, required: bool})  
  - `forbidden_relationships`  
  - `expected_health_warnings` / `forbidden_health_warnings`  
  - `expected_high_value_reports` (titles or intent templates + required tables/columns)  
  - `expected_rejected_or_low_value`  
  - `post_join_cardinality_bounds` or `aggregate_sanity_checks` where relevant  
  - `known_traps` (fan, chasm, grain)  

---

## 1. Core Domain Bases (Clean Versions)

Implement these clean generators first. Each becomes the seed for multiple damage variants.

### 1.1 Classic Retail / Orders (Star Schema)

**Tables:** `customers`, `orders`, `order_items`, `products`, `categories`  
**Grain:** Order-item is the atomic fact.  
**Clean expected relationships:**  
- `order_items.order_id` → `orders.order_id` (required, high confidence)  
- `orders.customer_id` → `customers.customer_id`  
- `order_items.product_id` → `products.product_id`  
- `products.category_id` → `categories.category_id`  
**Clean expected high-value reports:**  
- Revenue by category / month  
- Top customers by order value  
- Product popularity (quantity sold)  
**Clean health:** No critical warnings; keys unique; low nulls on join columns.  

### 1.2 HR / Employees

**Tables:** `departments`, `employees`, `salaries` (or `pay_history`), `titles`  
**Special:** Multi-language name columns optional.  
**Clean expected:** Clear employee → department, employee → salary history (temporal).  

### 1.3 Himalayan Expeditions (existing flagship – formalise)

**Tables:** expeditions, members, peaks, (optionally weather / routes)  
**Clean expected:** Well-documented relationships already known to the project.  

### 1.4 Medical / Clinical

**Tables:** `patients`, `encounters` / `visits`, `lab_results`, `diagnoses`, `medications`  
**Characteristics:** High sparsity on some labs, temporal alignment critical, multi-grain (patient vs encounter vs lab).  

### 1.5 IoT / Sensor Readings

**Tables:** `devices`, `locations`, `readings` (timestamp, value, device_id)  
**Grain:** Reading is atomic; heavy temporal.  

### 1.6 Education

**Tables:** `students`, `courses`, `enrollments`, `grades`, `instructors`  
**Composite keys common (student+course).**  

### 1.7 Inventory / Supply Chain

**Tables:** `warehouses`, `products`, `stock_levels`, `shipments`, `shipment_items`  
**Fan-out and temporal (ship dates) important.**  

### 1.8 Financial Transactions

**Tables:** `accounts`, `transactions`, `categories`, `customers`  
**Non-additive metrics (balances) and temporal windows.**  

### 1.9 Sports / Events

**Tables:** `teams`, `players`, `matches`, `match_events` / `results`  
**Composite and multi-hop relationships.**  

### 1.10 Simple Two-Table Baseline

**Tables:** `parent`, `child` with clean 1:N.  
Used as minimal unit for many damage recipes.

---

## 2. Named Damage Recipes & Fixture Families

For each recipe, generate both the clean base and the damaged variant(s). Metadata must record the exact seed and the damage parameters so results are reproducible.

### 2.1 Referential Integrity Failures

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `RI-orphan-basic` | Randomly delete 5–15 % of parent rows that have children | All FKs resolve; high-confidence relationships | Orphan warnings in health; relationship confidence drops or becomes partial; proposals that rely on the missing parents are rejected or caveated | Orphan detection, partial IND handling |
| `RI-orphan-severe` | 40 %+ orphans | Same | Strong health warnings; many relationships fall below usable confidence; almost no high-value multi-table proposals survive validation | Severity threshold behaviour |
| `RI-missing-parent-key` | Parent PK column renamed or removed | — | Relationship discovery fails for that edge; health flags missing key | Key presence |
| `RI-duplicate-pk` | Introduce duplicate primary-key values | Unique keys | Health uniqueness warnings; relationship discovery may still find inclusion but confidence/quality scores should degrade | Uniqueness enforcement |

### 2.2 Grain, Cardinality & Trap Failures

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `GRAIN-fanout` | Turn a clean 1:N into accidental N:M (e.g. remove bridge or add extra matching values) | Correct join cardinality bounds | Health or planner detects fan-out risk; proposals that would inflate aggregates are rejected or carry strong caveats | Fan-out / grain mismatch detection |
| `GRAIN-chasm` | Two fact tables sharing a dimension; measures on both facts | Individual facts report correctly | Any proposal that joins both facts without proper aggregation is forbidden or heavily caveated | Chasm trap detection |
| `GRAIN-fan-trap` | Classic 1-N-N (customer → orders → order_items) with measures on both many sides | Correct when aggregated properly | Proposals that sum the intermediate table incorrectly are rejected | Fan trap |
| `GRAIN-size-imbalance` | One table 50–500× larger than others | Relationships still found | Health notes size imbalance; planner prefers reports that do not explode memory / row counts | Size-aware proposal ranking |
| `GRAIN-mismatch-explicit` | Two tables that look joinable on a column but have fundamentally different grains (e.g. daily vs monthly) | — | Relationship may be proposed at low confidence; high-value reports that ignore grain are rejected | Grain awareness |

### 2.3 Naming, Semantic & Schema Failures

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `NAME-collision` | Two tables both have a column named `id` or `code` with different meanings | Distinct relationships | Health flags name collision; relationship discovery must use value statistics / other signals, not name alone | Name-collision robustness |
| `NAME-semantic-ambiguity` | Columns with similar names but low value overlap (or synonym pairs) | Correct high-overlap edges | Low-confidence or rejected relationships for the ambiguous pairs | Semantic vs syntactic matching |
| `NAME-multilingual` | Headers and/or values in mixed languages / scripts | Still discovers value-based relationships | Health notes encoding / language diversity; relationships still found if values overlap | Multilingual robustness |
| `SCHEMA-denorm-pivot` | A table that is a pivot / denormalised wide format | — | Health recognises atypical structure; relationship discovery limited or warns | Denormalised data handling |
| `SCHEMA-multi-meaning` | Single column used for different entity types (discriminated by another column) | — | Health or relationship stage flags multi-meaning risk | Multi-meaning column detection |
| `SCHEMA-mixed-record` | One CSV file contains two different record types (e.g. header rows mixed with data, or two entity types) | — | Health detects mixed structure; relationships may be incomplete | Mixed-record files |

### 2.4 Temporal Failures

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `TEMP-misalign` | Child dates systematically before parent existence or non-overlapping windows | Temporal reports work | Health or advisory notes temporal misalignment; time-window reports rejected or caveated | Temporal integrity |
| `TEMP-predated` | Specific children with dates before their parent’s start | — | Targeted warnings; proposals that assume temporal order are affected | Predated events |
| `TEMP-late-arriving` | Facts that arrive after the dimension snapshot they should belong to | — | Similar to above | Late-arriving facts |
| `TEMP-scd-mix` | Slowly-changing dimension mixed with snapshot facts without proper keys | — | Relationship quality drops; proposals need caveats | SCD handling |

### 2.5 Signal Strength & Sparsity

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `SIG-weak-ind` | Reduce inclusion ratio to 60–80 % (partial IND) | Full inclusion | Relationship still proposed but with lower confidence; may be demoted in ranking | Partial IND handling |
| `SIG-null-key` | 10–40 % nulls on candidate join columns | Low nulls | Health null warnings; relationship confidence reduced | Null handling on keys |
| `SIG-high-sparsity` | Many columns >80 % null | — | Health sparsity flags; proposals that rely on sparse columns are low-value / rejected | Sparsity awareness |
| `SIG-cardinality-skew` | Extreme Zipf / power-law distribution on join keys | Moderate | Health notes skew; planner avoids proposals that would be dominated by a few keys | Skew awareness |

### 2.6 Key Structure Failures

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `KEY-composite-only` | No single-column PK/FK; only composite keys work | Composite relationships discovered | If engine only looks for unary INDs it will fail; expected that composite support is tested | Composite key / n-ary IND support |
| `KEY-cyclic` | Introduce a cycle in the relationship graph | Acyclic | Health or planner detects cycle; proposals that traverse the cycle are limited | Cycle detection |
| `KEY-overlapping` | Overlapping composite keys or multiple candidate keys | Clear primary | Ranking of candidates is exercised | Key ranking |

### 2.7 Type, Format & Data-Quality Noise

| Fixture ID | Damage | Expected clean | Expected damaged | What it tests |
|------------|--------|----------------|------------------|---------------|
| `TYPE-drift` | Same logical column stored as string in one table, typed date/number in another | Consistent types | Health type-inconsistency warnings; join may still succeed via cast but confidence affected | Type consistency |
| `TYPE-format` | Dates in multiple formats, numeric with different decimal separators, etc. | Clean formats | Health format warnings | Format detection |
| `DQ-duplicates` | Exact or near-duplicate rows | Unique | Health duplicate warnings | Deduplication signals |
| `DQ-outliers` | Extreme values on measure columns | Normal ranges | Health or advisory outlier notes (optional for reports) | Outlier awareness |

### 2.8 Scale & Edge Cases

| Fixture ID | Description | Expected behaviour |
|------------|-------------|--------------------|
| `EDGE-empty-table` | One table has 0 rows | Health notes empty; relationships involving it are absent or low-confidence |
| `EDGE-single-row` | Extremely small tables | Still produces correct (trivial) relationships and reports |
| `EDGE-many-tables` | 15–30 tables with sparse relationships | Relationship discovery remains selective; does not explode proposals |
| `EDGE-wide-table` | One table with 100+ columns | Health and relationship stages remain efficient; proposals stay focused |
| `EDGE-no-relationships` | Completely unrelated tables | Zero high-confidence relationships; no multi-table report proposals |

### 2.9 Report-Proposal Selectivity Specific Fixtures

These focus on the planner / interestingness stage rather than pure discovery.

| Fixture ID | Setup | Expected clean | Expected damaged / hard | What it tests |
|------------|-------|----------------|-------------------------|---------------|
| `PROP-low-value-bait` | Many possible joins, most of which produce uninteresting or high-cardinality results | Only 3–5 high-value proposals survive | If interestingness is weak, many junk proposals appear | Selectivity / interestingness ranking |
| `PROP-hostile-join` | A join that is technically valid but produces >10× row explosion or non-additive nonsense | That join is never proposed or is explicitly rejected | — | Execution-hostile rejection |
| `PROP-sparse-signal` | Only one or two weak but real relationships | Those are still proposed (with caveats) | — | Ability to surface weak but useful signals |
| `PROP-multi-hop` | Useful report requires 3-table join path | Multi-hop proposals appear when valuable | — | Multi-hop proposal generation |

---

## 3. Recommended Generation Order (for the Dataset Factory)

**Phase A – Minimal viable factory**  
1. Two-table parent/child clean + `RI-orphan-basic`, `GRAIN-fanout`, `NAME-collision`, `SIG-weak-ind`, `TEMP-misalign`  
2. Classic Retail clean + the same damages  
3. Metadata schema + `eval_runner` comparison for the above  

**Phase B – Coverage expansion**  
4. Medical + HR + Inventory bases  
5. All grain/trap recipes (`chasm`, `fan-trap`, `size-imbalance`)  
6. Composite-key and cyclic fixtures  
7. Multilingual + type-drift + sparsity  

**Phase C – Planner stress**  
8. `PROP-*` selectivity fixtures  
9. Multi-damage combinations (e.g. orphan + grain mismatch + name collision)  
10. Scale edges  

**Phase D – Domain richness**  
11. Formalise Himalayan, add Sports, Financial, Education, IoT  
12. Re-encode existing adversarial datasets as factory recipes for regression continuity  

---

## 4. Control Metadata Template (minimal)

Every generated fixture directory should contain a `metadata.json` roughly of the form:

```json
{
  "fixture_id": "RI-orphan-basic-retail",
  "seed": 42,
  "base_domain": "retail",
  "damages_applied": ["orphan_fk:0.12"],
  "expected_relationships": [
    {"left": "order_items.order_id", "right": "orders.order_id", "min_confidence": 0.95, "required": true},
    ...
  ],
  "forbidden_relationships": [],
  "expected_health_warnings": ["orphan_fk_detected", "partial_inclusion:order_items->orders"],
  "expected_high_value_reports": [
    {"intent": "revenue_by_category", "required_tables": ["order_items", "products", "categories"]},
    ...
  ],
  "expected_rejected_or_low_value": [
    {"intent_pattern": ".*customer.*detail.*without.*order", "reason": "orphans make incomplete"}
  ],
  "post_join_cardinality_bounds": {
    "order_items \u22c8 orders": {"max_rows": 1.05 * order_items_rows}
  },
  "known_traps": []
}
```

`eval_runner` should produce a clear pass/fail + diff against this metadata.

---

## 5. Coverage Checklist (for completeness)

A mature factory should eventually cover:

- [ ] All RI variants (orphan severity levels, missing keys, duplicate PKs)  
- [ ] All grain / trap variants (fanout, chasm, fan-trap, size imbalance, explicit grain mismatch)  
- [ ] Naming / semantic / multilingual / multi-meaning / mixed-record  
- [ ] Temporal (misalign, predated, late-arriving, SCD mix)  
- [ ] Signal strength (weak IND, null keys, sparsity, skew)  
- [ ] Key structure (composite-only, cyclic, overlapping)  
- [ ] Type / format / DQ noise  
- [ ] Scale & edge cases  
- [ ] Report-proposal selectivity (low-value bait, hostile joins, multi-hop, sparse signal)  
- [ ] At least 6 distinct domain bases  
- [ ] Controlled multi-damage combinations  
- [ ] Regression of all previously hand-crafted adversarial sets  

---

## 6. Advice for Implementation & GROK Build

- Keep generators pure Python, seeded, deterministic, and free of external LLM calls.  
- Damage recipes should be independent functions that take a clean multi-CSV world + parameters and return the damaged world + a damage report (for the metadata).  
- Start small (Phase A) and get the `eval_runner` loop closed before expanding.  
- Every new damage recipe must come with at least one concrete expected-results example in metadata.  
- The specialist sidecar (when built) will be trained/evaluated on exactly these labelled pairs; the richer and more precise the expected results, the better the specialist can learn “what is worth reporting vs what is junk”.

This catalogue is intentionally extensive so that CSV Fusion can be stress-tested across the full spectrum of real-world multi-CSV pathologies while remaining fully reproducible and under deterministic control.

---

*End of specification. Cross-reference: main architecture document, RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md, RESEARCH_ADVICE_AND_CRITIQUE.md.*
