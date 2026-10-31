# The Stocks — MVP Backlog

> Scope and decisions: [MVP.md](MVP.md). Decision references (D1–D9) point there.

**Conventions**

- IDs: `E<n>` epic · `S<n>.<m>` story · checkbox items are subtasks.
- Size: **S** ≤ 1 day · **M** 2–3 days · **L** 4–5 days.
- **Stretch** stories are not required for the MVP to be done.
- Every story implicitly includes: tests, docs updated, CI green.

## Overview

| Epic | Name | Stories | Milestone | Depends on |
|---|---|---|---|---|
| E0 | Platform foundations | 6 | M1 | — |
| E1 | Batch ingestion to Bronze | 10 | M1–M2 | E0 |
| E2 | Security master & canonical Silver | 8 | M2 | E1 |
| E3 | Gold financial metrics | 4 | M2 | E2 |
| E4 | Event streaming | 6 | M3 | E0, S1.3 |
| E5 | Market event detection | 5 | M3 | E3, E4 |
| E6 | Financial semantic layer | 5 | M4 | E3 |
| E7 | Research agent | 7 | M5 | E5, E6 |
| E8 | Intelligence API & demo | 6 | M5 | E6, E7 |

---

## E0 — Platform foundations

**Goal:** one command brings up a reproducible local platform.
**Done when:** `make up` runs MinIO, Redpanda and Dagster with health checks green, and CI runs lint + tests.

### S0.1 Repository skeleton and tooling — S
As a developer, I want a standard project layout so every component lands in a predictable place.

AC: `uv sync` installs; `make lint test` passes on an empty project; layout documented in README.

- [ ] `pyproject.toml` with uv, Python 3.12, ruff, mypy, pytest
- [ ] Layout: `src/stocks/{ingestion,streaming,events,semantic,agent,api}`, `dbt/`, `semantic/`, `infra/`, `tests/`
- [ ] pre-commit hooks (ruff, ruff-format, end-of-file, yaml check)
- [ ] `Makefile` targets: `up`, `down`, `lint`, `test`, `backfill`, `replay`
- [ ] README with quickstart stub

### S0.2 Local stack with docker-compose — M
As a developer, I want the infrastructure running locally so pipelines run end to end on my machine.

AC: `make up` starts all services; buckets `bronze`, `silver`, `gold` exist; Redpanda console and Dagster UI reachable.

- [ ] MinIO service + init job creating buckets
- [ ] Redpanda single node + Redpanda Console
- [ ] Dagster webserver + daemon (SQLite/Postgres instance storage)
- [ ] Health checks and `depends_on` conditions
- [ ] `.env.example` with all ports and credentials

### S0.3 Source registry loader — S
As a pipeline, I want to read `sources.yaml` as typed config so adapters are driven by the registry.

AC: registry parses and validates; disabled sources cannot be ingested; invalid registry fails CI.

- [ ] Remove the Markdown code fence wrapping `docs/sources.yaml` (move to `config/sources.yaml`)
- [ ] Pydantic models for source, defaults, tiers, authority
- [ ] `get_source(name)` honouring `enabled` and `defaults.ingestion`
- [ ] Add `mvp: true` flag (or MVP tier) for MVP sources
- [ ] Validation test in CI

### S0.4 Settings and secrets — S
AC: all config comes from env via pydantic-settings; no secret committed; missing API keys fail fast with clear errors.

- [ ] `Settings` class (MinIO, Redpanda, API keys: price provider, EIA, Anthropic)
- [ ] `.env` gitignored; `.env.example` documented
- [ ] Startup validation per component

### S0.5 Continuous integration — S
AC: GitHub Actions runs on every PR: lint, type-check, unit tests, `dbt parse`, registry validation.

- [ ] Workflow `ci.yml` with uv cache
- [ ] `dbt parse` job (after S2.3)
- [ ] Status badge in README

### S0.6 Spike: table format for Silver/Gold — S (timeboxed)
As the platform owner, I want an informed choice between Iceberg and plain Parquet so we don't churn later.

AC: ADR `docs/adr/0001-table-format.md` with decision, evidence and fallback (D9).

- [ ] Test dbt-duckdb writing Iceberg to MinIO (DuckDB iceberg ext / PyIceberg + SQL catalog)
- [ ] Test time-travel read
- [ ] Compare against external Parquet materializations
- [ ] Write ADR

