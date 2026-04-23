# Common Swaarm workflows

These are the multi-tool recipes a human operator runs regularly. Read one when the user's request maps onto it — don't reinvent the sequence each time.

Organized by intent, not by tool. If a workflow has persona-specific variants (e.g., Account Manager skips financial columns), the `personas/<role>.md` file overrides the generic version here.

## Table of contents

1. [Performance snapshot: "how is X doing this week?"](#performance-snapshot)
2. [Conversion investigation: "why didn't this convert/pay?"](#conversion-investigation)
3. [Publisher onboarding onto an offer](#publisher-onboarding)
4. [Payout change on an existing offer](#payout-change)
5. [Budget change or setup](#budget-change)
6. [New offer rollout to a subset of publishers](#new-offer-rollout)
7. [Audit trail / "who changed this"](#audit-trail)
8. [Reconcile numbers with an advertiser or publisher report](#reconcile-numbers)
9. [Network adapter setup (import offers from external network)](#network-adapter-setup)
10. [Out-of-band topics (hand off to docs.swaarm.com)](#out-of-band)

---

## Performance snapshot

**Trigger:** "How's offer X doing?" / "Show me last week's performance for publisher Y" / "Top advertisers this month."

**Recipe:**
1. If the entity is specific and known, use `get_entity_stats` with the ID, entity_type, date range, and the metrics the user cared about.
2. If the entity is vague ("my top offers"), use `run_report` with the relevant dimension (OFFER), a reasonable metric set (CLICKS, CONVERSIONS, WE_GET, CR, EPC), sort by the user's implied ranking metric, and a limit of 10-20.
3. Present the result in a readable format. Include units. Call out anomalies if any metric is zero, negative, or dramatically off a prior period.
4. Offer one obvious follow-up: a time breakdown, a country split, or a sibling entity (advertiser of that offer, publishers driving it).

**Gotchas:**
- If the user is in a restrictive role (Account Manager), some financial metrics will be blanked — call that out in the response; don't let them wonder.
- For anything larger than ~90 days, warn the user that the query may be slow and offer to narrow.

## Conversion investigation

**Trigger:** "Why didn't this conversion pay?" / "Partner X says they're missing postbacks" / "This conversion looks wrong."

**Recipe:**
1. Get the click ID or postback ID from the user. If they only have a vague complaint, ask for the specific ID or a narrow time window + publisher ID.
2. Call `get_conversion_report` with filters on POSTBACK_ID or on `time` range + PUBLISHER_ID + OFFER_ID. Include the fields: POSTBACK_ID, OFFER_ID, PUBLISHER_ID, EVALUATION_STATUS, EVALUATION_DECISION, WE_GET, THEY_GET, COUNTRY, DEVICE_*.
3. **Convert microcents to display currency.** 1,500,000 microcents = $1.50. Never show microcents raw.
4. Interpret the status/decision pair:
   - APPROVED + PASSED → paid; if the user thinks it wasn't, check the publisher's postback endpoint configuration outside MCP
   - APPROVED + FAILED → payable but postback not fired; common causes: tripped SOFT budget, optimization rule blocking
   - PENDING + anything → not yet approved; common during a review window
   - REJECTED + anything → not counted; the audit log on that conversion (via list_audits) or the offer's Optimization Rules (UI-only) will say why
5. If a budget is the suspect, call `get_offer` and look at the budgets; check `consumed` vs `amount`.
6. If an Optimization Rule is the suspect, tell the user to check the Optimization Tool UI — the MCP can't inspect rule logic directly.

**Output template:** "Postback {id} at {time} for {publisher} on {offer}: evaluation status {X}, decision {Y}. {Plain-language reason}. Suggested next step: {specific action}."

## Publisher onboarding

**Trigger:** "Approve publisher X on offer Y" / "Get this partner running on our new offer."

**Recipe:**
1. `get_offer` to confirm the offer is ACTIVE and see `isApprovalRequired`.
2. `list_publishers` with keyword to confirm the publisher is ACTIVE (not BLOCKED or REJECTED).
3. If both check out, `manage_offer_access` with `state: APPROVED`.
4. If the publisher is BLOCKED globally, or the offer has a restriction the publisher doesn't meet, surface it — don't just force APPROVED.
5. For a batch (multiple publishers on one offer, or one publisher on multiple offers), confirm the full list with the user once, then execute — don't stop for per-item confirmation.

**Special case — publisher needs a different payout:** run the approval first, then `update_payout` with `publisher_id` to set the override.

## Payout change

**Trigger:** "Change offer X's payout to $Y" / "Give publisher Z a special rate on offer X."

**Recipe:**
1. **`get_offer`** to see the current payout(s). Echo them to the user.
2. Check `list_audits` with `entity_ids: [offer_id]` and `types: [PAYOUT]`, recent time window, to see if anyone changed this payout recently. Mention it if so.
3. Confirm the new values with the user explicitly: "Change we_get from $X to $Y, they_get from $A to $B. Proceed?"
4. `update_payout` with the right combination of `publisher_id` / `event_type_id` to target the right bucket:
   - Neither → default payout
   - `publisher_id` → publisher-specific override
   - `event_type_id` → event-specific override
   - Both → publisher-and-event-specific
5. After the update, show the new state and remind the user that the change is logged in `list_audits` under their name.

## Budget change

**Trigger:** "Set a daily cap of $1000 on offer X" / "Stop publisher Y from burning budget" / "Publisher says they're not getting postbacks."

**Recipe:**
1. **`get_offer`** to see current budgets. Echo them.
2. Ask the user which threshold they want — HARD (stops traffic) or SOFT (blocks publisher postbacks only). Default HARD for "cap spending" requests; SOFT is usually only chosen deliberately for a specific reason.
3. Ask validity — HOURLY/DAILY/MONTHLY/FOREVER. Default DAILY for spending caps; FOREVER for total lifetime caps.
4. Confirm the full setup with the user: "HARD DAILY WEGET budget on offer X at $1000. Proceed?"
5. `update_budget` with the fields.
6. If the user is trying to "unstick" a tripped budget, note that `consumed` cannot be manually reset — raise `amount` as a temporary workaround, and mention it resets at the validity boundary.

**Common miscue to catch:** "No postbacks" often means a SOFT budget tripped. Before creating a new budget, check existing ones with `get_offer` → budgets[] and look for any with `consumed >= amount` and `threshold: SOFT`.

## New offer rollout

**Trigger:** "We have a new offer, roll it out to my publishers in country X" / "Announce this offer to the top 20 partners."

**Recipe:**
1. `get_offer` on the new offer — confirm it's ACTIVE, and check `isApprovalRequired` and `targeting`.
2. Build the publisher list:
   - By existing performance: `run_report` dimension=PUBLISHER, metrics=CONVERSIONS or WE_GET, sorted desc, with a country filter if relevant.
   - By account manager ownership: ask the user; the MCP doesn't expose accountManager filters on `list_publishers` directly.
3. Confirm the list with the user before doing anything destructive.
4. For each publisher, call `manage_offer_access` with `state: APPROVED`. Batch without per-item confirmation.
5. Notify publishers — this is out-of-band; the MCP doesn't send emails. Tell the user to trigger the notification via their preferred channel (Slack, email, or an Automation Rule in the UI).

## Audit trail

**Trigger:** "Who changed this payout yesterday?" / "What did <operator> do this week?" / "What auto-changes fired last night?"

**Recipe:**
1. `list_audits` with targeted filters:
   - Entity question: `entity_ids: [id]` or `offer_ids / advertiser_ids / publisher_ids`
   - Person question: `changed_by: userId`
   - Type question: `types: [PAYOUT, BUDGET, OFFER_ACCESS, ...]`
   - Auto vs human: `is_robot: [true]` or `[false]`
2. For each entry with `op: UPDATE`, the `fields[]` array shows `{ name, previous, current }` — this is the actual change.
3. Present chronologically, oldest first unless the user wants most-recent-first.
4. If the user asks about an undoable change, remind them that while the audit flags it as undoable, there's no `undo_audit` MCP tool — rollback is manual via `update_*`.

## Reconcile numbers

**Trigger:** "Advertiser says we have X conversions but we show Y" / "These numbers don't match my report."

**Recipe:**
1. Pin down the date range, timezone, and exact metric definition the other side is using. Timezone mismatches cause most discrepancies.
2. Run the same query two ways:
   - Aggregated: `run_report` or `get_entity_stats`
   - Conversion-level count: `get_conversion_report` with the same filters, then count rows
3. Compare to the advertiser's/publisher's number. Common causes of gaps:
   - Timezone boundary (Swaarm defaults to UTC unless customer overrides)
   - Approval vs paid conversions (APPROVED vs APPROVED+payable)
   - Late-arriving postbacks (if the remote side's window closed earlier)
   - Rejected conversions being counted by one side, not the other
4. If the numbers still don't reconcile, fall back to `query_clickhouse` for a custom query — that's what SQL Studio is for in the UI.

## Network adapter setup

**Trigger:** "Integrate Advertiser X with their HasOffers account" / "Set up automatic offer import from Affise."

**Recipe:**
1. `list_network_adapters` to confirm the adapter type exists.
2. `list_network_configs` with `advertiser_id` to confirm no duplicate exists already.
3. Gather credentials from the user (API keys, endpoints, network-specific parameters). Never guess these.
4. **Call `preview_network_config`** first with the proposed settings — this shows what offers would be imported without creating anything.
5. Review the preview with the user. If it looks right:
6. `create_network_config` with `syncingActive: false` to start.
7. Tell the user: "Config created, syncing is off. Flip syncingActive to true via `update_network_config` when you're ready to start the auto-import."
8. After activation, monitor with `list_offers` filtered by advertiser — new offers should start appearing.

## Out-of-band topics

The MCP server doesn't cover these. When the user asks about them, acknowledge and point at docs.swaarm.com:

| Topic | Link |
|-------|------|
| SDK integration (iOS) | https://docs.swaarm.com/en/articles/8016044-ios-sdk-step-by-step-implementation-guide |
| SDK integration (Android) | https://docs.swaarm.com/en/articles/8016038-android-sdk-step-by-step-implementation-guide |
| SDK integration (Flutter) | https://docs.swaarm.com/en/articles/9776920-flutter-sdk-step-by-step-implementation-guide |
| SDK integration (React Native) | https://docs.swaarm.com/en/articles/8615473-react-native-sdk-step-by-step-implementation-guide |
| MMP integration (Appsflyer, Adjust, Singular, Branch, Kochava, Appmetrica) | https://docs.swaarm.com/en/articles/6092956-mmp-integration-with-swaarm |
| Leadflows overview | https://docs.swaarm.com/en/articles/4579666-leadflows-supported-in-swaarm |
| Advertiser tracking details | https://docs.swaarm.com/en/articles/4612478-advertiser-tracking-details |
| Publisher tracking details | https://docs.swaarm.com/en/articles/4612464-publisher-tracking-details |
| Publisher postback setup | https://docs.swaarm.com/en/articles/4676908-publisher-postback-setup |
| Testing publisher postbacks | https://docs.swaarm.com/en/articles/5904349-testing-publisher-postback |
| Optimization Tool | https://docs.swaarm.com/en/articles/4710700-optimization-tool |
| Postback Decision rules | https://docs.swaarm.com/en/articles/4711489-postback-decision-optimization-rules |
| Postback Status rules | https://docs.swaarm.com/en/articles/4711495-postback-status-optimization-rules |
| Automation Rules | https://docs.swaarm.com/en/articles/4575003-automation-rules |
| Alerts | https://docs.swaarm.com/en/articles/6777745-alerts |
| Publisher signup form | https://docs.swaarm.com/en/articles/4679150-creating-a-publisher-signup-form |
| Publisher groups | https://docs.swaarm.com/en/articles/5634871-publisher-groups |
| Margin and revenue share | https://docs.swaarm.com/en/articles/5104105-margin-and-revenue-share |
| User roles | https://docs.swaarm.com/en/articles/6083897-user-roles |
| SQL Studio | https://docs.swaarm.com/en/articles/4665217-sql-studio |
| View-Through Attribution | https://docs.swaarm.com/en/articles/5394154-view-through-attribution-vta |
| Rewarded API | https://docs.swaarm.com/en/articles/11745040-rewarded-app-development-guide-how-to-use-the-swaarm-rewarded-api |
| GraphQL API | https://docs.swaarm.com/en/collections/9052793-graphql-api |

When pointing the user at a doc, quote the most relevant line if you can — it's kinder than "here's a link, go figure it out."
