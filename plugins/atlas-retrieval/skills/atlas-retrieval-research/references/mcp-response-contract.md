# Atlas Retrieval MCP response contract

This reference defines every stable field returned by the eight public tools in the current Atlas
Retrieval implementation. Fields described as dynamic JSON may contain source-specific nested keys;
do not invent or depend on undocumented nested keys.

## Shared `ToolResult` envelope

Seven tools return this envelope; `prepare_search_query` is the exception.

| Field | Meaning |
| --- | --- |
| `result_type` | Stable discriminator for the tool result shape. |
| `data` | Tool-specific object documented below. |
| `citations` | Resolved `Citation[]` supporting returned facts. |
| `degraded_modes` | Retrieval stages that were unavailable or deliberately disabled. Empty means no reported degradation. |
| `warnings` | Evidence, comparability or derived-signal cautions that must be preserved. |
| `snapshot_id` | Immutable Evidence/Lance snapshot identity, or `null` when unavailable. |
| `configuration` | Active server configuration identity. |
| `stage_latency_ms` | Per-stage timings as `{stage, elapsed_ms}`. Timings are diagnostics, not relevance evidence. |

### `Citation`

| Field | Meaning |
| --- | --- |
| `citation_id` | Stable Atlas citation identifier passed to `fetch_evidence`. |
| `entity_id` | Canonical Atlas project/entity identifier. |
| `card_version` | Version of the card supported by the citation. |
| `kind` | `card`, `source` or `metric`. |
| `locator` | Field or observation location inside the versioned projection. |
| `source_key` | Provider/source identity, when known. |
| `source_url` | Original source URL, when recorded; may be `null`. |
| `observed_at` | Observation timestamp/date, when known. |
| `data_state` | Evidence state such as observed, missing or zero. Do not equate missing with zero. |
| `value` | Resolved scalar or JSON payload supported by the citation. |

### `ClaimEvidence`

| Field | Meaning |
| --- | --- |
| `claim_id` | Stable claim identity within the project. |
| `field_path` | Project field supported by this claim. |
| `value` | Claimed scalar or JSON value. |
| `data_state` | State of the claimed value. |
| `citation_ids` | One or more citations required to use the claim. |

## `prepare_search_query`

This tool returns a plain object rather than `ToolResult`.

| Field | Meaning |
| --- | --- |
| `required` | Always `true`; exact query preparation is required. |
| `state` | Combined `ready`, `pending`, `failed` or offline `missing` state. Search only when ready. |
| `embedding_state` | Dense-query job state: `missing`, `pending`, `processing`, `ready` or `failed`; omitted in frozen-only mode. |
| `decomposition_state` | Query-plan job state; omitted in frozen-only mode. |
| `dense_kind` | Dense model family, currently `openai`. |
| `source` | `frozen_artifact`, `dynamic_cache`, `document_cache` or `null`. |
| `attempts` | Durable preparation attempt count; omitted in frozen-only mode. |
| `error_code` | Sanitized preparation error, or `null`. |
| `retry_after_ms` | Suggested polling delay, or `null`. |
| `provider_calls_in_request_path` | Always `false`; provider work occurs in a separate worker. |

## Search structures

### `SearchHit`

Returned in search results, similarity results, feature examples and research bundles.

