# Atlas Retrieval plugin

Install this as one plugin. The installation exposes both the
`atlas-retrieval-research` skill and the production `atlas-retrieval` MCP server; do not install
them separately for normal closed-beta use.

Production retrieval uses LanceDB only to find candidates with OpenAI embeddings and full-text
search, then reranks them with Nemotron. Current cards, metrics, bounded history and citations are
hydrated from Evidence/PostgreSQL. The public package contains neither database access nor internal
Evidence credentials.

## Install in Codex

The closed-beta distribution repository is
[`AleksGuru/discovery-mcp-public`](https://github.com/AleksGuru/discovery-mcp-public). After this
package version is published there, colleagues install it without access to the private Atlas
source repository:

```shell
codex plugin marketplace add AleksGuru/discovery-mcp-public --ref main
codex plugin add atlas-retrieval@atlas
```

The plugin requests authentication during installation. Complete the browser login with the
individual username and one-time password issued in Atlas Admin, then start a new Codex task so it
loads the installed skill and MCP tools.

The Codex app can also install a shared copy from **Plugins → Shared with me**. Installing the
standalone skill is only a local development fallback; it does not configure or authenticate MCP.

Atlas maintainers validate package changes in the private `atlas-platform` repository first and
publish only the secret-free plugin directory and marketplace manifest to the public distribution
repository.

Agent Plugins-compatible clients can load this directory directly. The portable files are
`plugin.json`, `mcp.json`, and `skills/`; `.codex-plugin/` and `.mcp.json` are Codex-specific
metadata.

## Access

No credentials are bundled with this package. Obtain your individual full name, username, and
one-time password from Atlas Admin. Compatible clients use MCP OAuth 2.1 discovery, browser login,
and PKCE S256; the client stores short-lived tokens in its own credential store. Atlas stores only
hashes of authorization codes and tokens.

Never paste a password or token into `plugin.json` or `mcp.json`, and never commit either. The MCP
endpoint currently returns `401` until client-managed authentication completes. Contact the Atlas
operator to revoke access or reset your credentials.
