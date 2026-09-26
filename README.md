# SOCES Systematic Review Screener

**Similarity-Ordered Consecutive-Exclude Screening (SOCES)** — AI-assisted title and abstract screening for medical systematic reviews.

Based on: *SOCES: Similarity-Ordered Consecutive-Exclude Stopping with Typed Probabilistic LLMs in Medical Systematic Reviews*

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/bigwisu/SOCES/blob/main/soces_screening.ipynb)

---

## What you need

| Requirement | Notes |
|---|---|
| OpenRouter API key | https://openrouter.ai — free tier sufficient for small reviews |
| A `.bib` file | Every entry **must** have `title` + `abstract` fields; max **2 000 records** |

No local Python install required if running on Google Colab.

---

## Quick start — Google Colab (recommended)

1. Click the **Open in Colab** badge above  
2. Run **Cell 1** — installs dependencies  
3. Run **Cell 2** — paste your OpenRouter key and SR objective  
4. Run **Cell 3** — upload your `.bib` file using the file picker  
5. Run **Cell 4** — embeds corpus with `bge-m3`, saves `embeddings.parquet`  
6. Run **Cell 5** — runs SOCES screening, saves `results.csv`  
7. Run **Cell 6** — renders PRISMA-trAIce flow and included-records table  
8. Run **Cell 7** — exports all output files  
9. Run **Cell 8** *(optional)* — replays stopping sweep at n ∈ {5,10,20,30,50,100,∞}  

> **Cell 4 only needs to run once per corpus.** If you change the SR objective and want to re-screen, re-run Cell 4 onward (objective vector is re-embedded then). Cell 4 can be skipped entirely on subsequent runs if `embeddings.parquet` is already present in the session.

---

## Quick start — local

```bash
pip install jupyter pybtex pandas pyarrow numpy requests tqdm ipywidgets
jupyter notebook soces_screening.ipynb
```

---

## Notebook cells

| Cell | What it does |
|---|---|
| **1 — Install** | Installs `pybtex`, `pandas`, `pyarrow`, `numpy`, `requests`, `tqdm`, `ipywidgets` in-kernel |
| **2 — Configure** | Paste OpenRouter key and SR objective; set SOCES parameters (defaults ready to use) |
| **3 — Upload `.bib`** | `ipywidgets` file-picker; parses `title` + `abstract`; skips entries missing either; enforces 2 000-record cap |
| **4 — Embed** | Batched `BAAI/bge-m3` embeddings via OpenRouter; saves corpus vectors + objective vector to **`embeddings.parquet`** |
| **5 — Screen** | Loads Parquet → cosine-ranks all records → scores sequentially with `typesafe/jev-1.13` via `/api/alpha/decisions` → SOCES consecutive-exclude stopping |
| **6 — PRISMA display** | PRISMA-trAIce flow block (AI excluded / forwarded for human review) + styled included-records table with scores |
| **7 — Export** | Writes `results.csv`, `results_included.csv`, `summary.json`, `v7_prompt_review.txt` |
| **8 — n sweep** *(optional)* | Replays stopping logic at multiple n values from saved scores — **zero additional API calls** |

---

## API endpoints used

| Purpose | Endpoint | Model |
|---|---|---|
| Embeddings | `POST /api/v1/embeddings` | `BAAI/bge-m3` |
| Screening | `POST /api/alpha/decisions` | `typesafe/jev-1.13` |

The screening model is a **decisions model** — it cannot use `/v1/chat/completions`. The notebook calls the correct `/api/alpha/decisions` endpoint with the Jev payload shape:

```json
{
  "model": "typesafe/jev-1.13",
  "state": { "text": "<CONTEXT: objective + title + abstract>" },
  "questions": {
    "noul_relevance": {
      "type": "noul",
      "instructions": "<QUESTION + TRUE IF + FALSE IF criteria>"
    }
  }
}
```

Response: `body["answers"]["noul_relevance"]["noul"]` → probability `p ∈ [0.0, 1.0]`

---

## Output files

| File | Contents | PRISMA-trAIce item |
|---|---|---|
| `embeddings.parquet` | Corpus vectors (`list<float32>` column) + objective vector in metadata | — |
| `results.csv` | One row per candidate: `rank`, `key`, `pmid`, `title`, `similarity`, `score`, `decision`, `source` | M5 |
| `results_included.csv` | Rows from `results.csv` where `decision == True` — the human-review queue | M5 |
| `summary.json` | PRISMA flow node counts, model names, thresholds | M7, M8 |
| `v7_prompt_review.txt` | Full V7 `noul_relevance` proposition rubric | M6 |
| `sweep.csv` | n-sweep results (no extra API calls) | — |

---

## `.bib` file requirements

Export from **Zotero**, **Mendeley**, **EndNote**, **PubMed**, or any tool that produces BibTeX. Entries missing `title` or `abstract` are skipped and counted in the parse summary.

```bibtex
@article{Smith2023,
  title    = {Effect of CBT on depression in type 2 diabetes},
  abstract = {Background: ... Methods: ... Results: ...},
  year     = {2023},
  pmid     = {36012345},
}
```

---

## SOCES parameters

| Parameter | Default | Notes |
|---|---|---|
| `N_CONSECUTIVE` | `20` | Consecutive-exclude stopping window |
| `P_CUTOFF` | `0.50` | Proposition inclusion threshold |
| `MAX_BIB_RECORDS` | `2000` | Hard cap enforced at parse time |
| `ABSTRACT_CHARS` | `1500` | Characters of abstract sent to screener |
| `PARQUET_PATH` | `embeddings.parquet` | Embedding cache file |

### Stopping parameter guidance (from paper, Table 1 — 70 medical SRs)

| n | Macro recall | Calls saved | Recommendation |
|---|---|---|---|
| 5 | 90.4% | 26% | ✗ Recall penalty |
| 10 | 95.0% | 11% | Rapid reviews only |
| **20** | **95.6%** | **6%** | **✓ Recommended default** |
| 30 | 95.6% | 5% | Marginal vs n=20 |
| ∞ | 95.7% | 0% | Full unstopped ceiling |

> **Note on stopping behaviour:** SOCES stopping only triggers when a long tail of irrelevant records follows the relevant cluster — typical of broad PubMed searches. On a curated journal export (e.g. all JMIR AI) where nearly everything is relevant to an AI-in-medicine objective, n=20 may never fire and the full corpus is scored. This is correct behaviour.

---

## PRISMA-trAIce compliance

| Item | What the notebook provides |
|---|---|
| M5 | `results.csv` — one row per candidate, `decision == false` count = AI exclusions |
| M6 | `v7_prompt_review.txt` — full rubric, no hidden variables |
| M7 | `p ≥ 0.50` + `n = 20` declared in `summary.json`; no per-corpus similarity floor |
| M8 | `typesafe/jev-1.13` + `BAAI/bge-m3` recorded in `summary.json` |

---

## Cost estimate

Per-record cost at standard OpenRouter pricing (check https://openrouter.ai/models for current rates):

| Corpus size | Embedding | Screening | Total |
|---|---|---|---|
| 300 records | ~$0.002 | ~$0.01 | ~$0.01 |
| 500 records | ~$0.003 | ~$0.02 | ~$0.02 |
| 2 000 records | ~$0.01 | ~$0.07 | ~$0.08 |

---

## License

MIT License — see [LICENSE](LICENSE) for full text.

You are free to use, modify, and distribute this notebook for any purpose, including commercial research, with attribution.