---

## E1 — Batch ingestion to Bronze

**Goal:** every MVP source lands raw, immutable and traceable in Bronze.
**Done when:** Dagster backfills ≥3 years for all MVP sources, each object has a metadata sidecar, and reruns are idempotent.

### S1.1 Provider adapter framework and Bronze writer — M
As a pipeline, I want a common adapter contract and writer so all sources land the same way (D2).

AC: `MarketDataProvider` and `QuoteStream` protocols defined; Bronze writer stores payload + metadata; retries/timeouts come from registry defaults.

- [ ] Protocols: `get_historical_prices`, `get_quotes`, `stream_quotes`; generic `SourceFetcher`
- [ ] Bronze path convention `bronze/{source}/{dataset}/ingest_date=YYYY-MM-DD/{run_id}.{ext}`
- [ ] Metadata sidecar: source, dataset, request params, `ingested_at`, HTTP status, checksum, record count
- [ ] HTTP client with retries, backoff, timeout, rate limiting
- [ ] Idempotency: skip when checksum already present
- [ ] Unit tests with recorded HTTP fixtures

### S1.2 Spike: B3 price source — S (timeboxed, do first)
As the platform owner, I want a legally and technically viable price source for B3 so the MVP isn't built on a dead end.

AC: ADR `docs/adr/0002-b3-price-source.md` naming the batch source, the streaming option (or "replay only") and the licensing position.

- [ ] Verify Twelve Data BVMF coverage and WebSocket access on free vs paid tiers
- [ ] Review B3 COTAHIST historical files and terms of use
- [ ] Evaluate Yahoo (`.SA`) as dev-only fallback
- [ ] Check IBOV index availability per option
- [ ] Write ADR; update `sources.yaml`

### S1.3 Daily price ingestion — M
As an analyst, I want daily OHLCV for the universe and IBOV so returns and anomalies can be computed.

AC: ≥3 years backfilled; daily incremental run appends only new days; split/dividend adjustment fields preserved raw.

- [ ] Adapter for the source chosen in S1.2
- [ ] Universe read from security master seed (S2.1)
- [ ] Backfill job (partitioned by month) + daily incremental
- [ ] Record-count and date-coverage checks

### S1.4 CVM company registry — S
AC: `cad_cia_aberta` downloaded daily to Bronze; changes detectable between snapshots.

- [ ] Fetcher for CVM open-data company registry CSV
- [ ] Encoding handling (latin-1 → UTF-8 noted in metadata)
- [ ] Daily Dagster asset

### S1.5 CVM financial statements (DFP/ITR) — M
AC: annual (DFP) and quarterly (ITR) packages for the last 4 years stored raw; universe companies identifiable by CVM code.

- [ ] Fetch yearly zip packages; store zip + extracted CSVs
- [ ] Capture receipt date (`DT_RECEB`) for point-in-time (D1)
- [ ] Track file-schema versions per year
- [ ] Backfill + periodic refresh asset

### S1.6 CVM IPE material facts and announcements — M
AC: IPE index for universe companies stored; document metadata (category, subject, delivery date, link) available; PDFs fetched for universe only.

- [ ] Fetch IPE yearly index files
- [ ] Filter to universe CVM codes
- [ ] Download referenced documents (rate-limited)
- [ ] Optional text extraction stored alongside PDF

### S1.7 BCB SGS macro series — S
AC: USD/BRL (PTAX), Selic and IPCA series stored with full history; series codes configured, not hard-coded.

- [ ] Generic SGS fetcher by series code
- [ ] Series config (confirm codes against SGS)
- [ ] Daily asset

### S1.8 EIA Brent spot — S
AC: Brent daily spot price history in Bronze.

- [ ] EIA API v2 fetcher with API key
- [ ] Daily asset

### S1.9 GDELT news for the universe — M
AC: per-company news (PT + EN) for the backfill window stored raw with query parameters.

- [ ] GDELT DOC API fetcher; queries built from company aliases (S2.1)
- [ ] Micro-batch schedule (e.g. every 30 min) + backfill
- [ ] Dedupe by URL at Bronze metadata level

### S1.10 Dagster ingestion assets and schedules — M
AC: all ingestion is Dagster assets with partitions, schedules and visible run metadata; `make backfill` triggers full backfill.

