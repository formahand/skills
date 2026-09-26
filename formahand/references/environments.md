# Environments: live, test, preview and governance

Every store has a live environment and can have a sandbox: a separate test store with test payments, free test labels and email only to the owner. A token's prefix decides which one it reaches. This file covers the sandbox, promote and refresh, preview links, launch mode and the rules a token works under.

## Live and test

A store has a live environment and, once enabled, a sandbox: a fully separate store at `<slug>-test.<domain>` with payments in test mode, test labels and email routed only to the owner with a “[Test]” subject. Orders there are not real: no usage, no bill, no plan wall.

A token acts on exactly one environment, fixed by its prefix: a live token (`fh_live_…`) reaches the live store, a test token (`fh_test_…`) the sandbox. `get_store` and `get_environment` return `mode`; there is no per-call switch.

`get_environment` shows both; `enable_sandbox` creates the sandbox (poll until `sandbox.runtimeStatus` is `ready`); `refresh_sandbox` overwrites it from live; `promote_to_live { dryRun: true }` previews, then `promote_to_live` copies sandbox content to live (see Promote). Live-only: custom domains, plan and billing, statements, the payment profile.

## The model

| | Live | Sandbox |
| --- | --- | --- |
| Data | your live products, orders, customers and media library | its own copies, kept apart from live in both directions |
| Storefront | the theme as published | the same theme, plus a fixed amber banner "Test mode, orders here are not real" on every page |
| Payments | your Formahand Payments account / PayPal keys / manual | its own payment account in test mode: a test-mode payment account (onboarded in the dashboard's embedded form with test data), sandbox PayPal keys, or manual |
| Shipping labels | Formahand Shipping, billed on your statement | free test labels with simulated tracking, and nothing accrued |
| Email | buyers and you | **only the store owner's address**, subject prefixed `[Test] `; a sandbox with no owner address on file sends nothing |
| Events | `mode: "live"` on every event | `mode: "test"`; subscriptions and flows are the sandbox's own |
| Tokens | a live token | a test token, `fh_test_…`, which only ever reaches the sandbox |
| Domains, plan, billing, team, card | live only | not applicable |

The value in the API and in every event payload is `mode`: `live` or `test`. "Sandbox" is simply what the dashboard calls the test environment.

## Promote and refresh

Copied, idempotently (products by slug, collections by handle, pages by slug, templates by key, flows by name, discounts by code):

| Resource | Promote (sandbox → live) | Refresh (live → sandbox) |
| --- | --- | --- |
| Storefront draft: theme settings, sections, module settings, hero image, navigation, content pages | becomes a **new live draft revision**; publishing stays explicit (Online Store → Publish, or `publish_storefront`) | imported and **published** so the sandbox mirrors the live storefront |
| Products with variants and images | created or updated live; images are copied into the target environment's media library and URLs rewritten to its address | same |
| Collections (members mapped through product slugs), pages, navigation, SEO | live immediately | same |
| Shipping settings and rates; the shipping mode when it is `self` or `formahand` | live immediately (a courier platform mode is noted, not switched: its credentials belong to the target) | same |
| Discounts | recreated on the target's own payment account; codes that already exist are skipped; without a ready payment account they are reported as conflicts | same |
| Email templates (edited or custom ones) | upserted by key | same |
| Flows | created or updated **disabled**; webhook secrets and agent bearers do not travel (a note names the flow) | same |
| Integration configs | config only, never secrets | same |

Never copied: orders, customers, events, deliveries, event subscriptions (their endpoints differ per environment), shipping labels, usage, statements, tokens, domains, payment accounts. `dryRun: true` returns the same summary without writing. A resource the target refuses (a reserved variant, a bad handle) is listed under `conflicts`; the rest still lands.

## Agent walkthrough

1. Mint a live token with `settings:write` (or use the dashboard) and call `enable_sandbox`; poll `get_environment` until `sandbox.runtimeStatus` is `ready`, usually within a minute.
2. Mint a test token (Settings → Agents & API → Test) with the scopes you need. `get_store` now answers `mode: "test"` and the `-test` address.
3. Build: `set_shipping`, `upsert_product` + `set_product_image`, `upsert_collection`, `get_storefront_draft` → edits → `publish_storefront`. Place an order on the sandbox address with a test card; the confirmation lands in the owner's inbox as `[Test] …`.
4. `promote_to_live { dryRun: true }` → read the summary → `promote_to_live`.
5. With the live token: `get_storefront_draft` (the promoted revision) → `publish_storefront`.

## Limits

- One sandbox per store, and it cannot be deleted from the dashboard.
- The sandbox has no custom domain, no plan of its own, no card and no statement.
- Promote copies content, not history: sandbox orders stay in the sandbox.
- Formahand Payments test-mode onboarding runs in the dashboard's own embedded form (with test data); an agent still cannot complete it.

## Launch mode

A store can be kept private until the merchant is ready (launch-mode). To look at it meanwhile, `get_preview_link` (scope `storefront:read`, also `GET /api/storefront-preview`) answers `{ published: { url, expiresAt }, draft: { url, expiresAt }, launchMode }`: two signed links, good for 24 hours, that open the storefront on any launch mode without changing it, one on what shoppers would see and one on the unpublished draft; nobody has to switch the store to password mode to see their own work. `get_launch_mode` reads `{ access: "open" | "password" | "coming_soon", message, hasPassword, updatedAt }`; `set_launch_mode { access, password?, message? }` replaces it. While `access` is `password` or `coming_soon`, every storefront HTML page answers one themed "Coming soon" page with the merchant's message, a password box and the newsletter signup. The storefront read API, the shopping feeds, signed download links and the shopper `/account` pages keep working, so a headless front end and a shopping engine are unaffected by a shop that has not opened yet.

Switching to `password` needs a `password` the first time; opening never needs a password. Setting a new password signs every preview visitor out. `coming_soon` is that page without the password box, because a store that has never opened has nothing to preview: it is where every new store starts (readiness), and opening one goes through `storefront_open` readiness, so a shop with nothing to sell, no way to take money, no delivery or no policy pages is refused with the checklist.

## Governance: expiry, propose mode and pauses

**Expiry.** A token may carry an end date: `POST /api/tokens { expiresInDays }`, 1–365, or omitted for a token that never expires (the mint form calls it "stops working after"). The tokens list shows it, and verification refuses the token from that moment; on `/api/*` the refusal is **401 `{ error, reason: "expired" }`**, so an agent can tell a stale credential from a wrong one and ask the merchant for a new token rather than re-trying. Three days before it expires, the owner is emailed once. Nothing is deleted: an expired token is simply inert.

**Propose mode.** A token minted (or later switched) with `approvalMode: "propose"` may read everything its scopes allow and **write nothing**. Every write tool call is stored instead of run:

```json
{ "proposed": true, "proposalId": "…", "summary": "upsert_product on “Blue mug”",
  "note": "Stored, not run: this token is in propose mode and the change is awaiting the owner's approval in Settings → Agents & API." }
```

The proposal keeps the tool, the exact arguments, a summary and a `status` of `pending`, `approved`, `rejected`, `expired` or `failed`. The owner approves or rejects it in Settings → Agents & API; approving replays the stored call with that same token's scopes and environment through the same tools `/mcp` exposes, and keeps the result with the proposal (a replay that fails is recorded as `failed`, with the error, rather than silently vanishing). A pending proposal lapses after seven days. Four tools keep the agent in the loop, `list_proposals`, `get_proposal`, `withdraw_proposal` and `describe_governance`, and are never themselves proposed. Propose mode is not a scope: a token with `products:write` in propose mode still needs that scope, it just cannot use it unattended.

**Pauses.** A token that writes unusually fast, or keeps failing the same way, can be paused.

A paused token answers **403 `{ error, reason: "paused", steps }`** on `/api/*` and the same body as an `isError` tool result on `/mcp`. Only the owner resumes (or revokes) a paused token, from the dashboard: stop and tell them.

### Auditing an agent

Every write a token makes is recorded with the actor `token:<id>`, so `GET /api/events?actor=token:<id>&limit=50` (and `list_events { actor }`) is one agent's complete trail: what it changed, when, and which fields. A trailing `*` matches a prefix (`actor=token:*` for every agent, `actor=system:*` for the platform's own jobs). Settings → Agents & API shows the same thing as **What your agents did**: every token with its last activity, the last 50 changes of the one you pick, and the revoke button next to it.

