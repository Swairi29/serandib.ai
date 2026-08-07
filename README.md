<div align="center">

# Serandib
### CSE Financial Intelligence · RAG Pipeline

A retrieval-augmented generation system built over annual reports of companies listed on the Colombo Stock Exchange. Ask natural language questions over 16+ financial filings — grounded, cited, and evaluated with RAGAS.

## What is Serandib?

Financial analysts spend hours searching annual reports to answer questions that should take seconds — *"How did People's Bank's capital adequacy ratio trend over the last three years?"* or *"Which CSE-listed conglomerate grew revenue fastest in 2023?"*

Serandib answers those questions directly.

The system ingests annual reports from CSE-listed companies across banking, manufacturing, and diversified sectors, builds a hybrid retrieval index over the corpus, and grounds every LLM response in retrieved document evidence. Every answer is traceable to a source chunk. Retrieval quality is measured using RAGAS — not estimated — with a curated evaluation set of domain-specific questions.

The result is a system that behaves less like a chatbot and more like a financial research assistant that cites its sources.

---

## Corpus

| Sector | Companies | Files |
|---|---|---|
| **Banking** | Commercial Bank, HNB, Sampath Bank, People's Leasing | COMB_2022, COMB_2023, HNB_2023, PLC_2022, SAMP_2022 |
| **Diversified** | JKH, Hayleys, LOLC, Capital Alliance | JKH_2022, JKH_2023, JKH_2024, LOLC_2022, CALT_2022 |
| **Manufacturing** | Ambeon, Central Finance, CIC, Ceylon Grain, Panasian, RIL | AMBN_2023, CFIN_2022, CIC_2022, GRAN_2022, PAP_2023, RIL_2023 |

All PDFs are sourced directly from `cdn.cse.lk` — no authentication required.

---

## Architecture

```
data/raw/<sector>/*.pdf
        │
        ▼
┌─────────────────┐
│  loader.py      │  PyMuPDF — extract text page-by-page with metadata
└────────┬────────┘
         │  [{text, page, company, sector, fiscal_year, ...}]
         ▼
┌─────────────────┐
│  chunker.py     │  RecursiveCharacterTextSplitter — 512 tokens / 50 overlap
└────────┬────────┘
         │  [LangChain Documents with metadata]
         ▼
┌─────────────────┐
│  embedder.py    │  text-embedding-3-small → ChromaDB (local persistent)
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌──────────────────┐
│  retriever.py   │────▶│  FastAPI /query  │
│  Hybrid search  │     │  POST endpoint   │
│  Dense + BM25   │     └──────────────────┘
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  chain.py       │  LangChain RAG chain — GPT-4o-mini + retrieved context
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  run_ragas.py   │  RAGAS evaluation — faithfulness, relevancy, precision
└─────────────────┘
```

---

## Project Progress

| Phase | Description | Status |
|---|---|---|
| **01** | Domain & data selection, corpus strategy, download script | ✅ Complete |
| **02** | PDF ingestion, chunking, embedding, ChromaDB | 🔄 In progress |
| **03** | Hybrid retrieval, RAG chain, FastAPI endpoint | ⬜ Upcoming |
| **04** | RAGAS evaluation, scorecard, iteration | ⬜ Upcoming |
| **05** | Streamlit demo, Docker, README polish | ⬜ Upcoming |

---

## Quickstart

### 1. Clone and install

```bash
git clone https://github.com/yourname/serandib.git
cd serandib
pip install -r requirements.txt
```

### 2. Set up environment

```bash
cp .env.example .env
# Add your OpenAI API key to .env
```

### 3. Download the corpus

```bash
# Verify CDN URLs are live
python serandib_download_cse.py --check

# Download all 16 PDFs into data/raw/
python serandib_download_cse.py
```

### 4. Run the ingestion pipeline

```bash
# Dry run — inspect chunks without writing to ChromaDB
python src/ingest/run_pipeline.py --dry-run

# Full run — embed and store (~3–6 minutes, costs ~$0.01)
python src/ingest/run_pipeline.py
```

### 5. Verify the index

```python
import chromadb
client = chromadb.PersistentClient(path="./data/chroma")
col = client.get_collection("serandib_cse_reports")
print(col.count())   # expect 1800–2400 chunks
```

---

## File Structure

