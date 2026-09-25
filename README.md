# @pipeworx/ena

The European Nucleotide Archive (EMBL-EBI) — the public record of raw
sequencing runs, assemblies and annotated sequence, searchable by organism,
study, platform, collection country or date, with FASTQ/BAM download URLs and
checksums for every run.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `ena_search(result?, query?, fields?, sortFields?, limit?)` — search any ENA
  result type with ENA's own query grammar (`tax_eq(2697049)`,
  `tax_tree(9606) AND library_strategy="WGS"`, `country="Kenya"`). Answers
  "what public sequencing data exists for this organism / disease / place".
- `ena_filereport(accession, result?, fields?, limit?)` — every data file behind
  a study, run, sample or experiment accession, with direct download URLs, byte
  sizes and MD5s. Answers "where do I get the reads".
- `ena_results_types(result?)` — the schema-discovery step: every searchable
  result type with its live record count, and for one named type, every field
  it accepts. Call this before composing a query.

## Auth

Keyless. No registration step.

## Data sources

- <https://www.ebi.ac.uk/ena/portal/api/search> — typed search over result types.
- <https://www.ebi.ac.uk/ena/portal/api/filereport> — per-accession file listing.
- <https://www.ebi.ac.uk/ena/portal/api/results> — result types + live counts.
- <https://www.ebi.ac.uk/ena/portal/api/returnFields> — fields per result type.

### Things that cost time to rediscover (measured 2026-09-17)

- **`format=json` is not the default.** Without it you get TSV, which downstream
  reads as a single malformed string rather than as an error.
- **`limit=0` means UNLIMITED**, the opposite of the usual convention. On
  `read_run` that is tens of millions of rows. This pack never sends 0.
- **Numeric columns are strings** on the wire (`"read_count":"4547927"`).
- **A search with no `fields` returns the accession column alone** — a
  valid-looking, useless answer. The pack sends a default field set per result
  type.
- **`filereport` rejects a mismatched accession with HTTP 200** and a
  `{"message": ...}` body naming the accepted regexes. We raise that message;
  returning an empty list would read as "this study has no runs".

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ena": {
      "url": "https://gateway.pipeworx.io/ena/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ena/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ena_search \
  -H 'Content-Type: application/json' \
  -d '{"result":"read_run","query":"tax_eq(2697049)","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ena_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ena": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ena"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ena
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ena data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