- [ ] Asset definitions per source/dataset
- [ ] Partitions (daily/monthly) and schedules
- [ ] Asset metadata: record counts, Bronze paths
- [ ] `make backfill` CLI wrapper

---

## E2 — Security master & canonical Silver

**Goal:** raw data becomes canonical, point-in-time, entity-resolved Silver.
**Done when:** every Silver row resolves to `company_id` / `security_id`, carries `event_time` + `known_at`, and dbt tests pass.

### S2.1 Seed security master — S
As the platform, I want a curated master of companies, securities and aliases so every source maps to the same entities.

AC: seeds for ~20 securities + IBOV; PETR3/PETR4 map to one company; sector and macro exposures captured.

- [ ] `companies.csv`: company_id, CNPJ, CVM code, legal name, trade name, sector
- [ ] `securities.csv`: security_id, ticker, class (ON/PN/UNIT), company_id, exchange
- [ ] `aliases.csv`: alias, alias_type, target id
- [ ] `exposures.csv`: company_id → macro indicator (e.g. Brent, USD/BRL, Selic), direction
- [ ] Cross-check CNPJ/CVM code against CVM registry

### S2.2 Entity resolver — M
As any consumer (news, agent, API), I want to resolve free text or tickers to canonical IDs.

AC: all PRD §3 aliases resolve to `br_petrobras` / `br_petrobras_pn`; ambiguous input returns candidates, not a guess.

- [ ] Normalisation: case, accents, suffixes (`.SA`, `BVMF:`), corporate suffixes (S.A.)
- [ ] Exact + fuzzy alias match with score
- [ ] Ambiguity handling (e.g. "Petrobras" → company, not a security)
- [ ] Python API + table-driven tests

### S2.3 dbt project setup — S
AC: dbt-duckdb project reads Bronze from MinIO via httpfs and writes Silver/Gold in the format chosen in S0.6.

- [ ] dbt project, profiles, sources over Bronze
- [ ] Macros for bitemporal columns and audit columns (`_run_id`, `_source`)
- [ ] Dagster ↔ dbt integration (dagster-dbt)

### S2.4 Silver prices — M
AC: one row per security per trading day; deduped; unadjusted + adjusted close; `event_time` = session date, `known_at` = ingested_at.

- [ ] Staging model per source → canonical `silver.prices`
- [ ] B3 trading calendar handling (holidays, missing days flagged)
- [ ] Corporate-action adjustment strategy documented
- [ ] Join to security master

### S2.5 Silver companies and financial statements — L
AC: canonical `silver.companies` and long-format `silver.financial_statements` (company, period, statement, line item, value, `known_at` = receipt date); restatements kept as versions.

- [ ] Company model from CVM registry + seed
- [ ] DFP/ITR staging with per-year column mappings
- [ ] Canonical line-item mapping (revenue, net income, equity…)
- [ ] Restatement versioning (latest-known-as-of)

### S2.6 Silver filings and news — M
AC: `silver.filings` (IPE) and `silver.news` linked to companies; news below relevance threshold excluded.

- [ ] IPE model: category, subject, `event_time` (reference date), `known_at` (delivery date), document ref
- [ ] News model: url, title, language, published_at, source, matched company, match score
- [ ] Entity linking via resolver (S2.2)

### S2.7 Silver macro indicators — S
AC: long-format `silver.macro_indicators` (indicator_id, date, value, unit, `known_at`) for USD/BRL, Selic, IPCA, Brent.

- [ ] Staging per source → canonical model
- [ ] Indicator catalogue seed (id, name, unit, frequency, source)

### S2.8 Silver data quality — M
AC: dbt tests cover uniqueness, not-null, referential integrity, freshness and domain rules; failures block Gold builds in Dagster.

- [ ] Generic tests on keys and FKs to security master
- [ ] Price sanity: high ≥ low, prices > 0, volume ≥ 0
- [ ] Freshness tests per source
- [ ] `known_at ≥ event_time` where applicable
- [ ] Dagster asset checks wired to dbt tests

---

## E3 — Gold financial metrics

**Goal:** reusable, reproducible analytics tables.
**Done when:** Gold rebuilds identically from Bronze and metric values match hand-computed fixtures.

### S3.1 `gold.security_performance` — M
AC: per security per day: close, return_1d/7d/30d/ytd, volatility_30d, volume_ratio (vs 30d avg), drawdown, rolling 52w high/low, return relative to IBOV.

