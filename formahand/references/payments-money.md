# Payments and money

Everything that touches the merchant's money: whether payments are ready, the switches on the payments account, tax, walls, the plan and the monthly statement. The rule for all of it: you read, prepare and relay; the merchant decides and pays.

## What you may never do

- Add a payment method, choose a paid package, upgrade a plan or pay a statement.
- Finish payments onboarding or type card, bank or identity details.
- Accept terms for the merchant.
- Refund above the owner's agent refund limit.
- Add a tax registration the merchant has not told you they hold.

Minting or revoking tokens, changing team members, accepting terms, payment onboarding, the payment profile, paying a statement, setting the agent refund limits, changing a token's own governance or approving its proposals, choosing the dashboard's environment, exporting the whole store and closing it: session-only routes refuse tokens with 401 or 403. Everything else is token work. A secret comes back once, from the call that creates it (`secretShownOnce`); reads report only `secretSet`.

## Payments

One active processor per store and per environment: Formahand Payments (card payments; the merchant finishes payouts on a form embedded in the dashboard), PayPal (their own REST app keys) or manual (bank transfer / cash on delivery). `get_started` reports whether payments are ready; an agent cannot complete onboarding or enter card details. `get_payout_profile` → `set_payout_profile` fills that form ahead (what the store sells, its website, charge timing, support contact) so it opens asking only for identity, bank and terms; `leftover` is what stays with the merchant. Discounts (`upsert_discount`) need a ready Formahand Payments account because the codes are validated at checkout.

## Card payments by country

Formahand Payments opens card accounts for businesses registered in Australia, Canada, Hong Kong, Japan, Malaysia, Mexico, New Zealand, Singapore, Switzerland, Thailand, the United Arab Emirates, the United Kingdom, the United States, Gibraltar, Liechtenstein, Norway and the European Union countries. Anywhere else, `describe_readiness` and `describe_account` carry a `countryNote` with this sentence, the country named:

> Card payments are not available for businesses registered in your country yet. PayPal, bank transfer and cash on delivery are coming soon; until one of them opens, this store cannot take payment at checkout. Details: https://formahand.com/docs/payments

## Walls and the account

A refused action returns a wall body `{ wall: true, feature, reason, plan, requiredPlan?, steps[] }`: 402 for `reason: plan`, 409 for `setup`, `paused` or `limit`. Each step names a dashboard href and, where an agent can act, the tool; a step that switches a paid feature on carries `activateHref`. The sandbox has no walls.

On any wall, or when a meter nears its limit, call `describe_account`: the plan, the payment profile, the statement and every feature the merchant can switch on (email package, labels, extra domains, media, a plan for unlimited runs), each with `agentCanDo`, `activateHref` and `askMerchant`, a ready sentence carrying that link. Read `paymentProfile.state` (`none`/`ready`/`expired`) before suggesting a package; never ask a merchant to pay for something twice. Say the sentence and stop: you never add a payment method, choose a package or pay. `describe_walls` lists used/limit, `get_plan` pricing and usage, `get_statement` one month, `start_upgrade` opens the page the merchant pays on.

## `describe_account`: the whole situation, and what to say

