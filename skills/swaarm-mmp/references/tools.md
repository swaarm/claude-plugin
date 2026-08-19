# MMP tool notes — constraints and gotchas beyond the schemas

Organized per tool. Only the non-obvious parts; the schemas carry the basics.

## Apps

### list_apps
- Filters: keyword, advertiser_ids, tags, store_app_ids (finds the *parent* apps of store apps).
- Each node embeds its store apps (id, name, store, storeId) — usually saves a follow-up call.
- Supports `export_to_s3` (stable APP_ID sort).

### get_app
- Returns the event type catalogue (with mapping IDs) and store apps **including SDK/S2S tokens**
  (`token` values are real credentials — show them only when the user is doing integration work,
  don't echo them into summaries).
- This is the UI-equivalent event type list: soft-deleted event types are already filtered out.

## App event types

### list_app_event_types
- With `app_id`: uses the `app(id){eventTypes}` shape — excludes soft-deleted rows. Preferred.
- Without `app_id`: paginated global listing; may include soft-deleted rows. Use only for
  cross-app sweeps.

### create_app_event_type / update_app_event_type
- `mapping_id` must equal the SDK/S2S wire name exactly (case-sensitive). Before creating, check
  `list_sdk_test_events` for what the app actually sends — create the mapping to match reality,
  not the other way round.
- Update is fetch-then-merge; `types` replaces the whole classification list.
- Changing a mapping ID only affects events from that moment on.

### delete_app_event_type
- Not undoable via API; unmapped incoming events are dropped from mapped reporting. Never delete
  `__open` / `__reinstall`. Confirm with name + app first.

## Partners

### create_publisher
- Required: name, status, account_type, country, account_manager_id (a *user* ID — find via
  `search` or ask). Say "partner" in all user-facing text.
- After creating: the partner still needs offer access (`manage_offer_access`) and usually a
  postback link (UI) before traffic flows.

### update_publisher
- Fetch-then-merge: pass only what changes. `tags`/`allowed_sub_ids` REPLACE the current lists.
- It carries billing details / referral / signup-form linkage over automatically — don't worry
  about wiping those, but DO check `list_audits` for concurrent edits first.
- Block = `status: BLOCKED` (reversible). Prefer it over delete in almost every case.

### delete_publisher
- Destructive and final; tracking links die. Echo name + ID and confirm.

## App analytics reports

Shared inputs for activity/cohort/stickiness/funnel: `start`/`end` (ISO datetimes),
`app_ids`/`store_app_ids`, `group_by` + `filters` over the AppReportDimension set
(APP_*, PUBLISHER_*, POSTBACK_DEVICE_*, POSTBACK_GEO_*, OFFER_*, INSTALL_DAY/WEEK/...),
`limit`/`offset`. Dimension values come back as typed union records
(`stringValue`/`intValue`/...) — unwrap before showing.

### activity_report
- Metrics: USERS (active users in the bucket), INSTALLS (new users), SESSIONS,
  SESSIONS_PER_USER, EVENTS, REVENUE_SUM/COUNT, COST_SUM/COUNT, ARPU, ROAS.
- `granularity`: MINUTE/HOUR/DAY/WEEK/MONTH (defaults DAY). Rows carry a `date`.
- Returns `csvDownloadUrl` — hand it over when the user wants the full data.

### cohort_report
- ONE `metric` per call (RETENTION_PERCENT/RETENTION_USERS/REVENUE_*/COST_*/ROAS/SESSIONS/
  SESSIONS_PER_USER/EVENTS), cohorted by install date.
- D0–D7 retention = `granularity: DAY, number_of_periods: 8, metric: RETENTION_PERCENT,
  value_type: ORIGINAL_VALUES`. Cohorted revenue/ROAS usually reads better CUMULATIVE.
- `min_cohort_users` suppresses noisy tiny cohorts — suggest ~50+ when grouping.
- Rows have `periods[]` with `number` (0 = install period). Also returns `csvDownloadUrl`.

### stickiness_report
- Metrics: DAU, TRAILING_MAU (trailing 30-day), STICKINESS (= DAU / trailing MAU). Daily rows.
- No CSV link is served for this report — page through instead.

### funnel_report
- `event_sequence` = app event type **IDs** in order (get them from `list_app_event_types`),
  `window_seconds` bounds step-to-step time.
- Metrics: USERS (reached step), RATIO_OF_USERS (conversion from previous step). Rows are steps
  (`completedSteps`, `appEventType`). No CSV link served.

## Row-level and debugging

### get_conversion_report (as the MMP event log)
- Scope with `app_ids`/`store_app_ids`; select the APP_* fields. Key MMP fields:
  - `APP_EVENT_ID`/`APP_EVENT_NAME` — which in-app event
  - `APP_VENDOR_USER_ID` — the App User ID (device vendor ID); join key to user_journey_report
  - `APP_CUSTOM_EVENT_VALUE`/`APP_CUSTOM_EVENT_NUMERIC_VALUE` — the event's value
  - `ATTRIBUTION_PARTNER_AGGREGATED_EVENT_VALUE` — running aggregated value for the user
  - `POSTBACK_CTIT` — click-to-install seconds (near-zero or huge → fraud suspicion)
  - `ATTRIBUTION_PARTNER_*` — install/click times, sale amount + currency, match type,
    retargeting, SDK/app version, rejection reason
  - `SKAD_POSTBACK_*` — SKAdNetwork postback details
- Money fields are microcents (÷100,000,000). Exports force POSTBACK_ID ASC (deterministic).

### custom_report (MMP mode)
- App dimensions: APP_ID/APP_NAME, STORE_APP_ID/STORE_APP_NAME, STORE, APP_EVENT_TYPE_ID/NAME.
- App metrics: **ACTIVE_USERS** (active users), **NEW_USERS** (deduplicated
  installs), SESSIONS, EVENTS, ARPU, ROAS, SALE_AMOUNT, APP_CUSTOM_NUMERIC_VALUE,
  APP_AVERAGE_CUSTOM_NUMERIC_VALUE.
- `app_event_type_ids` (max 50) adds one approved-count column per in-app event — the way to get
  "installs vs registrations vs purchases per partner" in one table. MMP-enabled deployments only.
- To scope by event, filter on dimension APP_EVENT_TYPE_ID; to scope by app, APP_ID/STORE_APP_ID.

### user_journey_report
- Give `user_id` = the device/vendor ID (from APP_VENDOR_USER_ID or list_sdk_test_events);
  or scope app + `app_event_ids` for a small cohort.
- Steps are impressions/clicks/events in time order, `install: true` marks the install, and
  pre-attribution touchpoints are included — perfect for "which partner actually drove this".
- MMP-enabled deployments only; a GraphQL error on older deployments is expected.

### list_sdk_test_events
- Only events sent with a **test** token appear. A `null eventType` on a row = the incoming event
  name matched no mapping ID → compare against `list_app_event_types` and fix the mapping.
- `vendorId` from a row feeds `user_journey_report`.

## Shared tools used the MMP way

- `search` — name → ID resolution across entities; use before anything else when the user gives
  names.
- `run_report` — quick aggregates with STORE_APP_ID/APP_ID dimensions and ACTIVE_USERS/NEW_USERS/
  SESSIONS/ARPU/ROAS metrics; simpler than custom_report for one-liners.
- `list_audits` — who changed the partner/app/event type and when; check before mutating.
- `manage_offer_access`, `update_payout`, `update_budget` — the offer-side controls; see the
  `swaarm` skill for their full semantics.
- `export_to_s3` on list/report tools for anything > ~200 rows or headed to a spreadsheet.
