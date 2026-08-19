# Swaarm MMP — entity glossary

The entities as the MMP product models them, in Swaarm's vocabulary, with the relations you need
to navigate them via the MCP tools.

## App and Store App

**App** — the logical product. Owned by exactly one **Advertiser** (shown as "Brand" in the MMP
UI). Carries: name, tags, the in-app **event type catalogue**, and one or more Store Apps.
`list_apps` / `get_app`.

**Store App** — one store listing / platform build of an App:

- `store`: APP_STORE, PLAY_STORE, SAMSUNG_STORE, HUAWEI_STORE, NO_STORE, WEB
- `storeId`: the bundle ID (iOS) or package name (Android) — how attribution matches the app
- `storeUrl`, icon, description, screenshots (can be auto-filled by scraping the store)
- **Tokens** — the credentials the app integration uses:
  - `SDK` tokens: embedded in the mobile SDK init
  - `S2S` tokens: for server-to-server event calls
  - `test: true` tokens route events into the **SDK Debug Report** (`list_sdk_test_events`)
    instead of production data
- iOS integration credentials (App Store Connect key) for SKAdNetwork / Apple flows
- Facebook install-referrer decryption key, per-store-app SAN integrations

Rule of thumb: users say "my app" meaning the App; tracking/attribution mechanics and SDK setup
live on the Store App. `conversionReport`, activity/cohort/stickiness/funnel all accept both
`app_ids` and `store_app_ids` — these are sugar for filters on APP_ID / APP_STORE_APP_ID.

## App Event Type

The catalogue entry for one kind of in-app event:

- `name` — display name (max 40 chars), e.g. "Purchase"
- `mappingId` — **the wire name the SDK/S2S event sends**. Must match the integration exactly,
  including case. This is the most commonly misconfigured field in the whole product.
- `types` — classification, any of ACQUISITION, MONETIZATION, QUALITY
- Built-ins: `__open` (app open / session start) and `__reinstall`. Never delete or re-map them.

Soft-deleted event types disappear from `get_app`'s `eventTypes` but may still appear in the
unscoped `list_app_event_types` listing — prefer passing `app_id`.

**Do not confuse with offer Event Types** (payable conversion events attached to an Offer,
managed by `create_event_type`/`update_event_type`). An in-app event only becomes a payable
conversion through the offer's event configuration.

## Partner (Publisher)

The media source driving installs. Same entity as Perform's publisher — MCP tools are named
`*_publisher`, but always say "partner" to MMP users.

- Required at creation: name, country, status, account type, account manager user ID
- `accountType`: DIRECT, NETWORK, MEDIA_BUYER, FBMARKETER, SMARTLINK_PUBLISHER
- `status`: ACTIVE, BLOCKED (suspends traffic, reversible), REJECTED, PENDING, DELETED
- The GraphQL input's required name field is the legacy `company`; the MCP tools set both from
  the single `name` argument
- Carries: postback links (how conversions are sent back to the partner), API keys (Feed/Stats),
  allowed sub IDs, tags, billing/payment details, SAN integrations (Apple Search Ads, Facebook,
  Google, TikTok)
- `update_publisher` is fetch-then-merge: pass only changed fields; list fields (tags,
  allowed_sub_ids) replace the whole list
- Deleting is destructive; blocking is the reversible alternative

## Offer / Campaign (in MMP context)

The tracked campaign a partner runs for a Store App. MMP docs say "Offers / Campaigns". Still the
Perform entity underneath: leadflow (usually CPI for installs, CPA for events), tracking link,
payouts (WeGet from the advertiser side, TheyGet to the partner), budgets, targeting, and
per-partner access (`manage_offer_access`: APPROVED/BLOCKED/PENDING). An offer points at its app
via `storeAppId`/`appStoreId` on the offer.

## Attribution

- Click/impression arrives with partner parameters (click ID, device IDs, sub IDs)
- Install = first SDK open (or S2S install) for that device; Swaarm dedups the SDK's repeated
  install calls — that's why `NEW_USERS` (unique) can be lower than raw install postback counts
- Attribution windows are configured per offer (click/view-through); `POSTBACK_CTIT` (click-to-
  install time, seconds) is the key sanity/fraud signal
- The per-user identifier in conversion rows is `APP_VENDOR_USER_ID` (device vendor ID, e.g.
  IDFV); it is the join key into `user_journey_report`
- **SKAdNetwork**: iOS privacy-preserving attribution. Swaarm stores SKAd networks (org level),
  per-offer campaign mappings and postback links; conversion rows expose `SKAD_POSTBACK_*` fields
  (conversion value, campaign, fidelity, won, valid)

## Values on events

- `APP_CUSTOM_EVENT_VALUE` / `APP_CUSTOM_EVENT_NUMERIC_VALUE` — the custom value one event
  occurrence carried (string / numeric)
- `ATTRIBUTION_PARTNER_AGGREGATED_EVENT_VALUE` — the running "Aggregated Value" accumulated
  across repeats of that event for the user (e.g. total revenue so far)
- Revenue metrics: `SALE_AMOUNT` (custom report), REVENUE_SUM/COST_SUM (app reports), ARPU
  (revenue per active user), ROAS (revenue / cost)

## Audiences

MMP audience builder: recursive AND/OR condition groups over users' attributed events, with a
lookback window; built asynchronously to a CSV (device-ID lists) and optionally synced to
destinations (Meta/Google/TikTok). UI-driven; the MCP server has no audience tools — point users
at the Audiences page and docs.

## Fraud

Eight fraud detection reports exist in the UI (CTIT anomalies, click spamming, device/geo
mismatch, bot detection, proxy, SDK fraud...). Via MCP, approximate with `custom_report`
(REJECTION_REASON dimension), `get_conversion_report` (`POSTBACK_CTIT`, evaluation fields), and
remediate with `update_publisher` (BLOCK) / `manage_offer_access`.
