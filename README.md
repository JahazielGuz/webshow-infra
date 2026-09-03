# webshow

A streaming-service style application built as a study in **production LLM engineering** —
retrieval-augmented search, embedding-based recommendations, and agentic orchestration,
deployed on a real budget.

This repository holds the shared infrastructure: the roadmap, local orchestration
(`docker compose`), and deployment configuration. Each service lives in its own repository.

## Architecture

| Repository | Runtime | Responsibility |
| --- | --- | --- |
| `webshow-core` | Node + Express + Prisma | System of record: users, catalog, ratings, watch history, subscriptions |
| `webshow-web` | Next.js | Frontend |
| `recs-service` | Python + FastAPI | Embedding-based recommendations (pgvector) |
| `discovery-service` | Python + FastAPI + LangGraph | Natural-language search over a RAG pipeline |
| `llm-gateway` | Node | OpenAI-compatible proxy: provider keys, per-service token accounting, caching, fallback |
| `webshow-contracts` | JSON Schema | Versioned event and JWT claim contracts shared across services |

The monolith owns all transactional data and is the only service that writes it. The AI
services consume domain events and own only **derived** data — embeddings and vectors — so
they can be rebuilt from scratch by replaying events.

## Design decisions

**Authentication is asymmetric.** `webshow-core` holds an EdDSA private key and is the only
service that can mint tokens; every other service verifies against a public JWKS endpoint.
No shared secret means no service can forge a token it was only meant to validate.

**Recommendation is a distance measurement, not a prompt.** Movie descriptions are embedded
once at ingest. At request time the system performs an approximate-nearest-neighbour lookup
and a language model only reranks the resulting shortlist. This is the standard
retrieve-then-rerank cascade.

**Retrieval is hybrid.** Lexical search (Postgres full-text + trigram) and semantic search
(pgvector) are fused with Reciprocal Rank Fusion, keeping a single datastore rather than
synchronising a separate search cluster.

**Evaluation ships with the service.** Each AI service carries its own golden dataset and
fails CI on regression, rather than relying on manual inspection.

See [ROADMAP.md](./ROADMAP.md) for the build plan.
