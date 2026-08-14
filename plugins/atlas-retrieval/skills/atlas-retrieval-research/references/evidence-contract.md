# Atlas evidence contract

## Source boundary

In production, LanceDB contains a derived search projection: OpenAI document embeddings, FTS text
and the fields needed to select and rerank candidates. Evidence/PostgreSQL is authoritative for the
current card, current metrics, bounded history, claims and citations returned by the MCP.

Use `fact_source=evidence_http` as the runtime proof of this boundary. `snapshot_id` identifies the
active Lance generation; `facts_as_of` identifies the Evidence read. They are different clocks.

## Fact ledger fields

For every material claim retain:

- entity ID, project name, current card version and entity grain;
- observed value and data state;
- metric key, unit, denominator, geography and measurement window when present;
- `observed_at`, bounded history and its scope;
- `source_key`, confidence and every citation ID;
- classification as direct match, adjacent analogue, benchmark, counterexample or excluded;
- `indexed_card_version`, `current_card_version` and `index_stale` when returned.

Index freshness fields explain candidate-selection freshness. They do not downgrade a newer
Evidence fact: when `index_stale=true`, use the current Evidence card and state that discovery text
came from an older indexed version when it matters.

## Comparability rules

- Compare values only inside the same metric family, unit, entity grain and compatible window.
- A cumulative value is adoption, not current activity or recent growth.
- A single observation is current scale, not direction.
- Derive a delta only from compatible observations for the same entity and metric definition.
- Treat rank direction explicitly: a lower rank number can represent improvement.
- Keep provider estimates distinct from company claims and repository/store observations.
- State the covered numerator and denominator for concentration or coverage claims.
- Treat installs per week/month as velocity, not growth.
- Treat update staleness as maintenance risk, never as product-quality proof.

## App momentum rules

- Preserve top-10 milestone grain: store, app, country, collection, category, date and rank.
- `app_update_stale_90d=true` only when the latest known update across known applications is
  strictly older than 90 days.
- Missing `store_last_updated` produces unknown/null, never `false`.
- Do not substitute `release_date` for company founding or canonical launch date.
- Do not calculate time-to-scale until a canonical founding or launch date exists.

## Resolution gate

Before answering:

1. Deduplicate the citation IDs used in prose and tables.
2. Reuse citation objects already resolved in search/detail envelopes.
3. Call `fetch_evidence` only for used IDs that are not present in those envelopes.
4. Verify each used ID has one source record and supports the nearby claim.
5. Preserve null URLs as an auditability limitation; never invent a URL.
6. Exclude unresolved citations from factual prose and list the resulting evidence gap.

## Failure and coverage rules

- Evidence timeout or authorization failure is a sanitized service failure. Never fall back to
  Lance-stored facts.
- `unindexed_matches` means current Evidence records satisfied a structured filter but were absent
  from the active Lance generation. Report the warning; they cannot become semantic results until
  indexed.
- `degraded_modes` applies to retrieval channels. It does not authorize unsupported facts.
- Query-plan filters are proposals and are disabled by default. Trust `applied_filters`, not the
  plan explanation.

## Minimum answer shape

1. Direct conclusion and confidence.
2. Relevant cohort table.
3. Comparable metrics with dates and units.
4. Typed momentum, when available.
5. Interpretation explicitly separated from facts.
6. Counterevidence and alternative explanations.
7. Missing evidence and next validation step.
8. Complete source bundle for every citation used.
