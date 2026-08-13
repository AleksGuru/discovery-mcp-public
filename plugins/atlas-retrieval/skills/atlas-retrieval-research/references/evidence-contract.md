# Atlas evidence contract

## Fact ledger fields

For every material claim retain:

- entity ID, project name, card version and entity grain;
- observed value and data state;
- metric key, unit, denominator, geography and measurement window when present;
- `observed_at`, bounded history and `history_scope`;
- `source_key`, confidence and all citation IDs;
- classification as direct match, adjacent analogue, benchmark, counterexample or excluded.

## Comparability rules

- Compare values only inside the same metric family, unit, entity grain and compatible window.
- A cumulative value is adoption, not current activity or recent growth.
- A single observation is current scale, not direction.
- Derive a delta only from compatible observations for the same entity and metric definition.
- Treat rank direction explicitly: a lower rank number can represent improvement.
- Keep provider estimates distinct from company claims and observed repository/store values.
- State the covered numerator and denominator for concentration or coverage claims.

## Resolution gate

Before answering:

1. Deduplicate intended citation IDs.
2. Resolve them with `fetch_evidence`.
3. Verify each used ID has one source record and supports the nearby claim.
4. Preserve null URLs as an auditability limitation; never invent a URL.
5. Exclude unresolved citations from factual prose and list the resulting evidence gap.

## Minimum answer shape

1. Direct conclusion and confidence.
2. Cohort or landscape table.
3. Comparable metrics with dates and units.
4. Interpretation explicitly separated from facts.
5. Counterevidence and alternative explanations.
6. Missing evidence and next validation step.
7. Complete source bundle for every used citation.
