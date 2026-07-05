# Recall — Build Plan

A search engine that indexes my school notes and answers questions with cited sources. Built as a real IR system: hand-rolled inverted index, BM25 ranking, and hybrid (keyword + vector) retrieval in Go, with an LLM answer layer on top that cites its sources and refuses to answer when retrieval confidence is low.

## Why not just paste the notes into an LLM?

A whole semester of notes doesn't fit a context window, and re-sending everything per question is slow and expensive. The right architecture is a persistent index with retrieval: fetch the few relevant chunks, then let the model answer from those with citations. This project builds that index from scratch rather than using a search library, because the index *is* the point.

## Architecture

```
[ ingest — once, Python ]  folder of files → chunks → (text + embeddings) → data file
                                      │
                                      ▼
[ Go service ]
   • load chunks → build inverted index (word → chunks) + load vectors
   • GET /search?q=...             → BM25 keyword search → top-k via min-heap
   • GET /search?q=...&mode=hybrid → BM25 fused with vector similarity
   • GET /ask?q=...                → retrieve top chunks → LLM writes cited answer,
                                     refuses if no good match
   • (later) shard the index + scatter-gather coordinator
```

## Stack

- **Go** for the service: index, ranking, HTTP API, concurrency. No search libraries; the inverted index and BM25 are implemented from scratch.
- **Python** for one-time ingestion: parse PDFs/slides/docs, chunk (~300–500 words with overlap, tagged with subject/file/page), fetch embeddings via concurrent API calls, write a single data file.
- **Storage:** one local data file (chunks + metadata + embeddings); the Go service loads it into memory at startup. No external database.
- **Packaging:** Docker, `docker compose up`.

## Phase 1 — Core engine

- [ ] **A1 Ingest** (Python): walk notes folder → parse → clean → chunk with overlap → tag (subject/file/page) → embeddings via worker pool → data file. *Verify: chunk count sane; a printed chunk maps to its source; timed with 1 vs N workers.*
- [ ] **A1b Vision pass** *(optional)*: describe diagram-heavy slides with a vision model so figures become indexable chunks. *Verify: a diagram question retrieves the described chunk.*
- [ ] **A2 Index + BM25 + top-k** (Go): tokenize → lowercase → stem → stop-words; inverted index; BM25 scoring (IDF, k1, b); top-k min-heap; `GET /search`. *Verify: topic query returns correct chunks, best-first.*
- [ ] **A3 Hybrid retrieval**: embed query → brute-force cosine top-k → fuse with BM25 → `mode=hybrid`. *Verify: a meaning-level query surfaces synonym chunks BM25 alone misses.*
- [ ] **A4 RAG `/ask`**: retrieve top chunks → build prompt → LLM → cited answer; refuse when the top retrieval score is below threshold. *Verify: real exam question gets a sourced answer; off-topic question gets a refusal.*
- [ ] **A5 Eval harness**: 15–20 question→expected pairs; measure retrieval hit-rate, answer quality, refusal correctness; log every failure case in `FAILURES.md`. *Verify: metrics print; refusals fire on unanswerable questions.*
- [ ] **A6 Benchmarks (initial)**: p50/p95 search latency, index build time, index size, cost per `/ask` query, ingestion speedup from the worker pool. One table in the README.

## Phase 2 — Demo & write-up

- [ ] **P1** Minimal static demo page (single HTML file: query box, answer, citations)
- [ ] **P2** Failure analysis: 3–5 real failure cases from `FAILURES.md`, one line each on why
- [ ] **P3** Project write-up
- [ ] **P4** 3-minute demo video
- [ ] **P5/P6** Submission checklist for the LaunchPad Challenge (deadline 31 July 2026)

## Phase 3 — Scaling roadmap (post-submission)

- [ ] **B1** Shard the index across N processes; scatter-gather coordinator (goroutine fan-out, channel fan-in, k-way merge)
- [ ] **B2** Full `BENCHMARKS.md`: p50/p95/p99, QPS, sharded vs unsharded, cache hit ratio
- [ ] **B3** Tests: unit (tokenizer, BM25, heap merge, cosine, fusion) + API integration test
- [ ] **B4** Rate limiting (token bucket), input validation, basic auth
- [ ] **B5** Concurrency-safe LRU cache for hot queries
- [ ] **B6** Deploy docs, `DESIGN.md`, README polish
- [ ] **B7** *(optional)* Agentic query expansion: rewrite the query and re-search when the first pass is weak
- [ ] **B8** *(optional)* Postings compression (delta + varint), skip pointers

## Design constraints

- The index is read-only after build → lock-free reads by design; the rate limiter and cache (Phase 3) are the parts that need synchronization.
- Local-first: no external database, one command to run.
- Real numbers only: every metric reported is measured, failures included.
