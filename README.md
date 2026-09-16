# @pipeworx/ca-debtwatch

California municipal bond issuance — every state and local government bond, note, certificate of
participation and Mello-Roos special-tax deal reported to the California Debt and Investment
Advisory Commission (CDIAC) since 1984, at issue level: principal, sale date, debt type, purpose,
ratings, interest cost, issuance fees and the underwriter, municipal advisor and bond counsel on
the deal.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

This is **primary-market issuance**, not the secondary trade tape. It carries no MSRB/EMMA content
and no CUSIP dependency — CDIAC publishes it as California public data.

## Tools

- `ca_debt_search_issues(...)` — search 76,000+ California issues by issuer, county, debt type,
  purpose, tax status, ESG label, sale date and principal amount. The workhorse.
- `ca_debt_issue(cdiac_number)` — full detail for one issue, including issuance costs broken out by
  participant (underwriter takedown, advisor fee, bond and disclosure counsel fees, trustee fee,
  rating agency fee) and any Annual Debt Transparency Report rows filed against it in later years.
- `ca_debt_issuer_profile(issuer)` — one issuer's borrowing history: total principal, issue count,
  date range, breakdown by debt type and purpose, largest and most recent issues.
- `ca_debt_statewide_totals()` — the state treasurer's headline totals (debt issued YTD, local debt
  issued YTD, proposed but unsold, recent long-term volume).
- `ca_debt_filter_options(field?)` — the exact wording CDIAC uses for each enumerated filter.
- `ca_debt_resolve_issuer(name)` — partial name → the exact issuer names CDIAC uses.

## Auth

Keyless. No registration, no key, no quota published.

## Data sources

- <https://debtwatch.treasurer.ca.gov/env.json> — the API base the SPA itself reads. Resolved once
  per isolate rather than hardcoded, with a fallback to the current value.
- `PUT /api/dataset/issues/search` — issuance search. **PUT, not GET or POST.**
- `GET /api/dataset/issues` — column metadata, including the accepted values for every enumerated
  filter.
- `GET /api/dataset/issues/element/Issuer/search?q=&pageNumber=N` — issuer typeahead (7,600+
  issuers). **Paginates at ten and ignores every page-size parameter** (`pageSize`, `take`, `limit`
  all no-op) while reporting the true count in `totalValues`; only `pageNumber` works. Reading page
  one alone makes "san diego" look like 10 issuers when it is 47, and makes an exact name that
  sorts onto a later page (San Diego Unified Port District) come back as ambiguous with candidates
  that do not contain it. This pack pages, bounded, and reports when it stopped early.
- `GET /api/report/issuance-detail-with-history/{cdiacNumber}` — single-issue detail plus later ADTR
  filings.
- `GET /api/fast-stat` — headline statewide totals.

### Things worth knowing before you touch this

- **A malformed filter is an empty HTTP 500, not a 400.** No body, no validation message. Filters
  are keyed by column id with a `type` discriminator, decoded from the SPA bundle:
  `StringOneOf` / `StringContainsOneOf` (`options: []`), `DateBetween` / `DateOnOrAfter` /
  `DateOnOrBefore` / `DateEqualTo` (`minimum` / `maximum` / `exactly`), `NumberBetween` /
  `NumberGreaterThanOrEqualTo` / `NumberLessThanOrEqualTo`, `BooleanEqualTo` (`exactly`). Anything
  else is a 500.
- **`IssuerCounty` is the multi-value column** and needs `StringContainsOneOf`; sending
  `StringOneOf` against it matches nothing without erroring.
- **Text filters are exact-match on CDIAC's spelling**, and a near-miss returns a clean zero-row
  200 rather than an error — `Los Angeles` is a different issuer from
  `Los Angeles Unified School District`. Every enum argument in this pack is resolved against the
  upstream's own option list first, and every issuer argument through the typeahead endpoint, so a
  caller naming things the way a person would still gets rows. An argument that cannot be resolved
  is answered with candidates rather than dropped.
- **There is no `values` endpoint per filter.** The option lists live inline in
  `GET /api/dataset/issues`, under `columns[].filter.values.all`, for columns whose strategy is
  `AllAtOnce`. High-cardinality columns are `ByAsyncSearchOnly` and only reachable through the
  `element/{col}/search` typeahead.
- `SaleDate` carries future dates for proposed issues, so a descending sort surfaces deals that
  have not happened yet. `sold_status` separates them.
- Scope is California only. There is no national equivalent on a free path — MSRB/EMMA's terms bar
  building a redistributable database from it.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ca-debtwatch": {
      "url": "https://gateway.pipeworx.io/ca-debtwatch/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ca-debtwatch/mcp` returns the tools in the table
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
    "ca-debtwatch": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ca-debtwatch"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ca-debtwatch
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ca Debtwatch data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
