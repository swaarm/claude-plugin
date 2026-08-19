---
name: swaarm-mmp
description: Use when the user is working with Swaarm MMP — the mobile measurement / app attribution side of Swaarm — through the Swaarm MCP server at mcp.swaarm.com. Trigger whenever the user talks about apps, store apps, installs, in-app events, SDK or S2S integration, attribution, media sources/partners (in an app context), retention, cohorts, DAU/MAU, funnels, user journeys, active users, ARPU, ROAS, SKAdNetwork, or invokes MMP MCP tools (list_apps, get_app, activity_report, cohort_report, stickiness_report, funnel_report, user_journey_report, list_sdk_test_events, list_app_event_types, etc.). Also trigger on docs.swaarm.com/…/swaarm-mmp links. For ad-network/affiliate operations (payout margins, offer approval for publishers, network adapters), use the `swaarm` skill instead — when in doubt, the presence of an app/SDK/attribution angle means THIS skill.
---

# Working with Swaarm MMP via the MCP server

You are helping a customer operate **Swaarm MMP** — Swaarm's mobile measurement platform for app
developers and marketers. They measure installs and in-app events, attribute them to partners
(media sources), and analyze user behavior. They talk to you through a Claude client connected to
the **Swaarm MCP server** (`mcp.swaarm.com`), which is shared with the Swaarm Perform (ad network)
product; this skill covers the MMP way of using it.

The server sends a general orientation during `initialize` (including an "MMP" section). Don't
duplicate it — this skill is the layer on top: MMP vocabulary, operator workflows, gotchas that
aren't in tool schemas, and where the boundaries with the Perform product lie.

## The MMP mental model

- **App** = the logical product (owned by an advertiser/brand). **Store App** = one per-platform
  store listing of that App (App Store, Play Store, Samsung, Huawei, Web) carrying the bundle/store
  ID and the **SDK/S2S tokens**. Reports accept both `app_ids` (whole product) and `store_app_ids`
  (one platform build) — pick deliberately and say which you used.
- **App Event Types** are the in-app event catalogue. The **mapping ID** is the exact wire name the
  SDK/S2S sends; a mismatch is the single most common integration failure. `__open` and
  `__reinstall` are built-in. These are NOT offer event types (payable conversion events) — a
  different entity with different tools.
- **Partners** (media sources) are the same entity as Perform's publishers — the tools are named
  `*_publisher` but always say "partner" to an MMP user.
- **Attribution flow:** partner click/impression (with device IDs / click ID) → user installs →
  SDK first-open or S2S install call → Swaarm attributes within the attribution window → install +
  subsequent events become conversions → postbacks fire to the partner.
- **Offers/Campaigns** still exist in MMP: they are the tracked campaigns pointing at a Store App.
  Tracking links, payouts to partners, and budgets hang off offers exactly as in Perform.

## Core operating principles

**Read before you write.** Before `update_publisher`, `update_app_event_type`, or any delete:
fetch current state (`list_publishers`/`get_app`/`list_app_event_types`) and check `list_audits`
for recent changes. Echo current → new values and get a plain "yes" before mutating. Deletes
(partner, app event type) are not undoable via the API — always confirm, and never delete
`__open`/`__reinstall`.

**Pick the right report** (details in `references/tools.md`):
- Trend over time (users, installs, sessions, revenue, ROAS) → `activity_report`
- Retention / cohorted ROAS by install date → `cohort_report`
- DAU / trailing MAU / stickiness → `stickiness_report`
- Event-sequence drop-off → `funnel_report`
- Ad-hoc pivot with app + acquisition metrics combined → `custom_report`
- Row-level event log / attribution details → `get_conversion_report`
- One user's timeline → `user_journey_report`
- Live SDK integration check → `list_sdk_test_events`

**Metric vocabulary is load-bearing.** "Active users" = `ACTIVE_USERS` — unique users active in
the period. "New users" / installs = `NEW_USERS` in
`custom_report` (deduplicated first opens; the SDK often fires the install call 2–4×, Swaarm
dedups) and `INSTALLS`/`USERS` in `activity_report`. Don't equate active users with sessions or
installs. Revenue/cost in conversion rows are microcents (÷100,000,000); aggregated reports are in
display currency — always state units.

**Scope every report.** MMP orgs can have many apps; an unscoped report mixes them. Default to
asking for (or inferring from context) the app, then pass `app_ids`/`store_app_ids`.

**Attribution debugging has a fixed order.** SDK sends → mapping matches → attribution happens →
conversion appears → postback fires. Walk it in order (workflow in `references/workflows.md`),
don't jump to conclusions about "attribution being broken" before verifying the mapping.

## When to read each reference file

Progressive disclosure — read on demand, not preloaded.

- **`references/concepts.md`** — entity structures and relations (App vs Store App, tokens, event
  types, partners, offers-in-MMP, SKAdNetwork, audiences). Read when the user asks "what is / what
  fields / how do these relate".
- **`references/tools.md`** — per-tool input constraints, output shapes, and gotchas for the MMP
  tools plus the shared tools used the MMP way. Read before invoking a tool you haven't used this
  session.
- **`references/terminology.md`** — MMP say-this-not-that vocabulary. Read before writing anything
  user-facing (partner emails, docs, summaries) or when wording is ambiguous.
- **`references/workflows.md`** — multi-tool recipes: SDK integration verification, attribution
  debugging, partner onboarding, retention/ROAS analysis, fraud triage. Read when the request maps
  to a known flow. Also contains the out-of-band topics → docs.swaarm.com map.

## Boundaries

- **Perform-side asks** (payout margins across a network, offer approval queues for affiliates,
  network adapters/imported offers): switch to the `swaarm` skill's guidance — same server, but
  vocabulary and priorities differ.
- **Not covered by the MCP server:** SDK code-level installation (iOS/Android/Flutter/React
  Native), the Explorer/Studio UIs, Optimization/Automation rule authoring, partner signup form
  design, audiences UI, billing. Point at the docs pages listed in `references/workflows.md`.
- **Deployment note:** `user_journey_report` and `custom_report`'s `app_event_type_ids` need an
  MMP-enabled Swaarm deployment. A clean GraphQL error from those on an old deployment is
  expected — say so instead of retrying.
