# Financial Documents RAG Assistant

> A retrieval-augmented generation system that lets analysts query thousands of financial documents in natural language, with source citations.

## Client

**HWP** (anonymised) -- A mid-size independent financial advisory firm serving high-net-worth clients across the UK.

| Detail | Value |
|--------|-------|
| Industry | Financial Advisory / Wealth Management |
| Size | 25 advisors, 8 compliance staff |
| Location | City of London, UK |
| Document corpus | ~4,000 regulatory docs, fund prospectuses, client reports |

## The Problem

HWP's analysts spent 3+ hours per client meeting preparing by searching through regulatory documents, fund prospectuses, and internal research notes. Their challenges:

- Documents scattered across shared drives in PDF, Word, and Excel formats
- Full-text search missed context: searching "ESG requirements" returned 200+ results with no ranking
- Compliance team needed audit trails: "which document says we can/cannot do X?"
- Junior analysts asked the same questions senior staff had answered dozens of times

**Pain:** 3+ hours prep per meeting x 15 meetings/week = 45+ analyst hours/week wasted on document search. Compliance reviews took 2x longer because staff couldn't cite exact sources.

## The Solution

A RAG (Retrieval-Augmented Generation) chatbot powered by n8n that ingests financial documents into a vector database (Qdrant), then lets anyone ask questions in plain English and get answers with exact source citations.

### How It Works

```
[Document Ingestion Pipeline]
New Document (PDF/Word)
       |
       v
[File Trigger] --> [Read File] --> [Text Splitter]
       |
       v
[Generate Embeddings] --> [Store in Qdrant Vector DB]
       |
       v
[Check for duplicates] --> [Update or Insert]

[Query Pipeline]
User Question (Chat UI)
       |
       v
[Chat Trigger] --> [Embed Question]
       |
       v
[Vector Store Retrieval] --> [Top-K relevant chunks]
       |
       v
[QA Chain with LLM] --> [Answer + Source Citations]
```

### Architecture

| Component | Technology |
|-----------|-----------|
| Orchestration | n8n (self-hosted) |
| Vector Database | Qdrant |
| Embeddings | OpenAI text-embedding-3-small |
| LLM | OpenAI GPT-4o (swappable) |
| Document Processing | Recursive Character Text Splitter |
| Interface | n8n Chat Trigger (embeddable widget) |
| Deployment | Docker Compose (n8n + Qdrant) |

## Key Design Decisions

- **Why Qdrant over Pinecone/Weaviate?** Self-hosted, no per-query costs, GDPR-compliant (data stays on-premise). Critical for a financial firm handling sensitive client data.
- **Why recursive text splitting?** Financial documents have complex structure (tables, footnotes, appendices). Recursive splitting preserves semantic coherence better than fixed-size chunks.
- **Why duplicate detection?** Documents get updated versions. The pipeline checks if a document already exists and updates the vectors rather than creating duplicates.
- **Why source citations?** Compliance requirement. Every answer must reference the exact document and section it came from.

## Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Meeting prep time | 3+ hours/meeting | 15-25 minutes (depends on topic complexity) | ~85-90% reduction |
| Compliance citation time | 45 min per query | 1-3 minutes | ~95% reduction |
| Analyst hours saved | 0 | ~35-40 hrs/week | Reallocated to client advisory |
| Document search accuracy | ~60% (keyword) | ~88-92% (semantic, varies by doc type) | Significant improvement |

> Accuracy measured on a sample of 200 queries evaluated by senior analysts over 4-week pilot. Regulatory documents scored lower (~85%) due to dense legal language.

## Setup

### Prerequisites
- Docker & Docker Compose
- OpenAI API key

### Quick Start
```bash
docker compose up -d
# n8n at http://localhost:5678
# Qdrant at http://localhost:6333
# Import workflow/workflow.json
# Set OpenAI credentials
# Drop documents into the watched folder
# Open the chat interface to query
```

### docker-compose.yml
```yaml
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
      - ./sample-data:/data/documents
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changeme

  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

volumes:
  n8n_data:
  qdrant_data:
```

## Example Run

### Query
```
"What are the fee structures for the Balanced Growth Fund, and are there any
ESG disclosure requirements I should mention to the client?"
```

### RAG Retrieval
The system embeds the query and retrieves the top 3 most relevant chunks from Qdrant:

| Rank | Source Document | Section | Relevance Score |
|------|----------------|---------|-----------------|
| 1 | fund_prospectus.pdf | Section 4 -- Fee Structure | 0.94 |
| 2 | regulatory_update_FCA_PS25-3.pdf | Section 3 -- Key Requirements | 0.91 |
| 3 | regulatory_update_FCA_PS25-3.pdf | Section 4 -- Effective Dates | 0.87 |

