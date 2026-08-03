# Marrying CSV Fusion with a Specialist Small Language Model

**Document type:** Architecture & product exploration (Grok Build session close-out)  
**Date:** 2026-08-02 (written up for GitHub / mobile / Grok Heavy follow-on)  
**Local project:** `C:\AI\GROK_Build\CSV_Fusion_Reports`  
**Status:** Design decisions locked; specialist model **not** implemented yet  

**Audience:** Human operator, Grok Heavy, Grok Build agents  

---

## 1. Executive summary

**CSV Fusion** is a working, local, privacy-safe engine that takes a folder of related CSV files **with no prior knowledge of the domain** and produces:

- health / schema profiling  
- relationship (join) candidates  
- deterministic report proposals  
- advisory notes and Excel outputs  

It works, but it is imperfect. Quality bottlenecks are mainly **selectivity** (too many low-value or execution-hostile proposals), weak feedback from evaluation into heuristics, and noisy health signals—not “can’t find any joins at all.”

**Proposal explored:** improve the soft decisions with a **specialist small language model** (narrow “report planner”), trained or prompted on **verifiable gold data** from CSV Fusion’s own eval harness and from **deterministic synthetic datasets**—**not** from AI-hallucinated CSVs.

**Non-proposal:** replace the deterministic pipeline with a general LLM at every stage, or fine-tune a “Himalaya-only” domain expert.

---

## 2. Terminology: “small large language model”

The phrase is informal and sounds oxymoronic. Preferred language:

| Prefer | Avoid as product language |
|--------|---------------------------|
| **Small specialist LM** | “Small large language model” |
| **Fine-tuned small model** / LoRA on a 7B-class base | Implying frontier Grok-class general intelligence |
| **Local specialist model** | “Just use a bigger general LLM everywhere” |

**Meaning in this document:** a **narrow** model specialised for *multi-CSV report planning*—not a general chat model and not the full CSV Fusion product by itself.

---

## 3. What CSV Fusion is today

### 3.1 Pipeline (deterministic authority)

```text
CSV folder
  → Health check          (profiles, types, warnings)
  → Relationship discovery (join candidates, confidence)
  → Deterministic planning (report proposals; semantic templates)
  → Advisory notes
  → Execution / Excel     (joins, aggregates, sheets)
```

Optional: **LLM hypothesis sidecar** (`csv_fusion/llm/`) that *suggests* report intents; Python **validates** feasibility. Hypotheses are **not** auto-merged as sole planner.

### 3.2 Design laws (already established)

1. **Generic** — no prior knowledge of subject or schema required.  
2. **Deterministic first** — reproducible, local, privacy-safe.  
3. **LLM is optional sidecar**, never sole planner.  
4. **Eval suite** — adversarial datasets + `eval_runner.py` for continuous measurement.  
5. **Long runs outside agent TUI** — full orchestrator/eval in a separate PC terminal.  
6. **Do not mirror** the full project to Drive (huge Excel/CSV artifacts).

### 3.3 Flagship & test assets

- Demos: Himalayan Expeditions, World Economic Indicators  
- Adversarial datasets under `datasets/` (e.g. column name collisions, grain mismatch, size imbalance, weak signals, multilingual HR, …)  
- Rich JSON artifacts per run: `health_report.json`, `relationship_candidates.json`, proposals, eval summaries  

---

## 4. Why a specialist model was considered

### 4.1 Problem

Multi-stage **general** LLMs sprinkled through the pipeline:

- are inconsistent across stages  
- are expensive and hard to regression-test  
- still need Python validation for join/column truth  

Observed residual weaknesses (see local `docs/DIAGNOSIS_WEAKNESSES_2026-06-01.md`):

- planner emits **too many low-value** or hostile proposals  
- weak **eval → heuristic** feedback loop  
- noisy health flags on clean IDs  
- execution brittle when proposals are imperfect  

### 4.2 Niche product angle

The same “niche expert small model” idea (explored elsewhere as income/productisation) fits a **concrete skill**:

> Given multi-table CSV **schemas + relationship candidates + health stats**, propose **high-value, executable** report intents and **reject junk**.

