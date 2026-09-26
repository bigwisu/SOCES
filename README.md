# SOCES Systematic Review Screener — Notebook

A self-contained Jupyter notebook implementing **Similarity-Ordered Consecutive-Exclude Screening (SOCES)** for AI-assisted title and abstract screening in medical systematic reviews.

Based on:  
> *SOCES: Similarity-Ordered Consecutive-Exclude Stopping with Typed Probabilistic LLMs in Medical Systematic Reviews*

---

## What you need

| Requirement | Notes |
|---|---|
| Python ≥ 3.10 | Tested on 3.11 |
| OpenRouter API key | https://openrouter.ai — free tier sufficient for small reviews |
| A `.bib` file | Every entry must have `title` + `abstract` fields; max **2 000 records** |

---

## Quick start

```bash
pip install jupyter pybtex pandas pyarrow numpy requests tqdm ipywidgets
jupyter notebook soces_screening.ipynb
```

Or open directly in **VS Code**, **JupyterLab**, or **Google Colab** (upload the `.ipynb`).

---

## Notebook cells

| Cell | What it does |
|---|---|
| **1 — Install** | Installs all Python dependencies in-kernel |
| **2 — Configure** | Paste your OpenRouter key and review objective; tune SOCES parameters |
| **3 — Upload `.bib`** | File-picker widget; parses titles + abstracts; enforces the 2 000-record cap |
| **4 — Embed** | Calls `BAAI/bge-m3` via OpenRouter; saves all vectors to **`embeddings.parquet`** |
| **5 — Screen** | Ranks by cosine similarity → scores with `typesafe/jev-1.13` → SOCES stopping |
| **6 — PRISMA display** | Renders a PRISMA-trAIce flow block + included-records table |
| **7 — Export** | Writes `results.csv`, `results_included.csv`, `summary.json`, `v7_prompt_review.txt` |
| **Appendix** | Replays the stopping logic at n ∈ {5,10,20,30,50,100,∞} — no extra API calls |

---

## Output files

| File | Contents | PRISMA-trAIce item |
|---|---|---|
| `embeddings.parquet` | Embedded corpus (title, abstract, vector, metadata) | — |
| `results.csv` | One row per candidate: `rank`, `key`, `pmid`, `title`, `similarity`, `score`, `decision`, `source` | M5 |
| `results_included.csv` | Subset of `results.csv` where `decision == True` | M5 |
| `summary.json` | PRISMA flow node counts, model names, thresholds | M7, M8 |
| `v7_prompt_review.txt` | Full V7 `noul_relevance` proposition rubric | M6 |
| `sweep.csv` | Stopping-parameter sweep (n = 5 … ∞) replayed from saved scores | — |

---

## `.bib` file format

Export from **Zotero**, **EndNote**, **PubMed**, or any reference manager that produces BibTeX.  
Entries **without both `title` and `abstract`** are silently skipped and counted in the parse summary.

Minimum valid entry:

```bibtex
@article{Smith2023,
  title    = {Effect of CBT on depression in type 2 diabetes},
  abstract = {Background: ... Methods: ... Conclusions: ...},
  year     = {2023},
  pmid     = {36012345},
}
```

---

## SOCES parameters

| Parameter | Default | Notes |
|---|---|---|
| `N_CONSECUTIVE` | `20` | Consecutive-exclude stopping window — paper's recommended default |
| `P_CUTOFF` | `0.50` | Proposition inclusion threshold |
| `MAX_BIB_RECORDS` | `2000` | Hard cap enforced at parse time |
| `ABSTRACT_CHARS` | `1500` | Characters of abstract fed to the screener |
| `PARQUET_PATH` | `embeddings.parquet` | Where embeddings are persisted |

### SOCES stopping parameter guidance (from paper, Table 1)

| n | Macro recall | Inference calls saved | Use case |
|---|---|---|---|
| 5  | 90.4% | 26% | Not recommended — recall penalty |
| 10 | 95.0% | 11% | Acceptable for rapid reviews |
| **20** | **95.6%** | **6%** | **Recommended default** |
| 30 | 95.6% | 5% | Marginal gain over n=20 |
| ∞  | 95.7% | 0% | Full unstopped ceiling |

---

## PRISMA-trAIce compliance

SOCES satisfies items M5–M10 of the PRISMA-trAIce checklist (Holst et al., 2025) **by architecture**:

- **M5** — every decision is a single JSON row in `results.csv`; AI exclusion count = rows where `decision == false`  
- **M6** — full proposition rubric in `v7_prompt_review.txt`; no hidden prompt variables  
- **M7** — declare `p ≥ 0.50` and `n = 20`; no per-corpus similarity floor to calibrate  
- **M8** — tool versions recorded in `summary.json`  

---

## Cost estimate

| Scale | Embedding cost | Screening cost | Total |
|---|---|---|---|
| 200 records | ~$0.001 | ~$0.008 | < $0.01 |
| 500 records | ~$0.003 | ~$0.02 | ~$0.02 |
| 2 000 records | ~$0.01 | ~$0.07 | ~$0.08 |

*Based on OpenRouter pricing for `bge-m3` and `jev-1.13` as of 2025. Check current rates at https://openrouter.ai/models*

---

## Reproducibility note

Re-running Cell 4 overwrites `embeddings.parquet`. If you want to screen the same corpus against a **different objective**, change `SR_OBJECTIVE` in Cell 2 and re-run from Cell 4 onward — only the objective embedding changes, so you can also manually update `obj_embedding` in the Parquet metadata if you want to skip re-embedding the full corpus.
