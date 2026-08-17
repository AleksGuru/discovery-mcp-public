# Atlas Retrieval plugin

Public Codex distribution for the Atlas Retrieval research plugin. Installing it adds:

- the `atlas-retrieval-research` skill, for evidence-backed project discovery and comparison;
- the `atlas-deep-research` skill, for a cited report drawing on every Atlas stream at once;
- the production Atlas MCP connection, authenticated through an Atlas beta account.

Version 1.5.0 reaches five evidence streams: companies, research papers, news, vacancies and
book-derived product ideas. Companies use LanceDB for candidate discovery and Evidence/PostgreSQL
for the current facts, metric history and citations returned by MCP. The other four are derived
research artifacts and never override a company fact.

Every returned item carries a citation that resolves, and a link back into Atlas.

## Install

```bash
codex plugin marketplace add AleksGuru/discovery-mcp-public --ref main
codex plugin add atlas-retrieval@atlas
```

Restart Codex after installation. On first use, sign in with the login and one-time password created for you in Atlas Admin. The repository contains no credentials.

See [`plugins/atlas-retrieval/README.md`](plugins/atlas-retrieval/README.md) for workflow and authentication details.
