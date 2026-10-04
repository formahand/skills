# Commerce: catalog, orders, shipping and customers

Day-to-day store work: products and collections, orders and fulfilment, returns and edits, gift cards, customers and campaigns, and shipping. These tools change the live store the moment they return (or the sandbox, with a test token); there is no draft.

## Set up a whole store

An agent with `products:write`, `settings:write`, `storefront:write` and `storefront:publish` can take a new store from empty to live:

1. `get_started`, the checklist for this store, then `get_store` to confirm `runtimeStatus: "ready"` and note `primaryHostname`.
2. `set_shipping { allowedCountries: ["US","CA"], rates: [{ name: "Standard", amountMinor: 700, currency: "USD", freeOverMinor: 7500, minDays: 3, maxDays: 7 }], origin: { … } }`.
3. For each product: `upsert_product { product: { name, slug, priceMinor, currency, inventory, description, tags, kind?, backorder? } }` → note the returned `id`, then `set_product_image { productId, url }` or `{ productId, dataBase64, contentType }` (and `slot: "gallery"` or `{ variantId }` for more photos). A `digital` product then takes `upload_product_file`.
4. Anything the catalog has no box for goes on the object with `set_custom_fields` (call `describe_custom_fields` first).
5. `upsert_collection { collection: { name, slug, kind: "manual", productIds } }` or a tag rule, then `set_collection_image`.
6. `set_seo { title, metaDescription, socialTitle, socialDescription }` store-wide, and per product or collection with `{ ownerType, ownerId }`.
7. `get_storefront_draft` → `revision`; `apply_storefront_commands { revision, commands: [{ type: "set-copy", heading, intro }, { type: "set-theme", presetId }] }`; `set_hero_image`; `set_navigation`; `set_content_pages` (each call returns the next revision).
8. `publish_storefront { revision }`.
9. `describe_readiness` and `describe_walls` for what is left with the merchant (payments, a card, a domain), and `install_recipe` for any pattern the store needs from day one.
10. `set_launch_mode { access: "open" }` when `describe_readiness` says the shop is ready, a new store starts closed, so nothing sells until this call.

**Payments are the merchant's own step.** An agent can read whether they are ready (`get_started`, `get_plan`) but can never finish them: the merchant finishes payouts in **Payments**, on a form embedded in the dashboard, no other site, no separate login, or enters their PayPal keys or bank-transfer instructions there. Say that in those words when reporting it; do not send them a processor URL.

## Catalog

`upsert_product` (full product each time; `slug` is the identity; up to 3 options and 100 variants) then `set_product_image` (primary, gallery or variant); `upsert_collection` (manual `productIds` or tag rules) then `set_collection_image`. Images come from an https URL or inline base64 (`upload_image` keeps one for later).

Content pages and header/footer links are draft edits (`set_content_pages`, `set_navigation`); store-wide SEO is live (`set_seo`).

Custom fields (`set_custom_fields`, namespaced keys like `supplier.order_id` on any object) hold anything else the store needs and appear in views and events. Products have a `kind` (physical, digital with expiring download links, service) and a backorder flag. Pricing, shipping and payment rules are checkout data (`upsert_pricing_rule`, `upsert_checkout_rule`, customer groups); check one with `simulate_checkout` first. Missing a feature (free gifts, bundles, personal codes)? Build it as store code: `describe_store_functions`.

## Shipping

