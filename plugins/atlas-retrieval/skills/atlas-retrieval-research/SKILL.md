---
name: atlas-retrieval-research
description: Build evidence-backed market, product, competitor, trend, opportunity, or hypothesis analysis from Atlas Retrieval MCP. Use when a user asks Codex to discover or compare projects, evaluate scale or growth, inspect features or funding, find semantic alternatives, or turn current Atlas facts and metric history into a decision-ready answer. Also use when, in the Atlas Retrieval context, a user asks what Atlas MCP can do, how to use it, or requests a short onboarding. Use LanceDB for candidate discovery and Evidence/PostgreSQL for returned facts; resolve every cited Atlas claim before answering.
---

# Atlas Retrieval Research

Use Atlas Retrieval MCP to discover candidates through LanceDB dense/full-text retrieval and
Nemotron reranking. Treat current cards, metrics, bounded history, claims and citations hydrated
from Evidence/PostgreSQL as the factual substrate. Retrieval scores and similarities are discovery
signals, not evidence.

Read [mcp-response-contract.md](references/mcp-response-contract.md) before interpreting tool
responses and [evidence-contract.md](references/evidence-contract.md) before synthesizing claims.

## Step 0: connect Atlas before anything else

Do this first, on the first Atlas request of a session, before any research step.

If the Atlas tools are not available — no `search_projects` in your tool list, or a call fails with
an authorization error — stop and walk the user through connecting, in their language:

1. Say plainly that Atlas is installed but not yet connected, so no data can be read yet.
2. Ask them to complete the Atlas sign-in that Codex offers for this plugin. It opens a browser
   window; the plugin authenticates through Atlas OAuth and stores nothing locally.
3. Tell them the credentials are the individual full name, username and one-time password issued in
   Atlas Admin. Never ask them to paste a password, a token or a one-time code to you, and never
   accept one — the browser flow is the only place those belong.
4. After they confirm, ask them to start a new Codex thread so the tools load, then retry.

If a call fails after a successful sign-in, report the failure verbatim and stop. Do not retry a
different phrasing, guess headers, or fall back to your own web search and present it as Atlas data.

Once the tools respond, continue with the onboarding below or the research workflow.

## Short onboarding

When the user asks what Atlas MCP can do or how to use it, answer in the user's language without
calling MCP unless the user also asks to check the live connection. Keep the answer short and use
this shape:

1. Describe Atlas MCP as a project discovery and evidence service. It can find projects by meaning
   and keywords, compare scale and momentum, inspect current metrics and dated history, find similar
   projects, assemble support and counterarguments, resolve citations, and save selected projects
   with the user's hypothesis and feedback.
2. Give a three-step guide: ask a normal research question; optionally name the segment, platform,
   geography, metric or time window; then ask for evidence, comparison, counterarguments or saving.
3. Offer a few concrete prompts, adapted to the user's language:
   - "Find AI projects that help children learn math through play."
   - "Compare the 10 strongest products and show scale, growth signals and sources."
   - "Find evidence for this hypothesis and separate counterarguments."
   - "Save this hypothesis with these projects and my feedback."

Do not dump internal tool names, architecture or authentication details in the onboarding. Do not
claim that the live MCP connection works unless it was checked in the current task.

## Access and effort

Read the plugin [access guide](../../README.md) before connecting. Authentication is client-managed;
never place credentials in package files or guess headers after an authorization failure.

Work until the question is actually covered. There is no call budget and no time limit that ends
the research early. Keep issuing searches from new angles while they still return relevant material
that is not already in your ledger, and stop when new angles stop producing new relevant results —
not when a counter is reached.

Never silently reduce the user's requested scope. If you stop before full coverage, say plainly what
is still uncovered and why. Report the coverage you achieved rather than implying completeness you
did not verify.

## Discovery workflow

1. **Define relevance.** Restate the request as a decision and define direct relevance, comparison
   axes, key scale metrics, growth windows, counterevidence and exclusions.
2. **Take every relevant project.** There is no target count and no cap. Keep every project that is
   materially relevant to the question and show all of them. Do not trim a cohort to a round number,
   to fit a table, or to save effort. If Atlas holds few relevant projects, show every match and
   state the shortfall explicitly.
