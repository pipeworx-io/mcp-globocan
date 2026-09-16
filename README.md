# GLOBOCAN — Global Cancer Observatory (IARC/WHO)

Cancer incidence, mortality and 5-year prevalence for **238 countries and territories** across **41 cancer sites**: new cases, deaths, age-standardised rates per 100,000, and cumulative risk to age 74.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Attribution — a condition of use, not a courtesy

IARC permits free use of these data **but not sale**, and requires the source to be acknowledged. Every response carries the credit, and the note travels with the mirrored rows so it cannot be dropped downstream.

> IARC / WHO Global Cancer Observatory (GLOBOCAN). Free to use, not for sale; source acknowledged as a condition of use.

## Tools

| Tool | Answers |
|---|---|
| `globocan_country_profile` | *How much cancer is there in Japan, and which sites lead?* |
| `globocan_cancer_in_country` | One cancer, one country — cases, deaths, prevalence, ASR, cumulative risk |
| `globocan_countries` | The countries covered, with ISO3, region and HDI band |
| `globocan_cancer_types` | The 41 sites with their ICD-10 ranges |

## Why this is a mirror

`gco-api.iarc.fr` refuses both of our compute environments while serving a laptop in about a second:

| From | Result |
|---|---|
| Laptop | 200, ~1s |
| Cloudflare Worker | silent hang — 45s, zero bytes, no status |
| Supabase edge function | `Connection reset by peer` (os error 104) at connect |

TLS is not the cause — the certificate is valid for the host (SAN `*.iarc.fr`) and verifies clean — so this is a deliberate refusal of datacenter ranges. The standard egress-relay fix does **not** work, because the block is not Cloudflare-specific.

**The block affects serving, not ingesting.** Data is fetched once from an allowed connection and read here from Postgres. GLOBOCAN is an annual estimate release, so a mirror costs nothing in freshness.

## Refreshing

```bash
<store credentials in the environment> node scripts/ingest-globocan.mjs
```

**Run it from a laptop, not CI or a Worker** — see above. If it starts hanging, you are probably on a cloud host. It is idempotent (upserts on the natural key), so re-running is safe. Once a year is enough.

Schema: `supabase/migrations/067_globocan_mirror.sql`.

## Gotchas

1. **The dimensions are bare integers and the measure is the dangerous one.** Upstream calls it `type`; the mirror stores it as `measure` precisely because `type` says nothing. **0 = incidence, 1 = mortality, 2 = prevalence.** Handing back a raw row lets 632,166 US cancer *deaths* read as new cases, or an 8-million prevalence figure read as an annual count. Every number this pack returns is labelled.

   The encodings were pinned against published US figures — 2,462,729 / 632,166 / 8,069,773 — not assumed.

2. **`cancer_code` 39 is "All cancers"** (C00-97), not a site. It is excluded from the leading-site rankings; leaving it in puts a row worth 100% at the top of every list.

3. **Sex-specific sites have no both-sexes row.** Asking for a male-only site with `sex: "female"` returns `no_estimate` with the reason — the standard being coherent, not a gap in the data.

4. **These are modelled estimates, not case counts.** IARC estimates national burden from registry data of varying completeness. For *observed* registry counts see CI5 (Cancer Incidence in Five Continents), which is a different dataset answering a different question.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "globocan": {
      "url": "https://gateway.pipeworx.io/globocan/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/globocan/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "globocan": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-globocan"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-globocan
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Globocan data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