Three modes, chosen with `set_shipping { mode }`: `self` (flat rates, the merchant ships), `formahand` (Formahand Shipping: `get_shipping_label_rates` → `buy_shipping_label` from a paid order, billed on the monthly statement, tracking included) or a courier platform id from `list_integrations` (`connect_integration` with the merchant's credentials, `test_shipping_rates` to check). `get_shipping` returns the settings and the modes this deployment offers. Labels need the ship-from `origin` (with email and phone) and, live, the merchant's payment profile.

Orders ship with `fulfill_order` (with `lines` it ships part: `partially_fulfilled` until the last line) or `set_order_tracking`; `get_order_tracking` refreshes where a parcel is; `add_order_note` and `cancel_order` do the rest. `refund_order` needs the `orders:refund` scope and the owner's refund limit; over it, a `refunds` limit wall.

## Orders and fulfilment

`POST /api/order-actions` is the single route behind every button on an order, and it answers a browser session and a bearer token the same way: the same body, the same JSON back, the same entry in the event log (with the actor `token:<id>` when an agent did it). Body: `{ storeId?, orderId, action, … }`.

| Action | Scope | Extra fields | Does |
| --- | --- | --- | --- |
| `fulfill` | `orders:write` | `carrier?`, `trackingNumber?`, `trackingUrl?`, `notifyCustomer?`, `lines?` | Marks the order fulfilled, registers tracking and, with `notifyCustomer`, sends the store's `order-shipped` email. With `lines` it ships part of the order |
| `refund` | `orders:write` **and** `orders:refund` | `amountMinor?`, `note?`, `restock?` | Refunds through the processor that took the payment; without `amountMinor` the whole unrefunded remainder. `restock: true` returns the goods to stock (off by default). Agents also meet the refund limit |
| `cancel` | `orders:write` | none | Cancels an unfulfilled order, returns stock (a paid order restocks from its own lines) and closes an open checkout. Refused once any line has shipped |
| `tags` | `orders:write` | `tags` | Replaces the order's tags; an empty list clears them |
| `track` / `set-tracking` / `mark-paid` | `orders:write` | `carrier?`, `trackingNumber?`, `trackingUrl?`, `notifyCustomer?` | Refreshes tracking, records a number you shipped yourself, or records payment on a bank-transfer order |
| `send-payment-link` | `orders:write` | `notifyCustomer?` | Opens a card checkout for an order you created yourself and emails the customer the link (see below) |

The MCP tools `fulfill_order`, `refund_order`, `cancel_order`, `add_order_note` and `tag_order` call this route, so the rules below apply identically.

### Orders you create yourself

Not every sale starts on the storefront. A phone order, a counter sale or an invoice is created with `POST /api/orders` (`orders:write`, or the `create_order` tool):

```json
{ "customer": { "email": "buyer@example.com", "name": "Ada", "phone": "+1555" },
  "lines": [{ "productSlug": "granola", "variantId": null, "quantity": 2 }],
  "shipping": { "amountMinor": 500, "label": "Courier" },
  "discountMinor": 200, "note": "Ordered over the phone.",
  "shippingAddress": { "line1": "1 Loop Road", "city": "Portland", "state": "OR", "postalCode": "97201", "country": "US" } }
```

The store prices every line from its **published catalog** and reserves the same stock a shopper's checkout reserves, so the order is real inventory, not a note; a product with options must name a `variantId`, an unpublished slug is a 404 and not enough stock is a 409, in both cases before anything is held. Up to 20 lines and 10 of each. The answer is `{ order }`: `pending`, `source: "manual"`, with the next order number and `totalMinor` = subtotal + shipping − discount.

A line may also carry `unitPriceMinor`, the price the merchant agreed, instead of the catalog's. **A token cannot**: that field from a bearer token is refused with 403 `Only a signed-in merchant can set a price of their own on an order line.`, so an agent can only ever sell at the published price.

Then collect the money, two ways:

- **`POST /api/order-actions { action: "send-payment-link" }`** (or `send_payment_link`) opens a card checkout for that order and emails the customer the store's **Payment request** template, which carries `data.paymentUrl`. The answer is `{ order, paymentUrl, expiresAt, emailSent, emailError }`; `notifyCustomer: false` returns the link without sending it. The store needs card payments set up and the order must still be awaiting payment; anything else is a 409. The link may be **emailed** at most 3 times a day for one order and 10 times a day to one customer; past that the answer still carries `paymentUrl` with `emailSent: false` and an `emailError` saying so, so the merchant can pass the link on themselves.
- **`{ action: "mark-paid" }`** records cash or a transfer that already arrived, exactly as it does for a bank-transfer order.

Either way the order then behaves like any other: fulfil it, refund it, cancel it, export it. In the sandbox the payment link is a test-mode checkout on fake money. Merchants do the same from Orders → **Create order**.

### Partial fulfilment

Ship what you have: `fulfill` with `lines: [{ lineId, quantity }]`, where `lineId` is `items[].id` from `GET /api/orders` / `get_order`. Quantities are cumulative across calls and can never exceed what was ordered. While anything is still open the order reads `fulfillmentStatus: "partially_fulfilled"` and the result carries `fulfillment: { status, complete, lines: [{ lineId, productName, quantity, fulfilledQuantity }] }`; the call that ships the last line marks the order `fulfilled` exactly as a whole-order fulfilment does. A partly shipped order can no longer be cancelled, refund it instead.

Carrier, tracking and the buyer's shipping email belong to the whole order (one tracking number per order), so send them with the call that ships the last line, or use `set-tracking` afterwards; a partial call that carries them is refused with a 400 saying so.

### Refund limits for agents

Refunds are the one order action that moves money out, so besides the `orders:refund` scope there is a ceiling the owner sets in **Settings → Agents & API → Guardrails** (`#settings/agents/guardrails`): the most one refund may be, and the most all of a store's tokens may refund in a day (UTC, counted across every token and per environment). A merchant refunding in the dashboard has no limit and never uses up the agents' allowance.

Over either limit the refund is refused with **409** and the usual wall body, so an agent can hand the merchant the link instead of retrying or splitting the refund into smaller ones:

```json
{ "error": "An agent can refund at most <limit> on one order. …",
  "wall": true, "feature": "refunds", "reason": "limit",
  "steps": [{ "id": "dashboard", "label": "Refund this order from the dashboard", "done": false, "href": "/dashboard#orders/ord_…" },
            { "id": "limit", "label": "Raise the agent refund limit", "done": false, "href": "/dashboard#settings/agents/guardrails" }],
  "usage": { "used": …, "limit": … } }
```

## Returns

A buyer opens a return on the store's own order page; everything after that is agent work. `POST /api/return-actions` is the one route behind every move, and it answers a session and a bearer token the same way. Body: `{ storeId?, returnId, action, … }`.

| Action | Scope | Extra fields | Does |
| --- | --- | --- | --- |
| `approve` | `orders:write` | `label?` (`instructions`, `labelUrl`, `carrier`, `trackingNumber`), `merchantNote?`, `notifyCustomer?` | Approves a requested return and emails the buyer the store's `return-approved` template |
| `decline` | `orders:write` | `merchantNote` (required; the buyer reads it), `notifyCustomer?` | Declines it and emails the buyer |
| `receive` | `orders:write` | `lines: [{ lineId, quantity, condition, restock }]` | Records what actually came back, line by line |
| `refund` | `orders:write` **and** `orders:refund` | `amountMinor?`, `merchantNote?`, `notifyCustomer?` | Refunds through the order's own processor; without `amountMinor` the returned goods' value. The same cap and the same agent money limit as `refund_order` |
| `restock` | `orders:write` | none | Puts the received units of the ticked lines back on the shelf, once per return |
| `close` | `orders:write` | `merchantNote?` | Ends the return. Final |

The tools `list_returns`, `get_return`, `approve_return`, `decline_return`, `mark_return_received`, `refund_return`, `restock_return` and `close_return` call this route, so the rules are identical; `set_return_policy` writes the window, the policy and the instructions (`settings:write`).

**What an agent should check first.** `get_return` carries `status`, `linesMinor` (what the returned goods are worth) and `refundableMinor` (what the order still has left). A refund can never exceed `refundableMinor`, and a move the state machine forbids comes back as `A <status> return cannot become <status>.` rather than a silent no-op, read the status, do not retry.

**Return labels are not bought here.** No carrier module sells one, so `approve_return` takes a `labelUrl` the merchant already has; without it the buyer gets `instructions` (the store's own, from `set_return_policy`, when you pass none).

Full behaviour: [orders](https://formahand.com/docs/orders) "Returns and refunds".

## Editing an order

A buyer who wants one more of something, or a different address, does not need the order cancelled and re-made. `POST /api/order-edits` (`orders:write`) is the one route behind every move, and it answers a session and a bearer token the same way. Body: `{ storeId?, action, orderId? | editId?, … }`.

| Action | Tool | Does |
| --- | --- | --- |
| `begin` | `begin_order_edit` | Opens a draft holding the order exactly as it is. An order that may not be edited is refused with the reason |
| `set` | `edit_order_draft` | Sets lines, address, shipping method, discount, contact details and the reason. Only the fields you send are touched |
| `preview` | `preview_order_edit` | Prices the draft and answers `{ before, after, diff, deltaMinor, consequence, problems }`. Changes nothing |
| `apply` | `apply_order_edit` | Writes the edit and settles the money |
| `cancel` | `cancel_order_edit` | Throws the draft away; the order was never touched |
| `waive` | `waive_order_edit_balance` | Closes an unpaid difference, forgiven, or collected another way |
| `list` | `list_order_edits` | The audit trail: who, before, after, the difference, how it settled |

**Always preview before you apply.** `consequence.sentence` is the one line a merchant would read, "Buyer owes 12.40 USD. We'll email a payment link", or "Refund 8.00 USD to the buyer's card", and `problems` lists everything that would refuse the edit (stock, an address the store does not ship to, a discount code that does not apply). An edit that costs a customer more money is not a thing to discover afterwards.

**What an agent may not do.**

* Set a price of its own on a line. `edit_order_draft` refuses `unitPriceMinor` from a token with 403, exactly as `create_order` does: an agent sells at the catalog price the merchant published.
* Move more than the owner's money limit. The single-refund limit (Settings → Agents & API) governs an edit **in both directions**, because an agent that could raise an order without limit could empty a customer's card as easily as it could empty the merchant's. Over it, `apply_order_edit` is refused with `{ wall: true, feature: "order_edits", reason: "limit", steps }`, hand that to the merchant rather than splitting the edit into smaller ones.
* Edit an order that is unpaid, cancelled, refunded, fully shipped, or already waiting on an earlier edit's balance. `get_order` carries `edits.editable` and `edits.reason`, so an agent can tell before it starts.

`lines` on `edit_order_draft` is the **complete** list the order should end up with, like `tag_order`'s tags: read the draft first, send every line you want kept. A line that has already shipped cannot be removed or taken below what shipped.

Full behaviour: [orders](https://formahand.com/docs/orders) "Editing an order".

## Gift cards

A gift card is money somebody already paid, waiting to be spent ([payments](https://formahand.com/docs/payments) § Gift cards). **Reading the till is order data; issuing credit is money**, and the surface draws that line and no other.

| Tool | Scope | Does |
| --- | --- | --- |
| `list_gift_cards` | `orders:read` | Every card: what it is worth now, what it was issued for, who issued it, and what is outstanding in total. |
| `get_gift_card` | `orders:read` | One card with its whole history, issued, held at a checkout, spent, given back, refunded onto, adjusted, stopped. This is where "why is this card worth what it is worth" is answered. |
| `issue_gift_card` | `gift_cards:issue` | Puts new credit in the till. Pass `deliver` and the store emails the code to whoever it is for, in the merchant's own words from their own address. |
| `void_gift_card` | `gift_cards:issue` | Stops a card. Its balance stays on the record so the books add up; stopping moves no money. |
| `adjust_gift_card` | `gift_cards:issue` | Moves a balance by hand. Adding is issuing, and counts the same way; taking off never goes below zero. |

Three things to hold on to:

Hand it on in the same breath, or pass `deliver` and let the store send it. An agent that loses it voids the card and issues another.

**It is idempotent per sale.** `orderId` + `orderLineId` means a retry whose answer you never saw issues one card, not two. Always pass both when a sale paid for the card.

**You are held to the owner's money limits.** The same two numbers a refund obeys, counted separately. Over one you get a sentence and a dashboard link, not a card, hand the merchant the link rather than retrying.

## Customers and marketing

`list_customers` (`q`, `tag`, `minOrders`, `minSpentMinor`, `marketingStatus`), `get_customer`, `tag_customer` and `export_customers` (own scope, CSV) are the CRM. Campaigns are draft → preview → send: `create_campaign` → `preview_campaign` → `send_campaign`, which meets the marketing wall (`describe_marketing`, or `describe_readiness { feature: "campaigns" }`). Only subscribed contacts are mailed, and a token can never subscribe someone or accept the terms.

## Checkout rules and customer groups

Three kinds of row decide what one cart is charged and what it is offered, and all three are data an agent can write ([checkout-rules](https://formahand.com/docs/checkout-rules)).

- **Pricing rules**, automatic discounts with no code for the shopper to type. Kinds: `percent`, `amount`, `bxgy` and `free_shipping`. `applies` says which lines the discount comes off (`all`, chosen `productIds`, or `collectionHandles`); `priority` orders them, lower first; `stackable` decides whether they combine. Routes: `/api/pricing-rules`.
- **Checkout rules**, `hide_rate`, `show_rate_only`, `hide_payment`, `pickup` and `local_delivery`. A rule can only take an option away, or add one the merchant configured themselves: the store's zero-cost "Pickup at …" option, or local delivery at `target.feeMinor` under `target.rateName`. It can never invent a price of its own. Routes: `/api/checkout-rules`.
- **Customer groups**, a handle, a price list (`{ "<productId or variantId>": priceMinor }`, or `{ "%": -20 }` for a percentage off list), a minimum order, a tax flag and payment terms. `terms: "invoice"` makes bank transfer the only payment method for that group. Routes: `/api/customer-groups`.

Both rule kinds share one condition grammar: `minSubtotalMinor`, `minQuantity`, `firstOrder`, `customerGroup`, `customerTag`, `destinationCountries`, `destinationPostalCodes`, `productIds`, `collectionHandles` and `customField { ownerType, namespace, key, equals }`. Every part you state must hold.

`destinationPostalCodes` is a list of postal-code **prefixes** the ship-to address must start with, compared with spaces, hyphens, dots and case removed, so `SW1` matches `sw1a 2aa`. Formahand geocodes nothing: a local-delivery radius is written as the prefixes that ring the shop, and a minimum order is the ordinary `minSubtotalMinor`. A cart with no "Ship to" estimate matches no destination condition at all.

A discount never takes a line below zero, nor below the line's `supplier.cost` custom field when the store keeps one, and each line's discount is rounded to a whole number of minor units per unit in the shopper's favour, so the cart, the payment session and the order all carry the same integer price. The order records what applied, and the thank-you page shows it.

**When there is no feature for it, build it as store code.** A free gift with purchase, a bundle, a quantity break, a personal code after purchase, a loyalty discount, a progress bar: a [store function](https://formahand.com/docs/store-functions) decides (`cart.transform` adds lines, `cart.price` prices them and shows offers, `order.event` makes a code and mails it), an [app](https://formahand.com/docs/apps) keeps records and routes, and a storefront interaction presents it. For new agent-authored interactions, use section [behavior](https://formahand.com/docs/section-behaviors) and only the bounded APIs it exposes. Existing owner-authored [legacy section scripts](https://formahand.com/docs/section-scripts) remain a more powerful same-origin option; they are not the safe agent behavior API. The platform clamps every function answer: store code names products and quantities, never prices. Worked examples include a free gift ([store functions](https://formahand.com/docs/store-functions)), a sign-up popup and spend-progress bar ([legacy section scripts](https://formahand.com/docs/section-scripts), §9b); `describe_store_functions` carries the free-gift function. Without code, a cart page body section places `{% widget "offers" %}`, `{% widget "progress" %}` and `{% widget "quick-add" product: settings.gift %}`, or draws from `offers` and `cart.progress` (`describe_sections { parts: ["cart-page"] }`). On the order confirmation page, a `thankyou.content` function shows up to three offers. Section interactions run only where enabled and not on protected checkout/account pages. An app granted `shopper:identify` may know the signed-in shopper at its own route; only the documented legacy script API can call it with `api.shopper.fromApp(name, path)` today. The daily code limits (2,000 per store, 200 per function, 500 per flow or app, 1,000 per token or person) are defaults: `set_code_limits` (settings:write) raises or lowers them up to 50,000 a day, and `list_discounts` shows the limits in force.

**Check before you enable.** `simulate_checkout { items, customerEmail?, destinationCountry? }` (`settings:read`; naming a customer also needs `customers:read`) runs the engine over a hypothetical cart and answers with the priced lines, the rules that would apply, and the shipping and payment options. It creates no order, holds no stock and counts no usage. Never switch a rule on without simulating the cart it is meant for and one it is not.

Conditions that name the shopper, `customerGroup`, `customerTag`, `firstOrder`, only hold for a shopper signed in to the store.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `list_products` | `products:read` | Products with their variants, newest first: { products, limit, nextCursor }. |
| `get_product` | `products:read` | One product by id or slug → { product }. |
| `upsert_product` | `products:write` | Create a product (no id) or replace one (with id) → { success: true, id, variants: [{ id, title, values, sku }] }, the variant ids are what set_product_image { slot: { variantId } } takes, so no second get_product is needed. |
| `list_collections` | `products:read` | All collections (manual or tag-based) with member products. |
| `upsert_collection` | `products:write` | Create a collection (no id) or replace one (with id). |
| `delete_collection` | `products:write` | Delete one collection by id. |
| `set_product_image` | `products:write` | Give a product a photo and attach it. |
| `set_collection_image` | `products:write` | Give a collection its cover image: { collectionId, url \| dataBase64 + contentType }. |
| `upload_image` | `products:write` | Store an image in the store's own media library without attaching it, for use anywhere an https image URL is accepted (upsert_product imageUrl/images, upsert_collection imageUrl, set-hero-image, content pages): { url \| dataBase64 + contentType, alt? }. |
| `upload_section_video` | `products:write` | Store one MP4 or WebM video in this environment's own media library: { dataBase64, contentType }. |
| `delete_product` | `products:write` | Delete one product for good: { id \| slug } → { deleted: { id, slug, name } }. |
| `bulk_products` | `products:write` | The dashboard's bulk bar for a token: { ids: string[], action: 'archive' \| 'activate' \| 'delete' } → { updated, deleted, skipped }. |
| `export_products` | `products:read` | The catalog as CSV text: { format, filename, csv, rows, products, total, truncated, columns, note }. |
| `import_products` | `products:write` | Import a product CSV, the same parser the dashboard's migration wizard uses: { csv, dryRun?, onExistingHandle?, removeMissingVariants? } → { mode, rows, outcomes: [{ handle, rows, action: 'create' \| 'update' \| 'skip', reason?, warnings, changes?,… |
| `list_media` | `products:read` | Every image, video and licensed font in the store's own media library: { media: [{ key, url, bytes, contentType, uploadedAt, inUse, usedBy, fontValue? }], nextCursor, totalBytes }. |
| `delete_media` | `products:write` | Delete one image from the store's bucket: { key, force? } → { deleted: { key, bytes, usedBy } }. |
| `list_product_files` | `products:read` | The files a digital product delivers: { files: [{ id, name, bytes, contentType }] }, newest last. |
| `upload_product_file` | `products:write` | Attach a file to a digital product (its kind must be digital) and store it in the store's own media library: { productId, url \| dataBase64 + contentType, name? }. |
| `delete_product_file` | `products:write` | Remove one file from a digital product and delete the object from the store's bucket: { productId, fileId }. |
| `regenerate_download_links` | `orders:write` | Re-issue the download links of a paid order: { orderId } → { downloads: [{ name, url, expiresAt }] }, each valid 72 hours from now. |
| `list_custom_fields` | `products:read` | Every custom field on one object: { fields: [{ ownerType, ownerId, namespace, key, type, value }] }, ordered namespace then key. |
| `set_custom_fields` | `products:write` | Write custom fields on one object. |
| `delete_custom_field` | `products:write` | Remove one custom field by namespace and key. |
| `describe_custom_fields` | `products:read` | What custom fields are and how to use them: the six owner types, the seven value types (text, number, money, date, boolean, json, file) with their shapes and limits, the namespace and key rules, how merging works, which scopes each owner type needs,… |
| `get_seo` | `storefront:read` | Store-wide SEO settings: { seo: { title, metaDescription, socialTitle, socialDescription, noindex, aiAssistants } } (seo is null until set; aiAssistants is 1 when AI assistants may read the store, the default). |
| `set_seo` | `storefront:write` | Replace the store-wide SEO settings: title (<=60), metaDescription (<=160), socialTitle (<=70), socialDescription (<=200), noindex, and aiAssistants (false refuses AI crawlers in robots.txt and hides /llms.txt; absent keeps the current value). |
| `list_seo_overrides` | `storefront:read` | Every per-object SEO override the store has set: { overrides: [{ ownerType, ownerId, title, metaDescription, canonical, noindex, ogImageUrl, url, updatedAt }] }. |
| `list_redirects` | `storefront:read` | The store's path redirects: { redirects: [{ id, fromPath, toPath, status, hits, createdAt }] }. |
| `set_redirect` | `storefront:write` | Send one address somewhere else: { fromPath, toPath, status?: 301 \| 302 } (301 by default). |
| `delete_redirect` | `storefront:write` | Remove one redirect: { id } or { fromPath }. |
| `list_posts` | `storefront:read` | The store's posts, newest first: { posts: [{ id, slug, title, excerpt, bodyMarkdown, coverImageUrl, author, tags, status, publishedAt, url }] }, drafts included. |
| `upsert_post` | `storefront:write` | Write or rewrite a post: { id?, slug, title, excerpt?, bodyMarkdown?, coverImageUrl?, author?, tags?, status?: 'draft' \| 'published' }. |
| `delete_post` | `storefront:write` | Remove a post: { id }. |
| `set_post_image` | `storefront:write` | Give a post its cover image: { postId, url \| dataBase64 + contentType }. |
| `get_seo_audit` | `storefront:read` | Checks the store for search problems and changes nothing: missing, duplicate, too long or too short titles and descriptions, pages with no main heading or several, products with no image, images with no description or over 500 KB, product… |
| `get_seo_insights` | `orders:read` | The store's SEO insights, one part at a time: { part?: "summary" \| "topics" \| "improve" \| "add" \| "titles" \| "duplicates" \| "remove" \| "answers" \| "apply", offset?, limit? }. |
| `list_tracked_links` | `storefront:read` | The store's tracked links, newest first: { links: [{ id, code, name, targetPath, source, medium, campaign, discountCode, active, createdByKind, createdBy, createdAt, updatedAt, clicks, clicksLast30Days, orders, revenueMinor, url }], total, offset,… |
| `upsert_tracked_link` | `storefront:write` | Make or change one tracked link: { id?, code?, name, targetPath, channel?, source?, medium?, campaign?, discountCode?, active? } → { link }. |
| `delete_tracked_link` | `storefront:write` | Remove one tracked link: { id } or { code } → { deleted, id, code }. |
| `list_orders` | `orders:read` | Orders, newest first, with payment and fulfillment status, tags and items: { orders, limit, nextCursor }. |
| `get_order` | `orders:read` | One order with its items, shipping address, tags and timeline. |
| `fulfill_order` | `orders:write` | Mark an order fulfilled with optional carrier and tracking details, and optionally email the shipping confirmation to the customer. |
| `refund_order` | `orders:refund` | Refund a paid order through the processor that took the payment, in full or in part: { id, amountMinor?, reason?, restock? } → { order, refund }. |
| `cancel_order` | `orders:write` | Cancel an order that has not shipped: { id, reason? } → the cancelled order. |
| `tag_order` | `orders:write` | Replace an order's tags: { id, tags } → the order. |
| `add_order_note` | `orders:write` | Replace the merchant-facing note on an order: { id, note } → the order. |
| `export_orders` | `orders:read` | The order book as CSV text, the same file the merchant downloads from Orders (/docs/agents "Bookkeeping exports"): { format, filename, csv, rows, orders, truncated, columns, note }. |
| `print_packing_slip` | `orders:read` | The packing slip for one order as HTML: { id } → { orderId, orderNumber, filename, contentType, html }. |
| `create_order` | `orders:write` | Create an order the merchant sells themselves, by phone, at a counter, on an invoice: { customer: { email, name?, phone? }, lines: [{ productSlug, variantId?, quantity }], shipping?: { amountMinor, label }, discountMinor?, note?, shippingAddress?,… |
| `send_payment_link` | `orders:write` | Email the customer a card checkout for an order the merchant created: { id, notifyCustomer? } → { order, paymentUrl, expiresAt, emailSent, emailError }. |
| `begin_order_edit` | `orders:write` | Start editing a paid order that has not shipped: { orderId } → { edit }. |
| `edit_order_draft` | `orders:write` | Set what an open draft changes: { editId, lines?, shippingAddress?, shippingName?, shippingMethod?, discountCode?, manualDiscount?, customerEmail?, customerPhone?, reason? } → { edit }. |
| `preview_order_edit` | `orders:write` | Changes nothing: what applying this draft *would* do, { editId } → { before, after, diff, deltaMinor, consequence, priced, problems, tax }. |
| `apply_order_edit` | `orders:write` | Apply a previewed draft to the order: { editId, reason?, notifyCustomer? } → { edit, order, deltaMinor, consequence, paymentUrl?, refundedMinor? }. |
| `cancel_order_edit` | `orders:write` | Throw away an open draft: { editId } → { edit }. |
| `waive_order_edit_balance` | `orders:write` | Settle the balance an edit asked the buyer for without collecting it here: { editId, reason? } → { edit }. |
| `list_order_edits` | `orders:read` | The store's order edits, newest first: { edits, limit }. |
| `create_return` | `orders:write` | Open a return on a buyer's behalf (the phone call, the email, the person at the counter): { orderId, reason, note?, lines: [{ orderLineId, quantity }] } → { return }. |
| `list_returns` | `orders:read` | The store's returns, newest first: { returns, limit, nextCursor }. |
| `get_return` | `orders:read` | One return with its order, lines, reason, notes, refund so far and the return label or instructions the buyer was given: { id } → { return }. |
| `approve_return` | `orders:write` | Approve a requested return and tell the buyer how to send the goods back: { id, instructions?, labelUrl?, carrier?, trackingNumber?, merchantNote?, notifyCustomer? } → { return, emailSent }. |
| `decline_return` | `orders:write` | Decline a requested return: { id, reason, notifyCustomer? } → { return, emailSent }. |
| `mark_return_received` | `orders:write` | Record what actually came back, line by line: { id, lines: [{ lineId, quantity, condition, restock }], merchantNote? } → { return }. |
| `refund_return` | `orders:refund` | Refund a return through the processor that took the order's payment: { id, amountMinor?, merchantNote?, notifyCustomer? } → { order, refund, return, emailSent }. |
| `restock_return` | `orders:write` | Put the received goods back on the shelf: { id } → { return, restocked }. |
| `close_return` | `orders:write` | Close a return that needs nothing more: { id, merchantNote? } → { return }. |
| `set_return_policy` | `settings:write` | The store's return rules: { windowDays, policy?, instructions? } → { returns }. |
| `list_gift_cards` | `orders:read` | The store's gift cards: what each is worth now, what it was issued for, who issued it, whether it is still good, and what is outstanding in total. |
| `get_gift_card` | `orders:read` | One gift card with its whole ledger: issued, held at a checkout, spent, released when a checkout was abandoned, refunded back onto it, adjusted by hand. |
| `issue_gift_card` | `gift_cards:issue` | Puts new credit in the store's till and answers the code ONCE, nothing can read it back afterwards, so hand it to whoever it is for in the same breath. |
| `void_gift_card` | `gift_cards:issue` | Stops a gift card. Its balance stays on the record so the ledger still adds up, but it can no longer pay for anything. |
| `adjust_gift_card` | `gift_cards:issue` | Moves a card's balance by hand: a goodwill top-up, or taking credit back off one issued in error. |
| `get_shipping` | `settings:read` | Shipping settings: allowedCountries, flat rates, the ship-from origin, parcel defaults, the shipping mode ('self', 'formahand' or a courier platform id) and the modes available on this deployment. |
| `set_shipping` | `settings:write` | Replace the shipping settings: allowedCountries (ISO alpha-2), rates (max 5: name, amountMinor, currency, freeOverMinor\|null, minDays\|null, maxDays\|null), origin (line1, city, state, postalCode, country) and parcel defaults. |
| `get_shipping_label_rates` | `orders:read` | Formahand Shipping: quote label rates for a paid order from the store's ship-from address to the order's address with the order's weights: { order, rates: [{ rateId, carrier, service, name, costMinor, amountMinor (what you pay: cost + markup),… |
| `buy_shipping_label` | `orders:write` | Formahand Shipping: buy the label for a paid order at one of the quoted rates (rateId from get_shipping_label_rates). |
| `void_shipping_label` | `orders:write` | Formahand Shipping: void the order's current label. |
| `get_order_tracking` | `orders:read` | Where an order's parcel is: refreshes tracking through the store's courier platform or, in any mode, through Formahand's tracking, records it on the order and returns { order (carrier, trackingNumber, trackingUrl, trackingStatus, trackingEta,… |
| `set_order_tracking` | `orders:write` | Carrier and tracking number for an order you ship yourself. |
| `list_customers` | `customers:read` | The store's CRM contacts, newest first: { customers: [{ id, email, name, phone, marketingStatus, tags, totalOrders, totalSpentMinor, currency, createdAt, updatedAt, accountStatus, lastSeenAt }], nextCursor, total }. |
| `get_customer` | `customers:read` | One customer's whole page by id or email: { customer (tags, marketing status, order count, lifetime spend, group, country, accountStatus / lastSeenAt), orders (the last 20: number, date, totalMinor, currency, status), timeline (created, orders,… |
| `upsert_customer` | `customers:write` | Create or update one CRM contact by email, the same write POST /api/crm makes: { email, name?, phone?, tags?, notes?, marketingStatus? } → { customer }. |
| `add_customer_note` | `customers:write` | Add a note to one customer's timeline: { customerId, note }. |
| `tag_customer` | `customers:write` | Add or remove tags on one contact (by id or email): { id \| email, add?: string[], remove?: string[] }. |
| `export_customers` | `customers:export` | The customer list as CSV text (the same columns the merchant downloads: email, name, phone, marketing_status, tags, total_orders, total_spent, currency, created_at, updated_at), returned as { format, filename, csv, rows, truncated, note }. |
| `list_campaigns` | `customers:read` | Email campaigns of this environment, newest first: { campaigns: [{ id, subject, previewText, audience, status (draft \| sent \| partial \| failed), createdAt, sentAt, recipientCount, deliveredCount, suppressedCount, failedCount }] }. |
| `create_campaign` | `marketing:send` | Create a campaign DRAFT: { subject, previewText?, bodyHtml \| bodyMarkdown, audience: { tag?, segmentId?, marketingStatus: 'subscribed' } }. |
| `preview_campaign` | `customers:read` | One campaign draft with the audience it would reach now, the recipients of the next send (at most 50), and the marketing wall as it stands: { campaign, audience, recipients, allowed, wall, steps, warnings, caps }. |
| `send_campaign` | `marketing:send` | Send a draft to its audience (at most 50 subscribed contacts per call). |
| `describe_marketing` | any | Read-only. |
| `list_segments` | `customers:read` | The store's customer segments: { segments: [{ id, name, filter, builtIn, createdAt, updatedAt }], counts }. |
| `upsert_segment` | `customers:write` | Save a segment: { id?, name, filter }. |
| `delete_segment` | `customers:write` | Delete a saved segment: { id }. |
| `preview_segment` | `customers:read` | Who a filter would reach, before saving it: { filter } or { id } for a saved or built-in segment → { count, customers } with the first 10 contacts. |
| `list_pricing_rules` | `settings:read` | The store's automatic discounts (live data): { rules: [{ id, name, kind, value, conditions, applies, priority, stackable, startsAt, endsAt, enabled, usageCount }] }, in the order they run. |
| `upsert_pricing_rule` | `settings:write` | Creates or replaces one automatic discount on the live store. |
| `delete_pricing_rule` | `settings:write` | Removes one automatic discount from the live store. |
| `list_checkout_rules` | `settings:read` | The store's shipping and payment rules (live data): { rules: [{ id, name, kind, conditions, target, enabled }] }. |
| `upsert_checkout_rule` | `settings:write` | Creates or replaces one checkout rule on the live store: hide_rate, show_rate_only, hide_payment or pickup. |
| `delete_checkout_rule` | `settings:write` | Removes one shipping or payment rule from the live store; carts go back to seeing every rate. |
| `list_customer_groups` | `customers:read` | The store's customer groups (live data): { groups: [{ id, name, handle, priceList, minOrderMinor, taxExempt, terms, members }] }. |
| `upsert_customer_group` | `settings:write` | Creates or replaces one customer group on the live store: its handle, its price list ({ "<productId or variantId>": priceMinor } or { "%": -20 }), its minimum order, tax exemption and payment terms. |
| `set_customer_group` | `customers:write` | Puts one customer in a group, or takes them out with groupHandle null. |
| `simulate_checkout` | `settings:read` | Prices a hypothetical cart the way checkout would and reports what the rules do: { currency, lines, subtotalMinor, discountMinor, totalMinor, freeShipping, applied: [{ ruleId, name, amountMinor }], customer, shippingOptions, paymentProvider,… |
| `describe_shopper_accounts` | `customers:read` | How shoppers sign in to this store and what is set up: { signInEnabled, accounts, signInsThisWeek, routes, howItWorks }. |
| `set_shopper_accounts` | `customers:write` | Turn the store's shopper sign-in on or off: { enabled } → { signInEnabled, accounts, signInsThisWeek }. |
| `revoke_shopper_sessions` | `customers:write` | Signs one shopper out everywhere: deletes every sign-in session of that customer, so their next visit to /account asks for a new code. |
| `get_analytics` | `orders:read` | Everything the store knows about a period: { from?, to?, compare? } → { period, currency, revenueMinor, taxCollectedMinor, orders, paidOrders, averageOrderMinor, refundedMinor, abandonedCheckouts, abandonedRate, newCustomers, repeatRate,… |
| `get_top_products` | `orders:read` | What sold most: { period: "7d" \| "30d" \| "90d" \| "365d" } → { period, currency, bestSellers: [{ productId, name, quantity, revenueMinor }], paidOrders, revenueMinor }. |
| `export_analytics` | `orders:read` | A period's numbers as CSV text: { period: "7d" \| "30d" \| "90d" \| "365d" } → { format, filename, csv, rows, period, currency }. |
| `import_search_performance` | `settings:write` | Upload search performance rows the owner exported themselves: { rows: [{ day, query, page, clicks, impressions, position, engine?, country?, device? }], source? } → { imported: { rows, days, from, to, source } }. |
| `get_search_performance` | `orders:read` | What search performance says for a period: { from, to, groupBy?: "query" \| "page" \| "engine" \| "country" \| "device" \| "day", engine?: "google" \| "bing" \| "other", country?, device?: "desktop" \| "mobile" \| "tablet", limit? } → { period, groupBy,… |
| `get_order_sources` | `orders:read` | Where sales came from: channel, source, campaign and tracked link, with orders and sales for a period. |
| `describe_search_engines` | `settings:read` | Which search engines this store is connected to, read-only: {} → { mode, primaryHostname, engines: [{ engine, available, state, way, hostname, lastReadAt, lastError, readAccess, identities? }], requests: [{ id, engine, hostname, requestedBy,… |
| `connect_search_engine` | `settings:write` | Asks the owner to connect a search engine: { engine: "google" \| "bing" \| "indexnow" } → { request: { id, engine, hostname, state: "waiting_for_owner" }, approveAt, next }. |
| `get_feeds` | `settings:read` | Where this store publishes itself for shopping engines and AI agents: { } → { hostname, feeds: { google, csv, json }, channelFeeds: { meta, pinterest, tiktok }, llmsTxt, wellKnown, mcp, docs, summary: { items, inStock, outOfStock, backorder,… |
| `check_feed` | `settings:read` | Fetch this store's published product feed and report on it: { channel?: "google" \| "meta" \| "pinterest" \| "tiktok" } → { ok, hostname, feeds, items, inStock, outOfStock, backorder, problems: [{ id, title, field, message }], problemCount, checkedAt, error? }. |

## Read more

- https://formahand.com/docs/orders
- https://formahand.com/docs/shipping
- https://formahand.com/docs/product-kinds
- https://formahand.com/docs/checkout-rules
- https://formahand.com/docs/email-marketing
- https://formahand.com/docs/content-seo
