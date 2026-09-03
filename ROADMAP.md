# webshow — Roadmap

Built as **vertical slices**, not horizontal layers. Every slice ships backend *and* frontend
together and ends deployed, so there is always something live to look at. Auth deliberately
comes after the catalogue is visible — invisible plumbing does not come first.

Three slices (6, 9, 11) have no visible output. They are marked, so a demo-less week is never
a surprise.

| # | Slice | Backend | Frontend added | Result |
| --- | --- | --- | --- | --- |
| 0 | Walking skeleton | `/health`, Dockerfile, CI | one page calling the API | **a live URL** |
| 1 | Catalogue | Prisma, `Movie`, TMDB seed | poster grid, detail page | it looks like a product |
| 2 | Search & browse | Postgres FTS + `pg_trgm` | search bar, genre filters | a usable catalogue |
| 3 | Auth | argon2, `jose` EdDSA + JWKS | register / login, session header | accounts |
| 4 | Ratings & history | `Rating`, `WatchEvent` | star widget, Continue Watching | it remembers you |
| 5 | Subscription tiers | tier gating middleware | pricing page, upgrade flow | it feels like a product |
| 6 | Async infrastructure | compose, Redis, RabbitMQ, contracts | *(none)* | — |
| 7 | Similar titles | recs-service, pgvector, ingest | More Like This | **first visible AI** |
| 8 | Personalisation | user vectors, nearline consumer | Top Picks For You | it learns from you |
| 9 | LLM gateway | OpenAI-compatible proxy, metering | *(none)* | — |
| 10 | Agentic search | discovery-service, LangGraph CRAG | natural-language search | the headline feature |
| 11 | Evaluation | golden datasets, LangSmith, CI gate | *(none)* | measurable quality |
| 12 | Hardening | OpenTelemetry, cost dashboards | architecture page | production readiness |

## Why this order

**Deploy on day one.** Slice 0 is a walking skeleton — a thin end-to-end slice through every
layer, deployed. Deployment is never a cliff at the end of the project; every subsequent change
simply ships.

**Slice 7 delivers AI early.** *More Like This* is item-to-item similarity: it needs a movie's
vector but no user profile. That means a working AI feature exists well before any
personalisation machinery does.

**Frontend is a dimension, not a phase.** Interface work grows with each slice rather than
arriving in one block.

## Local development

Prerequisites: Docker, Node 22+, `uv`, `gh`.

```bash
docker compose up -d      # Postgres, Redis, RabbitMQ
```

## Deployment

| Component | Platform |
| --- | --- |
| Frontend | Vercel |
| API | Fly.io (scale-to-zero) |
| Postgres + pgvector | Neon |

Target running cost: under $30/month.