That is a **report-planner specialist**, not a subject-matter expert for one vertical.

### 4.3 What the specialist must *not* do

- Invent tables/columns that do not exist  
- Propose joins not supported by relationship graph  
- Replace health, relationship discovery, or Excel execution  
- Be trained only on one demo domain (e.g. only expeditions)  

---

## 5. Target architecture: marriage of Fusion + specialist

```text
                 ┌──────────────────────────────────────────┐
  CSV folder ──► │  DETERMINISTIC PIPELINE (authority)      │
                 │  health → relationships → plans → Excel  │
                 └──────────────────┬───────────────────────┘
                                    │ optional
                 ┌──────────────────▼───────────────────────┐
                 │  SPECIALIST SIDECAR (future)               │
                 │  Input:  health JSON + relationships JSON  │
                 │          (+ optional metadata)             │
                 │  Output: ranked report intents / rejections│
                 │  Then:   Python validates → optional merge │
                 └──────────────────▲───────────────────────┘
                                    │
              trained / prompted on gold from below
                                    │
         ┌──────────────────────────┴─────────────────────────┐
         │  PRIMARY GOLD                                        │
         │  • Seeded Python generators + damage library         │
         │  • Control metadata (expected outcomes)              │
         │  • eval_runner artifacts from those fixtures         │
         │  SUPPORT                                             │
         │  • GROK Research: techniques + damage-recipe ideas   │
         │    → codified into Python (not AI CSVs as truth)     │
         └──────────────────────────────────────────────────────┘
```

**Marriage principle:**  
The specialist improves **judgment** (what is worth reporting, ranking, naming, caveats).  
The deterministic stack remains **truth and execution**.

---

## 6. Data strategy for the specialist (critical)

### 6.1 Primary gold — deterministic generate + damage + metadata + eval

**Agreed as primary and trustworthy.**

| Component | Role |
|-----------|------|
| **Generator scripts** | Build clean multi-CSV worlds with known keys, grains, metrics (seeded) |
| **Damage scripts** | Apply *named* faults (collisions, orphan FKs, grain blow-up, time mismatch, imbalance, weak links, …) |
| **Control metadata** | Document expected relationships, **forbidden** joins, minimum report quality |
| **eval_runner** | Run CSV Fusion; compare to metadata → pass/fail and training labels |

**Principle:**  
*If we did not generate or damage it under a seed and document expected outcomes, it is not gold.*

This yields:

- instant re-verification  
- self-documenting fixtures  
- clean input/output pairs for a specialist model  

### 6.2 Behavioural gold — existing eval JSON

Runs under `datasets/eval_runs/` and dataset `metadata.json` files show how **this engine** behaves on hard cases. Primary source for:

```text
(input: health + relationships [+ schema])
→ (labels: good proposals / bad proposals / required edges / forbidden joins)
```

### 6.3 Explicitly rejected — AI-generated datasets as ground truth

AI-invented CSVs can **hallucinate** structure and “expected” outcomes.  
Acceptable for brainstorming **damage recipe ideas**; **major mistake** as control truth for scoring Fusion or training a specialist.

### 6.4 GROK Research — supporting roles only

| Research role | Use |
|---------------|-----|
| Surface **techniques** (inclusion dependencies, grain detection, interestingness, …) | Feed deterministic code and specialist vocabulary |
| Suggest **damage recipes** and obscure failure modes | Codify into Python damage library |
| Optionally locate real public multi-table datasets | Only with **human/deterministic control docs** |

Research **does not replace** seeded generators for gold.

---

## 7. Training / deployment path (recommended order)

| Step | Deliverable | Status |
|------|-------------|--------|
| **A** | Dataset factory: `generate` + `damage` registry + metadata schema; re-encode one existing adversarial set as proof | Agreed, **not built** |
| **B** | Export script: eval artifacts + metadata → training/eval pairs | Not built |
| **C** | Optional GROK Research pack: techniques + adversarial recipes (no AI CSVs as gold) | Not built |
| **D** | Specialist v0: RAG / tight prompts over gold + pack **inside** existing LLM harness; still validator-gated | Not built |
| **E** | Fine-tune small model (LoRA etc.) only if D plateaus and product needs offline/portable expert | Deferred |

