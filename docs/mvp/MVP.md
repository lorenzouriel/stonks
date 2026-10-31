# The Stocks — MVP Definition

> Source documents: [PRD.md](../PRD.md), [sources.yaml](../sources.yaml).
> Backlog: [BACKLOG.md](BACKLOG.md).

## 1. MVP statement

> **Given a Brazilian ticker and a time window, the-stocks detects significant market moves and produces an evidence-backed explanation — observed facts, candidate drivers, cited evidence and a confidence level — computed from reproducible lakehouse data and exposed through an API.**

This is the PRD's killer feature (§12, "Explain market movements using data") delivered as a thin vertical slice through every layer:

```text
Sources → Bronze → Silver → Gold → Events → Semantic Layer → Agent → API
```

## 2. Why a vertical slice instead of V0 → V5

The PRD roadmap (§17) adds one technology layer per milestone, so nothing demonstrable exists until V4. The MVP instead takes a narrow slice of *each* layer, keeping the PRD's principle intact:

> **Agents reason. The semantic layer defines. The data platform provides evidence.**

Each layer is built "thin but real": production-shaped interfaces, small scope. Post-MVP work widens the slice (more sources, Trino, Flink, Cube, replay UI) without redesigning it.

## 3. Scope

### In scope

| Area | MVP |
|---|---|
| Market | Brazil (B3-listed equities) |
| Universe | ~20 liquid Ibovespa securities + IBOV benchmark |
| Ingestion | Daily batch + minimal streaming (Redpanda) |
| Lakehouse | MinIO object storage, Bronze/Silver/Gold, DuckDB + dbt |
| Orchestration | Dagster |
| Events | Rule-based anomaly detection (stream + batch), CVM material-fact events |
| Semantic layer | Custom YAML ontology + metric registry + SQL compiler |
| Agent | Single tool-calling research agent (Claude), deterministic investigation core |
| API | FastAPI: companies, metrics, events, anomalies, research |

### Out of scope (post-MVP)

Flink, Trino, Cube, Superset/Metabase, OpenTelemetry, historical-replay UI, alerts/notifications, multi-agent workflows, portfolio analysis, valuation metrics needing share counts (market cap, P/E, P/B), US market, multiple providers per domain, OpenFIGI/GLEIF resolution.

## 4. Universe

Seeded by hand in the security master. Chosen to exercise real modelling problems:

- **One company, several securities**: `PETR3` / `PETR4` → `br_petrobras`.
- **Commodity exposure**: `PETR4`, `PRIO3`, `VALE3`, `GGBR4`, `SUZB3` (Brent, USD/BRL).
- **Rate-sensitive financials**: `ITUB4`, `BBDC4`, `BBAS3`, `B3SA3`, `BPAC11` (Selic).
- **Domestic / other**: `ABEV3`, `WEGE3`, `ITSA4`, `ELET3`, `RENT3`, `EQTL3`, `RADL3`, `VIVT3`, `PETR3`.
- **Benchmark**: `IBOV`.

Final list is confirmed in story S2.1.

## 5. Sources

| Domain | Source | Mode | MVP role | Risk |
|---|---|---|---|---|
| Prices (daily) | Decided in spike S1.2 (Twelve Data / B3 COTAHIST / Yahoo dev-only) | Batch | OHLCV for universe + IBOV | B3 licensing; free-tier BVMF coverage unverified |
| Prices (stream) | Replay producer; live adapter if S1.2 finds one | Stream | `market.price.updated` | No free live B3 feed known |
| Company registry | CVM `cad_cia_aberta` | Batch | Company master (CNPJ, CVM code, sector) | Low |
| Financial statements | CVM DFP / ITR | Batch | Revenue, net income, growth | Schema drift between years |
| Material facts | CVM IPE | Batch | `company.filing.published` events, evidence | Document text is PDF |
| Macro | BCB SGS (USD/BRL, Selic, IPCA) | Batch | Macro context | Low |
| Commodities | EIA (Brent daily spot) | Batch | Oil context for energy names | Needs API key |
| News | GDELT DOC API | Micro-batch | News evidence, PT + EN | Noisy entity matching |

Deviations from `sources.yaml`:

