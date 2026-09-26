---
name: formahand
description: "Work with a Formahand store through its MCP tools and API. Use when working with a Formahand store: set up, design the storefront, products, shipping, payments readiness, apps, store functions, integrations, events, webhooks, environments, money. Covers the first calls of every session, the draft to publish loop, the walls and money decisions that stay with the merchant, and which reference to read for each task."
---
# Formahand

Formahand is a hosted store platform. A merchant's store has a storefront (a theme, sections and pages that change through a draft and reach shoppers only on publish), a catalog, checkout, orders, shipping, email, an event log and automations. Everything the dashboard does to a store, an agent can do with a store API token, through the MCP endpoint or the REST API; a short list of decisions (money, signatures, credentials) always stays with the merchant.

## Core principles

1. **Read before you write.** Start every session with the five calls below. They say which store and environment you are in, what the merchant has and has not set up, and what to do next.
2. **Money is the merchant's.** You never add a payment method, choose a paid package, accept terms or pay. When something costs money, relay the store's own sentence (`askMerchant`) and its link, then stop.
3. **Nothing reaches shoppers until `publish_storefront`.** Storefront work goes to a draft. Preview it with `get_preview_link` before you publish.
4. **Try it in the sandbox first.** A test token (`fh_test_`) works on a separate test store with test payments and owner-only email. Promote to live when it works.
5. **Follow the store's checklist, not a guess.** `describe_readiness` and `get_started` say what is missing and which tool finishes it; a requirement with no tool is the merchant's to do.
6. **Keep the key secret.** Keep the store key in `$FORMAHAND_KEY`; never print, echo or log it.
7. **Ask when a choice binds the merchant.** Prices that include tax, tax registrations, refunds, anything sent to their customers: confirm with the merchant first.

## Connecting

- MCP endpoint (Streamable HTTP, bearer token): https://formahand.com/mcp
- Claude Code: `claude mcp add formahand --transport http https://formahand.com/mcp --header "Authorization: Bearer $FORMAHAND_KEY"`
- Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.formahand]
url = "https://formahand.com/mcp"

[mcp_servers.formahand.http_headers]
Authorization = "Bearer <token>"
```

- Any MCP client: `{ "mcpServers": { "formahand": { "type": "http", "url": "https://formahand.com/mcp", "headers": { "Authorization": "Bearer <token>" } } } }`
- A config file holds the token itself (`<token>` above), so keep it out of version control and never paste it into chat.
- REST: the same work at `https://formahand.com/api/*` with `Authorization: Bearer $FORMAHAND_KEY`; the OpenAPI reference is https://formahand.com/api/openapi.json and https://formahand.com/api lists every tool with its scope.
- The whole documentation: https://formahand.com/llms.txt and https://formahand.com/docs.

A token carries scopes. A tool the token lacks the scope for answers an error naming what it needs; `get_started` and `describe_governance` say up front what this token can and cannot do.

| Scope | Lets the token |
| --- | --- |
| `storefront:read` | Read the storefront draft and module catalogue |
| `storefront:write` | Edit the storefront draft (nothing goes live) |
| `storefront:publish` | Publish the draft to shoppers |
| `products:read` | Read products and collections |
| `products:write` | Create and edit products and collections |
| `orders:read` | Read orders |
| `orders:write` | Fulfill and edit orders |
| `orders:refund` | Refund orders, within the agent limits |
| `gift_cards:issue` | Issue and adjust gift cards (store credit is money) |
| `settings:read` | Read shipping, discounts, integrations and event subscriptions |
| `settings:write` | Change shipping, discounts, integrations and event subscriptions |
| `events:read` | Read the store event log |
| `domains:read` | Read the store's domains and DNS setup instructions |
| `domains:write` | Add, check, remove and switch the primary custom domain |
| `customers:read` | Read and search customers |
| `customers:write` | Add customers and change their tags |
| `customers:export` | Download the customer list as a CSV file |
| `marketing:send` | Create and send email campaigns to subscribers |
| `email:send_template` | Send one of the store's own templates to one address |
| `payments:write` | Change the payment features and move the store's balance |

## How a store works

### What a store is

