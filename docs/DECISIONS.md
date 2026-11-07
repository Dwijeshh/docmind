# Decision log

Record new decisions with the date, reason, alternatives, and consequences. Revisit entries when implementation or evaluation evidence changes.

## 2026-11-07 — Documentation before implementation

- **Decision:** Establish the product requirements, architecture, and measurement plan before writing runtime code.
- **Reason:** The project is a portfolio piece where explainable tradeoffs and measured results matter more than feature count.
- **Consequence:** Roadmap items and evaluation values remain unchecked or TBD until implemented and measured.

## 2026-11-07 — PostgreSQL for metadata and retrieval

- **Decision:** Plan PostgreSQL with pgvector for embeddings and `tsvector` for lexical search.
- **Reason:** One transactional store can keep permissions, metadata, and search rows consistent.
- **Alternative:** A separate vector service and search engine may scale differently but adds operational complexity.
- **Consequence:** HNSW index tuning, vector-model compatibility, and SQL access predicates require careful tests.

## 2026-11-07 — Summary-first retrieval with fallback

- **Decision:** Store embedded document summaries separately and make summary search configurable.
- **Reason:** Broad questions may need document-level context before passage ranking.
- **Alternative:** Search every chunk directly.
- **Consequence:** Summary gating must be evaluated for missed specific facts; a fallback path is required.

## 2026-11-07 — Query-level permission enforcement

- **Decision:** Apply user and role grants in the retrieval SQL for summaries and chunks.
- **Reason:** Unauthorized passages must never reach ranking, prompts, or source panels.
- **Alternative:** Filtering retrieved rows afterward could both leak data and reduce recall.
- **Consequence:** Permission tests must include users with overlapping roles and duplicate content.

## 2026-11-07 — Provider and embedding abstractions

- **Decision:** Plan interfaces for the LLM provider and embedder, with Claude/Ollama and BGE-small as initial implementations/defaults.
- **Reason:** Local development should be possible without a paid API, and model choices should be measurable.
- **Consequence:** Store model identifiers and versions with derived data; reindex when embedding models change.

## 2026-11-07 — Evidence-first answers

- **Decision:** Require passage-backed citations and a consistent not-found response; display both sides of conflicts with dates.
- **Reason:** A fluent unsupported answer is a failure for document QA.
- **Consequence:** Citation identifiers need server-side validation, and streaming must not expose unchecked citations.