### AI Response
```
Based on the retrieved documents, here is the information you requested:

**Fee Structure -- HWP Balanced Growth Fund:**
The Fund charges an Annual Management Charge (AMC) of 0.75% of NAV, calculated
daily and deducted monthly. There is also a performance fee of 10% on returns
above the composite benchmark, subject to a high-water mark and capped at 1.5%
of NAV per year. The total Ongoing Charges Figure (OCF) is estimated at 0.92%.
No entry or exit charges apply. [Source: fund_prospectus.pdf, Section 4.2]

**ESG Disclosure Requirements:**
Under the FCA's new SDR regime (Policy Statement PS25/3), all client-facing
reports must include a standardised ESG summary panel from 1 January 2026
onwards. The panel must show the portfolio's sustainability rating on the
FCA's five-tier classification. Additionally, firms must not use sustainability-
related terms in portfolio names unless the strategy meets SDR label criteria,
with a compliance deadline of 30 September 2025. You should inform the client
that their quarterly reports will include this new ESG panel starting Q1 2026.
[Source: regulatory_update_FCA_PS25-3.pdf, Section 3.2]
```

### Confidence & Sources
- **Confidence:** 0.89
- **Sources cited:** 2 documents
- **Retrieval time:** 1.2s

---

## Challenges & Iteration

- **V1 used fixed 1000-token chunks.** Financial tables got split mid-row, causing nonsense answers about fund performance. Fix: switched to recursive character text splitter at 512 tokens with 50-token overlap, plus a table detection step that keeps tables as single chunks.
- **Embedding quality varied wildly by document type.** Annual reports embedded well but regulatory filings (dense legal text) returned irrelevant results. Fix: added metadata tagging (doc_type, date, fund_name) and filtered retrieval by doc_type when the question mentioned regulation vs performance.
- **First week: analysts reported "it makes stuff up."** Root cause: questions about recent fund changes hit old documents (2019 data answering 2025 questions). Fix: added date-weighted retrieval -- recent documents get a relevance boost. Also added explicit "I don't have recent data on this" fallback when confidence is low.
- **Ingestion pipeline crashed on password-protected PDFs** (3 out of ~400 documents). Silent failure -- no error, just missing from the vector store. Fix: added PDF validation step that flags protected files for manual handling.

## Constraints & Trade-offs

- **Why Qdrant over cloud vector DBs:** GDPR compliance requirement. Client data cannot leave UK infrastructure. Qdrant self-hosted on their own servers.
- **Why not fine-tuning:** Document corpus changes weekly (new fund reports, updated regulations). RAG lets you update by dropping new files, not retraining.
- **Why OpenAI embeddings over open-source:** Tested nomic-embed-text and sentence-transformers. OpenAI's ada-002 had 15-20% better retrieval accuracy on financial terminology. Worth the API cost for this corpus size.
- **Why 512-token chunks:** Tested 256, 512, 1024. 512 was the sweet spot -- 256 lost context (especially for table data), 1024 returned too much noise in retrieval.

## Edge Cases & Error Handling

| Scenario | System Response | Fallback |
|----------|----------------|----------|
| PDF parsing fails (scanned image PDF) | Flagged for OCR pre-processing, analyst notified | Manual: run through OCR pipeline first |
| Question outside knowledge base | LLM responds "I don't have information on this topic" with suggested search terms | Analyst uses traditional search |
| Contradictory sources (old vs new regulation) | Returns both with dates, flags the conflict | Analyst decides which is current |
| Very long documents (500+ pages) | Split into logical sections before chunking | Automatic, but may miss cross-section references |
| Qdrant disk full | Ingestion halts, alert to IT | IT expands storage, re-run failed ingestion batch |

## Monitoring & Maintenance

- **Weekly analytics:** most-queried topics, lowest-confidence answers, documents with zero hits (dead content)
- **Feedback loop:** analysts rate answers thumbs up/down. Low-rated answers flagged for review -- usually reveals a chunking issue or missing document
- **Monthly:** compliance team verifies that deleted/superseded documents are removed from the vector store (regulatory requirement)
- **Qdrant health check:** daily cron checks collection size, index status, and disk usage. Alerts on >80% disk.

## Customisation

- **Different LLM:** Swap the OpenAI chat model node for Mistral, Claude, or a local Ollama model
- **Different vector DB:** Replace Qdrant nodes with Pinecone or Weaviate
- **Add authentication:** Put the chat widget behind your existing SSO
- **Add feedback loop:** Let users rate answers to improve retrieval quality over time

## Tech Stack

`n8n` `Qdrant` `OpenAI Embeddings` `RAG` `Vector Search` `Document Processing` `Docker`

## Lessons Learned

1. Chunk size matters more than model choice for RAG quality -- 512 tokens with 50-token overlap worked best for financial documents
2. Duplicate detection saved the client from "which version is this answer from?" confusion
3. Source citations transformed this from "nice chatbot" to "compliance-approved tool"

---

*Built by [Kessog Chan](https://linkedin.com/in/kessogchan) -- AI Solutions | Workflow Automation | Finance & Banking*
