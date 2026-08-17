---
name: atlas-deep-research
description: Produce a decision-ready, fully cited research report from all five Atlas streams — companies, research papers, news, vacancies and book-derived product ideas — combined with your own web search. Use when a user asks for deep research, a market or competitive landscape, a thesis to be validated or challenged, or an answer that must hold up under scrutiny with sources attached. Unlike a quick lookup, this skill runs unbounded discovery across every stream, resolves each citation, and renders a report whose every number links back into Atlas.
---

# Atlas Deep Research

Answer the question with everything Atlas knows, plus what the open web adds, and render it so a
reader can check every number without asking you a follow-up.

Read [mcp-response-contract.md](../atlas-retrieval-research/references/mcp-response-contract.md)
before interpreting responses and
[evidence-contract.md](../atlas-retrieval-research/references/evidence-contract.md) before
synthesizing claims.

## Step 0: connect Atlas before anything else

If the Atlas tools are missing from your tool list, or a call fails with an authorization error,
stop and walk the user through connecting, in their language: Atlas is installed but not connected,
so no data can be read yet; complete the Atlas sign-in Codex offers for this plugin, using the
individual full name, username and one-time password issued in Atlas Admin; then start a new thread
so the tools load. Never ask for, or accept, a password or one-time code in chat — the browser flow
is the only place those belong.

Without Atlas, do not silently fall back to your own web search and present the result as Atlas
evidence. Say what is unavailable first.

## The five streams

| Stream | Tool | Marker | One item establishes | It never establishes |
| --- | --- | --- | --- | --- |
| Companies | `search_projects` | `C01` | Current Evidence facts about a product | — |
| Research | `search_research` | `R01` | A study measured this, in its publication year | That a company achieved it |
| News | `search_news` | `N01` | It was reported on that date | That the reported numbers were verified |
| Vacancies | `search_vacancies` | `V01` | Hiring intent and investment at that date | That the capability shipped |
| Book ideas | `search_book_ideas` | `B01` | A product hypothesis already exists | Market demand or feasibility |

`search_knowledge` queries the four non-company corpora in one call and returns them grouped.
Companies come only from `search_projects`; they are never placed in a shared index.

Company facts are canonical Evidence. The other four are derived research artifacts: they never
override a company value, never enter a company metric table, and never change a ranking.

## Effort

There is no call budget, no time limit, and no cohort cap. Work until new search angles stop
returning relevant material you do not already have.

Never trim results to a round number, to fit a table, or because the list feels long. If a search
returns 40 relevant companies, show 40. Truncating a cohort silently is the failure this skill
exists to prevent. When a knowledge group reports `truncated: true`, it returned exactly the page
you asked for and more may exist — say so, and raise the limit rather than implying completeness.

If you stop before full coverage, state plainly what remains uncovered. Report the coverage you
achieved; never imply completeness you did not verify.

## Workflow

1. **Restate the question as a decision.** Name the comparison axes, the metrics that would settle
   it, what would count as counterevidence, and what is out of scope.
2. **Companies first.** Call `prepare_search_query`, then `search_projects`, from several distinct
   but relevant angles: the direct workflow, adjacent substitutes, and counterexamples. Expand
   strong seeds with `find_similar_projects`. Do not pass `limit` unless you deliberately want a
   smaller cohort.
3. **Then a scale pass, always.** Issue a **separate** `search_projects` call for the same question
   with `metric_sort` set to a scale or adoption metric — monthly installs, website visits,
   downloads. A `metric_sort` request runs a corpus-wide Evidence pass, so it returns relevant
   projects the semantic pass never surfaced. This step is what finds the leaders of a crowded
   category: dozens of small products describe themselves in the exact words of the query, while
   the leader may use other words, another language, or a brand name that says nothing about the
   category, and can therefore rank below fifty near-identical clones. Repeat with a second metric
   family when the first has thin coverage — a project without that metric cannot appear in that
   ordering at all.
4. **Then the knowledge streams.** Call `search_knowledge` for the same question, and per-corpus
   tools when one stream deserves depth. Research tells you what was measured; news tells you what
   happened and when; vacancies tell you where money is going now; book ideas tell you which
   hypotheses already exist.
5. **Then your own web search.** It is a normal step and needs no permission. Keep every external
   finding in a separate lead ledger. External findings never receive an Atlas marker. For each
   useful external company, make at least one Atlas attempt by exact name or precise semantic
   query, and look for its closest Atlas analogue when it is absent.
6. **Resolve every citation.** Reuse citation objects already present in tool envelopes. Call
   `fetch_evidence` for unresolved Atlas company IDs and `fetch_knowledge_evidence` for unresolved
   knowledge IDs. Drop any claim whose citation does not resolve, or label the gap precisely.