A wall says what is refused. `describe_account` says what the account looks like and what to ask the merchant for, in one call ([billing](https://formahand.com/docs/billing)). Call it whenever a tool answers 402 or 409, or a meter is close to its limit:

```json
{ "plan": { "id": "free", "name": "Free", "monthlyUsd": 0, "status": "free" },
  "paymentProfile": { "state": "none", "onFile": false, "href": "https://formahand.com/dashboard#settings/plan/payment", "commitments": [] },
  "statement": { "period": "2026-09", "totalMinor": 0, "currency": "usd", "status": "running" },
  "features": [{ "id": "email", "title": "Email from your own domain", "state": "needs_card",
                 "used": 480, "limit": 500, "unit": "emails", "currentOption": "free",
                 "options": [{ "id": "starter", "label": "1,500 emails a month", "priceLine": "$5 a month", "monthlyUsd": 5, "requiresCard": true }],
                 "activateHref": "https://formahand.com/dashboard#activate/email?option=starter",
                 "agentCanDo": "Read the meter … Choosing a package or adding a payment method is the merchant's.",
                 "askMerchant": "Your store has used 480 of 500 emails this month. Before it runs out, add a payment method to your payment profile at https://formahand.com/dashboard#settings/plan/payment first, then choose 1,500 emails a month at https://formahand.com/dashboard#activate/email?option=starter. Only you can add it." }],
  "nextSteps": ["…"] }
```

`state` is one of `active`, `available`, `needs_card`, `needs_plan`, `limit_reached`. `agentCanDo` is what your token may do by itself; `askMerchant` is a sentence to say as it stands, with the deep link.

**Read `paymentProfile.state` before you suggest anything that costs money.** It is `none`, `ready` or `expired`, and `askMerchant` is already written for it: with `ready` the sentence says the package "will be billed to your card on file"; with `none` it asks the merchant to add a payment method at the profile link *first*, then choose the option; with `expired` it asks them to replace the card first. A feature that is already `active` is never offered again, it says so and stops. `nextSteps` puts the payment profile first whenever it blocks anything, so following the list in order never asks a merchant to pay for something twice.

**An agent never adds a payment method, chooses a paid package or pays**; `start_upgrade` only opens the page the merchant pays on, and `describe_readiness` requirements carry the same `activateHref` where one applies.

## The wall body

A wall is the structured refusal a feature returns until the store meets its conditions:

```json
{ "error": "…", "wall": true, "feature": "customDomains", "reason": "setup", "plan": "free", "requiredPlan": "growth",
  "steps": [{ "id": "card", "label": "Add a card for $2.00/month per extra domain", "done": false, "href": "/dashboard#integrations" }],
  "usage": { "used": 3, "limit": 3 } }
```

HTTP **402** for `reason: "plan"`, **409** for `setup`, `paused` and `limit`; MCP tools return the same body as an `isError` result.

## Payments: preparing the payout profile

Getting paid is the one part of a store an agent cannot finish. Identity, bank account and terms are the merchant's own signature, on the processor's own form. Everything *around* that is ordinary store knowledge, and an agent that has already read the catalog and the domains answers it better than a merchant typing into a form, so `get_payout_profile` and `set_payout_profile` prepare the form and then hand over ([payments](https://formahand.com/docs/payments#4d-payout-profile)).

`get_payout_profile {}` (`settings:read`) answers `{ profile, defaults, warnings, applied, leftover }`. `profile` is what the merchant saved, or `null`; `defaults` is the draft Formahand built from the store's own record, category, the live custom domain or the free address, a written product description, the owner as representative and support contact, and is what to send back when there is nothing to correct. `leftover` names, in plain language, what the secure form will still ask for.

`set_payout_profile { profile }` (`settings:write`) replaces every field, so read first and send the whole object:

```jsonc
{
  "businessType": "individual",        // or "company", then businessName
  "category": "clothing",              // one of the curated keys, see get_payout_profile
  "websiteMode": "store",              // "other" + websiteUrl, or "none"
  "productDescription": "…",           // ≥ 40 characters when websiteMode is "none"
  "chargeTiming": "at_checkout",       // or "on_ship", "deposit"
  "supportEmail": "orders@example.com",
  "supportPhone": "+44…",
  "representativeName": "Jane Doe",
  "representativeEmail": "jane@example.com"
}
```

Where a payout account already exists the answers are sent to it at once, so the form opens shorter; where none exists they are sent the moment the merchant opens one. Neither tool completes onboarding and neither switches the store's active processor.

Read the `warnings` back to the merchant rather than swallowing them, a storefront still showing "Coming soon" is the common one, because the website on the account is reviewed and a site under construction is usually refused; open the store first, or choose `websiteMode: "none"` and write a description that stands on its own. Then hand over: the answer's `handoff.href` is `/dashboard#settings/payments`, and only the merchant can finish there. `describe_readiness { feature: "checkout" }` carries the same thing as the `payout_profile` requirement, severity `recommended`.

## Payments: the switches on the account

The payout profile is what an agent prepares *before* money moves. This is what an agent can change *after*: the features on the store's card-payments account, what each one costs the merchant, and which of them are actually theirs to decide ([payments](https://formahand.com/docs/payments#4e-payment-features-and-what-you-control)).

`describe_payment_features {}` (`settings:read`) reads them live from the account, nothing here is a flag Formahand keeps, and answers `{ ready, mode, features, handoff }`. Each feature carries `id`, `title`, `summary`, `cost`, `control`, `state`, `value`, `blockers` and `whatChanges`, plus `methods` for the ways to pay and `payouts` for the schedule. Two fields decide what an agent should do next:

- **`control`** is `"you"` when the merchant can change it here, and `"provider"` when the payment provider decides and the row is status only. Fraud screening and dispute cover are `"provider"`: there are no rules to write, no allow or block lists to keep, and `set_payment_feature` refuses them with the reason rather than pretending.
- **`blockers`** says, in words a merchant reads, what is stopping a feature. "You have no tax registrations yet, so every order is taxed at nothing until you add one" is the one worth repeating out loud.

`set_payment_feature { feature, enabled, confirm }` (`payments:write`) is **two calls on purpose**. With `confirm: false`, the default, nothing is touched and the answer is `{ applied: false, consequence }`, one sentence saying what the change does and what the provider will charge for it. Read that to the merchant. Only then call again with `confirm: true`, which applies it and answers the refreshed list. An agent that skips the first call is asking the merchant to pay for something they were never told the price of.

| Feature | Extra arguments | What switching it on means |
| --- | --- | --- |
| `automatic_tax` | none | Orders are taxed where the merchant is registered, at 0.5% of each order. Registrations are theirs to add, and without one the order is taxed at nothing. |
| `strong_authentication` | `authentication` (`always`, `over`, `provider`), `authenticationOverMinor` | Buyers are asked to confirm with their bank more often. No extra cost; fewer disputes, and a few buyers who give up at the bank's screen. |
| `local_currencies` | none | A buyer abroad sees the total in their own currency. Nothing extra for the merchant; the buyer's converted price carries the conversion charge. |
| `instant_payouts` | `payoutInterval` (`daily`, `weekly`, `monthly`, `manual`) | The schedule *is* the switch: `manual` keeps the balance until the merchant asks, which is what makes an instant payout worth having. |
| `payment_methods` | `method` (an id from `describe_payment_features`) | One way to pay on or off. Pay-later methods cost noticeably more per order and the rate is on the row. |
| `fraud_controls`, `chargeback_protection` | none | Refused: the provider decides these. Report the `state` and the `blockers`. |

`request_instant_payout { amountMinor?, currency? }` (`payments:write`) sends the balance across now. It moves the merchant's money, so say the fee first, the provider charges a percentage of each instant payout, and Formahand adds nothing, and it is refused outright unless `describe_payment_features` shows `instant_payouts` with `state: "on"`. Leave `amountMinor` out to send everything ready.

### Tax registrations

`list_tax_registrations {}` (`settings:read`) answers where the store is registered to collect tax, read live from its payments account, with the attestation Formahand recorded for each one beside it. `read: false` means the account could not be asked, which is not the same as there being none, and the difference matters: **where there is no registration, orders are taxed at nothing**, silently. `guard` says whether tax is paused, locked or flagged.

`add_tax_registration { country, state?, type, activeFrom?, confirm, attested }` (`payments:write`) is two calls. With `confirm: false` nothing is touched and the answer carries the **attestation**, the exact sentence the merchant has to agree to, the effective date and the fee. Read all three back. Only then call again with `confirm: true` and `attested: true`.

**An agent's registration binds the owner.** The owner minted your token, so an attestation you make is theirs: the claim is recorded against the store with your token named as the actor, the owner is emailed every time an agent adds or ends one, and the owner carries the consequences of tax collected or not collected.

If the payment provider later rejects a registration, tax is paused for the whole store and the claim is marked `disputed`. `country` is a two-letter code; `state` is required for the United States and a Canadian `province_standard`; `type` must be one the country offers, and every refusal names the ones it does; `activeFrom` is `"now"` or `YYYY-MM-DD`, today or later, a registration is never backdated.

`expire_tax_registration { registrationId, expiresAt?, confirm }` (`payments:write`) ends one from a date. Nothing is deleted: an ended registration is still the record of what the store claimed.

`list_payment_notices { limit }` (`events:read`) is everything the provider has said about the account rather than about one order: disputes, fraud warnings, payouts that bounced, details the account still owes, changes to the tax setup. They are ordinary events on the store's log, `payments.account_changed`, `payments.dispute`, `payments.fraud_warning`, `payments.payout_failed`, `payments.tax_notice`, so a flow or a subscription reacts to them as they arrive rather than polling. `severity: "action"` means the merchant has to do something; a `dueAt` is a real deadline.

All of it acts on the token's own environment, so `fh_test_` changes the sandbox account and `fh_live_` the live one.

## Tax

Whether the store charges tax at all is data an agent can write ([payments](https://formahand.com/docs/payments#6c-tax)). `get_tax_settings` reads the row, `set_tax_settings` replaces it whole: `enabled`, `pricesIncludeTax`, `defaultTaxCode` and `shippingTaxCode`. A product overrides the default with its own `taxCode` on `upsert_product`.

Three things are worth knowing before writing any of it.

**Nothing here calculates tax.** The merchant's payments account holds the registrations and is liable for the money, so the processor works out every number. All these settings do is decide what to ask it. Until that account has tax switched on and a registration in the regions the merchant collects in, `enabled: true` still records a tax of 0, `describe_readiness` carries a `tax` requirement that says exactly which of the two halves is missing, and it is a recommendation, never a blocker: plenty of stores owe no tax and take money perfectly well without it.

**`pricesIncludeTax` changes the money, not the wording.** `true` means the price on the product page is what the shopper pays and the tax is inside it (the EU/UK way); `false` adds it at checkout (the US way). Flipping it on a live store changes what every shopper is charged, so ask the merchant rather than inferring it. A store that has never saved the row already reads the answer its currency implies.

**Tax categories are opaque.** A code looks like `txcd_99999999` and is never constructed, guessed or translated from another provider's code, the tool refuses anything that is not one. `txcd_99999999` is general tangible goods, `txcd_20030000` services, `txcd_10000000` electronically supplied services, `txcd_92010001` shipping, `txcd_00000000` nontaxable; a product left at `""` falls back to the store's default and then to the code its `kind` implies.

Tax also reaches the numbers an agent reads: `get_analytics` reports `revenueMinor` **net of tax** with `taxCollectedMinor` beside it, and `export_orders` carries `order_tax`, `order_tax_rate`, `order_tax_jurisdictions` and `customer_tax_id`. Tax is money the merchant holds for a tax authority, never theirs, so it is never counted as revenue.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `describe_payment_features` | `settings:read` | Every switch on the store's card-payments account, read live from the account itself in the token's own environment (fh_live_ the live account, fh_test_ the sandbox's): { ready, mode, features: [{ id, title, summary, cost, control, state, value,… |
| `set_payment_feature` | `payments:write` | Changes one switch on the store's card-payments account. |
| `request_instant_payout` | `payments:write` | Sends the store's balance to its debit card now instead of waiting for the normal payout. |
| `list_tax_registrations` | `settings:read` | Where the store is registered to collect tax, read live from its payments account, plus who told Formahand about each one. |
| `add_tax_registration` | `payments:write` | Tells the store's payments account that the business is registered to collect tax in one place, so orders going there are taxed from a date. |
| `expire_tax_registration` | `payments:write` | Ends one of the store's tax registrations from a date. |
| `list_payment_notices` | `settings:read` | Everything the payment provider has said about the store's account rather than about one order: disputes, fraud warnings, payouts that bounced, details the account still owes, changes to the tax setup. |
| `get_payout_profile` | `settings:read` | The business details that prepare the store's payout form, live data in both modes (fh_live_ reads the live account, fh_test_ the sandbox's): { profile, defaults, warnings: [{ id, message }], applied, leftover }. |
| `set_payout_profile` | `settings:write` | Replaces the live store's payout profile: business type, what it sells, the website it is reviewed against, the product description, when the customer's card is charged, and the support contact. |
| `get_checkout_settings` | `settings:read` | What the store's payment page asks a shopper for, and where it lives (live data): { checkout: { phone, company, addressLine2, orderNote, marketingOptIn, termsLink, embedded, tipping } }. |
| `set_checkout_settings` | `settings:write` | Replaces what the live store's payment page asks for and where it lives: the phone, company and second-address-line fields, the order note, the marketing consent checkbox, the terms link and embedded (the payment form on the store's own page, or the… |
| `get_notification_settings` | `settings:read` | Who on the staff hears about a new order or a low stock level (live data): { notifications: { newOrderEmails, lowStockEmails } }. |
| `set_notification_settings` | `settings:write` | Replaces the live store's staff notification addresses: who also receives the new-order email and the low-stock email, and whether a morning summary is wanted. |
| `get_tax_settings` | `settings:read` | Whether the live store charges tax, and how (live data): { tax: { enabled, pricesIncludeTax, defaultTaxCode, shippingTaxCode } }. |
| `set_tax_settings` | `settings:write` | Replaces the live store's tax settings: whether tax is charged, whether prices already include it, and the default product and shipping tax categories. |
| `get_plan` | `settings:read` | The store's plan, effective pricing and billing: { plan: { plan: free \| growth \| pro, subscribedPlan, status, entitlements: { stores, customDomains, marketing, flows, apiTokens, teamSeats, transactionFeeBps, shippingMarkupBps,… |
| `get_statement` | `settings:read` | A monthly statement (/docs/billing §5): { period? } (YYYY-MM, default this month) → { statement: { id, period, currency, lines: [{ kind: plan \| email_overage \| extra_domains \| media_overage \| label, description, quantity, unitMinor, amountMinor,… |
| `apply_promo_code` | `settings:write` | Apply a promotion code to this store: { code } → { promotion: { code, kind: fee_bps \| markup_bps \| plan_discount_pct \| email_overage \| domain_price \| fee_holiday, value, expiresAt }, pricing } (the new effective pricing, as get_plan.pricing). |
| `describe_walls` | `settings:read` | Every walled feature with its current state for this store: { plan, fees, pricing, usage, activity, features: { marketing: { allowed, wall, steps: [{ id, label, done, href?, tool?, detail? }], warnings, caps: { monthlyCap, monthSent, monthRemaining,… |
| `describe_account` | `settings:read` | The whole account in one answer, for deciding what to ask the merchant for: { plan: { id, name, monthlyUsd, status }, card: { onFile, brand?, last4? }, paymentProfile: { state: none \| ready \| expired, onFile, brand?, last4?, expires?, href,… |
| `start_upgrade` | `settings:write` | Start a plan upgrade: { plan: growth \| pro, interval?: monthly \| annual } → { url, sessionId, plan, interval }. |
| `describe_readiness` | `settings:read` | What each feature of this store still needs before it works, live data: { features: [{ feature, label, ready, requirements: [{ id, label, done, detail, href, tool?, severity? }], countryNote? }] }. |

## Read more

- https://formahand.com/docs/payments
- https://formahand.com/docs/billing
- https://formahand.com/docs/pricing
