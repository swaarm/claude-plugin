# Swaarm terminology — say this, not that

Swaarm has a specific vocabulary in its docs. Using the right words makes you sound like a Swaarm insider; using the generic industry terms makes you sound like a bot reading tool schemas. Read this file before writing anything the user will see externally (partner emails, onboarding messages, doc drafts) or when their wording is ambiguous and you want to mirror it back correctly.

## Product lines

| Use | Not |
|-----|-----|
| **Swaarm Perform** (performance marketing / affiliate network) | "the Swaarm platform" when the user specifically means Perform |
| **Swaarm MMP** (mobile measurement partner, for app installs/events) | "the MMP product," "Swaarm Mobile" |
| **Swaarm Marketplace** (partner discovery layer) | "the marketplace feature" |

When the user says "partner," they're almost certainly on the MMP side. "Affiliate" or "publisher" means Perform side. Match their side in your reply.

## Entities

| Swaarm word | Industry synonym (don't use) | Notes |
|-------------|------------------------------|-------|
| **Offer** | ad, campaign (Perform) | MMP docs sometimes say "Offer/Campaign"; both are fine there |
| **Advertiser** | merchant, brand | |
| **Publisher** (Perform) / **Partner** (MMP) | affiliate, network partner, source | Match the user's side |
| **Leadflow** | pricing model, payout type | This is load-bearing — don't paraphrase |
| **Conversion** | action, goal, event (wrong — see below) | |
| **Event** | conversion (wrong — events feed into conversions) | An event type is a *kind* of action on an offer (e.g., "registration"). A conversion is an attributed, paid-ready event occurrence. |
| **Postback** | webhook, callback, pixel | Mechanically similar but Swaarm says "postback" |
| **Click ID** | transaction ID, session ID | Capitalized in docs; threaded through tracking |
| **Macro** | placeholder, variable, token | URL template tokens with `#{...}` syntax |
| **Tracking Link** | offer URL, redirect link | |
| **Optimization Rule** | fraud rule, filter | Evaluates conversions, decides postback routing |
| **Automation Rule** | workflow, scheduler | Changes entity state on conditions/schedule — distinct from Optimization Rule |
| **Network Adapter** | integration, connector | The type. `list_network_adapters` |
| **Network Config** (a.k.a. AdvertiserConfig) | integration instance | The per-advertiser setup. `list_network_configs` |
| **Explorer** | dashboard, report builder | Dimensional report UI |
| **SQL Studio** | query tool, SQL console | Raw ClickHouse query UI |
| **OfferAccess** | offer approval, access state | The per-offer-per-publisher permission record |

## Metrics and money

| Swaarm word | Mean |
|-------------|------|
| **WeGet** | revenue the network collects from the advertiser per conversion |
| **TheyGet** | cost the network pays the publisher per conversion |
| **Margin** | `(WeGet − TheyGet) / WeGet`, as a percentage |
| **Profit** | `WeGet − TheyGet`, per conversion |
| **EPC** | earnings per click = `WeGet / Clicks` |
| **CR** | conversion rate = `Conversions / Clicks` |
| **Paid conversions** | conversions that are approved *and* payable |

Conversion-level revenue/cost from `get_conversion_report` is in **microcents**. Aggregated reports present in the user's display currency. Always show units explicitly; don't leave a raw number hanging.

## Statuses, thresholds, and enums

These are case-sensitive in the API. Use them uppercase in filters and mutations.

**Offer status:** ACTIVE, PAUSED, PENDING, ARCHIVED, DELETED
**Advertiser/Publisher status:** ACTIVE, BLOCKED, PAUSED, PENDING, REJECTED, DELETED (varies slightly)
**OfferAccess state:** APPROVED, BLOCKED, PENDING
**Leadflow:** CPI, CPA, CPL, CPS, CPC, CPM
**Budget type:** WEGET, THEYGET, EVENT, CLICK, IMPRESSION
**Budget validity:** HOURLY, DAILY, MONTHLY, FOREVER
**Budget threshold:** HARD, SOFT
**Conversion evaluation status:** APPROVED, PENDING, REJECTED
**Conversion evaluation decision:** PASSED, FAILED
**Granularity:** ALL, HOUR, DAY, WEEK, MONTH

## Macros

Swaarm tracking URLs use `#{namespace.path}` syntax. The `#` is part of the macro. Common ones:

- `#{click.publisher.clickId}` — the Click ID
- `#{payout.weGetInDollars}` — revenue amount formatted as dollars
- `#{payout.theyGetInDollars}` — cost amount formatted as dollars
- `#{gaid}` — Google Advertising ID (Android)
- `#{idfa}` — Apple IDFA (iOS)

Never strip the `#`, never add spaces, and don't invent new macros — consult [Advertiser Tracking Details](https://docs.swaarm.com/en/articles/4612478-advertiser-tracking-details) for the full list.

## Roles (as the docs define them)

- **Affiliate Manager / Account Manager** — restrictive role; see only their own publishers/advertisers; no profit/revenue/cost on the Dashboard
- **General Manager** — can see data owned by others; has Explorer and audits
- **Admin** — full control, including Optimization Rules and Automation Rules
- **Advertiser Account Manager** — specialized advertiser-relationship role; referenced in automation notifications

Don't collapse "Account Manager" into "admin" — they have fundamentally different visibility.

## Things the docs do NOT say

If you're tempted to use these words, check first — they're either wrong or don't exist in Swaarm's vocabulary:

- "Fraud" — docs don't lead with this word; feature names use "Optimization" or "Postback Decision" instead
- "Pixel" — Swaarm uses "postback"; pixels are a different mechanism
- "Goal" — use "Event" or "Conversion," depending on meaning
- "Click throughrate / CTR" — the standard metric is CR (conversion rate), not CTR. Impressions are tracked only on CPM offers.
- "Lookup" / "segment" — Swaarm uses "dimension" and "filter"

## Mirroring the user

If the user says "my partner X" and they're on MMP, reply with "your partner X." If they say "my publisher X" and they're on Perform, reply with "your publisher X." Don't force them to switch vocabularies — match them. This applies in both directions (internal consistency in a single reply matters more than cross-doc purity).