- **GDELT and EIA are promoted from tier 1 into the MVP**: the explanation feature cannot work without news, and oil is the PRD's own example driver.
- **SEC EDGAR and FRED (tier 0) are deferred**: the MVP is Brazil-only.
- **OpenFIGI/GLEIF are replaced by a curated seed security master** for the MVP universe.
- `sources.yaml` is currently wrapped in a Markdown code fence (```` ```yaml ````) and is not valid YAML; fixed in S0.3.

## 6. Architecture

```text
                    ┌──────────────────────────── SOURCES ─────────────────────────────┐
                    │ Price provider   CVM (cad, DFP/ITR, IPE)   BCB SGS   EIA   GDELT │
                    └──────┬──────────────────────┬────────────────────────────────────┘
                           │ Provider adapters    │
          stream_quotes()  │                      │  get_historical_prices(), fetch()
        (replay | live)    ▼                      ▼
                   ┌──────────────┐        ┌────────────────────────────────────┐
                   │  Redpanda    │        │  Dagster batch assets              │
                   │ market.price │        └───────────────┬────────────────────┘
                   │  .updated    │                        │
                   └──┬───────┬───┘                        ▼
                      │       │ sink           ┌──────────────────────────┐
                      │       └───────────────►│ BRONZE (MinIO) raw + meta│
                      ▼                        └────────────┬─────────────┘
           ┌────────────────────┐                           ▼ dbt-duckdb
           │ Stream detector    │              ┌──────────────────────────┐
           │ (shared rule lib)  │              │ SILVER canonical,        │
           └─────────┬──────────┘              │ bitemporal, security mst │
                     │                         └────────────┬─────────────┘
                     ▼                                      ▼
           market.anomaly.detected            ┌──────────────────────────┐
                     │                        │ GOLD metrics + events    │◄── batch detector
                     │                        └────────────┬─────────────┘
                     │                                     ▼
                     │                        ┌──────────────────────────┐
                     │                        │ SEMANTIC LAYER           │
                     │                        │ ontology · metric registry│
                     │                        │ SQL compiler (DuckDB)    │
                     │                        └────────────┬─────────────┘
                     │                                     ▼
                     │                        ┌──────────────────────────┐
                     └──────────────────────► │ RESEARCH AGENT           │
                        auto-investigation    │ deterministic core + LLM │
                                              └────────────┬─────────────┘
                                                           ▼
                                                    FastAPI (REST)
```

## 7. Key design decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | **Every Silver fact is bitemporal**: `event_time` (when it was true in the market) and `known_at` (when it became public / was ingested). | Point-in-time queries prevent look-ahead bias (PRD §13). Cheap now, very expensive to retrofit. For CVM statements `known_at` = filing receipt date, not fiscal period end. |
| D2 | **Provider adapters behind protocols** (`MarketDataProvider`, `QuoteStream`). | PRD §16 licensing constraint; lets the price source change after S1.2 without touching Silver. |
| D3 | **Streaming via replay producer first**, live adapter as stretch. | No free live B3 feed. Replay is deterministic, testable, and is the seed of the historical-replay feature. |
| D4 | **One event envelope** shared by Kafka topics and the `events` table. | Batch and stream detectors produce identical events; future topics need no rework. |
| D5 | **One rule library** used by both stream and batch detectors. | Same thresholds, same semantics, no drift. |
| D6 | **Semantic layer is custom YAML + compiler**, not Cube. | The ontology (entities + relationships + exposures) is the differentiator; Cube can later consume the same registry. |
| D7 | **The agent has no raw-SQL tool.** All data access goes through the semantic layer. | PRD §6: the LLM reasons over governed definitions. |
| D8 | **Investigation core and confidence are deterministic**; the LLM only plans, narrates and cites. | Reproducible, testable confidence; numbers in the narrative are validated against tool outputs. |
| D9 | **Table format for Silver/Gold decided in spike S0.6** (Iceberg vs plain Parquet). | Iceberg is the PRD target, but dbt-duckdb write support must be verified. D1 makes reproducibility independent of the format. |

## 8. Core contracts

### Identifiers

```text
company_id   = br_<slug>          e.g. br_petrobras        (natural key: CNPJ, CVM code)
security_id  = br_<slug>_<class>  e.g. br_petrobras_pn     (natural key: ticker)
aliases      = PETR4 · PETR4.SA · BVMF:PETR4 · Petrobras PN · Petróleo Brasileiro S.A.
```

### Event envelope

```json
{
  "event_id": "uuid",
  "event_type": "market.anomaly.detected",
  "event_version": 1,
  "producer": "market_events.stream_detector",
  "subject": { "company_id": "br_petrobras", "security_id": "br_petrobras_pn" },
  "event_time": "2026-10-30T17:05:00Z",
  "known_at": "2026-10-30T17:05:02Z",
  "payload": { "rule": "PRICE_DROP", "return_1d": -0.068, "volume_ratio": 2.4 },
  "lineage": { "inputs": ["silver.prices@<run_id>"], "rule_version": "1.0" }
}
```

MVP topics: `market.price.updated`, `market.anomaly.detected`, `dlq.market`.
MVP event types: `PRICE_DROP`, `PRICE_SPIKE`, `VOLUME_SPIKE`, `VOLATILITY_SPIKE`, `52_WEEK_HIGH`, `52_WEEK_LOW`, `company.filing.published`.

### Research response

```json
{
  "question": "Why did Petrobras fall this week?",
  "as_of": "2026-10-30T21:00:00Z",
  "subject": { "company_id": "br_petrobras", "security_id": "br_petrobras_pn" },
  "facts": [ { "metric": "return_7d", "value": -0.071, "metric_version": "1.0" } ],
  "drivers": [ { "rank": 1, "description": "...", "evidence_ids": ["ev_1", "ev_3"], "strength": "strong" } ],
  "evidence": [ { "id": "ev_1", "type": "filing", "source": "cvm_ipe", "known_at": "...", "ref": "..." } ],
  "events": [ "event_id" ],
  "confidence": { "level": "medium", "score": 0.62, "rubric_version": "1.0" }
}
```

## 9. MVP success criteria

1. `make up && make backfill` starts the stack and loads ≥3 years of history for the universe from a clean machine.
2. Rebuilding Silver/Gold from Bronze produces byte-identical Gold outputs (hash check in CI).
3. All dbt tests pass; every Silver row resolves to the security master.
4. `make replay` streams historical bars through Redpanda; the stream detector emits the same anomalies the batch detector finds for the same days.
5. `POST /research` answers "Why did <ticker> move <window>?" with ≥1 cited evidence item for detected events, and **100% of numbers in the response trace to a semantic-layer metric call**.
6. Every response honours `as_of`: no evidence with `known_at > as_of`.
7. On the labelled eval set (S7.6): citation validity 100%, zero unsupported numeric claims, driver recall reported.

## 10. Milestones

| Milestone | Epics | Demo |
|---|---|---|
| M1 — Data lands | E0, E1 (core) | Raw prices, CVM registry and BCB series in Bronze via Dagster |
| M2 — Facts | E1 (rest), E2, E3 | Canonical Silver + `security_performance` in Gold, tests green |
| M3 — Events | E4, E5 | Replay through Redpanda, anomalies detected in stream and batch |
| M4 — Meaning | E6 | Metrics queried by name with lineage |
| M5 — Explanation | E7, E8 | `POST /research` and auto-generated investigation reports |

Critical path: `E0 → E1 → E2 → E3 → E5 (batch) → E6 → E7 → E8`.
E4 can start once E0 and price ingestion (S1.3) exist. E6 can start in parallel with E4/E5 once E3 lands.

## 11. Risks and open questions

| Risk | Impact | Mitigation |
|---|---|---|
| No licensed/free B3 price source | Blocks E1 prices | Spike S1.2 first; Yahoo dev-only adapter behind D2; nothing redistributed |
| GDELT entity matching is noisy | Weak news evidence | Alias-based matching + relevance threshold; CVM IPE is the authoritative corporate signal |
| CVM schema drift across years | Silver statement gaps | Versioned column mappings + tests per year |
| Iceberg write path immature in dbt-duckdb | Format churn | Spike S0.6, D1 keeps reproducibility format-independent |
| LLM hallucinated numbers | Credibility | D7 + D8 + numeric-claim validator (S7.5) |
| Scope creep toward the full PRD | MVP never ships | Out-of-scope list above; stretch stories are marked |

Open questions:

1. Which price source passes S1.2 (affects whether live streaming is possible)?
2. Iceberg or Parquet for Silver/Gold (S0.6)?
3. History depth: 3 years assumed; 5 if the source permits.
