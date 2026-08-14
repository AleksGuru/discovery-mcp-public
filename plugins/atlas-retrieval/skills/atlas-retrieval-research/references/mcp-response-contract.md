# Atlas Retrieval MCP response contract

This reference documents the stable public fields needed by the research workflow. The production
path uses LanceDB for candidate retrieval and Evidence/PostgreSQL for current facts.

## Shared `ToolResult` envelope

Seven tools return this envelope; `prepare_search_query` returns a plain object.

| Field | Meaning |
| --- | --- |
| `result_type` | Stable discriminator for the tool-specific `data` shape. |
| `data` | Tool-specific object documented below. |
| `citations` | Resolved current Evidence citations supporting returned facts. |
| `degraded_modes` | Unavailable or disabled retrieval stages. Empty means no reported degradation. |
| `warnings` | Coverage, freshness or comparability cautions that must be preserved. |
| `snapshot_id` | Active immutable Lance generation identity. It is not the current-facts timestamp. |
| `configuration` | Active server configuration identity. |
| `stage_latency_ms` | Diagnostic `{stage, elapsed_ms}` timings; never relevance evidence. |

Evidence-backed responses put `fact_source=evidence_http` and `facts_as_of` inside `data`. The
service fails closed when Evidence cannot supply current facts; it does not return old Lance facts.

### `Citation`

| Field | Meaning |
| --- | --- |
| `citation_id` | Stable Atlas identifier accepted by `fetch_evidence`. |
| `entity_id` | Canonical Atlas entity ID. |
| `card_version` | Current card version supported by this citation. |
| `kind` | `card`, `source` or `metric`. |
| `locator` | Current card field or metric-observation location. |
| `source_key`, `source_url` | Recorded source identity and URL; either may be `null`. |
| `observed_at` | Observation date/time, when known. |
| `data_state` | Observed, zero, missing or another explicit evidence state. |
| `value` | Resolved scalar or JSON payload supported by the citation. |

### `ClaimEvidence`

`claim_id`, `field_path`, `value`, `data_state` and `citation_ids` bind a returned field to one or
more citations. Use all required IDs for a material claim.

## `prepare_search_query`

Call this once for each exact text passed to `search_projects`.

| Field | Meaning |
| --- | --- |
| `required` | Exact query preparation is required. |
| `state` | Combined `ready`, `pending`, `failed` or offline `missing` state. |
| `embedding_state`, `decomposition_state` | Durable OpenAI embedding and query-plan states. |
| `dense_kind` | Dense model family, currently `openai`. |
| `source` | `frozen_artifact`, `dynamic_cache`, `document_cache` or `null`. |
| `attempts`, `error_code`, `retry_after_ms` | Durable preparation diagnostics. |
| `provider_calls_in_request_path` | Always `false`; provider work runs in a separate worker. |

Search only when both components and the combined state are ready.

## Search structures

### `SearchHit`

Final search and similarity items contain:

- identity: `entity_id`, `project_name`, current `card_version`, `entity_grain`, `status`;
- current Evidence facts: `summary`, classifications, `platforms`, `metrics`, `momentum`,
  `claim_evidence`, `citation_ids`;
- retrieval diagnostics: `score`, `reranker_score`, `dense_rank`, `lexical_rank`, `graph_rank`,
  `reasons`, optional relation fields;
- freshness trace: `indexed_card_version`, `current_card_version`, `index_stale`.

`reranker_score` is relevance, not project quality. Scores compare only inside one result set.
`index_stale=true` means Lance selected the entity using older indexed text; the returned card and
metrics still come from the newer Evidence version.

### `MetricValue`

Each metric has `key`, `value`, `state`, `unit`, `observed_at` and `citation_ids`. Never infer a
definition from the number alone.

### `momentum`

The typed object may contain:

- `website_monthly_visits_growth_ratio`;
- `github_growth_61d`, `github_growth_6m`;
- `app_installs_weekly`, `app_installs_monthly` with `signal_kind=velocity`;
- `app_rating_count_weekly` with `signal_kind=weekly_delta`;
- `app_best_rank_ever`, `app_ever_top_10`, `app_first_top_10_at`;
- `app_rank_observation_days`, `app_top_10_milestones`;
- `app_store_last_updated`, `app_store_update_age_days`, `app_update_stale_90d`;
- `app_update_signal_kind=maintenance_risk`.