In propose mode the REST routes refuse writes outright with 403 `{ error, reason: "propose" }` (there is no stored call to replay there); only MCP tool calls become proposals. `describe_governance` reports which mode the token is in.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `get_environment` | `settings:read` | Both environments of the store: { live: { hostname, url, runtimeStatus, runtimeVersion }, sandbox: null \| { … }, sandboxEnabledAt, selected } where selected is this token's mode. |
| `enable_sandbox` | `settings:write` | Create the store's sandbox (a second, complete store at <slug>-test.<domain> with payments in test mode, free test labels and owner-only email) if it does not exist. |
| `promote_to_live` | `settings:write` | Copy the sandbox's content to the live store: products with variants and images, collections, pages, navigation, SEO, shipping settings and rates, discounts (recreated on the live payment account), email templates, flows (disabled on arrival,… |
| `refresh_sandbox` | `settings:write` | Overwrite the sandbox's content with the live store's (same resources as promote_to_live, in the other direction) and publish it in the sandbox so it mirrors what shoppers see. |
| `list_proposals` | any | The writes this token proposed instead of running, newest first, with their status (pending, approved, rejected, expired, failed). |
| `get_proposal` | any | One proposal this token made, with the exact arguments it stored and, once the owner has approved it, what running it returned. |
| `withdraw_proposal` | any | Take back a proposal this token made while it is still pending, for instance when you worked out a better change. |
| `describe_governance` | any | The rules this token works under: act or propose mode, when it expires, the money limits, and what pauses it. |
| `get_preview_link` | `storefront:read` | Two signed links that open the storefront whatever its launch mode, so you can look at a "Coming soon" or password-protected store without opening it or changing anything: { published: { url, expiresAt }, draft: { url, expiresAt }, launchMode }. |
| `get_launch_mode` | `settings:read` | Whether the live storefront is open or still private (live data): { access: { access: "open" \| "password" \| "coming_soon", message, hasPassword, updatedAt } }. |
| `set_launch_mode` | `settings:write` | Opens the live store to shoppers, or keeps it private. |

## Read more

- https://formahand.com/docs/environments
- https://formahand.com/docs/agents