- [ ] Window-function models in dbt
- [ ] IBOV-relative (abnormal) return
- [ ] Sector-relative return (peer average from security master)
- [ ] Fixture tests with hand-computed values

### S3.2 `gold.company_fundamentals` — M
AC: point-in-time revenue and net income TTM, revenue_growth and earnings_growth YoY, each row valid from its `known_at`.

- [ ] TTM aggregation from quarterly statements
- [ ] YoY growth metrics
- [ ] As-of join helper (latest known value at date X)
- [ ] Tests for look-ahead: values never visible before receipt date

### S3.3 `gold.market_snapshot` — S
AC: latest state per security plus IBOV, USD/BRL, Selic, Brent with their dates.

- [ ] Snapshot model
- [ ] Macro daily changes (USD/BRL, Brent)

### S3.4 Reproducibility check — S
AC: CI job rebuilds Gold from a fixed Bronze fixture and compares hashes; mismatch fails the build.

- [ ] Small committed Bronze fixture
- [ ] Deterministic ordering in models
- [ ] Hash-compare script + CI job

---

## E4 — Event streaming

**Goal:** a minimal, production-shaped event backbone (D3, D4).
**Done when:** `make replay` publishes historical bars to Redpanda, they are sunk to Bronze, and Silver merges batch and streamed prices.

### S4.1 Event envelope and contracts — S
AC: versioned Pydantic/JSON Schema envelope (MVP.md §8); topic naming convention documented; schemas tested.

- [ ] Envelope model + payload models per event type
- [ ] JSON Schema export to `contracts/`
- [ ] Compatibility rule (additive changes only within a version)

### S4.2 Topic provisioning — S
AC: topics `market.price.updated`, `market.anomaly.detected`, `dlq.market` created idempotently on `make up`, with retention set.

- [ ] Provisioning script (rpk or admin client)
- [ ] Partition key = `security_id`

### S4.3 Replay producer — M
As a developer, I want to stream historical bars as if live so streaming is testable without a live feed.

AC: replays a date range at configurable speed preserving original `event_time`; output is deterministic for the same inputs.

- [ ] `ReplayStream` implementing `QuoteStream` over Silver/Bronze bars
- [ ] Speed factor and date-range CLI (`make replay FROM= TO=`)
- [ ] Ordering by event_time per partition

### S4.4 Live quote producer — M (Stretch)
AC: if S1.2 identifies a streaming-capable source, a live `QuoteStream` publishes to the same topic with the same envelope.

- [ ] WebSocket adapter
- [ ] Reconnect/backoff, heartbeat
- [ ] Switch replay/live by config

### S4.5 Stream-to-Bronze sink — M
AC: consumer writes micro-batched Parquet to `bronze/stream/...`; idempotent by `event_id`; offsets committed after durable write; poison messages go to DLQ.

- [ ] Consumer with batching by size/time
- [ ] Exactly-once-effect via event_id dedupe
- [ ] DLQ routing + metrics log

### S4.6 Batch/stream convergence in Silver — S
AC: `silver.prices` unions batch and streamed bars with documented precedence (batch close supersedes stream for the same session).

- [ ] Staging model for streamed bars
- [ ] Precedence + dedupe logic and tests

---

## E5 — Market event detection

**Goal:** significant moves and corporate events become first-class, reusable events (PRD §9).
**Done when:** stream and batch detectors emit identical anomalies for the same replayed days, and all events land in `gold.market_events` with lineage.

### S5.1 Shared rule library — M
AC: rules (`PRICE_DROP`, `PRICE_SPIKE`, `VOLUME_SPIKE`, `VOLATILITY_SPIKE`, `52_WEEK_HIGH`, `52_WEEK_LOW`) are pure functions with thresholds in versioned config (D5).

- [ ] Rule interface: inputs → optional event payload
- [ ] Threshold config with `rule_version`
- [ ] Unit tests per rule including edge cases

### S5.2 Streaming detector — M
AC: consumes `market.price.updated`, maintains per-security state (prior close, 30d avg volume seeded from Gold), emits `market.anomaly.detected` with cooldown to avoid duplicates.

- [ ] Consumer + state store (in-memory, seeded on start)
- [ ] Cooldown/dedupe per security + rule + session
- [ ] Integration test with replay producer