| Field | Meaning |
| --- | --- |
| `entity_id` | Canonical Atlas entity ID; use for `get_project` and similarity expansion. |
| `project_name` | Display name. |
| `card_version` | Version of the source card. |
| `entity_grain` | Entity grain, for example app, company or project. Compare like grains. |
| `status` | Current Atlas lifecycle/status value. |
| `summary` | Compact text used for discovery; not a complete card. |
| `is_ai` | AI classification, or `null` when not established. |
| `is_asian` | Asian-market classification, or `null`. |
| `domain` | Controlled/derived domain value, or `null`. |
| `user_type` | User/customer type, or `null`. |
| `platforms` | Detected platform names from `github`, `ios`, `android`, `web`. |
| `rerank_text` | Bounded text supplied to reranking. It is retrieval context, not a structured fact source. |
| `score` | Fused retrieval score. Compare only inside the same result set. |
| `reranker_score` | Local Qwen reranker score, or `null`. It is relevance, not quality. |
| `lexical_rank` | Full-text rank, or `null` if absent from that candidate channel. |
| `dense_rank` | Vector rank, or `null`. |
| `graph_rank` | Graph-expansion rank, or `null`. |
| `relation_type` | Graph relation type for an expanded hit, or `null`; derived, not a citation. |
| `relation_basis` | Basis of that graph relation, or `null`. |
| `citation_ids` | Card/metric citations associated with the hit. |
| `metrics` | Hydrated `MetricValue[]`; populated for final search items but may be empty in `candidate_trace`. |
| `reasons` | Retrieval channels contributing to the hit, for example `dense` or `full_text`. |
| `claim_evidence` | Claim-level evidence when populated; search hits may return an empty list. |

### `MetricValue`

| Field | Meaning |
| --- | --- |
| `key` | Metric key; never infer its definition from the number alone. |
| `value` | Number, text, boolean or `null`. |
| `state` | Data state. |
| `unit` | Unit, or `null`. |
| `observed_at` | Observation date/time, or `null`. |
| `citation_ids` | Supporting metric citations. |

### `QueryPlan`

| Field | Meaning |
| --- | --- |
| `lexical_query` | Normalized full-text query used by retrieval. |
| `subqueries` | Up to four decomposition subqueries. |
| `is_ai`, `is_asian` | Proposed boolean filters, or `null`. |
| `platforms` | Proposed platform filters. |
| `user_types` | Proposed user-type filters. |
| `domains` | Proposed domain filters. |
| `metric_filters` | Proposed typed metric filters. |
| `explanation` | Short query-plan rationale. |

Each returned metric filter contains `key`, `operator`, `value`, optional `value_high`, `states`,
optional `unit`, `observed_after` and `observed_before`. Applied filters additionally contain
`entity_grains`, `statuses`, `is_ai`, `is_asian`, `platforms`, `user_types`, `domains` and
`metric_filters`.

## `search_projects`

`result_type=project_search`; `data` contains:

| Field | Meaning |
| --- | --- |
| `query` | Exact normalized user query. |
| `query_plan` | `QueryPlan` or `null`. |
| `applied_filters` | Actual validated filter object; inspect this rather than assuming proposed filters were accepted. |
| `items` | Final reranked and metric-hydrated `SearchHit[]`, limited by the request. |
| `candidate_trace` | Larger fused/reranked `SearchHit[]` before final metric hydration; diagnostic discovery only. |

Search combines dense vector retrieval and FTS, fuses the channels, optionally expands the graph,
then reranks locally. `dense_rank`/`lexical_rank` and `reasons` show which paths contributed.

## `get_project`

`result_type=project_detail`; `data.project` contains:

| Field | Meaning |
| --- | --- |
| `entity_id`, `project_name`, `status`, `card_version`, `entity_grain` | Versioned Atlas identity fields. |
| `updated_at` | Card update timestamp. |
| `snapshot_id` | Snapshot containing this card. |
| `local_card_url` | Public Atlas Platform project URL. Use this as the primary user-facing link. |
| `description` | Card description, or `null`. |
| `classification` | Object with `is_ai`, `is_asian`, `domain`, `user_type`, `entity_grain`, `platforms`. |
| `audience` | Source-projected audience JSON, or `null`; nested keys vary by available evidence. |
| `market` | Source-projected market-size JSON, or `null`; preserve unit, geography, range, confidence and methodology when present. |
| `badges` | Scalar badge labels retained by the Retrieval projection. |
| `tags` | String tags. |
| `links` | Source-projected link records. Common keys are `url` and `link_type`/`type`; other keys are dynamic. Use `local_card_url` as the main link. |
| `features` | Structured card feature strings. Empty means unavailable, not that the product has no features. |
| `funding` | Structured Crunchbase funding object documented below. |
| `aggregate_metrics` | Current and historical metric groups documented below. |
| `scores` | Dynamic Atlas score/formula JSON. Do not treat scores as source evidence or mix them with raw metrics. |
| `claim_evidence` | `ClaimEvidence[]` for card fields and observations. |
| `history_scope` | Overall metric-history scope, normally `evidence_metric_observations`. |
| `history_complete` | Whether the generation has a complete Evidence metric-history projection. |

