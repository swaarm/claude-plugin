# Swaarm concepts: the entity glossary

This file is the entity glossary in Swaarm's own voice. The MCP server's `initialize` instructions already introduce the high-level model (offers, advertisers, publishers, the WeGet/TheyGet model, budgets, conversions). This file goes deeper: the fields you'll actually encounter, how entities relate, and the vocabulary nuances that matter.

Read this when the user asks what fields an entity has, how entities relate, or what a specific term means in Swaarm. For how to *use* an entity in a workflow, see `workflows.md`. For tool-specific input/output caveats, see `tools.md`.

## Table of contents

1. [Advertiser](#advertiser)
2. [Publisher (a.k.a. Affiliate, a.k.a. Partner on MMP)](#publisher)
3. [Offer (a.k.a. Campaign)](#offer)
4. [Leadflow](#leadflow)
5. [Payout](#payout)
6. [Budget](#budget)
7. [Click](#click)
8. [Conversion (Postback)](#conversion)
9. [EventType](#eventtype)
10. [OfferAccess](#offeraccess)
11. [Network Adapter and NetworkConfig (AdvertiserConfig)](#network-adapter)
12. [Audit entry](#audit-entry)
13. [User (operator)](#user)
14. [Rewarded user](#rewarded-user)
15. [Swaarm Perform vs Swaarm MMP](#perform-vs-mmp)

---

## Advertiser

The entity that owns the apps or products being promoted. Pays the network for conversions.

**Key fields:**
- `id`, `name`, `status` (ACTIVE / BLOCKED / PAUSED / PENDING / REJECTED / DELETED)
- `accountType`, `country`, `phone`, `address`, `taxNumber`
- `salesManager`, `accountManager` — User objects; who owns the relationship
- `budgets[]` — advertiser-scoped budgets (rare; most budgets are offer-scoped)
- `blockedPublishers[]` — publishers this advertiser has blocked globally across their offers

**Relationships:**
- 1 → many **Offers**
- 1 → many **NetworkConfigs** (per-advertiser network adapter instances)
- many → 1 **accountManager** (User), 1 **salesManager** (User)

**Notes:**
- `PENDING` means awaiting approval during onboarding. Use it as a signal the relationship isn't fully live yet.
- MMP side sometimes says **"Direct Advertiser"** to distinguish from advertisers reached via a Network Adapter.

## Publisher

The entity that drives traffic to offers and earns payouts for valid conversions. Must be approved per-offer before promoting it.

**Important naming:**
- **Perform** product calls them **Publishers**; **Affiliate** is a synonym in some UI surfaces and role names ("Affiliate Manager").
- **MMP** product calls them **Partners**. Same concept, different label. Match the user's wording.

**Key fields:**
- `id`, `name`, `status` (ACTIVE / BLOCKED / REJECTED / PENDING / DELETED)
- `accountType`, `country`, `phone`, `address`, `taxNumber`
- `isApiPublisher`, `apiKeys` — whether this publisher pulls offers via API vs the portal
- `salesManager`, `accountManager` — User objects
- `createdAt`

**Relationships:**
- many → many **Offers** through **OfferAccess**
- 1 → many **PublisherSubIds** (traffic source identifiers within the publisher)
- 1 → many **PostbackLinks** (endpoints the network fires to)
- many → 1 **accountManager**, **salesManager**

**Notes:**
- `PENDING` commonly means awaiting approval; `REJECTED` means the network declined them.
- A publisher cannot promote an offer until their `OfferAccess` record for that offer is `APPROVED`. Status on the publisher alone is not enough.

## Offer

The central monetizable campaign entity. Offers belong to advertisers and are promoted by publishers.

**Key fields:**
- `id`, `name`, `description`, `status` (ACTIVE / PAUSED / PENDING / ARCHIVED / DELETED)
- `leadflow` — the billing model (CPI/CPA/CPL/CPS/CPC/CPM)
- `advertiser` — parent Advertiser
- `targeting` — countries, OS, device type
- `kpi` — plain-text KPI expected by the advertiser
- `note`, `additionalInformation`, `customRestrictions` — human-readable caveats
- `categories[]`, `verticals[]`, `tags[]`
- `isPrivate`, `isApprovalRequired` — access gating
- `previewUrl` — click-through preview for the landing page
- `payouts[]`, `budgets[]` — see below
- `createdAt`, `updatedAt`

**Status transitions that matter:**
- `ACTIVE` — live and accepting traffic. Syncs with advertiser's network API if network-adapter-backed.
- `PAUSED` — temporarily halted; still syncs with the advertiser's API so new/updated offer data keeps flowing in. Use PAUSED for short breaks.
- `PENDING` — development stage; *stops* syncing. Use PENDING when the offer isn't ready.
- `ARCHIVED` — inactive 90+ days; budgets and landing pages are removed. Reversible but disruptive.

**Relationships:**
- many → 1 **Advertiser**
- 1 → many **EventTypes** (per-offer event catalog)
- 1 → many **Payouts**, 1 → many **Budgets**
- many ↔ many **Publishers** via **OfferAccess**

## Leadflow

The offer's billing model. This is Swaarm's term — don't call it "pricing model" or "payout type" when the user is asking about it.

| Leadflow | Full name           | Pays on                           | Requires paid events? |
|----------|---------------------|-----------------------------------|-----------------------|
| CPI      | Cost per install    | Install                           | No                    |
| CPA      | Cost per action     | One or more defined post-install events | **Yes**       |
| CPL      | Cost per lead       | Lead (registration/form)          | No                    |
| CPS      | Cost per sale       | Sale / purchase                   | No                    |
| CPC      | Cost per click      | Click                             | N/A — no conversion   |
| CPM      | Cost per mille      | 1,000 impressions                 | N/A — no conversion   |

The leadflow is set on the offer and drives which metrics matter (CPC has no conversion rate; CPM has no click throughput). When debugging "no postbacks," look at the leadflow first — a CPC offer with no postbacks is expected.

## Payout

Revenue-and-cost pair for a single conversion event. This is the financial heart of Swaarm.

**Fields:**
- `weGet` — what the network collects from the advertiser per conversion (revenue)
- `theyGet` — what the network pays the publisher per conversion (cost)
- `margin` — auto-calculated as `(weGet - theyGet) / weGet * 100`
- `offer` — parent Offer
- `publisher` — optional; if present, this payout is publisher-specific
- `eventType` — optional; if present, this payout is event-specific
- `validity` — relevant when payouts vary over time

**Payout hierarchy (highest priority first):**
1. **Targeted payouts** — condition-based (geo, OS, sub-id pattern, etc.)
2. **Special payouts** — publisher-specific (different cost for one partner)
3. **Default payouts** — the offer's base payout, applied to everyone else

When an update is made via `update_payout`, the tool picks the right bucket based on which of `publisher_id` / `event_type_id` are present. Omit both to set the default; add one or both to create overrides.

## Budget

A spending or volume cap. Swaarm evaluates budgets against offers, publishers, or advertisers.

**Type × Validity × Threshold matrix:**

| Dimension   | Options                                                      |
|-------------|--------------------------------------------------------------|
| `type`      | WEGET (revenue), THEYGET (cost), EVENT (conversion count), CLICK, IMPRESSION |
| `validity`  | HOURLY, DAILY, MONTHLY, FOREVER (total, never resets)        |
| `threshold` | HARD (stops traffic) or SOFT (blocks postbacks only)         |

**Priority:** offer-level budgets take priority over publisher-level budgets. Multiple budgets of different types stack — a CLICK budget and a WEGET budget on the same offer both apply.

**What "exceeded" means:**
- HARD + exceeded → offer is removed from the publisher Feed API; clicks hit a blank page
- SOFT + exceeded → offer stays visible, clicks and conversions continue, but publisher postbacks are not forwarded

**Fields:**
- `type`, `validity`, `threshold`, `amount`
- `consumed` — how much of the budget is used so far in the current period
- `estimated` — projected consumption based on current pace
- `offer` (required), `publisher` (optional), `advertiser` (optional)

Budgets reset at the validity boundary (midnight for DAILY, start-of-month for MONTHLY, etc.) — `FOREVER` never resets.

## Click

A user redirect from Swaarm's tracker to the advertiser's landing page. Clicks carry the **Click ID** — the unique identifier threaded through tracking, postbacks, and attribution.

**Key fields:**
- `id` (the Click ID), `time`
- `publisher`, `publisherSubId` — traffic source
- `offer` — the offer being promoted
- `user` — geo, device (OS, make, model, UA), IP
- `url` — the tracking URL that was hit

**Relationships:**
- 1 → 1 **Publisher**, 1 → 1 **Offer**
- 1 → many **Conversions** (postbacks attributing to this click)

In URLs, the click ID is referenced via the macro `#{click.publisher.clickId}`. The `#` is part of the syntax — do not strip it.

## Conversion

A tracked action attributed to a click. In Swaarm these are often called **postbacks** because that's the mechanism — the advertiser (or MMP) fires an HTTP callback to Swaarm when the action happens, and Swaarm evaluates it and fires its own postback to the publisher.

**Key fields (as returned by `get_conversion_report`):**
- `id` — the postback ID
- `time` — postback receipt time
- `offer`, `publisher`, `advertiser`
- `weGet`, `theyGet` — **in microcents** (e.g., 1,500,000 = $1.50)
- `country` — user's geo
- `device` — OS, make, model, UA
- `connection` — IP
- `status` — APPROVED / PENDING / REJECTED
- `decision` — PASSED (postback forwarded to publisher) / FAILED (blocked)
- `click` — back-reference to the originating Click

**Status vs decision:**
- **Evaluation status** is the conversion's standing in the network's records (APPROVED = counted, PENDING = awaiting review, REJECTED = not counted).
- **Evaluation decision** is whether the publisher's postback was fired (PASSED vs FAILED). A REJECTED conversion can still have PASSED = false; an APPROVED conversion can have FAILED if something else blocked it (e.g., tripped SOFT budget).

## EventType

A conversion event category attached to an offer — e.g., "install," "registration," "purchase," "level_10_reached." Only leadflows that need events (CPA, sometimes CPL/CPS) have these.

**Fields:** `id`, `name`, `offer` (parent).

Payouts and budgets can reference a specific EventType to differentiate rates per event (e.g., $1 for install, $5 for purchase).

## OfferAccess

A publisher's permission state on a specific offer. This is the gate that controls whether a publisher can see the offer in their feed and promote it.

**Fields:**
- `offer`, `publisher`
- `accessState` — APPROVED / BLOCKED / PENDING
- `blockingReason` — free-text; populated when BLOCKED

**Created and updated via `manage_offer_access`** — which is an upsert: a missing record is created, an existing one is updated.

`PENDING` is the initial state when `isApprovalRequired` is true on the offer and a publisher requests access. `BLOCKED` is explicit denial with a reason. `APPROVED` is the go state.

## Network Adapter

A prebuilt integration with an external performance network (HasOffers, Affise, Everflow, Offer18, CPALead, Chimera, etc.). Adapters auto-import and sync offers via the external network's API.

**Two concepts, often confused:**
- **Network Adapter (type)** — the integration code. Listed via `list_network_adapters`; doesn't belong to any one advertiser.
- **Network Config (a.k.a. AdvertiserConfig)** — a specific instance of an adapter configured for one advertiser. Listed via `list_network_configs`; created via `create_network_config`.

**NetworkConfig fields:**
- `id`, `name`
- `advertiser` — parent Advertiser
- `adapter` — the adapter type
- `syncingActive` — whether offers are actively syncing
- `margin` — override margin applied to imported offers
- `apiConfig` — adapter-specific credentials and parameters
- `trackingUrlParams` — URL macro mappings
- `fieldSettings` — which offer fields to import/sync

Use `preview_network_config` as a dry-run to see what offers *would* be imported before committing.

## Audit entry

A record of who changed what, when, with before/after values. Returned by `list_audits`.

**Fields:**
- `id`, `changedAt`, `changedBy` — the User who made the change (or "robot" for automations)
- `type` — entity type (OFFER, ADVERTISER, PUBLISHER, PAYOUT, BUDGET, OFFER_ACCESS, ADVERTISER_CONFIG, AUTOMATION_RULE, and more)
- `affectedEntityId`
- `op` — CREATE / UPDATE / DELETE
- `undoable` — flag indicating whether Swaarm can roll back this change (note: no MCP tool to actually undo exists today; this is informational)
- `fields[]` — array of `{ name, previous, current }` for UPDATE ops

**Use cases:**
- "Who changed X's payout this week?" → filter by `entity_ids` or `types` and time range
- "What did this operator do today?" → filter by `changed_by`
- "What auto-changes fired?" → filter by `is_robot = true`

## User (operator)

A human operator of the Swaarm platform. Not directly managed via MCP tools (no `create_user` or `list_users`), but referenced as `changedBy` in audits and as `accountManager` / `salesManager` on advertisers and publishers.

The docs define four roles ([User Roles](https://docs.swaarm.com/en/articles/6083897-user-roles)):
- **Affiliate Manager / Account Manager** — restrictive; sees only their own publishers/advertisers; no profit/revenue/cost on Dashboard
- **General Manager** — broader; sees data owned by others; access to Explorer and audits
- **Admin** — full control; includes Optimization and Automation Rules
- **Advertiser Account Manager** — advertiser-relationship specialist; appears in automation actions

When the user tells you their role, adjust accordingly — see `personas/` for per-role playbooks.

## Rewarded user

Appears in the `get_rewarded_user` and `list_rewarded_users` tools. Rewarded users are end-users participating in a rewarded campaign — a distinct Swaarm sub-product covered in the docs' [Rewarded API guide](https://docs.swaarm.com/en/articles/11745040-rewarded-app-development-guide-how-to-use-the-swaarm-rewarded-api). If the user isn't running Rewarded offers, you won't encounter these.

## Swaarm Perform vs Swaarm MMP

Two product lines of the same platform:

- **Swaarm Perform** — performance marketing / affiliate network. Uses "Publisher," "Offer," classic affiliate-network vocabulary.
- **Swaarm MMP** — mobile measurement partner for app installs/events. Uses "Partner" (≈ Publisher), "Offer/Campaign," and has its own articles in the docs.

The MCP tool surface is unified — same tools serve both. But the *user's* vocabulary will tell you which side they're on. If they say "partner," they're in MMP mode; mirror that back. If they say "affiliate" or "publisher," they're in Perform mode.
