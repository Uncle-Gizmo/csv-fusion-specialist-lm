# Research Advice, Critique & Guidance for GROK Research / GROK Build / Human Operator

**Document purpose:** Distil the team research into actionable guidance, critique of the current design trajectory, and high-value prompt patterns for the GROK Research tool.  
**Date:** 2026-08-03  
**Audience:** Human (Tony / Uncle-Gizmo), GROK Build agents, GROK Research tool, future Grok Heavy sessions.

---

## 1. Core Design Laws Remain Sound — Reinforce Them

The original architecture document correctly locks:

- Deterministic pipeline = sole authority for truth and execution.
- Specialist small LM = optional validated sidecar for judgment (ranking, naming, rejection of junk, caveats).
- Gold data = seeded generate + named damage + control metadata + eval_runner artefacts **only**.
- Never treat AI-hallucinated CSVs (or undocumented public dumps) as ground truth.

Our research strongly supports this stance. Public multi-table datasets and academic techniques are excellent *support* material, but they almost never supply the full control metadata (expected high-value report intents, forbidden joins, quality thresholds, post-join cardinality bounds) required for safe specialist training or for rigorous regression of the planner.

**Critique of any temptation to relax the gold rule:** Doing so would re-introduce exactly the inconsistency, non-reproducibility and hallucination risk that the deterministic-first design was created to avoid. Keep the factory pure.

---

## 2. Highest-Value Work for GROK Research

Order of priority for future research sessions:

1. **Damage-recipe expansion**  
   Extract concrete, named failure modes from:
   - TPC-DI quality-problem papers
   - Classic data-cleaning benchmarks (HOSP, beers, flights, etc.)
   - RelBench / open-data-lake studies (sparsity, temporal, denormalization)
   - BI literature on fan traps / chasm traps  
   Then ask the research tool to propose precise Python implementations or pseudo-code that can be dropped into the local damage library.

2. **Obscure or under-documented deterministic techniques**  
   - Exact ranking features and pruning rules used by HoPF
   - Practical parameter regimes for Faida / Binder on folders of CSVs (not just RDBMS)
   - Recent open-source interestingness libraries or multi-relational SI implementations
   - Grain / trap detection heuristics used in commercial or research BI tools
   - Any post-2019 advances in scalable IND discovery that are not yet in Desbordante

3. **Public multi-table sets that *do* have usable control documentation**  
   Prefer sets that publish schema diagrams, expected FKs, or known quality issues in machine-readable or clearly human-readable form. Use them as seeds, not as gold.

4. **Evaluation harness patterns**  
   CleanML, REIN, MaDI-Bench, Spider-join-data style ground-truth formats that can inspire the control-metadata schema.

**Prompt style that works well for GROK Research:**  
“Locate deterministic algorithms or open-source code for X. Prefer papers or repos that give ranking features, parameter settings, or pseudo-code that can be re-implemented in pure Python. Summarise the failure modes they were designed to handle so we can turn them into named damage recipes. Do not recommend AI-generated synthetic data as ground truth.”

---

## 3. Immediate Local Priorities (Dataset Factory First)

From the main document’s recommended order (Step A before any specialist fine-tune):

1. Define the control-metadata JSON schema (expected edges, forbidden joins, min quality, cardinality bounds, known traps).
2. Implement 5–8 high-value damage recipes first:
   - `orphan_fk` / `missing_parent`
   - `grain_mismatch` / `fanout_explosion` / `chasm_trap`
   - `name_collision` / `semantic_ambiguity`
   - `temporal_misalign`
   - `size_imbalance`
   - `weak_ind`
3. Re-encode one existing adversarial fixture as a factory proof.
4. Only then build the export path that turns eval artefacts + metadata into specialist training pairs.
5. Prototype the specialist as a tightly-prompted / RAG sidecar *inside* the existing LLM harness, still fully validator-gated. Fine-tune only if that plateaus and offline/portable need appears.

---

## 4. Critiques & Risks Observed in Research

| Observation | Implication for project |
|-------------|-------------------------|
| Most public “dirty” multi-table sets lack full expected-report labels | Synthetic factory is not optional; it is the only reliable gold path |
| Many modern insight-generation papers are heavily LLM-based | Correct to keep the specialist as a narrow, validated sidecar rather than replacing the pipeline |
| IND discovery literature is mature but ranking/precision features are under-used in practice | Big opportunity for CSV Fusion to improve selectivity by adopting HoPF-style ranking + interestingness filters |
| Fan/chasm traps are well-known in BI but rarely automated in open tools | Explicit detection of these traps should be a first-class health or planner feature |
| Scale of real open data lakes (LakeBench etc.) far exceeds typical local CSV folders | Focus first on quality of proposals on realistic but manageable sizes; scalability is secondary for the specialist |

**Risk of over-research:** The temptation to keep searching for the “perfect” public dataset. Resist. The factory gives perfect control; public material is for inspiration and external validation only.

---

## 5. Advice for the Human Operator & GROK Build Sessions

- Keep long-running factory generation and full eval outside the agent TUI (as already designed).
- When briefing GROK Research or a new Grok Heavy session, always restate the gold-data law and the “deterministic authority” principle in the first paragraph.
- Prefer concrete, implementable outputs (pseudo-code for a damage recipe, list of ranking features, parameter table) over high-level surveys.
- After each research session, immediately codify any new damage recipe or interestingness formula into the local Python library so the knowledge does not remain trapped in chat logs.
- The specialist’s value will be highest if the deterministic baseline already has strong selectivity; do not expect the LM to compensate for a weak planner.

---

## 6. Suggested Structure for Future Research Handoffs

When GROK Research or a later session produces new material, add it under:

- `research/damage_recipes/` — named recipes with seed + metadata examples  
- `research/techniques/` — short notes + links + Python sketches  
- `research/datasets/` — curated seeds with notes on what control metadata they *do* and *do not* provide  

This keeps the GitHub repo documentation-focused and avoids accidental inclusion of large artefacts.

---

## 7. Closing Assessment

The original design decisions are robust. The research we conducted supplies exactly the kind of deterministic, under-represented techniques and realistic failure modes that the brief requested. The path of maximum leverage is:

1. Expand the local damage library with recipes inspired by the sources above.  
2. Strengthen the deterministic planner with interestingness scoring and trap detection.  
3. Only then train or prompt the specialist on the resulting clean gold pairs.

This keeps the system privacy-safe, reproducible, and generically applicable — while still allowing a specialist model to improve the soft judgment that pure heuristics struggle with.

---

*End of document. Cross-references: RESEARCH_HANDOFF_DATASETS_AND_HARD_CASES.md, RESEARCH_HANDOFF_DETERMINISTIC_TECHNIQUES.md, main CSV_FUSION_SPECIALIST_LM.md.*
