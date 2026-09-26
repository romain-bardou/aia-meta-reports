# AIA Meta Reports — Make.com Automation

This repo is a workspace for a set of **Make.com (Integromat) scenarios**, not a
traditional codebase. There is no application code here — the actual "code" lives
in Make.com scenario blueprints, edited live via the `mcp__Make__*` MCP tools.
This file captures context and hard-won lessons so future sessions don't
re-discover them the hard way.

## Project status

**Finished, on this personal/test Make workspace.** The user has recreated the
final pipeline in the company's own Make instance. The scenarios described below
exist in this workspace for reference/history only — do **not** reactivate them
or treat their `isActive: false` state as a problem to fix. If asked to touch
Make scenarios again, confirm with the user whether they mean this workspace or
the company one before assuming anything is broken.

## Make.com account (this workspace)

- Zone: `eu1.make.com`
- Organization: "My Organization" — id `7391173`
- Team: "My Team" — id `1535519`
- Get these via `mcp__Make__environment_get` if they ever change.

## Scenarios

| Scenario | ID | Purpose |
|---|---|---|
| Meta Ads - Collecte quotidienne | 7339570 | Daily Meta Ads performance data (facebook-insights) into Google Sheets, "Untitled" tab. |
| Meta Ads - Catalogues (voitures par set) | 7477181 | Daily product-set counts per catalog (facebook-catalogs `listProductSets`) into Google Sheets, "Voitures par catalogue" tab. |
| Meta Ads - Rapport Hebdo | 7354781 | Per-account Meta Ads performance narrative to Slack via `anthropic-claude:createAMessage`. Pre-existing; not modified this session beyond reading it as a reference pattern (router-per-account + TextAggregator + Claude + Slack). |
| Meta Ads - Rapport Quotidien | 7340180 | Older/simpler daily performance report variant. Not touched. |
| **Meta Ads - Rapport Catalogue Quotidien** | **7601111** | **Built and fully debugged this session.** Daily Slack report of catalog/product-set counts per account, with day-over-day deltas, written by Claude. This is the final, working version — see below for its exact format and construction. |

A prior session (different branch, different local clone) recorded having built a
scenario called "Meta Ads - Rapport Catalogue" (id 7573317). That scenario does
**not** exist in this team's scenario list — either it was never actually
persisted, or it was superseded. **7601111 is the real, current, tested one.**
No duplicate exists.

## Data source

Google Sheet `1b-K8h1Dv69IWznJcWcjinkqwxfrpQciT58pBEpTm2jo` ("Meta Ads - Collecte
quotidienne"), sheet tab **"Voitures par catalogue"**. Columns (0-indexed, as
referenced in Make formulas via `` {{moduleId.`N`}} ``):

- `0` — date, written as plain `"YYYY-MM-DD"` text (e.g. `2026-09-25`). Unlike
  the "Untitled" tab in the same spreadsheet, this column has **not** shown the
  Sheets auto-conversion-to-Date artifact (no stray time suffix) — plain
  `text:equal` against a `formatDate(now; "YYYY-MM-DD")` string is reliable here.
- `1` — account name (`Northshore VW`, `ALTO MG NORTHSHORE`, `ALTO OMODA JAECOO`)
- `2` — dealership/catalog display name (e.g. `Alto Volkswagen North Shore`)
- `3` — set name (e.g. `All products`, `All Vehicles`, or a named product set)
- `4` — product count in that set

The "total catalog" row per account is **not consistently named** — it's
`"All products"` for Northshore VW but `"All Vehicles"` for both MG and Omoda.
This has to be hardcoded per account/branch; there's no reliable generic rule
(the naive "it's the row with the max count" heuristic mostly holds but isn't
guaranteed — a real-world glitch this session saw ALTO OMODA JAECOO's
`All Vehicles` drop from 73 to 1 in one day, which is a legitimate data event to
report, not a bug to work around).

## Account / catalog / business IDs (from the existing collection scenarios)

