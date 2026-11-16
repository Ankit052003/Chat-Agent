# Since: architecture and design

Since is a chat agent for questions whose answer changes over time. It remembers what it told the user, when it learned it, and whether current public evidence still agrees. The important design choice is to treat conversation as a dated record of statements, decisions, claims, and evidence instead of as a transcript that an LLM must reread and interpret from scratch.

## The core workflow

Every turn runs through one compiled LangGraph. The graph is deliberately small and observable:

```text
question
  -> analyze
  -> gather: memory recall || web research
  -> normalize evidence
  -> extract observations
  -> compare with the historical baseline when needed
  -> generate a streamed answer
  -> validate claims
  -> deterministic repair of unsupported sentences
  -> persist the episode, claims, decisions, and trace
```

`src/graph/build.ts` owns the graph; each step is a focused node under `src/graph/nodes/`. Conditional edges keep the fast paths short: greetings are handled deterministically, recall-only questions do not search, and comparison runs only for a recheck with targets.

### 1. Analyze intent and time

`analyze` classifies a turn as `chat`, `fresh`, `recall`, or `recheck`. For non-trivial questions, a fast structured model call returns entities, aliases, date expressions, decisions, volatile attributes, comparison targets, and a bounded search plan. Zod validates the response before it enters graph state.

Dates are resolved with `chrono-node` and Luxon into half-open UTC intervals using the configured user timezone. An unparseable date is omitted and recorded in the trace rather than guessed. For dated recall/recheck, entities are also backfilled from episodes in that date range so a follow-up such as "have they changed?" can recover what "they" means.

### 2. Gather memory and web evidence concurrently

`gather` starts memory recall and, when required, research with `Promise.all`, then joins the results in one node. This explicit join avoids a LangGraph fan-in running the downstream chain twice when parallel branches finish in different supersteps.

Memory recall in `src/memory/retrieve.ts` uses two independent signals:

- exact date-range lookup when the question contains a usable date;
- hybrid retrieval across all episodes using SQLite FTS5 ranks and cosine similarity over Gemini embeddings, fused with reciprocal rank fusion.

The top candidates can be narrowed by a small structured selector. Retrieval returns an explicit outcome: `exact`, `likely`, `multiple`, `not_found`, or `insufficient_context`. It also returns current claims for the recalled entities and stored decisions such as shortlists. A missing match is a product result, not an instruction to invent history.

Research in `src/graph/nodes/research.ts` plans one public-web query per known entity plus a bounded recency query. Up to eight first-pass searches run in parallel. Results are cached in SQLite by normalized query and volatility class: prices for 10 minutes, features for one day, and stable facts for 30 days. Each source is deduplicated, ranked, and stored with its URL, domain, title, passage, publication date, fetch time, and evidence ID.

The only model-visible tool is `web_search`, defined in `src/search/tool.ts`. It accepts a query, optional news/general topic, recency window, and domain restrictions. The deterministic planner uses the same adapter directly for predictable parallelism; if first-pass evidence leaves gaps, a fast model may make at most three refinement calls through that one tool. Tavily is an adapter, not part of the agent's reasoning boundary.

### 3. Normalize, extract, and compare

`normalize` removes unusable or duplicate source rows and renumbers evidence IDs once. `extract` uses a structured fast-model call to turn source passages into observations with provenance, verification, confidence, dates, pricing units, plans, regions, conditions, and effective dates. The extractor cannot promote an assistant statement into web evidence.

`compare` is code, not a model opinion. It chooses a comparable historical claim and compares fields such as amount, currency, period, billing mode, unit, plan, region, limits, and conditions. It reports `changed`, `unchanged`, or `unclear`, with reasons such as `price_increased`, `unit_changed`, `plan_changed`, `no_material_change`, `no_baseline`, or `insufficient_evidence`. An unverified seed baseline remains unverified; a newer page cannot retroactively make the old observation verified.

### 4. Generate, validate, and repair

`generate` uses the stronger configured model to answer from the structured context. It streams tokens through the server as they arrive. The prompt requires evidence IDs such as `[e3]`, not model-written URLs; the UI maps those IDs to stored source links. Recheck answers include the deterministic comparison rows before prose so the model cannot hide a mismatch in fluent language.

`validate` checks generated sentences against evidence and comparison rows. Comparison-backed sentences can be proven deterministically; web-backed claims are checked with a structured fast-model call. If validation finds unsupported claims, `repair` replaces those sentences with an explicit unverified statement. Repair is deterministic and limited to one pass, so the safety path cannot create a second unsupported claim while trying to fix the first.

### 5. Persist an auditable record

`persist` writes the completed assistant turn, trace, searchable episode, embedding, extracted claims, and decisions in one per-session queue. SQLite uses WAL mode and foreign keys. Claims are append-only: a new observation can point to the claim it supersedes, but the historical record is retained. Every turn and relevant row carries `is_synthetic` so seeded demo history cannot be confused with a user's real history.

