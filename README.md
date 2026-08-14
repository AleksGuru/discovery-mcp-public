# Atlas Retrieval plugin

Public Codex distribution for the Atlas Retrieval research plugin. Installing it adds both:

- the `atlas-retrieval-research` skill, which guides evidence-backed project research;
- the production Atlas MCP connection, authenticated through an Atlas beta account.

Version 1.2.1 uses LanceDB for candidate discovery and Evidence/PostgreSQL for the current facts,
metric history and citations returned by MCP.

## Install

```bash
codex plugin marketplace add AleksGuru/discovery-mcp-public --ref main
codex plugin add atlas-retrieval@atlas
```

Restart Codex after installation. On first use, sign in with the login and one-time password created for you in Atlas Admin. The repository contains no credentials.

See [`plugins/atlas-retrieval/README.md`](plugins/atlas-retrieval/README.md) for workflow and authentication details.