| Account name | Ad Account ID | Catalog ID | Total-set name |
|---|---|---|---|
| Northshore VW | `act_189058208834173` | `263250252225004` | All products |
| ALTO MG NORTHSHORE | `act_615616278281772` | `1114156783953669` | All Vehicles |
| ALTO OMODA JAECOO | `act_2280337522378487` | `749620024334945` | All Vehicles |

Business Manager ID common to all three: `446755469247878`.

## Connections (IMTCONN ids, this workspace)

- Google Sheets/Drive: `6880131`
- Anthropic Claude: `10742354`
- Slack (bot): `10740939`, channel `#meta-ads-reporter`
- Facebook: `10739826` (not needed by the catalogue report scenario — it only
  reads the sheet the collection scenarios already populated)

## Rapport Catalogue Quotidien (7601111) — final design

One shared `google-sheets:filterRows` read of the whole "Voitures par catalogue"
sheet, then a `builtin:BasicRouter` with one route per account. Each route:

1. `util:TextAggregator` (feeder = the shared read) filtered to this account +
   today's date → builds `"set name: count"` lines for today.
2. `util:SetVariable2` (scope `"roundtrip"`) storing that aggregator's `.text` —
   needed only so the value stays selectable/referenceable in the Make UI's
   variable picker once a second aggregator later reuses the same upstream
   feeder (see the bug below); at the formula level `{{<aggregatorId>.text}}`
   would have worked fine too.
3. **Its own independent** `google-sheets:filterRows` (a second, separate read
   of the same sheet — not a reuse of the shared one), then a `TextAggregator`
   on *that* module filtered to this account + yesterday's date → builds
   yesterday's lines.
4. `anthropic-claude:createAMessage` (model `claude-sonnet-5`) given both text
   blocks and the account's hardcoded total-set name, instructed to: compute
   the header line's product/set counts, add the total's own delta in the
   header line if it changed, bullet only the individual sets whose count
   actually changed (`* set_name has now X products (+N/-N since yesterday)`),
   and write `No changes since yesterday.` when nothing changed.
5. `slack:CreateMessage` to `#meta-ads-reporter`, wrapping the account name in
   `*asterisks*` via `replace()` for Slack bold, matching the existing Rapport
   Hebdo convention.

Final message shape (English, per spec):

```
*Northshore VW*: 264 products in the catalog, 15 sets:

* Tayron has now 26 products (+1 since yesterday)
* Tiguan has now 59 products (-1 since yesterday)
```

or, if the total itself moved: `*ALTO OMODA JAECOO*: 1 products (-72 since
yesterday) in the catalog, 4 sets:` — or `No changes since yesterday.` when
literally nothing (including the total) changed.

## Hard-learned Make.com bugs found and fixed this session

**`max_tokens` / `temperature` on `anthropic-claude:createAMessage` must be
JSON numbers, not strings.** Writing `"max_tokens": "1500"` (quoted) passes
Make's structural blueprint validation but fails at actual execution with
`RuntimeError [400] max_tokens: Input should be a valid integer` from the
Anthropic API. `mcp__Make__validate_module_configuration` catches this
statically (`"Expected type 'number', got type 'string'"`) — always run it
before pushing a blueprint that sets these fields from raw JSON rather than
through the Designer UI.

**`util:SetVariable2` needs `"scope"` explicitly set in the mapper**, even
though its schema shows a default (`"roundtrip"`) and `validate_module_configuration`
reports it as an "applied default." That default is only applied for the
design-time validation call — omitting it from the actual pushed blueprint
causes a runtime `BundleValidationError: Validation failed for 1 parameter(s)`
(the missing required `scope` field) with no further detail exposed via the
API. Always include `"scope": "roundtrip"` (or `"execution"`) explicitly.

**`text:startswith` is not a valid Make filter operator** (at least not for a
Google Sheets text field in this context). Using it produces a filter
condition whose operator dropdown renders as the raw, untranslated i18n key
(`##filter.text:startswith.label`) in the Designer's filter inspector, and the
condition never matches anything (confirmed live: 0 bundles passed for every
account, for both "today" and "yesterday" conditions that used it). Use
`text:equal` for an exact string match instead — safe here since this sheet's
date column is a clean, unsuffixed `"YYYY-MM-DD"` string (see Data source
above). If a date column *does* have Sheets' auto-converted time-suffix
artifact, prefer a `time:greaterorequal` / `time:less` range instead of
`text:equal`/`text:startswith`.