**Do D before E.** Many gains may come from better deterministic selectivity + validated sidecar prompts without fine-tuning.

---

## 8. Example specialist I/O (conceptual)

**Input (from pipeline artifacts):**

- Table list, column types, cardinality hints  
- Relationship candidates with confidence  
- Health warnings (sanitised)  

**Output (JSON-like intents):**

- Ranked report titles / intents  
- Required tables & columns  
- Suggested group-bys / metrics  
- Explicit **rejections** of low-value or unsafe ideas  
- Optional caveats (“grain mismatch risk”)  

**Python then:**

- Drop any intent referencing missing tables/columns  
- Drop joins not in the relationship graph  
- Cap fan-out / post-join row estimates  
- Merge survivors with deterministic proposals under policy  

---

## 9. Product / income framing (context only)

The wider conversation noted niche small models as a productisation path. CSV Fusion is a **concrete** niche:

- real operator pain  
- generic multi-CSV → reports is differentiated  
- eval harness already provides measurement  

Monetisation is **optional and later**. Architecture must stay correct even if never sold.

---

## 10. Risks and non-goals

| Risk | Mitigation |
|------|------------|
| Specialist invents impossible reports | Validator gate (existing harness pattern) |
| Fine-tune replaces deterministic core | Forbidden by design |
| Gold polluted by hallucinations | No AI CSVs as ground truth |
| Domain-specific model kills genericity | Train on **planner** skill + synthetic multi-domain fixtures |
| Huge repo / Drive mirror | Publish **docs only** to GitHub; keep data/code local |

---

## 11. Local project pointers (not all in this repo)

On the operator machine:

| Path | Content |
|------|---------|
| `C:\AI\GROK_Build\CSV_Fusion_Reports` | Full engine, datasets, eval runs |
| `docs/SESSION_CLOSE_2026-08-02_specialist_llm_exploration.md` | Session close / resume |
| `docs/LLM_HARNESS_DESIGN.md` | Sidecar design |
| `docs/DIAGNOSIS_WEAKNESSES_2026-06-01.md` | Quality bottlenecks |
| `docs/WAYPOINT_2026-06-09_end_of_session.md` | Pipeline ops checkpoint |

**Commands (local):**

```powershell
cd C:\AI\GROK_Build\CSV_Fusion_Reports
python -m csv_fusion.orchestrator datasets/himalayan_expeditions --output-dir analysis_himalayan_expeditions
python datasets/eval_runner.py
```

Long runs: separate PC terminal, not agent TUI.

---

## 12. Resume checklist for Grok Heavy / Grok Build

When continuing research or implementation:

1. Reaffirm: **deterministic authority**, specialist = **sidecar only**.  
2. Design or implement **dataset factory** before fine-tune.  
3. Define metadata contract: expected edges, forbidden joins, proposal quality criteria.  
4. Map existing adversarial datasets into factory recipes where possible.  
5. Prototype **validated** specialist outputs inside `llm/harness` before any fine-tune.  
6. Never treat AI-generated CSVs as gold.  
7. Keep GitHub copy **documentation-focused** unless explicitly publishing code.

---

## 13. One-paragraph abstract (for search / mobile)

CSV Fusion is a generic multi-CSV report engine driven by a deterministic pipeline with an optional LLM sidecar. To improve proposal quality without sacrificing safety or genericity, a **small specialist language model** may be added as a **validated report-planner**, trained or prompted on gold from **seeded generators, controlled damage scripts, control metadata, and eval_runner results**—not on AI-hallucinated datasets. GROK Research supports techniques and damage-recipe design, which must be codified deterministically. Fine-tuning is optional and late; architecture marriage means judgment from the specialist, truth and execution from the pipeline.

---

## 14. Document history

| Date | Note |
|------|------|
| 2026-08-02 | Exploration closed in Grok Build; decisions locked; this public/research write-up prepared for GitHub + private notes handoff to Grok Heavy |

---

*End of document*
