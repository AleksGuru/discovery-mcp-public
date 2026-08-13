---
name: atlas-retrieval-research
description: Build evidence-backed market, product, competitor, trend, opportunity, or hypothesis analysis from Atlas Lance Retrieval MCP. Use when a user asks Codex to discover or compare projects, evaluate scale or growth, inspect features or funding, find semantic alternatives, or turn Atlas facts and metric history into a decision-ready answer. Resolve every cited Atlas claim before answering.
---

# Atlas Retrieval Research

Use Atlas Lance MCP as the factual substrate. Treat retrieval scores and similarities as discovery
signals, not evidence. Read [mcp-response-contract.md](references/mcp-response-contract.md) before
interpreting tool responses and [evidence-contract.md](references/evidence-contract.md) before
synthesizing claims.

## Access and timebox

Read the plugin [access guide](../../README.md) before connecting. Authentication is client-managed;
never place credentials in package files or guess headers after an authorization failure.

Start an elapsed-time clock with the first retrieval call. The practical research limit is 30
minutes and the absolute safety ceiling is 1,000 MCP calls. At 30 minutes, return the best complete
partial result, name unfinished gaps, and ask for explicit permission to continue. Do not silently
reduce the requested scope merely to finish sooner.

## Discovery workflow

1. **Define relevance.** Restate the request as a decision and define direct relevance, comparison
   axes, key scale metrics, growth windows, counterevidence and exclusions.
2. **Target at least 10.** Seek at least 10 unique relevant Atlas projects. If more materially
   relevant projects are available within the timebox, show more. If Atlas cannot support 10 after
   the expansion steps below, show every supported match and explicitly report the shortfall.
3. **Prepare every text search.** Call `prepare_search_query` for the exact wording and wait until
   both embedding and decomposition are ready before `search_projects`. Start searches with
   `limit=20`; increase within the server limit when useful.
4. **Search from several angles.** Use distinct but still query-relevant formulations for the
   direct workflow, adjacent substitutes and counterexamples. All retained projects must remain
   semantically related to the request; different angles are not permission to add generic popular
   companies.
5. **Expand from good seeds.** If `search_projects` returns fewer than 10 strong results but at
   least one or two good seeds, call `find_similar_projects` for up to three strongest seeds,
   deduplicate by `entity_id`, then validate each neighbour against the original request. Similarity
   is non-canonical and does not by itself prove that a project is a competitor.
6. **Use web only for a named coverage gap.** If Atlas still has too few companies, Codex may use
   its built-in web search without separate permission. Keep external leads separate. For every
   useful external company, make at least one Atlas attempt using its exact name or a precise
   semantic query; if the exact company is absent, try to find its closest Atlas analogue. Never
   overwrite Atlas values with web values.
7. **Hydrate every displayed project.** Call `get_project` for every project included in the final
   tables. Search hits and `analyze_feature` examples are discovery records, not sufficient support
   for project facts. Use structured `features` from `get_project`; feature-analysis matches are
   retrieval-derived rather than canonical feature labels.
8. **Resolve evidence.** Build a fact ledger, collect every citation ID used in prose or tables and
   call `fetch_evidence` in bounded batches. Remove unsupported claims or label the precise gap.

## Default analytical views

Unless the user asks for another organization, present three possibly overlapping views. Do not
force a project into only one bucket:

1. **Closest semantic matches** — strongest original-query relevance using reranker, dense/lexical
   reasons and verified project features.
2. **Largest relevant projects** — sort only the relevant cohort by comparable key scale metrics,
   such as website visits, safe active-user observations, downloads, installs, funding or GitHub
   scale. Never combine unlike units into one invented size score.
3. **Fastest-growing relevant projects** — use compatible dated histories or explicit growth
   metrics. A single current observation is not growth.

Use `rank_trending_projects` as supporting discovery or validation, then intersect its entities with
the semantically relevant cohort. Do not present a globally trending but irrelevant company.

## Presentation rules

- Make every project name link to `get_project.data.project.local_card_url`, the Atlas Platform
  card. Product websites and provider pages belong only in secondary evidence/source columns.
- Format human-facing numbers compactly and without false precision: `3,139,013` → `3.1M`,
  `12,430` → `12.4K`, `$2,500,000` → `$2.5M`, and percentages to one sensible decimal.
- For returned website visits or safe MAU values below 1,000, display `<1K`, not a precise small
  number. Current MCP intentionally excludes unsafe `mau` aliases; never substitute visits for MAU.
- For a returned company-scale value of `0`, say the data has too few observations to establish
  scale. Do not describe the company itself as having zero users or traction. Preserve an explicit
  `data_state=zero` in the evidence notes when relevant.
- Keep visits, MAU, WAU, downloads, installs, users, revenue, ranks, ratings, funding and GitHub
  signals in separate metric families with dates and units.
- When website-visit history is returned, describe direction, period and rounded start/end values.
  When application-rank history is returned, state that a lower rank number is better and describe
  improvement, decline or volatility. If both are available, an overall dynamics paragraph is
  mandatory.
- When funding exists, show the rounded total and list every returned investor name from
  `project.funding.investors`; include round/category when returned. If the list is empty, say that
  investor names are unavailable rather than inferring them.
- Keep factual statements and interpretations separate. Cite the exact Atlas citation ID next to
  each material fact or number.

## Answer shape

1. Calibrated conclusion and coverage count.
2. Closest semantic matches table.
3. Largest relevant projects table.
4. Fastest-growing relevant projects table.
5. Features, funding/investors and combined dynamics where available.
6. Counterevidence, external leads attempted in Atlas, and limitations.
7. Complete resolved source bundle and the next validation step.

Prefer compact tables. If a project appears in several views, link the same Atlas card each time and
avoid duplicating its full description.

## Evidence boundaries

- Never call website visits MAU, infer growth from a point observation, turn missing into zero, or
  combine incompatible units, windows, geographies or entity grains.
- Treat `current_snapshot_fallback` as a point observation, not history.
- Treat `features` and funding investors as card fields only when returned by `get_project` and
  backed by its claim/citation bundle.
- Treat `find_similar_projects`, `analyze_feature`, graph expansion and retrieval ranks as derived
  discovery signals, not canonical relationships or causal evidence.
- Report insufficient evidence rather than padding the cohort with weak matches.