**Two sequential Aggregator-type modules cannot both source from the same
upstream multi-bundle "feeder" module within one linear branch.** The first
aggregator (e.g. `util:TextAggregator` with `feeder: 1`) consumes/closes
module 1's bundle stream for that branch. A second aggregator placed right
after it, also configured with `feeder: 1`, cannot re-open that already-closed
loop — its filter conditions referencing `` {{1.`0`}} ``/`` {{1.`1`}} `` never
resolve (confirmed live via the Designer's filter inspector: the referenced
fields show as unavailable/empty, "n'arrive pas à les récupérer," guaranteeing
the filter never matches, silently, with no error). **Fix: give the second
aggregation its own independent upstream read** — a brand-new
`google-sheets:filterRows` module (or whatever the source is), not a reuse of
the already-consumed one. This cost 3 extra cheap search operations (one per
account branch) and fully resolved it. A previous session's notes described a
different-looking workaround (a *nested* aggregator, inside the per-row
iteration of a first search module, re-referencing that first module's id as
its own feeder) for a similar-sounding lookup problem — that pattern was not
re-verified this session; if resurrected, test it live before trusting it,
since the general rule above ("a closed loop can't be re-opened for a second
aggregation in the same branch") should still apply to it structurally unless
the nesting depth genuinely changes the scope.

**`google-sheets:filterRows` output field naming, confirmed via the
`rpcSheetOutput` RPC**: even with `"includesHeaders": true`, bundle fields are
named by **numeric column index as a string** (`"0"`, `"1"`, `"2"`, ...),
labelled in the UI as `"header name (A)"` etc., not by the header text itself.
Reference them as `` {{moduleId.`0`}} ``, `` {{moduleId.`1`}} ``, etc. — this
matches every existing scenario in this project and is not a legacy quirk.

**`scenarios_update` wholesale-replaces the blueprint** — no merging happens.
Always `scenarios_get` first, edit that exact JSON, and send the complete
result back.

**The Make Designer UI can silently overwrite an API-pushed change if the user
has the scenario open in their browser at push time** — this happened
repeatedly this session (confirmed via `lastEdit` timestamps ticking forward
seconds after a push, with the pushed content gone). After every
`scenarios_update`, re-fetch via `scenarios_get` and diff the persisted content
against what was sent; if it reverted, ask the user to close/reload the editor
tab before you push again, and re-push.

**Make can auto-deactivate a scenario after repeated execution errors**
(`metadata.scenario.maxErrors`, default `3` in this project's blueprints).
Several manual test runs failing in a row (from the two bugs above) appears to
have silently flipped `isActive` to `false` on 7601111 with no explicit
deactivate call from either the user or any session. Don't assume a scenario
being unexpectedly inactive means someone turned it off on purpose — check
recent execution history for a run of errors first.

## Process history: dead ends and open threads from a prior session

A parallel session (branch `claude/relaxed-ritchie-qy6w4w`, now superseded by
this file) worked the same problem independently and left its own notes before
this session's final, tested design was reached. Preserved here verbatim in
substance, even where superseded, abandoned, or unverified — including where it
disagrees with this file's own conclusions above:

- **A production incident from an earlier iteration: 204 operations in one run,
  Slack got rate-limited, and the channel was flooded with duplicate messages.**
  Their diagnosis: an unclosed multi-bundle module was placed directly before a
  `builtin:BasicRouter`, so its multiplicity multiplied across every branch, and/or
  two independent unclosed multi-bundle sources were feeding the same downstream
  chain. Their stated fix/rule: any multi-bundle module must live *inside* a
  single router branch and be *immediately* followed by its own closing
  aggregator before anything else in that branch runs. This session's design
  (shared read → router → per-branch aggregator immediately after) is
  consistent with that rule, but the incident itself was not reproduced or
  re-verified here — flagged as a real cost of getting this wrong, not a
  hypothetical.

- **Dead end: using `map()` / `join()` / `length()` directly on a module id**,
  expecting it to behave like an array of that module's bundle history. Their
  notes state this **throws `'N' is not a valid array'` at runtime** (a live
  execution error, not a static one) and that there is no array-emitting
  aggregator in Make's `util` app to make this work — `map()`/`join()` need a
  genuine array-typed value, which nothing in this scenario produces. Not
  re-tested this session; this session avoided the whole approach by using
  independent `TextAggregator`s and plain string comparison instead.

- **An alternative "lookup without arrays" pattern they used for the delta
  calculation**, structurally different from this session's fix for the
  two-aggregators-one-feeder bug: (1) a `filterRows` *inside* the branch,
  self-filtered via its own `filter` property to today's rows for that account
  minus the total row; (2) a **nested** `util:FunctionAggregator2`, with
  `feeder` set to the *original top-level* `filterRows` module id, filtered to
  yesterday + this account + `set == {{<today-row-module-id>.` `3` `}}` (the
  current iteration's own set name, referenced dynamically) — always emits
  exactly one bundle, so `ifempty(X.result; 0)` never loses data for a
  brand-new set; (3) a `TextAggregator` (feeder = the today's-rows `filterRows`)
  filtered to `delta != 0`, building the bullets. This was **not re-verified
  this session** and may or may not actually work — it superficially looks like
  it re-triggers the same "closed loop, same feeder" problem this session hit
  and fixed differently (see above), but the *nesting* (inside a per-row
  iteration rather than as a sibling module) might change the scope in a way
  that avoids it. If a future session wants the "nested aggregator" approach
  instead of "give every aggregation its own independent read" (this session's
  simpler, confirmed-working fix), test it live before trusting either the old
  claim or this session's skepticism of it.

- **A direct disagreement on `text:equal` reliability for the date column** on
  this exact sheet/tab. Their notes: *"This sheet has silently converted plain
  `YYYY-MM-DD` text cells into typed Date cells (sometimes with time suffixes)
  after repeated writes. Always use `time:greaterorequal` / `time:less` range
  comparisons for date filters — `text:equal` on a date string is fragile and
  has caused silent '0 rows found' bugs before."* This session's own findings
  (above) are the opposite: `text:equal` worked reliably once the *actual*
  bug — the invalid `text:startswith` operator — was fixed, and the sheet's
  date column was observed clean and unsuffixed at the time. Both observations
  may be true at different points in time (the sheet could drift between clean
  text and auto-converted Date cells after repeated writes, as their note
  describes). **If a "0 rows found" bug reappears on this date column after
  `text:equal` previously worked, check for the Date-conversion drift they
  describe before assuming the operator is wrong again.**

- **Their stated workflow preference**: build/edit the blueprint JSON in a
  Python script in the scratchpad directory first, then read that file back
  into context before pushing, specifically to reduce hand-transcription risk
  versus editing raw JSON directly in a tool call. This session edited JSON
  directly in each `scenarios_update` call instead (with `validate_blueprint_schema`
  / `validate_module_configuration` as the safety net) and that worked fine
  here, but the scratchpad-script approach is a reasonable alternative worth
  trying if a future blueprint edit is large enough that direct JSON editing
  becomes error-prone.

## Workflow for editing a scenario blueprint

1. `mcp__Make__scenarios_get` the current blueprint — never hand-transcribe
   from memory or from an old copy.
2. Edit the complete JSON, keeping every existing field you're not
   intentionally changing.
3. `mcp__Make__validate_blueprint_schema` (structural) and, for any module
   whose mapper you hand-wrote rather than copied from a known-good example,
   `mcp__Make__validate_module_configuration` (catches wrong types like the
   `max_tokens` string bug above) before pushing.
4. `mcp__Make__scenarios_update` with the complete blueprint.
5. `mcp__Make__scenarios_get` immediately after to verify persistence (guards
   against both your own mistakes and a concurrent Designer-tab overwrite).
6. Ask the user to test (manually, or via `scenarios_run` if inactive is
   acceptable to activate first) — live execution has surfaced real bugs
   (invalid operators, missing required fields, closed-loop aggregator reuse)
   that static validation alone did not catch.
