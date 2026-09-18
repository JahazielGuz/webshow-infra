# webshow — Roadmap

Built as **vertical slices**, not horizontal layers. Every slice ships backend *and* frontend
together, so interface work grows with the system rather than arriving in one block. Auth
deliberately comes after the catalogue is visible — invisible plumbing does not come first.

Three slices (6, 9, 11) have no visible output. They are marked, so a demo-less week is never
a surprise.

| # | Slice | Backend | Frontend added | Result |
| --- | --- | --- | --- | --- |
| 0 | Catalogue on screen | Postgres, Prisma, layered API, TMDB seed | home page, genre rows | **movies on a page** |
| 1 | Containerise & deploy | Dockerfile, Neon, Fly.io, CI | *(goes live)* | **a live URL** |
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

**Something worth looking at comes first.** Slices 0 and 1 were originally the other way round —
a deployed `/health` endpoint before any catalogue. That is the textbook walking skeleton, and it
was swapped deliberately.

The textbook argument for deploying first is real: the first deploy always surfaces surprises
(port binding, environment variables, build context, database reachability), and meeting them one
at a time beats meeting them all at once. The cost of deferring is that slice 1 now carries more
moving parts, so a failure there has more candidate causes.

It was still the right call. A live URL serving `{"status":"ok"}` proves the pipeline works and
demonstrates nothing. Two days of deferral is a scheduling choice, not an architectural one —
nothing in slice 0 is harder to containerise for having been written first.

**Accounts moved ahead of the deploy** (2026-09-17). The order actually taken is 0, then 3, then 1:
with the catalogue on screen, accounts are the next thing that makes it feel like a product, and
everything after them (ratings, history, tiers) needs a user to hang off. Deferring the deploy costs
one extra secret to provision when it happens, the token signing key, and no rework: nothing in the
auth slice is harder to containerise for having been written first.

**Similar titles moved ahead of the deploy and the event bus** (2026-09-17). The order actually taken is
0, 3, 7, then 1. Two consequences, both deliberate. `recs-service` has no domain events to consume yet, so its
ingest **reads the catalogue over webshow-core's public API** instead; the property that matters, that its data
is derived and rebuildable from the system of record, holds either way, and the consumer swaps in at slice 6.
And the first deploy will carry three services rather than one.

**Slice 7 delivers AI early.** *More Like This* is item-to-item similarity: it needs a movie's
vector but no user profile. A working AI feature therefore exists well before any personalisation
machinery does.

**Frontend is a dimension, not a phase.** Interface work grows with each slice rather than
arriving in one block.

## Local development

Prerequisites: Docker, Node 22+, `uv`, `gh`.

```bash
docker compose up -d      # Postgres only, for now
```

Docker currently runs the database and nothing else. The services themselves run on the host
via `npm run dev`; containerising them is slice 1. Redis and RabbitMQ join the compose file in
slice 6, when there is asynchronous work for them to do.

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
