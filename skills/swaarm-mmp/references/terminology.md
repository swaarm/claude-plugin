# Swaarm MMP terminology — say this, not that

Read before writing anything the user will see externally, or when their wording is ambiguous.
The MMP docs and UI have their own vocabulary; using it correctly matters more here than on the
Perform side because the tools reuse Perform names underneath.

## Entities

| Say | Not | Notes |
|-----|-----|-------|
| **Partner** (media source) | publisher, affiliate | The MCP tools are named `*_publisher` but MMP users say partner. Mirror the user; default to partner. |
| **App** | product, title | The logical product. |
| **Store App** | app version, listing, bundle | The per-platform build. "Store app" is two words in the docs. |
| **Brand** / **Advertiser** | client, merchant | The UI nav says Brands; the API says Advertiser. |
| **Offer / Campaign** | ad, placement | MMP docs use both; either is fine, be consistent within a reply. |
| **(App) Event** / **Event Type** | goal, action, conversion event | The in-app event. Its **mapping ID** is the wire name — never call it "event key" or "token". |
| **In-app event** | custom event | when distinguishing from the install |
| **Install** | first open, download | Attribution-wise the install IS the first open; "download" is wrong (store downloads aren't measured). |
| **Postback** | webhook, callback, pixel | How conversions reach the partner. |
| **Tracking link** | click URL, redirect | |
| **SDK Debug Report** | test console, sandbox | The UI page backed by `list_sdk_test_events`. |
| **User Journey** | timeline, event stream | The per-user report. |
| **Attribution window** | lookback window | |
| **SKAdNetwork / SKAd** | SKAN (ok informally), Apple attribution | Docs write SKAdNetwork. |

## Metrics

| Say | Meaning | API name |
|-----|---------|----------|
| **Active users** | unique users active in the period | `ACTIVE_USERS` (custom/run report), `USERS` (activity report) |
| **New users** / installs | deduplicated first opens | `NEW_USERS` (custom/run report), `INSTALLS` (activity report) |
| **Sessions** | app opens (`__open` events) | `SESSIONS`, `SESSIONS_PER_USER` |
| **Events** | in-app events other than the default/install | `EVENTS` |
| **Retention** | % or count of the install cohort active in period N | `RETENTION_PERCENT` / `RETENTION_USERS` (cohort report only) |
| **Stickiness** | DAU ÷ trailing MAU | `STICKINESS` |
| **ARPU** | revenue ÷ active users | `ARPU` |
| **ROAS** | revenue ÷ cost | `ROAS` |
| **Revenue / Sale amount** | user-generated revenue (purchases) | `REVENUE_SUM`, `SALE_AMOUNT` |
| **Cost / Spend** | what's paid to partners | `COST_SUM`, TheyGet-side metrics |
| **CTIT** | click-to-install time (seconds) | `POSTBACK_CTIT` |
| **Aggregated value** | running total of an event's value per user | `ATTRIBUTION_PARTNER_AGGREGATED_EVENT_VALUE` |
| **App User ID** | per-device vendor ID of the app user | `APP_VENDOR_USER_ID` |

Conversion-row money is in microcents (÷100,000,000); aggregated reports are display currency.
Always state units.

## Statuses and enums (case-sensitive in the API)

- Partner status: ACTIVE, BLOCKED, REJECTED, PENDING, DELETED
- Partner account type: DIRECT, NETWORK, MEDIA_BUYER, FBMARKETER, SMARTLINK_PUBLISHER
- Event classification: ACQUISITION, MONETIZATION, QUALITY
- Store: APP_STORE, PLAY_STORE, SAMSUNG_STORE, HUAWEI_STORE, NO_STORE, WEB
- Token type: SDK, S2S (plus `test` flag)
- App report granularity: MINUTE, HOUR, DAY, WEEK, MONTH
- Journey sources: IMPRESSION, CLICK, EVENT
- Evaluation status/decision: APPROVED/PENDING/REJECTED, PASSED/FAILED

## Mirroring the user

If the user says "media source", "channel", or "network" for a partner, you may mirror it once
but anchor on "partner" — that's what their UI says. If they say "publisher", they may be an
operator who also runs Perform; keep their word. Never mix "publisher" and "partner" for the same
entity within one reply.
