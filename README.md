# Data Pipeline Continuous Improvement

**A self-updating vector pipeline that processes 373 AI conversations (1.2M words) into 17,428 searchable chunks — with incremental updates that run in <10 seconds instead of 5+ minutes.**

---

## The Problem

The obvious way to build a semantic search pipeline is also the wrong way: parse everything, embed everything, store everything, repeat when anything changes. At small scale it works. At 373 conversations and 224MB of raw JSON, a full rebuild takes 5+ minutes and makes the pipeline too expensive to run frequently — so you run it less often, the index drifts from reality, and retrieval quality degrades silently.

The deeper problem is that most pipelines are built for a static dataset. Production systems aren't static. Conversations accumulate. Documents change. New content arrives continuously. A pipeline that can't incorporate incremental changes without full rebuilds isn't a production pipeline — it's a batch job pretending to be one.

There's a third problem specific to conversational data: conversations don't chunk like documents. A 10,000-word document has natural section breaks. A 10,000-word conversation has topic shifts, digressions, and context that spans 50 exchanges. Naive chunking by token count destroys the retrieval signal. The chunk boundaries matter as much as the embeddings.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    INGESTION LAYER                      │
│                                                         │
│  Raw JSON conversations (224MB, 373 files)              │
│  Parse → extract message content + metadata             │
│  Detect conversation type (human/AI turns)              │
│  Apply chunking strategy per content type               │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  CHANGE DETECTION                       │
│                                                         │
│  Hash-based fingerprinting per chunk                    │
│  Compare against stored hashes in pgvector table        │
│  Identify: new chunks / modified chunks / deleted       │
│  Skip unchanged content entirely                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   EMBEDDING LAYER                       │
│                                                         │
│  sentence-transformers (all-MiniLM-L6-v2)              │
│  Process only new/modified chunks                       │
│  Batch embedding for efficiency                         │
│  Handle API rate limits + retry logic                   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                    STORAGE LAYER                        │
│                                                         │
│  Supabase pgvector (migrated from ChromaDB local)       │
│  Upsert changed vectors, preserve unchanged             │
│  doc_embeddings table with metadata columns             │
│  Cloud-accessible for n8n webhook retrieval             │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   RETRIEVAL LAYER                       │
│                                                         │
│  n8n webhook → HuggingFace embed query →                │
│  pgvector cosine similarity search →                    │
│  ranked results with source metadata                    │
│  Sub-second latency on 17,428 chunks                    │
└─────────────────────────────────────────────────────────┘
```

**Migration path:** Started with ChromaDB local (7,396 chunks) → migrated to Supabase pgvector (17,428 chunks) to enable cloud access and n8n webhook integration. The migration doubled the index size while reducing query latency through pgvector's native indexing.

---

## Key Insight

**Hash-based incremental updates turn a 5-minute rebuild into a <10-second sync.**

The approach is simple in retrospect: before embedding any chunk, compute a hash of its content. Store the hash alongside the vector. On subsequent runs, hash every candidate chunk and compare against stored hashes. Only chunks with changed or missing hashes need to be embedded and stored.

In practice, on any given day, 98%+ of the corpus is unchanged. A full rebuild embeds everything anyway. Incremental updates only touch what changed. The difference — 5+ minutes vs <10 seconds — makes the pipeline practical to run after every session instead of once a week.

The second insight is about **migration cost vs. migration benefit.** Moving from ChromaDB local to Supabase pgvector meant rebuilding the entire index (unavoidable), but unlocked cloud accessibility that enabled the n8n webhook retrieval layer. The one-time rebuild cost was worth it for the architectural capability gained. Pipeline migrations should be evaluated on downstream enablement, not just migration effort.

---

## Results

- **17,428 total chunks** indexed: 14,335 conversation chunks + 3,093 document chunks across 506 files
- **373 conversations processed** — 1.2M words, 224MB raw JSON
- **Incremental updates: <10 seconds** vs 5+ minutes for full rebuild (hash-based change detection)
- **Sub-second retrieval latency** on full corpus via pgvector cosine similarity
- **Migration completed:** ChromaDB local (7,396 chunks) → Supabase pgvector (17,428 chunks), 2.4x index growth
- **Production use:** Pipeline used to mine own conversation history — 50 semantic queries across 485 result chunks produced a verified 33-accomplishment inventory
- **Monthly distribution:** Nov 2025 (4,975 chunks), Jan 2026 (6,069 chunks) — heaviest building periods visible in embedding density

---

## Built With

- **Python** — ingestion, parsing, chunking, hash computation
- **sentence-transformers** (`all-MiniLM-L6-v2`) — embedding model
- **Supabase / PostgreSQL + pgvector** — vector storage and similarity search
- **HuggingFace Inference API** — cloud embedding for n8n workflow layer
- **n8n** — webhook-triggered retrieval workflow
- **MRMINOR semantic-index-update-skill v1.1** — packages incremental update logic as reusable capability
- **Conversation data from October 2024 onward** comprising the corpus

---

## Related Work

This pipeline is part of a larger [Human-AI Operating System (HAIOS)](https://mrminor-dev.github.io) — a production infrastructure for human-AI collaboration, in production since October 2024.

Other components:
- [semantic-search-framework](https://github.com/MrMinor-dev/semantic-search-framework) — the retrieval layer built on top of this pipeline
- [knowledge-mining-framework](https://github.com/MrMinor-dev/knowledge-mining-framework) — used this pipeline to mine 33 accomplishments from 373 conversations
- [ai-session-orchestration](https://github.com/MrMinor-dev/ai-session-orchestration) — the session system that generates the conversation corpus
- [database-security-framework](https://github.com/MrMinor-dev/database-security-framework) — the access controls governing the pgvector database

---

## Author

**Jordan Waxman** — AI Systems & Operations  
14 years operations leadership — building human-AI infrastructure since 2025  

[Portfolio](https://mrminor-dev.github.io) · [GitHub](https://github.com/MrMinor-dev) · [Email](mailto:waxmanj@mac.com)

---

*Part of the MRMINOR HAIOS/AOS production infrastructure. Every number verified from production systems.*