3. **Prepare every text search.** Call `prepare_search_query` for the exact wording and wait until
   both embedding and decomposition are ready before `search_projects`. Do not pass `limit` unless
   you deliberately want a smaller cohort; the server default already returns its full cohort.
4. **Search from several angles.** Use distinct but still query-relevant formulations for the
   direct workflow, adjacent substitutes and counterexamples. All retained projects must remain
   semantically related to the request; different angles are not permission to add generic popular
   companies. Russian/English lexical variants are generated by the service; do not duplicate them
   manually.
5. **Expand from good seeds.** When a search returns strong seeds, call `find_similar_projects` on
   them, deduplicate by `entity_id`, then validate each neighbour against the original request.
   Similarity is non-canonical and does not by itself prove that a project is a competitor.
6. **Run a scale pass — this is mandatory, not optional.** Text relevance alone cannot find the
   leaders of a category. When a query names a crowded category, dozens of small products describe
   themselves in exactly the words of the query, while the market leader may describe itself in
   other words, in another language, or under a name that says nothing about the category. Such a
   leader can rank below fifty near-identical clones and never enter the returned cohort at all.

   So issue a **separate** `search_projects` call for the same question with `metric_sort` set to a
   scale or adoption metric such as monthly installs, website visits or downloads. This is not a
   re-sort of the cohort you already have: a `metric_sort` request runs a corpus-wide Evidence pass
   and can return relevant projects that the semantic pass never surfaced. The response order is the
   metric's, not relevance — the reranker is skipped and `degraded_modes` says
   `rerank_skipped_for_metric_sort` — so the head of the list is the category's leaders by that
   metric. Repeat with a second metric family when the first has thin coverage, since a project
   without that metric cannot appear in that ordering at all.

   When the question is about growth or momentum, pass `include_metrics_history=true`: each item
   then carries `metrics_history` — dated observation series (≤12 points per metric, whitelisted
   metric families, series shorter than 3 observations omitted). Read growth strictly from those
   dated points.
7. **Search the knowledge corpora.** When available, call `search_knowledge` — or the per-corpus
   `search_research`, `search_news`, `search_vacancies`, `search_book_ideas` — for the same
   question. They answer what companies cannot: what studies measured, what was reported and when,
   where teams are hiring, and which product hypotheses already exist. See the knowledge section
   below for what each corpus may and may not support.
8. **Use your own web search freely.** Your built-in web search is a normal step and needs no
   separate permission or named coverage gap. Keep external findings in a separate lead ledger:
   they never receive an Atlas citation marker and never overwrite an Atlas value. For every useful
   external company, make at least one Atlas attempt using its exact name or a precise semantic
   query; if the exact company is absent, look for its closest Atlas analogue.
9. **Use final search items as current facts.** Production `search_projects.data.items` and
   `find_similar_projects.data.items` are batch-hydrated from Evidence and can support returned
   summaries, current metrics, momentum and attached citations. Do not call `get_project` merely
   to repeat those fields. Call it when a displayed project needs detailed fields, official links,
   features, funding/provider detail, or bounded metric history. `candidate_trace` remains
   retrieval-only diagnostic data.
10. **Resolve evidence.** Build a fact ledger and collect every citation ID used in prose or tables.
   Reuse citation objects already resolved in search/detail envelopes; call `fetch_evidence` in
   bounded batches only for used IDs absent from those envelopes, and `fetch_knowledge_evidence`
   for knowledge IDs. Remove unsupported claims or label the precise gap.

## Current-fact and filter contract

- Require `fact_source=evidence_http` for production factual answers and retain `facts_as_of`.
- Treat `snapshot_id` as the active Lance generation, not the current-facts timestamp.
- Preserve `indexed_card_version`, `current_card_version` and `index_stale`. A stale index affects
  discovery text; returned facts still use the current Evidence card.
- Trust `applied_filters`, not query-plan proposals. Query-plan filters are disabled by default.
  Explicit categorical and metric filters are checked against current Evidence values, never a
  historical value that merely once satisfied the condition.