### S5.3 Batch detector — S
AC: daily Dagster asset applies the same rules over `gold.security_performance`.

- [ ] Python asset calling rule library
- [ ] Parity test vs streaming detector on a fixture window

### S5.4 Corporate filing events — S
AC: new IPE filings for universe companies produce `company.filing.published` events.

- [ ] Event builder from `silver.filings`
- [ ] Category mapping (material fact, earnings release, dividend notice…)

### S5.5 Unified `gold.market_events` — S
AC: one table for all events (stream sink + batch), envelope fields as columns, lineage preserved, queryable by subject and time.

- [ ] dbt model unioning sources of events
- [ ] Tests: unique event_id, valid subject FK, valid event_type

---

## E6 — Financial semantic layer

**Goal:** governed meaning — entities, relationships and metrics — that every consumer, especially the agent, must go through (D6, D7).
**Done when:** any MVP metric can be queried by name for any entity and time range, returning values plus lineage.

### S6.1 Ontology specification — M
AC: `semantic/entities.yaml` defines Company, Security, Exchange, Sector, Filing, NewsArticle, MacroIndicator, MarketEvent with keys, table bindings and relationships (issues, belongs_to, trades_on, publishes, affected_by).

- [ ] YAML schema + Pydantic loader with validation
- [ ] Bindings to Silver/Gold tables and keys
- [ ] `affected_by` relationship driven by `exposures.csv`

### S6.2 Metric registry — M
AC: `semantic/metrics.yaml` defines each MVP metric (name, description, entity, grain, expression, unit, depends_on, version); registry validated against dbt models.

- [ ] Metrics: return_1d/7d/30d/ytd, volatility_30d, volume_ratio, drawdown, abnormal_return, revenue_growth, earnings_growth, macro change metrics
- [ ] Validation: referenced columns exist in dbt manifest
- [ ] Versioning rules

### S6.3 Query compiler — L
AC: `query(metrics, entity filters, time range, grain, as_of)` → validated SQL → DuckDB result with lineage (SQL, metric versions, tables); unknown metrics rejected; `as_of` enforced via `known_at`.

- [ ] Request model and validation
- [ ] SQL generation (templated, parameterised; no string injection)
- [ ] Point-in-time filter on `known_at`
- [ ] Result + lineage object
- [ ] Golden-SQL tests

### S6.4 Relationship traversal helpers — S
AC: functions for company → securities, security → sector peers, company → macro exposures, entity → events in window.

- [ ] Traversal API over ontology
- [ ] Tests on PETR3/PETR4 and sector peers

### S6.5 Generated semantic docs — S
AC: Markdown catalogue of entities and metrics generated from YAML in CI.

- [ ] Generator script
- [ ] Docs published to `docs/semantic/`

---

## E7 — Research agent

**Goal:** explain market moves with traceable evidence (PRD §8, §12), deterministic where it matters (D8).
**Done when:** the agent answers movement questions with validated numbers and citations, meets the eval thresholds, and auto-investigates detected anomalies.

### S7.1 Agent tool layer — M
AC: typed tools over the semantic layer only: `find_company`, `query_metric`, `get_market_events`, `search_filings`, `search_news`, `get_macro_context`, `compare_to_benchmark`; every tool accepts `as_of`.

- [ ] Tool schemas (JSON Schema) from Pydantic models
- [ ] Implementations calling E6 APIs
- [ ] Each result carries evidence ids + lineage

### S7.2 Deterministic investigation pipeline — L
As an analyst, I want a reproducible investigation of a move so the explanation doesn't depend on LLM randomness.

AC: `investigate(subject, window, as_of)` returns an `Investigation`: the move (absolute, vs IBOV, vs sector), market-wide vs idiosyncratic classification, and candidate drivers with evidence from filings, news and macro exposures in the window.

- [ ] Move decomposition (abnormal return vs IBOV and peers)
- [ ] Evidence gathering per driver type with time-window rules
- [ ] Driver candidates ranked by evidence strength
- [ ] Unit tests on fixture episodes

### S7.3 Confidence rubric — S
AC: confidence score/level computed from evidence (authoritative filing in window, aligned macro move on known exposure, corroborating news count, share of move unexplained by market) with `rubric_version`.

- [ ] Rubric spec in `docs/`
- [ ] Implementation + tests