```
serandib/
├── data/                          # gitignored
│   ├── raw/
│   │   ├── banking/               # COMB_2022.pdf, HNB_2023.pdf …
│   │   ├── diversified/           # JKH_2022.pdf …
│   │   ├── manufacturing/         # CIC_2022.pdf …
│   │   └── manifest.json
│   ├── processed/
│   │   └── chunks.jsonl           # one JSON object per chunk
│   └── eval/
│       ├── questions.json         # 50 curated QA pairs  [Phase 04]
│       └── ragas_results.json     # RAGAS scores per run [Phase 04]
│
├── src/
│   ├── ingest/
│   │   ├── loader.py              # ✅ PDF text extraction
│   │   ├── chunker.py             # ⬜ chunk + attach metadata
│   │   ├── embedder.py            # ⬜ embed + write to ChromaDB
│   │   └── run_pipeline.py        # ⬜ orchestrator
│   ├── retrieval/
│   │   ├── retriever.py           # ⬜ hybrid dense + BM25
│   │   └── reranker.py            # ⬜ optional cross-encoder
│   ├── generation/
│   │   ├── chain.py               # ⬜ LangChain RAG chain
│   │   └── prompts.py             # ⬜ prompt templates
│   ├── evaluation/
│   │   ├── build_eval_set.py      # ⬜ QA pair builder
│   │   └── run_ragas.py           # ⬜ RAGAS scoring runner
│   └── api/
│       └── main.py                # ⬜ FastAPI app
│
├── app/
│   └── ui.py                      # ⬜ Streamlit demo  [Phase 05]
│
├── notebooks/
│   ├── 01_data_exploration.ipynb  # ⬜
│   └── 02_chunking_experiments.ipynb # ⬜
│
├── serandib_download_cse.py       # ✅ corpus downloader
├── requirements.txt               # ⬜
├── docker-compose.yml             # ⬜ [Phase 05]
├── Dockerfile                     # ⬜ [Phase 05]
├── .env.example
├── .gitignore                     # ✅
└── README.md                      # ✅
```

---

## Chunk Metadata Schema

Every chunk stored in ChromaDB carries these fields, enabling filtered retrieval by company, sector, or year:

```python
{
    "source_file": "COMB_2023.pdf",
    "company":     "COMB",
    "sector":      "banking",
    "fiscal_year": 2023,
    "page_number": 47,
    "chunk_index": 12
}
```

---

## Stack

| Layer | Technology |
|---|---|
| PDF extraction | PyMuPDF (fitz) |
| Chunking | LangChain RecursiveCharacterTextSplitter |
| Embeddings | OpenAI text-embedding-3-small |
| Vector store | ChromaDB (local) → Pinecone (production) |
| Retrieval | Hybrid: dense vectors + BM25 |
| Generation | GPT-4o-mini via LangChain |
| Evaluation | RAGAS (faithfulness, relevancy, precision, recall) |
| API | FastAPI |
| Experiment tracking | MLflow |
| Demo UI | Streamlit |
| Deployment | Docker + docker-compose |

---

## Environment Variables

```bash
# .env.example — copy to .env and fill in values

OPENAI_API_KEY=sk-...
CHROMA_PERSIST_DIR=./data/chroma
EMBEDDING_MODEL=text-embedding-3-small
LLM_MODEL=gpt-4o-mini
```

---

## What's Coming

The following will be added as each phase completes:

- `requirements.txt` — pinned dependencies for full reproducibility
- `src/ingest/chunker.py` — chunking with metadata attachment
- `src/ingest/embedder.py` — embedding and ChromaDB write
- `src/ingest/run_pipeline.py` — full ingestion orchestrator
- `src/retrieval/retriever.py` — hybrid search (dense + BM25)
- `src/generation/chain.py` — RAG chain with citation support
- `src/api/main.py` — FastAPI `/query` endpoint
- `data/eval/questions.json` — 50 curated QA pairs over the corpus
- **RAGAS scorecard** — faithfulness, answer relevancy, context precision benchmarked across chunking and retrieval configurations
- `app/ui.py` — Streamlit chat interface
- `Dockerfile` + `docker-compose.yml` — one-command deployment
- Loom walkthrough video

---

## Why Serandib?

*Serandib* is the ancient Arabic name for Sri Lanka — used by traders navigating to Asian financial centres for centuries. The name reflects the project's dual identity: rooted in Sri Lanka's financial ecosystem (the CSE corpus), built with the tools of modern financial AI (RAG, hybrid retrieval, LLM evaluation).

---

<div align="center">
<sub>Built as a portfolio project · CSE data sourced from cdn.cse.lk · Not financial advice</sub>
</div>