- Preserve `rescue_ran` and `unindexed_matches` warnings. Evidence rescue can return only projects
  present in the active Lance generation for semantic results. `rescue_ran=true` means the response
  reached past the semantic candidates, which is the intended behaviour of a `metric_sort` request.
- `unindexed_matches` names current Evidence records that satisfied the structured request but are
  absent from the active Lance generation. Report the count; those projects cannot be returned.
- If Evidence is unavailable, the MCP fails closed. Never fall back to Lance-stored cards or
  metrics.

## Saving user research

Save research only when the user explicitly asks to save, remember, pin or send a hypothesis/list
to Atlas. Never treat ordinary positive language as permission to write.

- Use only canonical `entity_id` values already returned by Atlas in the current research flow.
- Call `save_hypothesis` once with a caller-generated stable idempotency key, a concise title and
  statement, the source query when known, and 1–50 selected projects.
- Mark each project as `supports`, `contradicts`, `example` or `related` and preserve the user's
  project-specific feedback without upgrading it into an Evidence fact.
- Report the returned hypothesis ID and whether the call was an idempotent replay. If a project is
  rejected as unknown, resolve the intended Atlas entity instead of dropping it silently.
- Use `list_my_hypotheses` only when the user asks to review their saved Atlas work. It is scoped to
  the authenticated user; operators review all users separately in Atlas Review.

## Default analytical views

Unless the user asks for another organization, present three possibly overlapping views. Do not
force a project into only one bucket:

1. **Closest semantic matches** — strongest original-query relevance using reranker, dense/lexical
   reasons and verified project features.
2. **Largest relevant projects** — built from the scale pass in step 6, not by re-sorting the
   semantic cohort. Compare within one metric family at a time: website visits, safe active-user
   observations, downloads, installs, funding or GitHub scale. Never combine unlike units into one
   invented size score.
3. **Fastest-growing relevant projects** — use compatible dated histories or explicit typed growth
   metrics. A single current observation is not growth.

`rank_trending_projects` is supporting discovery for view 3.

**Judge a scale-pass or trend result on its own merits.** A project that a metric ordering returned
is relevant if it genuinely answers the question — not because it also appeared in the semantic
cohort. Never drop a project merely for being absent from that cohort: absence usually means its
card is worded differently, is written in another language, or carries a brand name that does not
restate the category. That is exactly how a category leader disappears from an answer.

Still exclude what does not answer the question. A metric-sorted request runs a corpus-wide Evidence
pass bounded by structured filters rather than by meaning, so it will return large products from
unrelated categories. Check each one against the original request and drop the irrelevant, but do
that on the evidence of what the product is, never on which retrieval path found it.

When `rescue_ran` is `true`, results reached beyond the semantic candidates. Treat that as a
coverage gain to verify, not as a warning to discard.

## Knowledge corpora

Four derived corpora sit beside companies. They are research artifacts, not canonical Evidence: they
never override a company card, metric, score or rank, and they never join a company metric table.
Each returned item carries `marker`, `citation_id`, `atlas_url`, `dated` with the `date_field` that
produced it, `evidence_kind` and an `interpretation` line. Honour these boundaries:

| Corpus | Tool | Marker | What one item proves | What it never proves |
| --- | --- | --- | --- | --- |
| Companies | `search_projects` | `C01` | Current Evidence facts | — |
| Research | `search_research` | `R01` | A study measured this, in its publication year | That a company achieved it |
| News | `search_news` | `N01` | It was reported on that date | That the numbers were verified |
| Vacancies | `search_vacancies` | `V01` | Hiring intent at that date | That the capability shipped |
| Book ideas | `search_book_ideas` | `B01` | A product hypothesis exists | Market demand |

Book items are Atlas restatements attributed to a work through `attribution`. Never present one as a
quotation from the book, and never reproduce book text verbatim.

A vacancy snapshot may be historical. When `dated` came from `first_seen_at` or `last_seen_at`
rather than `published_at`, say it is an observation date, not a posting date.

