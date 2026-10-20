# @pipeworx/sk-rpo

Slovakia's primary legal-entity register — search companies, sole traders and
public bodies by name or IČO, then pull the full record (statutory bodies,
stakeholders, equity, predecessors, change history) for one entity.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1854+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `rpo_search_entity(fullName?, identifier?, legalForm?, addressMunicipality?, addressStreet?, establishmentAfter?/Before?, terminationAfter?/Before?, onlyActive?, stakeholderCompanyName?, stakeholderPersonFamilyName?, statutoryBodyFamilyName?, limit?)` — search for entities. Matching is a diacritics-insensitive substring match, not relevance-ranked, so a short or common query can return hundreds of hits; at least one filter is required. Returns each match's internal `id`, current/historical names, IČO, addresses and establishment/termination dates.
- `rpo_entity_detail(id?, ico?, includeHistory?, includeOrganizationUnits?)` — the full record for one entity: legal forms, address history, statutory bodies (directors/executives), stakeholders (owners/partners), equity and shares, predecessors (mergers/splits), statistical classification codes, and the source register (Obchodný register / Živnostenský register / etc.) with its registration number. Pass `id` from a prior search, or `ico` to resolve by IČO first.

## Auth

Keyless. No registration, no API key.

## Data sources

- Statistical Office of the Slovak Republic (ŠÚ SR), Register právnických osôb,
  podnikateľov a orgánov verejnej moci (RPO) — the national register of every
  legal entity, entrepreneur and public authority with a Slovak IČO. Published
  under a CC-BY 4.0 licence; every API response carries the licence text (this
  pack strips it from tool output and cites it here instead).

ORSR (orsr.sk, the Ministry of Justice's commercial register — adds share
capital / partner / executive detail for commercially-registered entities) was
evaluated as the secondary source for this build. It is an HTML site with no
public API, so it is not wired in. RPO alone already returns statutory bodies,
stakeholders, equity and shares for every entity this pack was tested against —
re-evaluate ORSR only if a caller needs something RPO's API does not expose.

finstat.sk, a commercial Slovak company-data reseller, was ruled out as a
candidate per the build task (it is not a primary register).

### Quirks worth knowing

- **Matching is substring, not fulltext-ranked.** `fullName=ESET` returns 272
  results (anything containing "eset" once diacritics are folded — e.g.
  "Rešeta" matches, since "š"→"s" makes "reseta" contain "eset") and the exact
  target may not be first. Use the fullest name string you have, or search by
  `identifier` (IČO) for a precise hit.
- **Diacritics are folded, not required.** "Zilinska" and "Žilinská" match the
  same records; you do not need to get the diacritics right.
- **`identifier` requires 3+ characters to exact-match**; shorter strings are
  not treated as fulltext and may return nothing.
- **The upstream caps search results at 500** regardless of how many actually
  match; there is no documented pagination/offset parameter, so a query that
  hits the cap should be narrowed rather than paged.
- **`rpo_entity_detail` takes the register's internal `id`, not the IČO**,
  for its path parameter — `GET /entity/{ico}` 404s. Pass `ico` to this tool
  and it resolves to the internal id for you via an exact-identifier search
  first.
- **The documented query-parameter names are not discoverable from the API
  itself** — `GET /search` with no recognized parameter returns a generic
  `{"message":"No parameters specified in search request"}` for both "no
  params at all" and "params I don't recognize," and there is no swagger/docs
  endpoint served at `api.statistics.sk`. The confirmed parameter set (verified
  live): `fullName`, `identifier`, `legalForm`, `legalStatus`,
  `addressMunicipality`, `addressStreet`, `establishmentAfter`/`Before`,
  `terminationAfter`/`Before`, `onlyActive`, `mainActivity`, `esa2010`,
  `sourceRegister`, `stakeholderType`, `stakeholderPersonGivenName`/`FamilyName`,
  `stakeholderCompanyName`, `statutoryBodyType`, `statutoryBodyGivenName`/`FamilyName`,
  `statutoryBodyCompanyName`, and the `orgUnits*` equivalents of the
  organization-unit fields. This pack exposes the subset verified to return
  real matches during build (`legalStatus` and `mainActivity` returned zero
  results for every value tried and are left out pending real test values).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sk-rpo": {
      "url": "https://gateway.pipeworx.io/sk-rpo/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sk-rpo/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1854+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/rpo_search_entity \
  -H 'Content-Type: application/json' \
  -d '{"fullName":"ESET, spol","onlyActive":true}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/rpo_search_entity`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "sk-rpo": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sk-rpo"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-sk-rpo
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Sk Rpo data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