### S7.4 LLM agent orchestration — M
AC: Claude tool-use loop (model configurable) that interprets the question, calls tools / `investigate`, and writes the narrative in the research response schema, citing evidence ids.

- [ ] System prompt: cite evidence ids, never state numbers not returned by tools, say "insufficient evidence" when applicable
- [ ] Structured output into the response schema
- [ ] Max-steps / timeout / cost guard
- [ ] Prompt and model versions recorded in the response

### S7.5 Guardrails and validation — M
AC: every number and evidence id in the final response is checked against tool outputs; failures trigger one repair attempt, then a degraded deterministic response; no evidence with `known_at > as_of`.

- [ ] Numeric-claim extractor + matcher
- [ ] Citation validator
- [ ] Point-in-time validator
- [ ] Fallback to Investigation-only response

### S7.6 Evaluation set — M
AC: 10–15 labelled historical episodes (taken from detected anomalies, causes labelled by hand); `make eval` reports citation validity, unsupported-number rate and driver recall.

- [ ] Episode selection from `gold.market_events`
- [ ] Labelling format and labels
- [ ] Eval runner + report

### S7.7 Event-triggered investigations — M
AC: consumer of `market.anomaly.detected` runs the investigation (and narrative) and stores the report in `gold.investigation_reports` linked to the event.

- [ ] Consumer with concurrency limit
- [ ] Report persistence + idempotency per event_id
- [ ] Replay demo: anomaly → report

---

## E8 — Intelligence API & demo

**Goal:** expose the platform as a Financial Intelligence API (PRD §15).
**Done when:** all MVP endpoints are documented in OpenAPI, tested, and the end-to-end demo runs from the README.

### S8.1 API skeleton — S
AC: FastAPI app with settings, health/readiness, error model, OpenAPI docs; runs in docker-compose.

- [ ] App factory, routers, dependency injection for semantic layer
- [ ] `/health`, `/ready`
- [ ] Problem+JSON error responses

### S8.2 Company endpoints — M
AC: `GET /companies/{id}`, `/companies/{id}/metrics`, `/companies/{id}/events`; `{id}` accepts canonical id, ticker or alias via resolver; metrics endpoint goes through the query compiler; `as_of` supported.

- [ ] Routes + response schemas
- [ ] Resolver integration and ambiguity response
- [ ] API tests

### S8.3 Market endpoints — S
AC: `GET /market/anomalies` (filter by date, rule, sector) and `GET /market/movers` (top/bottom by return window).

- [ ] Routes over `gold.market_events` and semantic metrics
- [ ] Pagination
- [ ] API tests

### S8.4 Research endpoint — M
AC: `POST /research {question, as_of?}` returns the research response schema (MVP.md §8) within a timeout; validation failures return the degraded deterministic response.

- [ ] Route + request/response schemas
- [ ] Timeout and error handling
- [ ] Contract tests against schema

### S8.5 Investigation reports endpoint — S (Stretch)
AC: `GET /investigations` and `GET /investigations/{id}` list and fetch auto-generated reports.

- [ ] Routes over `gold.investigation_reports`

### S8.6 End-to-end demo — S
AC: README walkthrough from clean clone: `make up` → `make backfill` → `make replay` → curl `/market/anomalies` → `POST /research`; recorded sample outputs committed.

- [ ] Demo script
- [ ] README walkthrough + architecture diagram
- [ ] Sample responses in `docs/examples/`

---

## Suggested build order

1. **Spikes first:** S1.2 (price source), S0.6 (table format) — they can invalidate assumptions.
2. **M1:** S0.1–S0.5 → S1.1 → S2.1 → S1.3, S1.4, S1.7 → S1.10
3. **M2:** S2.3 → S2.4, S2.7 → S3.1, S3.3 → S1.5/S2.5 → S3.2 → S1.6, S1.8, S1.9 → S2.6 → S2.8 → S3.4
4. **M3:** S4.1, S4.2 → S4.3 → S4.5 → S4.6 · S5.1 → S5.3 → S5.2 → S5.4 → S5.5
5. **M4:** S6.1 → S6.2 → S6.3 → S6.4 → S6.5
6. **M5:** S7.1 → S7.2 → S7.3 → S7.4 → S7.5 → S7.6 → S8.1 → S8.2–S8.4 → S7.7 → S8.6
