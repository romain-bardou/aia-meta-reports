# AIA Meta Reports — Make.com Automation

This repo is a workspace for a set of **Make.com (Integromat) scenarios**, not a
traditional codebase. There is no application code here — the actual "code" lives
in Make.com scenario blueprints, edited live via the `mcp__Make__*` MCP tools.
This file captures context and hard-won lessons so future sessions don't
re-discover them the hard way.

## Make.com account

- Zone: `eu1.make.com`
- Organization: "My Organization" — id `7391173`
- Team: "My Team" — id `1535519`
- Get these via `mcp__Make__environment_get` if they ever change.

## Scenarios

| Scenario | ID | Purpose |
|---|---|---|
| Collecte quotidienne | (not touched this session) | Daily Meta Ads data collection into Google Sheets |
| Rapport Hebdo | 7354781 | Sends the daily Meta Ads performance report to Slack (uses `anthropic-claude:createAMessage` for narrative). User modified its schedule to run **daily** (not weekly) in the company's Make instance. Keep this scenario in its clean, catalog-free state — the catalog report was deliberately split into its own scenario (see below) to avoid coupling catalog bugs to this Claude-dependent report. |
| Meta Ads - Rapport Catalogue | 7573317 | Standalone scenario reporting catalog size (product/set counts) per account to Slack. Built from scratch this session. Currently `isActive: false` — activate or run manually to test. |

## Data source

Google Sheet `1b-K8h1Dv69IWznJcWcjinkqwxfrpQciT58pBEpTm2jo`, sheet tab
**"Voitures par catalogue"**. Columns (0-indexed, as referenced in Make formulas):

- `0` — date (e.g. `2026-09-24`)
- `1` — account name (`Northshore VW`, `ALTO MG NORTHSHORE`, `ALTO OMODA JAECOO`)
- `2` — dealership display name (e.g. `Alto Volkswagen North Shore`)
- `3` — set name (e.g. `All products`, or a named product set)
- `4` — product count in that set

The row where `set == "All products"` holds the **total catalog size** for that
account/day. All other rows are individual product sets; "sets" count for
reporting purposes **excludes** the `All products` row.

⚠️ This sheet has silently converted plain `"YYYY-MM-DD"` text cells into typed
Date cells (sometimes with time suffixes) after repeated writes. **Always** use
`time:greaterorequal` / `time:less` range comparisons for date filters —
`text:equal` on a date string is fragile and has caused silent "0 rows found"
bugs before.

## Connections (IMTCONN ids)

- Google Sheets: `6880131`
- Slack: `10740939`
- Slack channel: `#meta-ads-reporter`

## Hard-learned Make.com semantics

**Module bundle multiplicity.** Check a module's `_annotations.returnsMultipleBundles`
via `mcp__Make__app-module_get` before assuming its behavior:
- Search-type modules (e.g. `google-sheets:filterRows`, labelled "Search Rows")
  return **multiple bundles** — one per matching row. Everything downstream of
  them re-executes once per bundle until something "closes" the multiplicity.
- Aggregator-type modules (`util:FunctionAggregator2` — sum/count/avg/min/max,
  `util:TextAggregator`, `util:AggregateAggregator` table aggregator) always
  return **exactly one bundle**, regardless of how many input bundles they
  consumed (even zero — the result field is then just empty/undefined, not an
  error). These are the only reliable way to "close" a multi-bundle stream.
- **There is no array-emitting aggregator in Make's `util` app.** Do not use
  `map(moduleId; ...)`, `join(moduleId; ...)`, `length(moduleId; ...)` directly
  on a module id expecting it to behave like an array of that module's bundle
  history — **this is invalid and throws `'N' is not a valid array'` at
  runtime** (confirmed by live execution error this session). `map()`/`join()`
  only work on genuine array-typed values, which nothing in this scenario
  currently produces.

**The cartesian-multiplication trap (caused a real production incident: 204
operations in one run, Slack rate-limited, channel flooded with duplicates).**
Never let two independent unclosed multi-bundle sources feed the same
downstream chain, and never place an unclosed multi-bundle module directly
before a `builtin:BasicRouter` — it multiplies across every branch. Safe
pattern used throughout this project: any multi-bundle module lives **inside**
a single router branch and is **immediately** followed by its own closing
aggregator before anything else in that branch runs.

**Lookup pattern without arrays (used for "delta vs. yesterday" per set).**
To compare "today's" per-row value against "yesterday's" matching row without
a real array type:
1. A `filterRows` inside the branch, filtered (via its own `filter` property,
   self-referencing its own module id) to today's rows for this account,
   excluding `All products` — emits one bundle per set.
2. A **nested** `FunctionAggregator2` (feeder = the *original* top-level
   `filterRows`, e.g. module `1`) filtered to yesterday + this account +
   `set == {{<today-row-module-id>.\`3\`}}` (the current iteration's own set
   name, referenced dynamically) — always emits exactly one bundle (0 or the
   real count), so no data is lost for brand-new sets. Wrap with
   `ifempty(X.result; 0)`.
3. A `TextAggregator` (feeder = the today's-rows `filterRows` module) filtered
   to `delta != 0`, building the bullet lines.

**Filter conventions.** A module's blueprint `filter` property's condition
`"a"` fields reference the id of whatever module the filter is conceptually
attached to:
- For an aggregator, that's its own `feeder` parameter's module id (the
  upstream module being aggregated/filtered as it streams in).
- For a plain search module (like `filterRows`) filtering its own emitted
  rows before passing them downstream, conditions self-reference **its own**
  module id.

**`scenarios_update` wholesale-replaces the blueprint** — no merging happens.
Always `scenarios_get` first, edit that exact JSON, and send the complete
result back. Never hand-transcribe a large blueprint into the tool call without
diffing the source file against what actually persisted afterward — manual
transcription has caused subtle bugs before (e.g. a stray self-reference where
a sibling module's id was meant).

**Trust but verify, every push.** The Make Designer UI can silently overwrite
an API-pushed change if the user has the scenario open in their browser at
push time. After every `scenarios_update`, re-fetch via `scenarios_get` and
confirm the persisted content actually matches what was sent — don't assume
success from the update call's response alone.

## Workflow for editing a scenario blueprint

1. Build/edit the blueprint JSON in a Python script in the scratchpad
   directory (reduces manual-transcription risk versus hand-editing raw JSON).
2. Read the generated file back before pushing, to get its exact content in
   context.
3. `mcp__Make__scenarios_update` with the complete blueprint.
4. `mcp__Make__scenarios_get` immediately after to verify persistence.
5. Ask the user to run/test, since live execution has surfaced real bugs
   (invalid `map()` usage, self-reference typos) that static review missed.
