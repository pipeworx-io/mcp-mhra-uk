# @pipeworx/mhra-uk

UK medicines register — the MHRA products database behind products.mhra.gov.uk.
Post-Brexit the EMA register no longer covers the UK, so this is the UK
authorisation surface.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1573+ live data sources.

## What this register actually is

It is an index of **documents** — Summaries of Product Characteristics, patient
information leaflets and public assessment reports, as PDFs — with metadata
about each. It is **not** a register of structured product records. There is no
indication field, no pathway field and no authorisation-status field in it.

That shapes every tool here. `mhra_product` returns the document metadata and
the authoritative PDF URL rather than a summary of the document, because the
authorised indication text is written in section 4.1 of the SmPC and nowhere
else. `authorisation_status` is always `unknown`, with a note saying why: the
only status-shaped field is `release_state`, which is `Y` on all 79,112 released
documents and means "published", not "currently authorised". Reporting it as
authorisation status would be a guess wearing a field name.

## Tools

- `mhra_search(query, doc_type?, territory?, limit?, offset?)` — by product,
  brand or active substance. One row per document.
- `mhra_product(pl_number)` — every document for one licence, grouped by type.
  Accepts `PL 29831/0798`, `PL29831/0798`, `PL298310798` and the PLGB / PLNI
  forms.
- `mhra_recent_changes(since, doc_type?, limit?)` — documents added or revised
  since a date, newest first. Idempotent on replay.
- `mhra_snapshot()` — total count, counts per document type, newest document
  date, and the paging recipe. Counts, never documents.

## Licence prefixes are not spellings of each other

`PL` is UK-wide, `PLGB` is Great Britain only, `PLNI` is Northern Ireland only.
Measured 2026-09-07: UK 64,126 documents, GB 8,428, NI 48. Treating them as
interchangeable answers "is this authorised in the UK?" with a licence that may
cover a small fraction of it.

## Enumeration

There is no bulk download. Walk the register with
`mhra_search({query: "*", limit: 50, offset: N})`, stepping `offset` by 50.
`mhra_snapshot()` returns the current total so you know when to stop.

## recent_changes reports new and changed, NOT withdrawn

The register records documents, and a withdrawal is not published as an event.
`covers` on every response says so. Do not read an absence as a withdrawal.

## Fragility worth knowing

Search runs on Azure Cognitive Search using MHRA's own client key — the one
shipped to every browser that loads products.mhra.gov.uk. No account, payment or
agreement stands between anyone and this data. But if MHRA rotates that key this
pack stops working, and the fix is to read the current one out of their public
bundle again. `medicines.api.mhra.gov.uk/graphql` also appears in their bundle
and 503s on every request; do not build on it.

`created` is sortable but **not** filterable on this index, which is why
`recent_changes` orders newest-first and stops at the cutoff rather than pushing
a date filter upstream.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "mhra-uk": {
      "url": "https://gateway.pipeworx.io/mhra-uk/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/mhra-uk/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1573+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "mhra-uk": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-mhra-uk"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-mhra-uk
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Mhra Uk data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
