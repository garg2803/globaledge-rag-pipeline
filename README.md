# GlobalEdge Market Intelligence — RAG Pipeline

**No-Code Track | n8n + Pinecone + OpenAI | 2026**

---

## Problem

Equity brokers at GlobalEdge cover 450 clients each and handle 30–50 daily interactions. Between 7–9:15 AM, 60–80 financial news articles and SEC filings are published — but brokers can only absorb 20–30% before client calls begin. Recommendations are made from memory rather than today's data. 100+ page SEC filings go unread. No firm-wide standard for pre-call intelligence exists.

**Result:** Poor quality, unverified advice to clients.

---

## Solution

A 2-phase no-code RAG (Retrieval-Augmented Generation) pipeline built in n8n that lets brokers query financial news, stock prices, and SEC filings in plain English — and receive grounded, source-backed answers in seconds.

---

## Architecture

### Phase 1 — Document Ingestion
- Reads 3 data sources in parallel: `global_news.csv`, `stock_price_details.csv`, `sec_filings_10q.pdf` (126 pages)
- Splits text using Recursive Character Text Splitter (chunk size 500/300/800, overlap 50)
- Embeds using OpenAI `text-embedding-3-small` (1536 dimensions)
- Stores all vectors in Pinecone index `globaledge-rag`

### Phase 2 — Question Answering & Evaluation
- Chat Trigger accepts broker query in plain English
- Vector Store Retriever fetches top-K relevant chunks from Pinecone
- GPT-4o-mini generates a structured answer: **Signal / Supporting Data / Risks & Caveats**
- A parallel eval pipeline retrieves chunks independently and runs LLM-as-a-Judge scoring
- Evaluation scores logged to n8n Data Table for quality monitoring

---

## Tech Stack

| Component | Tool |
|---|---|
| Workflow Automation | n8n |
| Vector Database | Pinecone |
| Embedding Model | OpenAI text-embedding-3-small (1536 dims) |
| LLM | GPT-4o-mini |
| Evaluation | LLM-as-a-Judge (GPT-4o-mini) |

---

## Configuration Comparison

Three retrieval configurations were tested across 5 broker queries each (15 total test cases):

| Parameter | Config 1 (Baseline) | Config 2 (Precision) | Config 3 (Breadth) |
|---|---|---|---|
| Chunk Size | 500 | 300 | 800 |
| Top-K | 6 | 4 | 8 |
| Temperature | 0.2 | 0.1 | 0.3 |
| Max Tokens | 1000 | 800 | 1200 |

---

## Results

| Metric | Config 1 | Config 2 | Config 3 |
|---|---|---|---|
| Relevance | 4.6/5 | 4.6/5 | 4.4/5 |
| Groundedness | 4.0/5 | 4.2/5 | **4.4/5** ⭐ |
| Completeness | 4.4/5 | 4.0/5 | 3.6/5 |
| Overall | 4.0/5 | 4.0/5 | **4.4/5** ⭐ |

**Key Finding:** Config 3 (Breadth) achieved the highest overall and groundedness scores — counterintuitively, larger chunks and broader retrieval improved factual accuracy rather than reducing it.

**Recommended Config:** Config 3 (Breadth) — 800 token chunks, Top-K 8, Temperature 0.3

---

## Sample Output

**Query:** *"Based on this week's news sentiment around Amazon, would you classify the current signal as bullish or bearish, and does the price data support that?"*

> **Signal:** Bullish
>
> **Supporting Data:** Amazon received double upgrades from Bank of America (price target $298) and KeyBanc (price target $325) ahead of Q1 earnings. Both firms forecast AWS growth accelerating to 28–30%, up from 24% in Q4.
>
> **Risks/Caveats:** Actual price movement depends on earnings outcomes. Disappointing results or guidance could cause volatility despite current bullish sentiment. Regulatory and macroeconomic risks remain.

*Evaluation: Relevance 5/5, Groundedness 5/5, Completeness 4/5, Overall 5/5 — PASS ✅*

---

## Files in This Repo

```
├── README.md
├── Phase_1_Document_Ingestion.json        # n8n workflow — import to run Phase 1
├── Phase_2_Question_Answering_Eval.json   # n8n workflow — import to run Phase 2
├── Project_2_GlobalEdge.pdf               # Full project report
└── screenshots/                           # n8n workflow and output screenshots
```

---

## How to Run

1. Create a [Pinecone](https://app.pinecone.io) account and index named `globaledge-rag` (1536 dims, cosine)
2. Add OpenAI and Pinecone credentials in n8n
3. Upload your 3 data files via n8n Manage Files
4. Import `Phase_1_Document_Ingestion.json` → run once to populate Pinecone
5. Import `Phase_2_Question_Answering_Eval.json` → open Chat Trigger URL → ask questions

---

## Key Learnings

- RAG-grounded inference derives bullish/bearish signals from retrieved evidence — no pre-labeling needed
- Breadth beats Precision: larger context windows improved groundedness, not the other way around
- LLM-as-a-Judge enables continuous quality monitoring without manual review
- System prompt structure (Signal / Supporting Data / Risks) enforces consistent, actionable output format