One merchant store = one separate store with its own data and media library, one or more hostnames, one payment account, one plan. `get_store` names it (id, hostnames, runtime status, theme preset, live version, `mode`). Everything a token can do the dashboard can do; there is no separate agent data path.

Every write is validated by the store and recorded in the event log (`list_events`).

### Live vs test (the sandbox)

A store has a live environment and, once enabled, a sandbox: a fully separate store at `<slug>-test.<domain>` with payments in test mode, test labels and email routed only to the owner with a “[Test]” subject. Orders there are not real: no usage, no bill, no plan wall.

A token acts on exactly one environment, fixed by its prefix: a live token (`fh_live_…`) reaches the live store, a test token (`fh_test_…`) the sandbox. `get_store` and `get_environment` return `mode`; there is no per-call switch.

`get_environment` shows both; `enable_sandbox` creates the sandbox (poll until `sandbox.runtimeStatus` is `ready`); `refresh_sandbox` overwrites it from live; `promote_to_live { dryRun: true }` previews, then `promote_to_live` copies sandbox content to live (see Promote). Live-only: custom domains, plan and billing, statements, the payment profile.

### Draft → publish

Storefront appearance and copy (theme settings, sections, modules, hero image, navigation, pages) live in a revision-checked draft: `get_storefront_draft` → `apply_storefront_commands` / `set_hero_image` / `set_navigation` / `set_content_pages` with that `revision` (discover commands with `list_modules`) → `publish_storefront { revision }`. Nothing shoppers see changes until publish; a stale revision is a conflict, re-read and retry.

Everything else, products, collections, images, SEO, shipping, discounts, integrations, subscriptions, flows, templates, changes the store the moment the tool returns.

### Before you act: readiness

Anything that needs setting up first answers one checklist: `describe_readiness { feature? }` → `{ features: [{ feature, label, ready, requirements: [{ id, label, done, detail, href, tool?, activateHref?, severity? }] }] }`, over `storefront_open`, `checkout`, `payment_links`, `discount_codes`, `campaigns`, `transactional_email`, `shipping_labels` and `live_promotion`. Call it before `set_launch_mode`, `send_payment_link`, `upsert_discount`, `buy_shipping_label` and `promote_to_live`: each refuses with 409 and this checklist, not halfway. A requirement's `tool` finishes it, its `href` is the page the merchant does, `activateHref` the feature they switch on; `severity: "recommended"` never blocks. A new store starts closed, so opening it is the last step.

### Act or propose

A token either acts or proposes. In propose mode every write you make is stored as a proposal for the owner to approve, and the tool answers `proposed: true` with a `proposalId` instead of the result. That is not a failure: tell the merchant what is waiting for them. `describe_governance` says which mode this token is in, when it expires and what the owner's money limits are; `list_proposals` shows what is pending. A token can also be paused by the owner or expire: stop and ask the merchant for a new one rather than retrying.

### Never possible with a token

Minting or revoking tokens, changing team members, accepting terms, payment onboarding, the payment profile, paying a statement, setting the agent refund limits, changing a token's own governance or approving its proposals, choosing the dashboard's environment, exporting the whole store and closing it: session-only routes refuse tokens with 401 or 403. Everything else is token work. A secret comes back once, from the call that creates it (`secretShownOnce`); reads report only `secretSet`.

## The first five calls

Make these at the start of every session, in this order, before you change anything:

| # | Call | Scope | Why |
| --- | --- | --- | --- |
| 1 | `get_store` | any | Which store and which environment this token reaches (`mode` is `live` or `test`), and its hostnames. |
| 2 | `describe_account` | `settings:read` | The plan, the payment profile (`paymentProfile.state`) and every paid feature with the sentence to relay. Live store only: a test token gets an error, which is expected. |
| 3 | `describe_readiness` | `settings:read` | What each feature still needs before it works. Read it before opening the shop, sending payment links, codes, labels, custom code, store functions or promoting. |
| 4 | `get_started` | any | The store's state and an ordered checklist of the next calls; `next` is the one to make. Re-run it after each step. |
| 5 | `list_modules` | `storefront:read` | The storefront editor commands and the modules a section is built from. Pass `{ id }` for one module's full settings. |

Then follow `get_started.next`, re-running `get_started` after each step. In a long session, `describe_governance` tells you why a write may wait for approval.