Null means unknown. A lower app rank is better. Preserve every returned top-10 milestone grain.

### `QueryPlan`

The plan contains `lexical_query_ru`, `lexical_query_en`, up to four `subqueries`, proposed
classifications/filters and an `explanation`. It is prepared by OpenAI but is not evidence.
Query-plan filters are disabled by default; use `query_plan_filters_applied` and `applied_filters`
to determine what actually constrained results.

Typed metric filters use `key`, `operator`, `value`, optional `value_high`, states, unit and optional
observation bounds. In production they are checked against current Evidence values, never an old
Lance value or a historical value that merely once satisfied the threshold.

## `search_projects`

`result_type=project_search`; `data` contains:

| Field | Meaning |
| --- | --- |
| `query` | Exact normalized query. |
| `query_plan` | Prepared plan or `null`. |
| `query_plan_filters_applied` | Whether plan-proposed filters were actually applied. |
| `lexical_queries` | Exact plus generated Russian/English FTS variants. |
| `applied_filters` | Explicit current-value filter contract used by search. |
| `metric_sort` | Optional current Evidence metric sort. |
| `items` | Final reranked results, batch-hydrated from Evidence, bounded by `limit`. |
| `candidate_trace` | Larger retrieval-only diagnostic trace before final hydration. |
| `fact_source`, `facts_as_of` | Current-fact source and Evidence read timestamp. |
| `rescue_ran` | Whether a bounded structured Evidence rescue was attempted. |

The pipeline is OpenAI dense + Lance FTS, RRF fusion, optional graph expansion and Nemotron
reranking. Explicit filters are verified in one Evidence batch after Lance selection. If too few
candidates remain, or `metric_sort` is requested, a bounded Evidence rescue may add only entities
already present in the active Lance generation. Missing indexed entities produce an
`unindexed_matches` warning.

Final `items` are current fact records and can support the summary/current-metric table without an
N+1 `get_project` loop. Use `candidate_trace` only to debug retrieval coverage and ranks.

## `get_project`

`result_type=project_detail`; `data` contains:

| Field | Meaning |
| --- | --- |
| `project` | Current Evidence object: identity, `summary`, `detail`, `links`, current `metrics`, bounded `metric_history`, labels, scores and `updated_at`. |
| `classification` | Current `entity_grain`, `domain`, `user_type`, `is_ai`, `is_asian`, `platforms`. |
| `momentum` | Typed current momentum described above. |
| `claim_evidence` | Claim bindings from Evidence. |
| `fact_source`, `facts_as_of` | Current-fact source and read timestamp. |

History is bounded to at most 24 observations per metric. Use `summary.card_ui.primary_link` or an
official item in `links` for user-facing links. Nested Evidence card fields are dynamic JSON; do not
invent absent keys. `scores` are Atlas formulas, not raw source metrics.

## `find_similar_projects`

`result_type=similar_projects`; `data` contains `source_entity_id`, current Evidence-hydrated
`items`, `fact_source` and `facts_as_of`. Candidate similarity uses the stored source embedding and
is non-canonical; it does not prove competition or a graph relationship.

## `rank_trending_projects`

`result_type=trend_ranking`; `data` contains `rankings`, `fact_source` and `facts_as_of`.
Each ranking has:

- `metric_key`: one typed momentum signal;
- `observed`: current Evidence coverage before the minimum gate;
- `items`: current projects with identity, card version and full `momentum` object.

Signals are ranked separately. App install velocity is not growth; update staleness is maintenance
risk. `app_best_rank_ever` sorts ascending because lower is better.

## `analyze_feature` and `build_research_bundle`

`analyze_feature` returns retrieval-derived matches/examples, not a canonical feature label or
causal result. `build_research_bundle` wraps complete search data plus required answer sections and
the citation rule. Their returned project facts follow the same Evidence hydration boundary.

## `fetch_evidence`

`result_type=evidence`; `data` contains `requested`, resolved count, `missing`, `fact_source` and
`facts_as_of`. Citation objects appear in the envelope. Reuse citations already present in prior
tool envelopes and call this tool only for used IDs that remain unresolved.

## Failure behavior

- Evidence timeout, 401 or invalid response yields a sanitized service failure; no Lance-fact
  fallback is allowed.
- Query preparation that is not ready blocks dense search; follow `retry_after_ms` rather than
  issuing new phrasings.
- Preserve every warning and degraded mode. A missing retrieval channel does not validate a fact.
