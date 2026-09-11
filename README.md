# webshow

A streaming-service style application built as a study in **production LLM engineering** —
retrieval-augmented search, embedding-based recommendations, and agentic orchestration,
deployed on a real budget.

This repository holds **no code and no infrastructure**. It is the roadmap, this README, and
the issue tracker — the things that belong to the project rather than to any one service.
Nothing here runs and nothing deploys from here. Each service lives in its own repository.

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

## Where things live

**The rule: ownership follows the data.** Whichever repository owns a piece of data owns the
thing that stores it, and declares it in its own `docker-compose.yml`. `webshow-core` owns the
catalogue, so core's compose file runs the Postgres holding it. `recs-service` owns its vectors,
so it will run its own. The practical test is that any service can be cloned and started on its
own — a service you cannot run without also cloning a second repository is a worse service.

**If you are looking for a `docker-compose.yml`, it is in the service that owns the data.**
There is not one here.

**External services are always their own container.** A message broker or cache — RabbitMQ,
Kafka, Redis — is a separate program with its own storage, clustering and version lifecycle. It
is never "inside" an application service, because there is no inside to put it in. In production
these are normally managed (CloudAMQP, Confluent Cloud, Upstash) and the application holds only
a connection string; the local container is a stand-in for that managed service, which is why it
looks external locally too.

**Which repository declares the shared ones is deliberately undecided.** There is one service
today and no second consumer, so the question has no information behind it yet. It belongs to
slice 6, when domain events first have somewhere to go.

| Infrastructure | Holds | Declared in | In production | Arrives |
| --- | --- | --- | --- | --- |
| Postgres | catalogue, users, ratings, watch history | `webshow-core` | Neon | **slice 0** |
| Postgres | user profile vectors *(derived)* | `recs-service` | Neon | slice 7 |
| Postgres | content index *(derived)* | `discovery-service` | Neon | slice 10 |
| Redis | cache, job state | undecided | Upstash | slice 6 |
| RabbitMQ | cross-service domain events | undecided | CloudAMQP | slice 6 |

Every Postgres above is a separate **database**, not a separate server — locally they may share
one container to save laptop resources, but no service ever reads another's tables. Data crosses
service boundaries as events (something happened) or HTTP calls (I need an answer now), never as
a shared table.

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