## Money rules

- If a tool answers 402 or 409 with a wall, or a meter is nearly full, call describe_account and give the merchant the askMerchant sentence for that feature, with its link.
- Before suggesting a package, read paymentProfile.state and never ask the merchant to pay for something twice.
- Never add a payment method, choose a paid package or pay for anything: those are the merchant's.
- **402** is a plan wall (`reason: plan`): the feature needs a higher plan. `start_upgrade` only returns the page where the merchant pays; give them that link.
- **409** with `wall: true` is a setup, limit or paused wall: follow its `steps`. A step with `tool` is yours to call; a step with only `href` or `activateHref` is the merchant's page. **409** with a `readiness` checklist means the feature is not set up yet (`describe_readiness`).
- Say `askMerchant` as it stands, with its link, and stop. It is already written for the store's `paymentProfile.state` (`none`, `ready` or `expired`), so it never asks the merchant to pay twice. A feature already `active` is never offered again.
- Payments onboarding is the merchant's own step, on a form inside the dashboard. You can read whether payments are ready (`get_started`, `describe_readiness`) and fill in the payout profile ahead (`get_payout_profile`, `set_payout_profile`), but you never finish onboarding or enter card, bank or identity details.
- Refunds need the `orders:refund` scope and stay inside the owner's agent refund limit; a refund over it is refused, so ask the merchant to do it in the dashboard.
- Never add a tax registration the merchant has not told you they hold, and never change whether prices include tax without asking: both change what every shopper pays.
- The sandbox has no walls and nothing in it is billed, so test freely there.

Details: references/payments-money.md.

## Designing the storefront

Before you change how the storefront looks, call get_custom_code and read `allowed`: the HTML and CSS the store keeps, the theme's colour and font tokens and keyframes, and the editor rules. If you have a design skill, load it first (in Claude Code: /frontend-design:frontend-design; in Codex, the design skill you have installed); otherwise follow https://formahand.com/docs/custom-code. Preview with get_preview_link before you publish, check the page at phone width (about 390px) as well as desktop, and honour prefers-reduced-motion.

The loop:

1. `get_custom_code` and read `allowed` (the rulebook below, in full) and `revision`.
2. `get_storefront_draft` for the current document and `revision`; `list_modules` for the commands and module settings.
3. Make the change: `apply_storefront_commands` (atomic batches), `set_hero_image`, `set_logo`, `set_navigation`, `set_content_pages`, and for your own code `set_theme_css`, `set_custom_block` or `set_script_embeds`. Every write returns the next `revision`; use it for the next call.
4. `get_preview_link` and open the `draft` link: check desktop and phone width (about 390px), with and without reduced motion.
5. `publish_storefront { revision }`. If something went wrong, `list_storefront_versions` and `restore_storefront_version` bring an earlier version back.

What your HTML and CSS keep:

- Not allowed in a block: the page already has its one h1 (the store or page title), and a second one confuses screen readers and search engines. Start a block's headings at h2; an h1 you send keeps its text and loses the tag.
- A block runs no JavaScript. For analytics or chat, switch on a provider with set_script_embeds.
- Any other element loses its tags and keeps its text. Comments, on* handlers and the style attribute are removed.
- CSS at-rules kept: `@media`, `@supports`, `@starting-style`, `@keyframes`, `@property`, `@font-face`; removed: `@import`, `@charset`, `@namespace`, `@layer`, `@container`, `@page`, `@font-feature-values`, `@counter-style`, any other at-rule.
- Also removed inside rules: position: fixed (a fixed layer can cover the checkout button); url() that is neither https nor a path under your store's own /media/.
- Every selector in the theme stylesheet is published under [data-fh-storefront], the storefront's own <body>. :root, html and body mean that element, so page-wide colours and fonts go there.
- Every selector in a block's CSS is prefixed with that block's own class, so it only reaches inside the block. :scope or & means the block itself.
- Honour reduced motion: put animations inside @media (prefers-reduced-motion: no-preference), or switch them off under prefers-reduced-motion: reduce.
- Build on the theme tokens so the merchant's palette follows: `--fh-background`, `--fh-foreground`, `--fh-accent`, `--fh-on-accent`, `--fh-muted`, `--fh-border` and `--fh-font-heading`, `--fh-font-body`.

