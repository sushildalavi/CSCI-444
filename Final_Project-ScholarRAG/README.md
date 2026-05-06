# ScholarRAG

**A trust-first literature assistant with sentence-level grounding and a calibrated per-citation confidence score.**

[![Python 3.11](https://img.shields.io/badge/python-3.11-blue.svg)](https://www.python.org/downloads/release/python-3110/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-green.svg)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-18.3-61DAFB.svg?logo=react)](https://react.dev/)
[![pgvector](https://img.shields.io/badge/pgvector-0.7-336791.svg)](https://github.com/pgvector/pgvector)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED.svg?logo=docker)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> Final project for **USC CSCI 444 — Natural Language Processing**.
> Authors: **Sushil Dalavi**, **Parvathi Sanjana Pericherla**, **Eshna Gupta**.
> Repository:
> [`sushildalavi/CSCI-444 → Final_Project-ScholarRAG`](https://github.com/sushildalavi/CSCI-444/tree/main/Final_Project-ScholarRAG).

ScholarRAG answers questions about a set of research papers and binds **every answer sentence to a specific cited passage**, then attaches a calibrated confidence score combining three orthogonal signals: **M** (NLI entailment between claim and passage), **S** (retrieval stability under query paraphrases), and **A** (lexical corroboration across distinct sources). The logistic blend of these features is fit on **530 claim–evidence pairs** that the three project authors labeled independently.

The system runs in two modes: **uploaded** (over the user's own PDFs, indexed in pgvector) and **public** (a parallel fan-out across seven scholarly APIs).

---

## Headline Results

| Hypothesis | Metric | Value |
|---|---|---|
| **H3** — Trustworthy gold | Avg pairwise Cohen's κ | **0.47** (moderate) |
|  | Three-way unanimous | **59.8 %** |
| **H2** — Reliable retrieval | Recall@5 | **0.992** |
|  | Recall@10 | **1.000** |
|  | MRR | **0.981** |
|  | nDCG@10 | **0.986** |
| **H1** — Faithful confidence | Brier (full fit) | **0.160** |
|  | AUC-ROC (full fit) | **0.852** |
|  | Brier (5-fold CV) | **0.163 ± 0.015** |
|  | AUC (5-fold CV) | **0.845 ± 0.028** |

Evaluated on a corpus of 15 landmark ML/NLP papers (1997–2023) and 120 GPT-4o-mini-generated queries spanning four query types (definitional, methodology, factual, limitations).

---

## Table of Contents

- [Slides and Dataset](#slides-and-dataset)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [Benchmark Results](#benchmark-results)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Design Decisions](#design-decisions)
- [Calibration Pipeline](#calibration-pipeline)
- [Re-indexing after Model Change](#re-indexing-after-model-change)
- [Environment Variables](#environment-variables)
- [Authors](#authors)

---

## Slides and Dataset

- **Class Presentation** — [`ScholarRAG_Class_Presentation.pptx`](ScholarRAG_Class_Presentation.pptx).
- **Calibration Dataset** — [`Evaluation/data/calibration/`](Evaluation/data/calibration/) contains the 530 majority-voted claim–evidence pairs, the three coder workbooks (`coder_A.xlsx`, `coder_B.xlsx`, `coder_C.xlsx`), the inter-annotator-agreement report (`iaa_report.json`), the fitted logistic weights (`calibration_fit.json`), and the reliability-diagram data (`reliability_diagram.xlsx`).
- **Retrieval Eval** — [`Evaluation/queries/queries_120.json`](Evaluation/queries/queries_120.json) (120 queries with `target_doc_id`) and [`Evaluation/data/retrieval_eval_120.json`](Evaluation/data/retrieval_eval_120.json) (Recall/MRR/nDCG report).

---

## Architecture

![System Architecture](images/system_architecture.png)

### Dual Retrieval Pipeline

![Dual Retrieval Pipeline](images/dual_retrieval_pipeline.png)

### M/S/A Confidence Pipeline

![MSA Confidence Pipeline](images/msa_confidence_pipeline.png)

### Database Schema (ER Diagram)

![ER Diagram](images/er_diagram.png)

---

## Key Features

- **Sentence-level citation grounding** — every output sentence is bound to a specific retrieved chunk; the UI surfaces the supporting passage alongside the claim.
- **M/S/A per-citation confidence** — calibrated logistic blend of NLI entailment (`M`), retrieval stability under paraphrases (`S`), and cross-source lexical corroboration (`A`). Weights are stored in Postgres and reloaded at inference time without a code change.
- **Hybrid dense + sparse retrieval** — pgvector cosine similarity over 1024-d `mxbai-embed-large` embeddings combined with BM25-style token overlap, fused with weight `α = 0.35`.
- **Public-mode aggregation** — concurrent `ThreadPoolExecutor` fan-out across **OpenAlex, arXiv, Semantic Scholar, Crossref, Springer, Elsevier, IEEE**, with DOI / title-fingerprint deduplication.
- **Embedding contract** — `provider`, `model`, `version`, `dim` stored on every vector row and filtered at query time, preventing silent vector mixing across model upgrades.
- **Citation-grounded generator** — GPT-4o-mini under a constrained prompt that abstains when retrieval evidence is insufficient instead of hallucinating.
- **Open annotation/calibration loop** — the same system that serves answers exports them as claim–evidence spreadsheets that humans label; a script computes IAA, takes a majority vote, and refits the logistic. New weights go live with no code change.
- **Closed-loop retrieval evaluation** — `scripts/eval_retrieval.py` computes Recall@K, MRR, and nDCG@K against the 120-query golden set.
- **Local-first stack** — React + Vite frontend, FastAPI backend, Postgres + pgvector, local Ollama embeddings, GPT-4o-mini for generation and NLI.

---

## Benchmark Results

### Calibration — Unified M/S/A Logistic

Calibration artifacts live in [`Evaluation/data/calibration/`](Evaluation/data/calibration/).

**Dataset**

| Item | Value |
|---|---|
| Claim–evidence pairs (binary rubric) | 530 |
| Independent human coders | 3 (the project authors) |
| Pairwise Cohen's κ (A-B / A-C / B-C) | 0.37 / 0.44 / 0.59 |
| **Average pairwise κ** | **0.47** (moderate) |
| Three-way unanimous agreement | 59.8 % |
| Gold label distribution | 50.4 % supported / 49.6 % unsupported |
| Uploaded mode / Public mode split | 262 / 268 |

**Fitted unified logistic** `P(supported | M, S, A) = σ(b + w₁·M + w₂·S + w₃·A)`

| Parameter | Value |
|---|---|
| `w₁` (M — NLI entailment) | **3.81** |
| `w₂` (S — retrieval stability) | **−0.29** |
| `w₃` (A — lexical multi-source corroboration) | **3.35** |
| `b` (bias) | **−4.86** |
| **Brier score** | **0.160** (random = 0.25) |
| Log-loss | 0.484 |
| **AUC-ROC** | **0.852** |

**Per-mode ablation** (justifies the unified fit)

| Fit | n | Brier | AUC |
|---|---|---|---|
| Pooled (unified) | 530 | 0.160 | 0.852 |
| Uploaded-only | 262 | 0.160 | 0.847 |
| Public-only | 268 | 0.153 | 0.853 |
| **Δ pooled vs per-mode avg** | — | **+0.003** | — |

Pooled Brier is within 0.003 of the per-mode weighted average — well below the 0.02 threshold at which separate per-mode fits would be warranted.

**Held-out generalization** (5-fold stratified CV, seed = 42; full report in [`Evaluation/data/calibration/cv_metrics.json`](Evaluation/data/calibration/cv_metrics.json))

| Metric | CV mean ± std | In-sample |
|---|---|---|
| Brier | **0.163 ± 0.015** | 0.160 |
| AUC | **0.845 ± 0.028** | 0.852 |
| Log-loss | 0.494 ± 0.038 | 0.484 |

CV Brier sits 0.003 above in-sample and within one fold-std — no meaningful overfitting on the 530-pair set.

**Feature ablation** (logistic refit for each subset)

| Model | Brier ↓ | AUC ↑ |
|---|---|---|
| `S` only | 0.249 | 0.548 |
| `A` only | 0.204 | 0.679 |
| `M` only | 0.190 | 0.786 |
| `M + S` | 0.190 | 0.790 |
| `M + A` | 0.161 | 0.850 |
| **`M + S + A`** | **0.160** | **0.852** |

The lift over `M`-only comes almost entirely from `A`. Adding `S` is essentially flat — our paraphrases were too mild to make stability informative beyond what entailment already captured.

### Retrieval Quality (Top-10 on the 120-Query Corpus)

Run with `python scripts/eval_retrieval.py --eval-set Evaluation/queries/queries_120.json --k 10`. Report in [`Evaluation/data/retrieval_eval_120.json`](Evaluation/data/retrieval_eval_120.json).

| Metric | Value |
|---|---|
| Cases | 120 |
| **Recall@5** | **0.992** (119 / 120) |
| **Recall@10** | **1.000** (120 / 120) |
| **MRR** | **0.981** |
| **nDCG@10** | **0.986** |

The target document surfaces in the top-10 for **every** query and at rank-1 for the vast majority — confirming the hybrid retriever reliably ranks the intended chunks first.

### Public Research Mode (7-API Aggregation)

Evaluated on 20 diverse ML/NLP queries with live API calls.

| Metric | Value |
|---|---|
| Queries tested | 20 |
| Total results returned | 200 |
| Avg results per query | 10.0 |
| Median search latency | 4.77 s |

| Provider | Results | Share |
|---|---|---|
| OpenAlex | 56 | 28.0 % |
| Elsevier / Scopus | 52 | 26.0 % |
| Semantic Scholar | 34 | 17.0 % |
| arXiv | 29 | 14.5 % |
| Crossref | 20 | 10.0 % |
| Springer | 9 | 4.5 % |

> Round-robin selection ensures provider diversity. 6 of 7 APIs contribute results; IEEE requires a separate API key. Latency is dominated by the slowest API in the concurrent fan-out.

### System Latency (uploaded mode, p50 / p95 / p99 ms)

| Stage | p50 | p95 | p99 |
|---|---|---|---|
| Embed query | 28 | 62 | 115 |
| Retrieve | 95 | 210 | 380 |
| Rerank | 18 | 45 | 90 |
| Generate | 310 | 720 | 1240 |
| **Total** | **420** | **980** | **1600** |

Measured on a 3-chunk context window with GPT-4o-mini, local Postgres pgvector, and local Ollama.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind |
| Backend | FastAPI, Python 3.11, Pydantic, Uvicorn |
| Database | PostgreSQL 16 with `pgvector` (HNSW index) |
| Embeddings | Ollama, `mxbai-embed-large` (1024-d) |
| Generation / NLI | OpenAI `gpt-4o-mini` |
| Public retrieval | OpenAlex, arXiv, Semantic Scholar, Crossref, Springer, Elsevier, IEEE |
| Containerization | Docker, Docker Compose |
| Testing | pytest, ruff |

---

## Quick Start

### Prerequisites

- Python 3.11+
- Node.js 18+
- Docker (for Postgres)
- Ollama running locally

### 1. Clone

```bash
git clone https://github.com/sushildalavi/CSCI-444.git
cd CSCI-444/Final_Project-ScholarRAG
```

### 2. Configure

Set the following in your shell (or in a `.env` file the backend reads):

```bash
export OPENAI_API_KEY=sk-...
export DATABASE_URL=postgresql://scholarrag:scholarrag@127.0.0.1:5432/scholarrag_db
export OLLAMA_BASE_URL=http://127.0.0.1:11434
```

For the frontend:

```bash
cp frontend/.env.example frontend/.env
```

### 3. Start Postgres and Ollama

```bash
docker compose up -d db
ollama pull mxbai-embed-large
ollama serve
```

### 4. Start the backend

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn backend.app:app --reload --host 127.0.0.1 --port 8000
```

### 5. Start the frontend

```bash
cd frontend
npm ci
npm run dev
# → http://localhost:5173
```

### 6. Run tests

```bash
pip install -r requirements-dev.txt
make test
```

---

## Project Structure

```
Final_Project-ScholarRAG/
├── backend/
│   ├── app.py                       # FastAPI entrypoint — CORS, routers, startup
│   ├── pdf_ingest.py                # PDF extraction, chunking, pgvector upsert
│   ├── public_search.py             # 7-API fan-out, dedup, hybrid rerank
│   ├── intent_resolver.py           # GPT-4o-mini intent + canonical term + variants
│   ├── sense_resolver.py            # Deterministic-lexicon fallback for ambiguous terms
│   ├── confidence.py                # M/S/A logistic blend (loads weights from Postgres)
│   ├── eval_metrics.py              # Recall@K, MRR, nDCG (pure functions)
│   ├── services/
│   │   ├── assistant_utils.py       # Citation-grounded generator, S and A computation
│   │   ├── nli.py                   # M (entailment) via GPT-4o-mini, lru-cached
│   │   ├── judge.py                 # LLM-as-judge faithfulness evaluation
│   │   ├── embeddings.py            # Ollama embedding contract
│   │   └── db.py                    # DB connection helpers
│   ├── utils/                       # 7 scholarly-API clients + config + logging
│   ├── scripts/                     # Calibration pipeline (see below)
│   └── tests/                       # pytest suite
├── frontend/
│   └── src/
│       ├── routes/                  # UploadedChat, PublicChat, Analytics
│       ├── components/chat/         # ChatShell, SourcesPanel (per-citation MSA chips)
│       ├── components/analytics/    # Reliability charts, latency, recall plots
│       ├── api/                     # HTTP client + TypeScript types
│       └── styles.css
├── db/
│   ├── init.sql                     # Postgres + pgvector schema
│   └── migrations/                  # Schema migrations
├── scripts/
│   ├── eval_retrieval.py            # Retrieval-metrics harness
│   └── reindex_embeddings.py        # Re-embed chunks after model change
├── Evaluation/
│   ├── papers/                      # 15-paper corpus (PDFs gitignored)
│   │   ├── download_corpus.sh       # Reproducible multi-source downloader
│   │   └── MANIFEST.md              # Paper list with official links
│   ├── queries/                     # 120 evaluation queries + claim/evidence pairs
│   └── data/calibration/            # 530 labeled pairs, gold, IAA, fit, reliability
├── images/                          # Architecture and pipeline diagrams (used in README)
├── ScholarRAG_Class_Presentation.pptx  # In-class slide deck
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── requirements-dev.txt
├── pyproject.toml                   # pytest + ruff config
└── Makefile                         # make test / lint / eval / run
```

---

## Design Decisions

### Why pgvector?

pgvector gives us approximate-nearest-neighbor search as a first-class PostgreSQL extension. That buys us transactional consistency, metadata filtering on the same row (`provider`, `model`, `version`, `dim` — the embedding contract), HNSW indexes for sub-millisecond search at this scale, and co-location of vector and relational data in a single query. No separate vector store to keep in sync.

### Why hybrid scoring?

Pure dense retrieval misses lexically specific terms (acronyms, model names, author names) that appear sparsely but are highly relevant. Pure sparse retrieval misses semantic synonymy. The hybrid score `(1 − α) · cosine_sim + α · sparse_overlap` with `α = 0.35` captures both. Most research queries are semantic, so dense retrieval dominates; sparse overlap is the correction signal for named-entity-heavy queries.

### Why M/S/A confidence vs. a single similarity score?

Cosine similarity measures retrieval proximity, not answer faithfulness. **M** (NLI entailment) captures whether the retrieved evidence actually supports the generated claim. **S** (retrieval stability) captures how consistently the same evidence surfaces under paraphrased queries. **A** (multi-source agreement) captures cross-provider corroboration. We deliberately keep `A` lexical rather than another NLI call so the calibration fit is not regressing the label on itself — `A` stays statistically independent of `M`. The logistic blend with calibrated weights tracks human judgment more closely than any single signal alone (see the ablation table above).

### Why open the calibration loop?

The same FastAPI service that answers queries can also export every claim–evidence pair as a workbook. Three coders label them independently, a script computes IAA + majority vote, and the logistic is refit. The new weights go into a row in the `confidence_calibration` Postgres table and are loaded by the inference path on the next request — no code change, no redeploy. This keeps the calibration tied to the system's actual current outputs rather than a frozen benchmark.

---

## Calibration Pipeline

End-to-end reproduction (see [`Evaluation/README.md`](Evaluation/README.md) for methodology notes):

| Step | Command | Output |
|---|---|---|
| 1. Ingest corpus | `python -m backend.scripts.ingest_corpus` | 15 documents in `documents` table |
| 2. Generate queries | `python -m backend.scripts.generate_queries` | `Evaluation/queries/queries_120.json` |
| 3. Build coder workbooks | `CODEBOOK_MAX_QUERIES=80 CODEBOOK_INCLUDE_PUBLIC=true PUBLIC_IEEE_LIMIT=0 python -m backend.scripts.build_codebooks` | 3 coder xlsx + `claim_evidence_pairs.json` |
| 4. (Manual) Label each workbook | — | three filled `coder_*.xlsx` files |
| 5. IAA + majority vote | `python -m backend.scripts.compute_iaa_majority` | `iaa_report.json`, `gold_labels.xlsx` |
| 6. M/S/A features | `python -m backend.scripts.extract_msa_features` | `features.xlsx` |
| 7. Fit + ablation + DB write | `python -m backend.scripts.fit_unified_calibration --write-db` | `calibration_fit.json`, `reliability_diagram.xlsx`, DB row `label='unified'` |
| 8. Activate | `export CONFIDENCE_USE_FITTED_WEIGHTS=true` | Calibrated logistic served at `/assistant/answer` |

---

## Re-indexing after Model Change

If you change the embedding model, provider, or version:

```bash
# 1. Update the embedding env vars (OLLAMA_EMBED_MODEL, EMBEDDING_VERSION, EMBEDDING_RAW_DIM)
# 2. Re-embed all chunks
source .venv/bin/activate
python scripts/reindex_embeddings.py --purge-all
```

The embedding contract (`provider`, `model`, `version`, `dim`) stored on every chunk row prevents the new vectors from being mixed into queries that ran against the old contract.

---

## Environment Variables

| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | OpenAI key for generation, intent resolution, and the NLI step |
| `RESEARCH_CHAT_MODEL` | Generation model (default: `gpt-4o-mini`) |
| `INTENT_RESOLVER_MODEL` | Intent-resolver model (default: `gpt-4o-mini`) |
| `EMBEDDING_PROVIDER` | `ollama` (default) or `openai` |
| `OLLAMA_BASE_URL` | Ollama endpoint (default: `http://127.0.0.1:11434`) |
| `OLLAMA_EMBED_MODEL` | Embedding model (default: `mxbai-embed-large`) |
| `EMBEDDING_VERSION` | Schema-compatibility tag (e.g. `mxbai-embed-large-v1`) |
| `EMBEDDING_RAW_DIM` | Raw embedding dimension (1024 for mxbai) |
| `VECTOR_STORE_DIM` | pgvector column dimension (1536 for backward compat) |
| `OPENAI_EMBEDDING_MODEL` | OpenAI embedding model when `EMBEDDING_PROVIDER=openai` |
| `OPENAI_EMBED_DIMENSIONS` | Requested embedding dimensions for OpenAI embeddings |
| `EMBEDDING_QUERY_PREFIX` | Optional prefix prepended to query strings before embedding |
| `EMBEDDING_DOC_PREFIX` | Optional prefix prepended to document chunks before embedding |
| `EMBEDDING_BATCH_SIZE` | Batch size for the embedding client (default `16`) |
| `EMBEDDING_TIMEOUT_SECONDS` | HTTP timeout for embedding requests (default `30`) |
| `EMBEDDING_RETRY_ATTEMPTS` | Max retries on embedding failure (default `3`) |
| `EMBEDDING_RETRY_DELAY` | Backoff delay between retries in seconds (default `1.5`) |
| `STORAGE_DIR` | On-disk directory for uploaded PDFs (default `./storage`) |
| `DATABASE_URL` | Postgres connection string |
| `CORS_ORIGINS` | Comma-separated allowed origins |
| `CONFIDENCE_USE_FITTED_WEIGHTS` | `true` to load fitted weights from `confidence_calibration` |
| `PUBLIC_SPARSE_WEIGHT` | Hybrid α (default `0.35`) |
| `PUBLIC_PROVIDER_MAX_WORKERS` | Public-mode fan-out concurrency (default `7`) |
| `IEEE_LIMIT` | Set to `0` to disable IEEE provider |

### Healthcheck

```bash
curl http://127.0.0.1:8000/health/embeddings
```

Returns Ollama reachability, embedding shape, the active `provider` / `model` / `version`, and the configured pgvector dimension.

---

## Development

```bash
make lint       # ruff check
make lint-fix   # ruff auto-fix
make test       # full pytest suite
make eval       # closed-loop retrieval evaluation
```

Conventions:
- Python 3.11 type hints on public functions
- No bare `except:` — always catch a specific exception
- Run `make lint && make test` before pushing
- Report Recall@5, MRR, and nDCG@10 in any change that affects retrieval

---

## Authors

- **Sushil Dalavi** — `sdalavi@usc.edu`
- **Parvathi Sanjana Pericherla** — `pperiche@usc.edu`
- **Eshna Gupta** — `eshnagup@usc.edu`

USC, CSCI 444 — Natural Language Processing (Spring 2026).

---

## License

MIT — see [`LICENSE`](LICENSE) (or the badge above) for terms.
