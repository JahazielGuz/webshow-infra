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

## Maturity levels

Every ticket carries a level label. The level is orthogonal to the slice: a slice cuts
*vertically* through the stack, a level says *how far* that cut is taken. The same
capability is revisited at higher levels as the system matures — the catalogue endpoint
is built at L1, cached and indexed at L3, and instrumented and cost-tracked at L4.

| Label | Question it answers | Representative work |
| --- | --- | --- |
| `level:1-foundations` | Does it work correctly? | Domain models, first working path, happy-path endpoints |
| `level:2-production` | Does it survive reality? | Deploys, migrations, secrets, CI, error paths, health checks |
| `level:3-scale` | Does it hold under load? | Async processing, caching, indexing, retrieval quality |
| `level:4-platform` | Is it governable? | Observability, evals, cost control, contracts others depend on |

Filter the board by level to see the ladder rather than the chronology:

```bash
gh issue list --label "level:3-scale"
```