Editor rules:

- The home page has four sections, each exactly once: hero, collection, more-products, styles. Reorder them with move-section (index 0 to 3; blocks keep their places) or set-layout.
- Every page is one ordered list of sections and custom blocks (the draft's document.layout): "home", "catalog", "product" and "page:<content page slug>", whose one section is its own body, "main". Place a block with position { placement: "before" | "after", target: a section id, a block id, "start" or "end", page?, slot? } on add-custom-block, move-custom-block or update-custom-block; reorder a whole page with set-layout { page, order }, which lists every section once and every block that should be there.
- The header.top slot (under the announcement bar, above the navigation) and the footer.top slot (above the footer) hold blocks too, on every page but the payment page: position { target: "start", slot: "header.top" }.
- sectionId without position still means right after that section and after the blocks already there, so blocks added one by one stay in the order they were sent. A block beside a hidden section keeps rendering.
- A block target that is not on the storefront, a section on the wrong page, or a set-layout that leaves out a block on that page is refused, naming the command.
- The collection section (the home page's product grid) must stay visible; the others can be hidden with set-section-visibility. To make it quieter, restyle it with set_theme_css or hide elements inside it with set-element-style.
- Hiding a section that is already hidden, or showing one already shown, changes nothing and is not an error.
- At most 30 commands per apply_storefront_commands call. A batch applies whole or not at all, and a refusal names the command's position, type and target.
- At most 40 custom blocks per store.
- Text colours must stay readable on the background; set-colors refuses a pair that is too close.
- A logo or favicon is one of the store's own uploads; set_logo and set_favicon upload for you.

Editor commands (`apply_storefront_commands`): `set-copy`, `set-colors`, `set-logo`, `set-favicon`, `set-fonts`, `set-social`, `set-badges`, `set-theme`, `set-section-visibility`, `move-section`, `set-section-content`, `set-hero-image`, `set-area-content`, `set-navigation`, `set-content-pages`, `set-module-settings`, `set-element`, `set-element-style`, `add-custom-block`, `update-custom-block`, `remove-custom-block`, `set-theme-css`, `set-script-embeds`, `move-custom-block`, `set-layout`. Fields for each are in references/storefront.md; `list_modules` answers the live schema.

## Where to look

| Task | Read |
| --- | --- |
| Look up any tool, its scope and its inputs | [references/tools.md](references/tools.md) |
| Change how the storefront looks: theme, sections, copy, blocks, custom HTML and CSS, publishing | [references/storefront.md](references/storefront.md) |
| Products, collections, images, orders, fulfilment, returns, gift cards, customers, campaigns, shipping | [references/commerce.md](references/commerce.md) |
| Payments readiness, payment features, tax, walls, statements, plan, anything that costs money | [references/payments-money.md](references/payments-money.md) |
| React to events: subscriptions, webhooks, flows, schedules, inbound URLs, secrets, integrations, recipes | [references/integrations-events.md](references/integrations-events.md) |
| Build an app: manifest, grants, records, blocks, timers | [references/apps.md](references/apps.md) |
| Custom logic in the cart or checkout: store functions, hooks, testing, going live | [references/store-functions.md](references/store-functions.md) |
| Sandbox, test and live tokens, promote and refresh, preview links, launch mode, governance | [references/environments.md](references/environments.md) |
| An error, a refusal, a conflict, a 402/409/429, a long answer or paging | [references/troubleshooting.md](references/troubleshooting.md) |

## Answers worth recognising

- `isError: true` on a tool result is the store refusing, with its reason in the text: read it, fix the input, and do not repeat the same call unchanged.
- A stale `revision` is a conflict: read the draft again and redo the change on the new revision.
- A refused storefront batch names the command: `Command 3 (set-section-visibility, collection): …` with `position` (0-based). Nothing in the batch was applied.
- `429` means slow down: wait the `Retry-After` seconds, then continue.
- A list with `nextCursor` has more: pass it back as `cursor` until it is empty. A `describe_*` tool with `sections` answers an overview first; ask for the part you need.

Details and every error shape: references/troubleshooting.md.