The database is intentionally small and local:

| Table | Purpose |
| --- | --- |
| `sessions`, `turns` | ordered conversations and turn status |
| `episodes`, `episodes_fts` | searchable question/answer memory |
| `entities` | canonical names, aliases, and embeddings |
| `claims` | append-only observations and supersession chains |
| `decisions` | shortlists, selections, rejections, and preferences |
| `checks` | append-only cached web-search results |

## API and UI boundary

Fastify serves the static client and a deliberately thin API in `src/server/app.ts`:

- `POST /api/chat` validates the session/message and returns an NDJSON stream;
- `GET /api/sessions` and `GET /api/sessions/:id` provide history;
- `GET /api/trace/:turnId` loads the persisted trace;
- `POST /api/demo` seeds labeled synthetic history;
- `GET /health` supports a basic process check.

The stream emits typed `status`, `token`, `diff`, `sources`, `verification`, `patch`, `done`, and `error` events. `public/app.js` renders these events without owning agent decisions. The white workspace provides session search, responsive history, suggested prompts, file-as-text context, copy, regenerate, and Markdown export, but the existing chat/session/memory contract remains unchanged.

## Why this architecture

- **Fast on the common path:** greetings are regex-routed, dated recall avoids web search, memory and research overlap, and cached results avoid repeated fetches.
- **Correct about time:** dates become explicit intervals; claims retain observed, published, and effective dates; comparisons are field-aware instead of numeric string matching.
- **Honest by construction:** retrieval has explicit failure outcomes, evidence has provenance, unverified baselines stay marked, and validation can downgrade unsupported prose.
- **Inspectable:** every node contributes a timestamped trace; source IDs, comparison rows, and validation results are available to the UI.
- **Testable:** provider seams accept injected search, embedding, selector, and model dependencies; domain comparison/date logic can be tested without live web calls.
- **Simple to operate:** Node, Fastify, LangGraph, SQLite, and provider adapters are enough for the local challenge. There is no vector database or background worker to deploy.

## Considered and rejected

- **Transcript-only memory:** rejected because it cannot reliably answer what was known on a date or preserve a claim's provenance.
- **Let the LLM compare prices:** rejected because `$20/user/month` and `$20/workspace/month` are not the same fact. Comparison belongs in typed code.
- **An unrestricted autonomous search loop:** rejected because latency, query count, and failure behavior become unpredictable. Search is planned, parallel, cached, and capped.
- **Many agent tools:** rejected because tool choice expands the failure surface. The agent gets one public-web capability; memory, persistence, and comparison stay application-controlled.
- **A vector database:** rejected for this single-user local workload. SQLite FTS5 plus in-process cosine similarity keeps the system inspectable and dependency-light.
- **Treating the newest source as truth:** rejected because source authority, publication date, plan, unit, and region matter; conflicting evidence must remain visible.
- **A second generative repair call:** rejected because a safety repair should reduce claims, not invent new ones.

## Evaluation and development workflow

The repository includes deterministic tests for date parsing, pricing comparison, source normalization, cache behavior, retrieval, and graph boundaries. `evals/run.ts` creates a scratch database, seeds synthetic sessions, runs memory-only and live-search scenarios, checks hard assertions, records latency and first-token time, and optionally asks a judge model for question-fit, concision, and honesty scores. The scenario set covers dated recall, shortlists, follow-ups, missing history, no-fabrication, source citation, rechecks, and unverified baselines.

The normal workflow is:

```bash
npm install
cp .env.example .env
# add OPENROUTER_API_KEY, TAVILY_API_KEY, and GOOGLE_API_KEY
npm run dev
npm run typecheck
npm test
npm run eval -- --offline
```

Provider configuration, retry counts, search caps, TTLs, timezone, database path, and port are centralized in `src/config.ts`. The model layer (`src/llm/`) validates structured responses and retries provider failures; the search layer (`src/search/`) owns the tool contract, Tavily adapter, and cache.

## Known limits

This is a local, single-user agent with no authentication, collaboration, scheduler, or source snapshot archive. Tavily cannot see logged-in or paywalled pages. Extraction is lossy, and a source may not expose enough context to compare two plans safely. The current turn stream has no user-facing cancellation control, and provider stalls can still dominate latency. Text attachments are bounded browser-side context, not durable uploads. Demo history is plausible synthetic data, not a record of real conversations.

## Another week

1. Add source snapshots and quote spans so a historical answer can be checked against the exact page state from that date.
2. Add cancellation and a hard end-to-end deadline, while preserving partial evidence and a clear timeout state.
3. Improve extraction with field-level confidence and human-reviewed fixtures for prices, limits, and plan changes.
4. Add scheduled rechecks and notifications built on the existing append-only claim chain.
5. Expand the eval set with adversarial date language, conflicting sources, ambiguous entities, and long multi-turn conversations.

The goal would remain the same: answer quickly when memory is enough, search only when freshness matters, and never make fluent uncertainty look like verified history.
