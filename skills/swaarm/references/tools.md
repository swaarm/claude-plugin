# Swaarm MCP tool reference

The MCP server advertises each tool's JSON Schema, so you can always see argument names and types. This file covers what the schema can't tell you: the intended use, the non-obvious input constraints, what to do with the output, and the known gotchas.

Read this when you're about to call a tool and want to avoid a surprise. Organized by workflow category (same grouping as the server's embedded instructions), then alphabetically within each.

## Table of contents

- [Listing & search](#listing--search)
  - `list_advertisers`, `list_publishers`, `list_offers`, `search`
- [Single-entity reads](#single-entity-reads)
  - `get_offer`
- [Reporting](#reporting)
  - `run_report`, `get_entity_stats`, `get_conversion_report`, `custom_report`
- [Financial operations](#financial-operations)
  - `update_payout`, `update_budget`
- [Offer management](#offer-management)
  - `update_offer`, `manage_offer_access`
- [Network adapters](#network-adapters)
  - `list_network_configs`, `list_network_adapters`, `get_network_config`, `create_network_config`, `update_network_config`, `delete_network_config`, `preview_network_config`
- [Audit & compliance](#audit--compliance)
  - `list_audits`
- [Raw data access](#raw-data-access)
  - `query_clickhouse`, `describe_clickhouse_tables`
- [Rewarded users](#rewarded-users)
  - `get_rewarded_user`, `list_rewarded_users`

---

## Listing & search

### `list_advertisers`, `list_publishers`, `list_offers`

All three accept `keyword`, `status[]`, `limit`, `offset`. `list_offers` additionally accepts `advertiser_ids[]` and `countries[]`.

- **Default limit is conservative.** If you expect more results than the first page, raise `limit` explicitly rather than paginating blindly.
- **Keyword matches name and ID.** Users commonly paste an ID into the keyword slot; this works.
- **Status filter uses the canonical enum values.** For offers: ACTIVE, PAUSED, PENDING, ARCHIVED, DELETED. Passing "active" lowercased may or may not work depending on server version — prefer uppercase.
- **`list_offers` with no filters can be slow on large networks.** Always add at least one filter (status, keyword, advertiser_ids) when you can.

### `search`

Cross-entity keyword search (offers + advertisers + publishers) in one call. Returns grouped results.

- Use this when the user says something like "find me everything matching 'Acme'" — it's faster than three separate list calls.
- The results are shallow (id + name). Use `get_offer`, `list_publishers`, etc. for full detail.

## Single-entity reads

### `get_offer`

The one detail-view tool. Required: `id`.

- Returns the offer plus its payouts and budgets inline — you don't need a separate call to enumerate them.
- Use this before calling `update_offer`, `update_payout`, `update_budget`, or `manage_offer_access` on the same offer. The echo is cheap, and it catches race conditions.
- If you need publisher access details for the offer, `get_offer` does not include them — query via `list_audits` with `types: [OFFER_ACCESS]` filtered to that offer, or ask the user which publisher.

(There is currently no `get_advertiser` or `get_publisher` — for those, use `list_advertisers` / `list_publishers` with the ID as `keyword`, or use `get_entity_stats` with `entity_type: ADVERTISER/PUBLISHER` if performance data is what you need.)

## Reporting

### `run_report` — aggregated reporting

The primary cross-entity performance report. Required: `start`, `end` (ISO datetimes), `metrics[]`.

**Metrics available** (common ones, not exhaustive):
- `CLICKS`, `CONVERSIONS`, `PAID_CONVERSIONS`
- `WE_GET` (revenue), `THEY_GET` (cost), `PROFIT`, `MARGIN`
- `CR` (conversion rate = CONVERSIONS/CLICKS), `EPC` (earnings per click = WE_GET/CLICKS)

**Dimensions** (optional; adds a `GROUP BY`):
- ADVERTISER, PUBLISHER, OFFER, COUNTRY, EVENT_TYPE — pick what the user's question needs

**Granularity** (optional time grouping): ALL, DAY, HOUR, WEEK, MONTH. Default is ALL (no time breakdown).

**Filters** (optional): per-dimension value lists.

**Gotchas:**
- Big date ranges with multiple dimensions can be slow. Narrow before you broaden.
- `PROFIT` and `MARGIN` are not visible to restrictive roles (Account Manager). If the report comes back with blanks on those columns, that's expected — not a bug. Explain it to the user.
- Time granularity + dimension groupings multiply. `DAY × OFFER × COUNTRY` over 30 days can return tens of thousands of rows. If that's what the user asked for, fine — but ask first if the intent was a summary.
- Sort with `sort_metric` + `sort_direction`. Default is no explicit sort.

### `get_entity_stats` — single-entity reporting

Preferred for "how is this one thing doing" questions. Required: `entity_type` (OFFER/PUBLISHER/ADVERTISER), `entity_id`, `start`, `end`, `metrics[]`.

- Cheaper than `run_report` for single-entity questions.
- Supports optional `dimensions` and `granularity` for sub-breakdowns (e.g., one offer's performance by country over days).
- Returns entity details alongside the report stats — saves a round trip.

### `get_conversion_report` — conversion-level reporting

Detail-level postback data. Required: `start`, `end`, `fields[]`.

- **Revenue/cost fields are in microcents.** Always convert and show units.
- Use `filters[]` for precise slicing (field, operator, values) — much more powerful than aggregated-report filters.
- Fields include POSTBACK_ID, OFFER_ID, PUBLISHER_ID, EVALUATION_STATUS, EVALUATION_DECISION, COUNTRY, DEVICE_*, IP, and more.
- Use this when the user says "investigate," "reconcile," "audit," "why didn't we get paid" — conversion-level is where the answer lives.

### `custom_report`

Flexible report wrapper. Parameters vary; consult the schema exposed by the server at runtime. Treat it as `run_report`'s more powerful cousin — reach for it when `run_report`'s shape doesn't fit, e.g., when mixing dimensions that `run_report` doesn't offer.

## Financial operations

### `update_payout`

Required: `offer_id`, and *some* combination of `we_get` / `they_get` (at least one). Optional: `publisher_id`, `event_type_id`.

**Which payout bucket this hits:**
- Neither `publisher_id` nor `event_type_id` → updates the offer's **default payout**
- `publisher_id` present → creates or updates a **publisher-specific payout** override
- `event_type_id` present → creates or updates an **event-specific payout** override
- Both present → publisher-and-event-specific payout

**Gotchas:**
- Amounts are in the user's display currency (not microcents here). If the user says "$1.50," pass `1.50`.
- Setting only `we_get` (no `they_get`) leaves cost unchanged — not zero. Same for `they_get` alone. This is idempotent merge.
- `margin` is auto-calculated; don't try to set it directly.
- **Always `get_offer` first** to show the user the current payout(s) before changing them. Echo → confirm → update.

### `update_budget`

Required: `offer_id`, `type`, `validity`, `amount`. Optional: `threshold` (default HARD), `publisher_id`, `advertiser_id`.

**Gotchas:**
- Offer-scope vs publisher-scope budgets are selected by whether `publisher_id` is present. Same tool, different target.
- **Threshold choice is the most common footgun.** HARD stops traffic; SOFT only blocks postbacks. If the user says "I want to cap spending at $100/day," default to HARD unless they specifically want the softer behavior.
- You cannot directly lower `consumed` — budgets reset at their validity boundary. If the user wants to "unstick" a tripped budget immediately, raise `amount` as an interim.
- `FOREVER` validity never resets. Use sparingly and be explicit with the user.

## Offer management

### `update_offer`

Required: `id`. All other fields optional — this is merge-on-write.

**Gotchas:**
- Status transitions have side effects. See `concepts.md` → Offer section. Moving to ARCHIVED removes budgets and landing pages; that's not reversible with a simple re-activation.
- `customRestrictions` and `note` are free text visible to publishers in the feed — treat them as customer-facing copy.
- `update_offer` does not change the advertiser (no re-parenting). It also doesn't touch payouts or budgets — use their dedicated tools.

### `manage_offer_access`

Required: `offer_id`, `publisher_id`, `state` (APPROVED / BLOCKED / PENDING).

- Upsert: creates the record if missing, updates if present.
- `BLOCKED` should be accompanied by a `blockingReason` when possible — the field is free-text and shows up in the publisher's feed. Don't leave it empty.
- Bulk approvals are a common workflow — loop through publisher IDs calling this tool. For batch clarity, confirm the full list with the user once, then run the batch.

## Network adapters

Seven tools in this family. All operate on NetworkConfigs (per-advertiser adapter instances), not on the adapter types themselves (which are enumerated via `list_network_adapters` and are read-only).

### `list_network_adapters`

No arguments. Returns the catalog of installable adapter types: HasOffers, Affise, Everflow, Offer18, etc. Use this to answer "what networks can we integrate with" and to validate an adapter name before `create_network_config`.

### `list_network_configs`

Returns existing NetworkConfig instances. Filters: `advertiser_id`, `keyword`, `syncing_active`, `limit`, `offset`.

- Use `syncing_active: true` to find only the live integrations.
- `keyword` matches config name.

### `get_network_config`

Full detail for one config. Required: `id`. Use before editing.

### `create_network_config`

Required: `advertiser_id`, `name`, `adapter`. Most remaining fields are adapter-specific (`apiConfig`, `trackingUrlParams`, `fieldSettings`).

- **Call `preview_network_config` first** before a create for adapters that would auto-import a lot of offers. Previewing avoids "oh no, we just pulled in 2,000 inactive offers."
- `syncingActive` defaults to false — new configs don't start syncing until you flip it.
- `margin` applied here overrides the offer's default for imported offers.

### `update_network_config`

Required: `id`. All other fields optional (merge-on-write).

- Use this to flip `syncingActive` on/off.
- Changes to `fieldSettings` affect subsequent syncs, not retroactive imports.

### `delete_network_config`

Required: `id`. Destructive. Confirm explicitly.

- Deletion does not remove offers that were already imported — it just stops the sync.

### `preview_network_config`

Dry-run of what would be imported. Use before create and before big config changes.

- Returns a preview list of offers without creating anything.
- Safe to call repeatedly while tuning `fieldSettings` / filters.

## Audit & compliance

### `list_audits`

The audit log. No required fields — everything's a filter.

**Useful filter dimensions:**
- `entity_ids[]` — direct targeting
- `types[]` — OFFER, ADVERTISER, PUBLISHER, PAYOUT, BUDGET, OFFER_ACCESS, ADVERTISER_CONFIG, AUTOMATION_RULE, and more
- `time_start`, `time_end`
- `changed_by` — user ID
- `offer_ids[]`, `advertiser_ids[]`, `publisher_ids[]` — lateral lookups (show all audits that touched entities related to this offer/advertiser/publisher)
- `is_robot[]` — `[true]` for automation-driven changes, `[false]` for human ones
- `undoable` — filter to changes Swaarm can technically roll back (no undo tool exists yet in MCP)

**Output:** list of audit entries with `fields: [{ name, previous, current }]` on UPDATE ops. This before/after pair is what makes `list_audits` so useful — you see exactly what changed.

**Patterns:**
- Before a mutation, `list_audits` on the target entity for the last 24h — checks for concurrent edits
- After a mutation, to confirm it landed and log the operator
- For diagnosis, filtered by `is_robot: [true]` to attribute auto-changes

## Raw data access

### `query_clickhouse`

Run read-only SQL against the tenant's ClickHouse (Dataxporter) tables. Required: `sql`. Optional: `limit` (default 1000, max 10000).

**This is escape-hatch territory.** Use it when the aggregated reporting tools can't express what the user needs — unusual groupings, complex joins, custom metrics. For routine reports, stick with `run_report` / `get_entity_stats` / `get_conversion_report` because they respect permissions and format consistently.

- SELECT only. The server enforces read-only.
- Tables are automatically scoped to the tenant.
- Use `describe_clickhouse_tables` first to find schemas before writing SQL.
- Output columns come back as strings — convert on display.

### `describe_clickhouse_tables`

Schema introspection. Use before writing a `query_clickhouse` SQL.

## Rewarded users

### `get_rewarded_user`, `list_rewarded_users`

Only relevant if the user runs Rewarded offers (see the [Rewarded API guide](https://docs.swaarm.com/en/articles/11745040-rewarded-app-development-guide-how-to-use-the-swaarm-rewarded-api)). These expose end-user reward-participation data, not the standard conversion reporting. If the user isn't running Rewarded, you won't need these.