7. **Check the streams against each other.** Agreement across streams is the strongest signal you
   can offer; a stream that contradicts the others is the most important thing in the report.

Judge a scale-pass result on what the product is, never on which retrieval path found it. Never drop
a project merely because the semantic pass missed it — that is how a category leader disappears from
a report. Do still exclude what does not answer the question: a metric-sorted pass is bounded by
structured filters rather than by meaning, so it returns large products from unrelated categories
too. When `rescue_ran` is `true`, the response reached beyond the semantic candidates; treat that as
coverage to verify, not as a warning to discard.

## Citation rendering

This is the part that most often goes wrong. A raw locator dumped into prose makes the report
unreadable and untrustworthy.

**Never** print a locator (`card:…:v7`, `metric:…`, `source:…`, `research:…`, `news:…`, `vacancy:…`,
`idea:…`), a `score`, a `rank` or an `item_id` in prose, headings, table cells or bullets.

**Always**:

- cite with a compact bracketed marker placed immediately after the number or claim it supports;
- combine several markers in one pair of brackets — `[R01, N03]`, not `[R01] [N03]`;
- render every marker as a Markdown link to that item's `atlas_url`;
- link every company name to its Atlas project URL, and keep the external origin URL out of prose;
- put each full `citation_id` exactly once, in the closing source table;
- state the date using the item's `dated` value with the meaning its `date_field` names.

Correct:

```markdown
Adoption reached 41% of surveyed teams in 2026 [[R01]](https://atlas.example.com/research-lab/doc/research/r-1),
and [Acme](https://atlas.example.com/project/acme) is hiring for the runtime that would ship it
[[V01]](https://atlas.example.com/research-lab/doc/vacancies/v-1) — intent, not a shipped feature.
```

Incorrect:

```markdown
Adoption reached 41% (research:r-1, publication_year 2026, score 0.9). Источники: research:r-1,
vacancy:v-1. Acme is hiring, so the runtime exists.
```

The second version leaks locators and scores, labels sources inside prose, links nowhere, and turns
a hiring signal into a shipped capability.

Every tool response carries a `presentation` block with these rules. When present, it is the
authoritative contract for that payload — follow it.

## Report shape

Write in the user's language. Use Markdown that renders well in a terminal client: short paragraphs,
compact tables, no decorative separators, no ASCII art.

1. **Answer** — the conclusion in the first paragraph, with its qualification and your confidence
   (`high` / `medium` / `low`). It must agree with the weakest stream: never claim support across
   all streams when one is weak or missing.
2. **Evidence across streams** — one row per stream, in order: companies, research, news, vacancies,
   book ideas. Each row names the strongest signal, its best date or metric, honest strength
   (`strong` / `medium` / `weak` / `missing`), and what it changes in the conclusion. Mark a stream
   `missing` rather than stretching a weak item into it. Distinguish `missing` (searched, nothing
   found) from an unavailable stream (not searched successfully) — they are not the same claim.
3. **Companies** — the union of the semantic pass and the scale pass, in full, not a sample. Name,
   Atlas link, what it does, and the comparable metrics with units and dates. Keep visits, MAU,
   downloads, installs, revenue, ranks, ratings, funding and GitHub signals in separate families;
   never invent a combined size score. If the largest products in the category came only from the
   scale pass, that is worth one sentence: it tells the reader the category is crowded with
   look-alikes.
4. **Findings** — ordered by decision importance. Each carries one claim, why it matters, and the
   strongest concrete number or date, cited.
5. **Counterevidence** — the strongest material against the thesis, plausible alternative
   explanations, and execution risks. Not a token devil's-advocate paragraph.
6. **Web leads** — external findings kept separate, marked as unverified by Atlas, with the Atlas
   attempt you made for each.
7. **Unknowns and next check** — the few missing facts that would materially change the answer.
8. **Sources** — every cited item: marker, stream, title, date, source name, full `citation_id`,
   Atlas link, and the external origin link when one exists.

## Boundaries

- Never call website visits MAU, infer growth from a single observation, turn missing into zero, or
  combine incompatible units, windows, geographies or entity grains.
- A returned company-scale value of `0` means too few observations to establish scale — not that the
  company has no users.
- Book items are Atlas restatements attributed through `attribution`. Never present one as a
  quotation and never reproduce book text verbatim.
- A vacancy dated from `first_seen_at` or `last_seen_at` carries an observation date, not a posting
  date; say which. A vacancy snapshot may be historical.
- Retrieval scores, reranker scores, similarity and graph edges are discovery signals, never
  evidence, and never appear in the report.
- Report insufficient evidence rather than padding a cohort or a stream with weak matches.
