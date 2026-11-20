# @pipeworx/ny-medicaid-fee-schedule

New York State Medicaid practitioner fee schedules, statewide, per procedure code: the fee for a CPT or HCPCS code (and dental CDT code) with the manual and section it comes from and that schedule's effective date. From the New York State Department of Health, Medicaid provider manuals (eMedNY).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2015+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ny_fee_lookup(code, manual?)` — the fee(s) for one 5-character procedure code: non-facility, facility and professional-component fees (physician manual) or the single fee (laboratory, DME, dental), anesthesia base units, by-report and prior-approval flags, follow-up days, the manual, section and effective date. A code listed in more than one schedule returns every row. An unmatched code returns the nearest codes, never a bare empty.
- `ny_fee_search(query?, code_prefix?, manual?, limit?)` — search by description words (all must appear) and/or code prefix, optionally within one schedule. An unmatched query returns the rows matching some of its words and the valid manuals.
- `ny_fee_manuals()` — the schedules the pack carries, each with its effective date, when the workbook was posted, its row count and the publisher's footnotes, plus how to read the fee columns and what is not covered.

## Auth

Keyless.

## Data sources

- New York State Department of Health, Medicaid provider manual fee schedules (eMedNY) — the Physician manual (medicine, drugs, radiology, surgery, anesthesia), Laboratory, DME and Dental fee schedules. Public, no account. Statewide fee-for-service fees per procedure code.

Things worth knowing:

- **This is the statewide practitioner fee schedule per procedure code.** It is not clinic or FQHC rates (those are per facility), not hospital outpatient or facility payment rates, and not what a Medicaid managed-care plan pays.
- **Eight schedules, each with its own effective date**: physician medicine, drugs, radiology and surgery, physician anesthesia, laboratory, DME and dental. The dates differ (the 2026-11 bake spans 2026-01-01 to 2026-10-01), so every row carries its own `effective_date`; `data_as_of` is the newest one.
- **Not covered**: Physician section 6 (the Enhanced Program fee schedule for HIV, MOMS and PPAC rate groups, whose fees are free text per rate group on three differently laid-out sheets) and every other manual's schedule (transportation, home care, nursing facilities, hospital outpatient clinics and so on). A CPT code the state does not price in these workbooks is absent (for example 36415).
- **No modifier column.** The workbooks list one row per code. A trailing modifier in the lookup is dropped and reported as `modifier_ignored`.
- Fee columns follow the publisher's headings: `fee_non_facility` ("non-facility global fee"), `fee_facility` ("facility global fee"), `fee_professional_component`, and `fee` for the single-fee schedules. `anesthesia_base_units` are base units, not dollars. Some fees carry four decimals because the state computes them that way; they are kept as published.
- **`fee_note`**: `cost` means the schedule prints the word cost instead of a fee (most drugs, and some radiology supplies), `BR` means by report, `NA` is printed as such. A row with no fee and no note is marked `no fee published`.
- `prior_approval` is the publisher's PA column: `PA`, `DVS`, or a numeric prior-approval type. The DME schedule's numeric types (1, 4, 6, 9) and the dental schedule's free-text conditions are kept as printed; a 0 means none and is omitted. `follow_up_days` is the physician manual's follow-up-days column (the dental schedule's post-op days). `change_flag` is the publisher's change mark.
- A footnote marker printed in front of a code in the source (`^G9012`, `#T2022`) is kept as `code_marker`; the manual's footnotes are in `ny_fee_manuals`.
- The physician, laboratory and DME descriptions are truncated by the publisher to 40 characters.
- Duplicate codes inside a schedule (DME has a few catch-all codes priced item by item) appear as separate rows with their own descriptions.
- Identifiers are strings; a leading zero is part of the code (`00840`).
- Every response carries `data_as_of`, `captured_at` and the source.

## Stored data and refresh

The state publishes these schedules only as Excel workbooks, one per manual, with no query interface, so they are read by `scripts/bake-index.mjs` into `src/ny-fee-data.ts` (about 11,700 rows, 1.8 MB), registered in `workers/gateway/src/pack-baked-indexes.json`; the deploy uploads it to KV and the gateway injects it on the first call into this pack, so nothing is evaluated at module scope. The legacy workbooks are read at bake time only, by a small dependency-free reader (`scripts/xls-reader.mjs`, checked cell for cell against xlrd on every workbook); nothing in the Worker parses a spreadsheet. Refresh with `node mcps/ny-medicaid-fee-schedule/scripts/bake-index.mjs`: it fetches the eight workbooks one at a time with a pause between, and refuses to write when a response is not a workbook, a header row changed, a fee is neither a number nor the publisher's own `cost` / `BR` / `NA`, a code is not five characters, a row is duplicated, an effective date is missing, a schedule falls below its row floor or more than 10% under the previous bake, or the index would exceed 25 MB. `--from-dir <dir>` bakes from already-downloaded files. The state revises each schedule a few times a year, so the stored copy can trail by a quarter; `effective_date` per row and `data_as_of` say which.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ny-medicaid-fee-schedule": {
      "url": "https://gateway.pipeworx.io/ny-medicaid-fee-schedule/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ny-medicaid-fee-schedule/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2015+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ny_fee_lookup \
  -H 'Content-Type: application/json' \
  -d '{"code":"99213"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ny_fee_lookup`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-ny-medicaid-fee-schedule` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/ny-medicaid-fee-schedule/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "ny-medicaid-fee-schedule": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ny-medicaid-fee-schedule"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ny Medicaid Fee Schedule data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
