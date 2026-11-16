# Since

Since is a chat agent for questions whose answers change over time.

It remembers what it told you, when it learned it, and whether current public evidence still agrees. Instead of treating a transcript as memory, Since stores dated episodes, decisions, claims, and source observations so it can answer questions like:

> “On Monday we shortlisted five CRM tools for a 20-person sales team. Has any of them changed pricing or shipped something big since?”

The agent has one model-visible tool: `web_search`.

## Why this approach

The difficult part of these questions is not producing fluent text. It is keeping time, provenance, and comparison semantics intact.

Since is designed around five decisions:

- **Temporal intent:** each turn is classified as `chat`, `fresh`, `recall`, or `recheck`; dates become explicit intervals in the user's timezone.
- **Hybrid memory:** dated questions use exact date-range lookup; open-ended recall combines SQLite FTS5 and embedding similarity with reciprocal-rank fusion.
- **Parallel gathering:** memory recall and web research run concurrently, then join through one deterministic graph node.
- **Typed comparison:** prices and limits are compared by amount, currency, plan, unit, billing period, region, and conditions—not by matching numbers in prose.
- **Evidence discipline:** sources receive stable evidence IDs, claims retain provenance, unsupported generated sentences are validated and downgraded, and historical claims are never deleted.

## Architecture

```text
question
  -> analyze intent, entities, dates, volatility
  -> gather memory || web evidence
  -> normalize sources
  -> extract typed observations
  -> compare against historical claims when needed
  -> generate a streamed answer
  -> validate evidence-backed sentences
  -> repair unsupported claims deterministically
  -> persist episode, claims, decisions, and trace
```

The workflow is implemented as a compiled LangGraph in [`src/graph/build.ts`](src/graph/build.ts). Each node has one responsibility:

| Stage | Responsibility |
| --- | --- |
| `analyze` | classify intent, resolve dates, identify entities, create a bounded search plan |
| `gather` | run memory recall and web research concurrently and join once |
| `normalize` | deduplicate sources and create stable evidence IDs |
| `extract` | convert source passages into typed, provenance-aware observations |
| `compare` | deterministically calculate changed, unchanged, or unclear outcomes |
| `generate` | synthesize an answer and stream tokens with evidence IDs |
| `validate` | check generated claims against evidence and comparison rows |
| `repair` | replace unsupported sentences with explicit uncertainty |
| `persist` | write the auditable record and trace to SQLite |

### Memory and evidence

SQLite stores sessions, turns, searchable episodes, entities, claims, decisions, and cached search checks. Episodes are indexed with FTS5 and optionally embedded with Gemini. Claims are append-only: a new observation can supersede an old claim without destroying the historical record.

Web research uses Tavily through [`src/search/tool.ts`](src/search/tool.ts). Search is bounded to eight parallel first-pass queries, cached by volatility class, and allowed up to three best-effort refinement calls when evidence gaps remain. The model sees only the `web_search` function; memory, comparison, persistence, and validation remain application-controlled.

### API and client

Fastify serves the static client and a small API:

- `POST /api/chat` — validates a message and streams typed NDJSON events;
- `GET /api/sessions` and `GET /api/sessions/:id` — session history;
- `GET /api/trace/:turnId` — persisted node trace;
- `POST /api/demo` — labeled synthetic history;
- `GET /health` — process check.

The browser renders status, tokens, diffs, sources, verification, repair, completion, and error events. The UI includes responsive session history, suggested prompts, copy/regenerate, Markdown export, and bounded text-file context without changing the backend's chat contract.

## Run locally

Requirements: Node.js 22+ and npm.

```bash
npm install
cp .env.example .env
```

Add the required keys to `.env`:

```dotenv
OPENROUTER_API_KEY=
TAVILY_API_KEY=
GOOGLE_API_KEY=
```

Then start the app:

```bash
npm run dev
```

Open [http://127.0.0.1:3000](http://127.0.0.1:3000).

Optional configuration:

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `3000` | local HTTP port |
| `USER_TIMEZONE` | `Asia/Kolkata` | interpretation of relative dates |
| `DB_PATH` | `data/since.sqlite` | SQLite database path |
| `FAST_MODEL` | configured Haiku model | analysis, extraction, selection, validation |
| `STRONG_MODEL` | configured Sonnet model | answer synthesis |
| `EMBEDDING_MODEL` | `gemini-embedding-2` | semantic episode recall |

## Try the hiring scenarios

Load the synthetic demo history in the UI, then try:

1. `On Monday we shortlisted 5 CRM tools for a 20-person sales team. Has any of them changed pricing or shipped something big since?`
2. `What did I ask you on 3 November? Is your answer still correct today?`
3. `Last Tuesday I asked you about Postgres hosting. Has any pricing changed since then?`

The seeded history is synthetic and clearly labeled. Its old values are plausible fixtures; the recall, research, comparison, citation, and uncertainty mechanisms are real.

For a CLI turn without the browser:

```bash
npm run turn -- "What did I ask you on 3 November?" --scratch
```

## Development commands

```bash
npm run dev                 # watch-mode server and UI
npm run typecheck           # TypeScript validation
npm test                    # Vitest unit/integration suite
npm run seed                # seed local demo history
npm run eval -- --offline   # memory/date/honesty evaluation without Tavily
npm run eval                # full evaluation when all provider keys exist
```

The evaluation runner uses a scratch database and checks dated recall, shortlists, follow-ups, missing history, no-fabrication behavior, citations, rechecks, and unverified baselines. It records latency, first-token time, search count, and optional judge scores for question fit, concision, and honesty.

## Repository map

```text
src/config.ts              provider, model, timeout, cache, and database config
src/server/                Fastify routes and NDJSON streaming
src/graph/                 LangGraph state, routers, prompts, schemas, and nodes
src/memory/                FTS5/vector retrieval and entity resolution
src/search/                web_search contract, Tavily adapter, and cache
src/domain/                date, pricing, and source-ranking rules
src/db/                    SQLite schema, queue, and repository operations
public/                    responsive chat client
seed/                      synthetic demo fixtures
evals/                     scenario assertions and evaluation runner
tests/                     deterministic provider-seamed tests
```

## Tradeoffs and limits

I rejected transcript-only memory, an unrestricted autonomous search loop, model-written comparisons, multiple agent tools, and a separate vector database. Those options make the system harder to bound, inspect, or test for this single-user local workload.

This is intentionally a local prototype: there is no auth, collaboration, scheduler, source snapshot archive, or durable binary upload. Tavily cannot see logged-in or paywalled pages. Extraction can be lossy, and provider stalls can dominate latency. When evidence is missing or incomparable, the intended behavior is to say so rather than guess.

The full architecture rationale, data model, evidence rules, and next-week roadmap are in [`DESIGN.md`](DESIGN.md).
