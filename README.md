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
| `recommendations-service` | Python + FastAPI + LangGraph | Retrieval: embedding-based recommendations and natural-language search over one pgvector index |
| `llm-gateway` | Node | OpenAI-compatible proxy: provider keys, per-service token accounting, caching, fallback |
| `webshow-contracts` | JSON Schema | Versioned event and JWT claim contracts shared across services |

The monolith owns all transactional data and is the only service that writes it. The AI
services consume domain events and own only **derived** data — embeddings and vectors — so
they can be rebuilt from scratch by replaying events.

## Where things live

**The rule: ownership follows the data.** Whichever repository owns a piece of data owns the
thing that stores it, and declares it in its own `docker-compose.yml`. `webshow-core` owns the
catalogue, so core's compose file runs the Postgres holding it. `recommendations-service` owns its vectors,
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
| Postgres | movie vectors, user profile vectors, content index, all *(derived)* | `recommendations-service` | Neon | slices 7, 8, 10 |
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

**One retrieval service, not two.** Recommendation and natural-language search were planned as separate
services, and were merged before either was built (2026-09-17). They embed the same catalogue and query the
same vectors, so the split meant embedding everything twice, running two databases, and deploying twice to
keep one index warm. What the merge costs is a shared failure domain and two very different cost profiles in
one process: a search request spends money on model calls while a similarity lookup spends nothing. That line
now has to be drawn *inside* the service, with its own budget and rate limits per entry point, rather than
around it. If search traffic ever needs to scale or fail separately from recommendations, splitting it back
out is a deployment change, not a rewrite, because the two already sit behind different endpoints.

**Retrieval is hybrid.** Lexical search (Postgres full-text + trigram) and semantic search
(pgvector) are fused with Reciprocal Rank Fusion, keeping a single datastore rather than
synchronising a separate search cluster.

**Evaluation ships with the service.** Each AI service carries its own golden dataset and
fails CI on regression, rather than relying on manual inspection.

See [ROADMAP.md](./ROADMAP.md) for the build plan.