Group `status` distinguishes `empty` (searched, nothing relevant) from `unavailable`, `error` and
`timeout` (not searched successfully). Never report an unavailable corpus as "no evidence found".
When `truncated` is true the group returned exactly the requested page, so more may exist — say so
and offer to raise the limit instead of implying the list is complete.

## Citation rendering

Raw locators in prose are the single worst failure mode of this skill: they make the answer
unreadable. A locator (`card:…:v7`, `metric:…`, `source:…`, `research:…`, `news:…`, `vacancy:…`,
`idea:…`) never appears in prose, headings, table cells or bullets.

- In prose, cite with a compact bracketed marker placed immediately after the number or claim it
  supports: `[C03]`, `[R01]`, `[N02, V04]`.
- Render each marker as a Markdown link to that item's `atlas_url`, so the reader clicks into Atlas
  rather than reading an identifier.
- The full `citation_id` appears exactly once, in the closing source table.
- No `Sources:` or `Источники:` labels inside prose, no repeated markers in one sentence, and never
  a marker for something you did not actually resolve.

Correct:

```markdown
Adoption reached 41% of surveyed teams in 2026 [[R01]](https://atlas.example.com/research-lab/doc/research/r-1),
and [Acme](https://atlas.example.com/project/acme) is hiring for the runtime that would ship it
[[V01]](https://atlas.example.com/research-lab/doc/vacancies/v-1) — intent, not a shipped feature.
```

Incorrect:

```markdown
Adoption reached 41% (research:r-1, publication_year 2026, score 0.9). Источники: research:r-1,
vacancy:v-1. Acme is hiring so the runtime exists.
```

The second version leaks locators and scores, labels sources inside prose, links nowhere, and
upgrades a hiring signal into a shipped capability.

Every returned tool response also carries a `presentation` block with these rules. When it is
present, follow it — it is the authoritative rendering contract for that payload.

## Presentation rules

- Link a project name to the current primary or official link returned by Evidence. If none is
  returned, leave the name unlinked rather than inventing a URL. Provider pages belong only in
  secondary evidence/source columns.
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
- When funding exists in current Evidence detail, show the rounded total and every returned
  investor name; include round/category when returned. If investor records are empty, say that
  names are unavailable rather than inferring them.
- Keep factual statements and interpretations separate. Cite the exact Atlas citation ID next to
  each material fact or number.

## Typed momentum rules

- Keep `website_monthly_visits_growth_ratio`, `github_growth_61d` and `github_growth_6m` separate.
- Call `app_installs_weekly` and `app_installs_monthly` velocity, not growth.
- Treat `app_rating_count_weekly` as a weekly delta.
- Preserve `app_best_rank_ever`, `app_ever_top_10`, `app_first_top_10_at`,
  `app_rank_observation_days` and every top-10 milestone grain. Lower rank is better.
- `app_update_stale_90d=true` means the latest known store update is strictly older than 90 days;
  `null` means unknown. Treat it as maintenance risk, not proof of poor quality.
- Never use application `release_date` as a company founding date or infer time-to-success without
  a canonical founding/launch date.

## Answer shape

1. Calibrated conclusion and coverage count.
2. Closest semantic matches table.
3. Largest relevant projects table.
4. Fastest-growing relevant projects table.
5. Features, funding/investors and combined dynamics where available.
6. Counterevidence, external leads attempted in Atlas, and limitations.
7. Complete resolved source bundle and the next validation step.

Prefer compact tables. If a project appears in several views, use the same returned official link
each time and avoid duplicating its full description.

## Evidence boundaries

- Never call website visits MAU, infer growth from a point observation, turn missing into zero, or
  combine incompatible units, windows, geographies or entity grains.
- Treat bounded history as Evidence observations; absence of history is not evidence of stability.
- Treat features and funding investors as card fields only when returned by current `get_project`
  detail and backed by its claim/citation bundle.
- Treat `find_similar_projects`, `analyze_feature`, graph expansion and retrieval ranks as derived
  discovery signals, not canonical relationships or causal evidence.
- Report insufficient evidence rather than padding the cohort with weak matches.