### `funding`

| Field | Meaning |
| --- | --- |
| `status` | Crunchbase/provider availability status, or `null`. |
| `source_key` | Funding provider identity, or `null`. |
| `profile_url` | Provider profile URL, or `null`. |
| `total_usd` | Provider total funding in USD, or `null`; the canonical metric may also appear in `aggregate_metrics`. |
| `round_count` | Known funding-round count, or `null`. |
| `disclosed_round_count` | Rounds with disclosed amounts, or `null`. |
| `investors` | Investor records. `name` is required for retained records; `round`, `category` and other provider fields are optional. |

### `aggregate_metrics[]`

| Field | Meaning |
| --- | --- |
| `key` | Metric family key. |
| `current` | Latest observation object. |
| `history` | Newest-first compatible observation objects. |
| `history_scope` | `evidence_metric_observations` or `current_snapshot_fallback`. The latter is not a time series. |
| `history_complete` | Whether history is complete for the generation. |

A current/fallback observation contains `value`, `unit`, `data_state`, `observed_at`,
`citation_ids`. A historical observation additionally contains `observation_id`, `value_kind`,
`source_key` and `confidence`. Current MCP excludes unsafe `mau` and `monthly_active_users` aliases;
website visits must never be relabelled as active users.

## `find_similar_projects`

`result_type=similar_projects`; `data` contains `source_entity_id` and `items` (`SearchHit[]`). It
uses the stored source-project embedding directly: no text query, FTS, query preparation or reranker.
The source entity is excluded. Similarity is retrieval-derived and non-canonical.

## `rank_trending_projects`

`result_type=trend_ranking`; `data.rankings[]` contains:

| Field | Meaning |
| --- | --- |
| `metric_key` | Metric ranked independently. |
| `observed` | Number of qualifying rows observed before the minimum-coverage gate. |
| `items` | Descending metric rows, or empty when coverage is below `min_observed`. |

Each item contains `entity_id`, `metric_key`, `value_json`, `numeric_value`, `unit`, `data_state`,
`observed_at`, `citation_id`. Rankings do not mix units/windows and do not prove causation. For app
rank histories, lower rank numbers are better; do not interpret this descending metric tool as a
rank-improvement calculation without inspecting dated history.

## `analyze_feature`

`result_type=feature_analysis`; `data` contains `feature`, `cohort_query`, `success_metric`,
`matching_projects` and `examples` (`SearchHit[]`). This performs retrieval over feature wording; it
does not return a canonical feature label or prove causation. Use `get_project.features` for the
structured feature list of a displayed project.

## `build_research_bundle`

`result_type=research_bundle`; `data` contains `question`, `retrieval` (the complete
`search_projects.data` object), and `answer_guidance`. `answer_guidance.required_sections` lists the
expected sections and `answer_guidance.claim_rule` requires citation IDs for factual/numeric claims.

## `fetch_evidence`

`result_type=evidence`; `data` contains:

| Field | Meaning |
| --- | --- |
| `requested` | Deduplicated bounded citation IDs requested. |
| `resolved` | Number resolved into the envelope's `citations`. |
| `missing` | Requested IDs not found in the active snapshot. |

Every citation actually used in the answer must appear in `citations`, not merely in `requested`.
