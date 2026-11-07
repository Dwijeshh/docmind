# Architecture

DocMind is planned as a small set of independently testable services. This document records the intended design; implementation details can change when benchmarks or tests provide evidence.

## Boundaries and data flow

The Next.js frontend handles chat, document browsing, and source highlighting. FastAPI authenticates requests, checks roles, accepts uploads, and streams answers. A worker parses and indexes documents. PostgreSQL stores document metadata, permissions, chunks, summaries, full-text terms, and pgvector embeddings. An LLM provider abstraction serves summaries and answers through Claude or Ollama.

Upload processing is asynchronous so OCR, embedding, and summarization do not hold an HTTP connection. The worker should use idempotent jobs keyed by document content hash and indexing version. Failed jobs need visible status and a safe retry path. The original upload should be stored only as long as required by the chosen deployment design; a future retention policy must cover deletion and reindexing.

## Ingestion decisions

PyMuPDF exposes PDF page boundaries; python-docx exposes document structure, but not stable rendered pages. TXT has no native pages. A citation therefore stores a page number only when source pagination is reliable, plus a section or paragraph locator and a passage offset. OCR runs on PDF pages with little or no extractable text, not indiscriminately on every page. Heading-aware chunks target 300–500 tokens with overlap; exact boundaries and overlap are configurable and must be checked against retrieval quality. Store source text and metadata alongside each chunk so highlights can come from the retrieved passage.

Content hashing avoids repeated indexing of identical bytes. File hashes alone do not detect semantically identical documents exported in different formats; that is a known limitation. If access rights differ between duplicate uploads, the storage model must retain distinct document ownership/grants while reusing safe derived content only where authorized.

Document summaries are separate from chunks because broad questions often require selecting several relevant documents before passage search. Generate a 3–5 sentence summary, embed it, and store the model identifier and generation version. Summary search narrows or prioritizes the document set; a configurable fallback must avoid losing a relevant document when its summary omits an important detail.

## Search and ranking

Use a swappable `Embedder` contract. BGE-small is the default candidate, but vector dimension and normalization depend on the exact model selected. Do not mix embeddings from different models in one search or HNSW index. Store model name per vector and plan a reindex when the model changes. HNSW accelerates approximate nearest-neighbor search on both chunks and summaries. PostgreSQL `tsvector` supports lexical matching of names, policy terms, and exact wording.

Optional query expansion produces 3–5 searches. Summary ranking identifies likely documents, then hybrid chunk retrieval merges vector and full-text results with reciprocal rank fusion (RRF). Keep roughly 20–30 candidates before diversity and reranking. A per-document cap is the first diversity strategy because it is simple to inspect; MMR can be evaluated later. Recency is a bounded ranking boost, not a filter or an authority rule. A cross-encoder selects roughly 5–8 passages for the prompt. Every switch is recorded in evaluation output, including model versions and retrieval parameters.

The access predicate must be part of each SQL retrieval query. Filtering after retrieval can distort top-k results and risks exposure through summaries, prompts, or source panels. A query for a user must join or test grants before it ranks candidate rows. Integration tests should seed overlapping roles, private documents, and identical text under different permissions, then assert that unauthorized rows never appear in retrieval or output.

## Answers and citations

The LLM receives only authorized retrieved passages and a grounded prompt. It should cite each factual claim with a stable passage ID resolved by the API to document title, page or locator, and date. The API should validate citation IDs against the supplied passages before streaming or finalizing the answer. If evidence is insufficient, return a consistent not-found response. Conflicts are shown with both dated sources; recency ranking cannot decide policy authority. Retrieved documents are untrusted input and should not be allowed to override system instructions.

Streaming complicates citation validation because incomplete output may already have been sent. The design should either stream structured citation events after validation or buffer claim-level units until their references can be checked. The first implementation may stream answer text while holding back unvalidated citation markup, with tests for interrupted streams.

## Evaluation and operations

Use a synthetic, versioned evaluation set of at least 50 questions across `SPECIFIC` and `BROAD`. Record expected source document and page or locator, and assess Recall@k and MRR from retrieval results. Define answer correctness and faithfulness rubrics before scoring; retain evidence for each judgment. Compare a vector baseline with each enabled stage, and report results by question type without inventing values.

Docker Compose will run API, worker, pgvector-backed PostgreSQL, and frontend locally. CI will lint and test code when it exists. The AWS guide should map those same boundaries to deployable services and explain secret storage, backups, migrations, OCR dependencies, and cost before any deployment. Cloud resources and paid API use require a separate decision.

## Tradeoffs to revisit

| Choice | Benefit | Cost or risk | Check |
| --- | --- | --- | --- |
| Summary-first search | Helps broad questions and large collections | Summary may omit a key fact | Compare Recall@k with and without summary gating |
| pgvector and `tsvector` in PostgreSQL | One transactional store | Index and query tuning at scale | Measure latency and recall on 200+ synthetic documents |
| Per-document cap | Easy to explain and test | May suppress a document with many relevant passages | Compare against MMR |
| Bounded recency boost | Surfaces recent evidence | Recent does not mean authoritative | Conflict evaluation cases |
| Local Ollama option | No paid API required for development | Model quality and hardware vary | Report provider and model per run |
