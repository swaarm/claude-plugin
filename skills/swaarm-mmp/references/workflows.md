# MMP workflows — multi-tool recipes

Step sequences for the requests MMP operators actually make. Follow the order; each step names
the tool and what to look for.

## 1. "Is my SDK integration working?" (integration verification)

1. `list_apps` / `get_app` — confirm the app + store app exist and have an SDK token with
   `test: true` for debugging.
2. Ask the user to fire test events from the app build using the test token.
3. `list_sdk_test_events` filtered by the store app — events should appear within moments.
   - Rows with `eventType: null` → the event name sent doesn't match any mapping ID.
4. `list_app_event_types` with the app — compare mapping IDs against what the SDK sends
   (case-sensitive). Fix with `create_app_event_type`/`update_app_event_type` — match the
   mapping to what the SDK actually sends, not vice versa.
5. Re-test until every intended event maps. Then the user switches to the production token.

## 2. "Why wasn't this install/event attributed (to partner X)?" (attribution debugging)

Walk the pipeline in order — don't skip ahead:

1. Get the user's device/vendor ID: from the user's report, from `get_conversion_report`
   (`APP_VENDOR_USER_ID` on a related row), or `list_sdk_test_events` (`vendorId`).
2. `user_journey_report` with that `user_id` (MMP-enabled deployments only) — the full
   impression/click/install/event timeline including pre-attribution touchpoints. Check: does a
   click from the expected partner exist BEFORE the install, inside the attribution window?
3. `get_conversion_report` scoped to the app + a filter on APP_VENDOR_USER_ID — check
   `ATTRIBUTION_PARTNER_*` (match type, install/click times, rejection reason),
   `EVALUATION_STATUS`/`EVALUATION_DECISION`, and `POSTBACK_CTIT`.
4. If the event exists but the partner never got it: evaluation FAILED (rules), a tripped SOFT
   budget, or a missing/broken postback link on the partner (postback links are UI-managed —
   direct the user there).
5. If nothing exists at all: back to workflow 1 (is the event even arriving and mapping?).

## 3. Partner onboarding ("wire up partner X")

1. `search` / `list_publishers` — make sure the partner doesn't already exist.
2. `create_publisher` — name, country, ACTIVE, account type, account manager. Confirm inputs
   with the user first; the account manager must be an existing user ID.
3. `manage_offer_access` — approve the partner on the relevant offer(s).
4. Payouts if partner-specific: `update_payout` with publisher_id (see the `swaarm` skill for
   payout semantics — microcents!).
5. Postback link + tracking link handover are UI steps — point at the Partner Details page and
   the docs: "MMP: Partner Tracking Details" and the "Swaarm MMP Partner Setup Guide".
6. Verify first traffic: `get_conversion_report` filtered by PUBLISHER_ID next day, or
   `custom_report` (PUBLISHER_NAME × NEW_USERS).

## 4. Retention / cohort analysis ("how is retention for app X?")

1. Resolve the app: `list_apps`.
2. `cohort_report`: RETENTION_PERCENT, granularity DAY, number_of_periods 8, value_type
   ORIGINAL_VALUES, scoped to the app; group_by PUBLISHER_ID (or POSTBACK_GEO_COUNTRY) to
   compare sources; min_cohort_users ~50 to cut noise.
3. Complement with `stickiness_report` (DAU/MAU trend) when the question is about habit rather
   than comeback-after-install.
4. For "is UA paying off": second `cohort_report` with metric ROAS, value_type CUMULATIVE, weekly
   periods — the crossing point past 100% is the payback period.

## 5. Campaign performance triage ("which partners deliver?")

1. `custom_report`: dimensions [PUBLISHER_NAME, STORE_APP_NAME], metrics [NEW_USERS,
   ACTIVE_USERS, SESSIONS, SALE_AMOUNT, ROAS], time range, sort by NEW_USERS.
2. Add `app_event_type_ids` for the funnel events (registration, purchase) to get per-event
   columns per partner (MMP-enabled deployments).
3. Suspicious partner (huge installs, zero events, CTIT near zero)? → workflow 6.
4. Scale winners / pause losers via `manage_offer_access` and budgets — echo-and-confirm first.

## 6. Fraud check on a partner

1. `get_conversion_report` filtered to the partner: fields POSTBACK_CTIT,
   ATTRIBUTION_PARTNER_MATCH_TYPE, APP_EVENT_NAME, EVALUATION_*, device fields. Near-zero CTIT
   clusters, single device models, no post-install events = classic signals.
2. `custom_report` with REJECTION_REASON dimension filtered to the partner — what the rules
   already caught.
3. `funnel_report` grouped by PUBLISHER_ID — a partner whose users never pass step 1 stands out.
4. Remediate: `manage_offer_access` BLOCKED on the offer, or `update_publisher` status BLOCKED
   for the whole partner. Confirm before blocking; blocking is reversible, tell the user so.
5. The dedicated Fraud Detection Reports (8 sub-reports) are UI-only — link the docs article.

## 7. Event taxonomy setup ("add purchase tracking")

1. `get_app` — current event types and mapping IDs.
2. Agree naming with the user: display name vs mapping ID (wire name), classification
   (MONETIZATION for revenue events).
3. `create_app_event_type`; verify via workflow 1 that real events map.
4. If the event should be payable to partners → offer-side `create_event_type` + payout
   (Perform-side semantics; see the `swaarm` skill).

## Out-of-band topics → docs.swaarm.com

Point users here for what the MCP can't do (collection root:
https://docs.swaarm.com/en/collections/4531320-swaarm-mmp):

- SDK installation (iOS/Android): SDK collection — https://docs.swaarm.com/en/collections/5099102-sdk
- S2S API: https://docs.swaarm.com/en/collections/7182994-mmp-server-to-server
- Attribution links / integration with Swaarm advertisers: https://docs.swaarm.com/en/articles/8929779-swaarm-integration-with-swaarm-mmp-clients
- Partner setup guide (tracking link + postback walkthrough): https://docs.swaarm.com/en/articles/13170769-swaarm-mmp-partner-setup-guide
- Partner tracking details (macros/parameters): https://docs.swaarm.com/en/articles/11010074-mmp-partner-tracking-details
- Partner onboarding checklist: https://docs.swaarm.com/en/articles/11066104-partner-onboarding-checklist
- App / Store App creation UI: https://docs.swaarm.com/en/articles/8020899-mmp-app-creation , https://docs.swaarm.com/en/articles/8020902-mmp-store-app-creation
- Reports (per-report articles incl. Explorer, Event Log, Fraud Detection): https://docs.swaarm.com/en/collections/5099064-mmp-reports
- User Journey report: https://docs.swaarm.com/en/articles/16354122-mmp-user-journey-report
- SDK Debug report: https://docs.swaarm.com/en/articles/8438859-sdk-debug-report
- Audiences: https://docs.swaarm.com/en/collections/19719791-mmp-audiences
- Integrations (SAN etc.): https://docs.swaarm.com/en/collections/8668426-mmp-integrations
