# SOCES validation data

This directory contains the replication package for:

> *SOCES: Similarity-Ordered Consecutive-Exclude Stopping with Typed Probabilistic LLMs in Medical Systematic Reviews*

A reviewer can verify every figure in Table 1 and Table 2 of the manuscript directly from `results.csv` without re-running inference.

---

## Files

| File | Rows | Description | PRISMA-trAIce item |
|---|---|---|---|
| `results.csv` | 15,895 | Per-record Jev scores and decisions for all 70 reviews | M5 |
| `summary.csv` | 70 | Per-SR confusion matrix, recall, precision, F1, latency | — |
| `n_sweep.csv` | 7 | SOCES stopping parameter sweep (*n* ∈ {5,10,20,30,50,100,∞}) — manuscript Table 1 | — |
| `cutoff_sweep.csv` | 8 | Probability cutoff sweep (0.30–0.90) under SOCES *n* = 20 — manuscript Table 2 | — |
| `v7_prompt_review.txt` | — | Full V7 `noul_relevance` typed proposition rubric | M6 |
| `sr_objectives.csv` | 70 | SR PMIDs, published titles, and stripped research objectives | — |
| `PRISMA_trAIce_log.json` | — | Structured M5–M10 compliance record with benchmark-level counts | M5–M10 |

---

## Column schemas

### `results.csv`

| Column | Type | Description |
|---|---|---|
| `review_id` | int | PubMed ID of the systematic review |
| `pmid` | int | PubMed ID of the candidate document |
| `score` | float | Jev `noul_relevance` probability *p* ∈ [0.0, 1.0] |
| `decision` | bool | True if *p* ≥ 0.50 (primary cutoff) |
| `gold_label` | bool | Ground-truth inclusion from the Bentegeac et al. benchmark |

### `n_sweep.csv`

| Column | Description |
|---|---|
| `n` | Consecutive-exclude stopping window (or `inf` for unstopped) |
| `macro_recall` | Mean per-SR recall across 70 reviews (%) |
| `macro_recall_ci_low/high` | 95% BCa bootstrap CI bounds (%) |
| `micro_recall` | Pooled recall across all candidate pairs (%) |
| `micro_precision` | Pooled precision (%) |
| `micro_f1` | Pooled F1 |
| `model_calls` | Total Jev inference calls made |
| `calls_saved_pct` | Inference calls eliminated vs. unstopped ceiling (%) |
| `perfect_recall_reviews` | Reviews achieving 100% recall |
| `early_stopped_reviews` | Reviews where stopping triggered before end of ranked list |

### `cutoff_sweep.csv`

| Column | Description |
|---|---|
| `cutoff` | Proposition probability threshold *t* |
| `micro_recall` | Pooled recall at this cutoff under SOCES *n* = 20 (%) |
| `micro_precision` | Pooled precision (%) |
| `micro_f1` | Pooled F1 |
| `workload_reduction` | Fraction of total candidate pool not forwarded to human review (%) |
| `forwarded_to_human` | Records forwarded above cutoff *t* |

---

## Reproduce Table 1 and Table 2 from `results.csv`

All manuscript tables can be regenerated from `results.csv` alone with standard Python. No additional API calls are needed.

```python
import pandas as pd, numpy as np

df = pd.read_csv("results.csv")
df["gold"] = df["gold_label"].astype(bool)

# SOCES n=20 stopping simulation
def soces_n20(grp, n=20, stop_cutoff=0.50):
    rows = grp.sort_values("score", ascending=False).reset_index(drop=True)
    consec, stopped_at = 0, len(rows)
    for i, row in rows.iterrows():
        if row["score"] >= stop_cutoff:
            consec = 0
        else:
            consec += 1
            if consec >= n:
                stopped_at = i + 1
                break
    return rows.iloc[:stopped_at], rows.iloc[stopped_at:]

# Table 2 — workload reduction at each cutoff under n=20
for t in [0.30, 0.40, 0.50, 0.60, 0.70, 0.75, 0.80, 0.90]:
    tp = fp = fn = 0
    for _, grp in df.groupby("review_id"):
        ev, rem = soces_n20(grp)
        tp += int(((ev["score"] >= t) &  ev["gold"]).sum())
        fp += int(((ev["score"] >= t) & ~ev["gold"]).sum())
        fn += int(((ev["score"] <  t) &  ev["gold"]).sum()) + int(rem["gold"].sum())
    recall = tp / (tp + fn)
    prec   = tp / (tp + fp)
    wl_red = 1 - (tp + fp) / len(df)
    print(f"t={t:.2f}  recall={recall*100:.1f}%  prec={prec*100:.1f}%  workload_red={wl_red*100:.1f}%")
```

---

## What is not included and why

**PubMed candidate titles and abstracts are not included.** The NLM licence for PubMed data does not permit bulk redistribution of title and abstract text. To retrieve the full text of any candidate record, use the NCBI E-utilities API with the PMIDs listed in `results.csv`:

```
https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=pubmed&id=<PMID>&rettype=abstract&retmode=text
```

**Dense embeddings (`embeddings.parquet`) are not included.** Embedding vectors are a near-lossless projection of PubMed text and fall under the same licence constraint.

**The gold-standard corpus** (ground-truth inclusion labels) was obtained from Bentegeac et al. (2026) with permission. The `gold_label` column in `results.csv` reproduces those labels for verification purposes. To obtain the full dataset, contact the original authors (Raphaël Bentegeac and Aghilès Hamroun, Lille University Hospital / UMR1167 RID-AGE, Pasteur Institute of Lille).

---

## Calibration set PMIDs

The five reviews used to select the V7 proposition rubric (calibration set, held out before the 70-review evaluation) are:

| sr_pmid | SR title |
|---|---|
| 22226047 | Group B streptococcal disease in infants aged younger than 3 months |
| 22323502 | Acute cannabis consumption and motor vehicle collision risk |
| 22422870 | White rice consumption and risk of type 2 diabetes |
| 24157497 | Antihypertensive treatments in patients with diabetes |
| 30326495 | Physician burnout prevalence |

These five reviews are included in the 70-review benchmark results reported in the manuscript. The V7 calibration was conducted on a stratified sample from these reviews before the full 70-review evaluation was run.

---

## Attribution

Gold-standard inclusion labels: Bentegeac R et al. *BibliZap: An exploratory evaluation of an automated multi-level citation searching tool for systematic and rapid reviews.* Research Synthesis Methods. 2026;17:816–829. doi:10.1017/rsm.2026.10079.

Proposition evaluation: TypeSafe AI. *Jev 1.13: Models and System One documentation.* 2026. https://docs.typesafe.ai/models.

Embedding model: Chen J et al. *BGE M3-Embedding: Multi-Lingual, Multi-Functionality, Multi-Granularity Text Embeddings.* arXiv:2402.03216. 2024.
