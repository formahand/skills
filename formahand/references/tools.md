# Formahand tools

Every MCP tool the Formahand endpoint serves (285 of them), grouped by the job they do: name, the scope a token needs ("any" means any valid token), what it does, and its top-level input (`?` marks optional). The full description of each tool is in the MCP `tools/list` answer and under `components.schemas` in https://formahand.com/api/openapi.json. Tools that say they act on "the token's environment" work in both live and test; the plan, billing and domain tools are live only.

## Orientation

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_store` | any | The store this token belongs to: id, name, mode (live \| test: the environment this token acts on), that environment's hostnames, runtime status, `theme` { id, name } (the look the draft wears, as list_themes names it; `themePresetId` is the older… | none |
| `describe_system` | any | How Formahand works, for an agent that has read nothing else: what a store is, live vs test mode and how a token's prefix binds it, the draft → publish loop, catalog/pages/navigation/SEO, shipping modes, payment providers, events → subscriptions →… | `{ sections?: string[] }` |
| `get_started` | any | The store's current state (mode, runtime status, product count, published?, draft pending?, payments ready?, shipping mode, custom domains, sandbox?, open walls) and an ordered checklist of the next tool calls with argument hints, so an agent can… | none |

## Storefront draft, brand, custom code, history and publishing

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_modules` | `storefront:read` | Discover what the storefront editor can do: the editor commands `apply_storefront_commands` accepts, as a JSON schema, and the storefront modules a section can be built from. | `{ id?: string }` |
| `get_storefront_draft` | `storefront:read` | The current storefront draft: { schemaVersion: 3, draft: { revision, document, publishedRevision, updatedAt, updatedBy }, liveVersion, publishedAt, modules }. | none |
| `apply_storefront_commands` | `storefront:write` | Apply editor commands (see list_modules) to the draft, at most 30 per call. | `{ revision: integer, commands: any[] }` |
| `publish_storefront` | `storefront:publish` | Publish the draft at this revision so shoppers see it. | `{ revision: integer }` |
| `set_hero_image` | `storefront:write` | Set the home page hero image of the storefront draft: { revision, url \| dataBase64 + contentType }. | `{ revision: integer, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `set_logo` | `storefront:write` | Set the storefront's header logo in the draft: { revision, url \| dataBase64 + contentType, alt?, width? }. | `{ revision: integer, alt?: string, width?: integer, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `set_favicon` | `storefront:write` | Set the browser-tab icon in the storefront draft: { revision, url \| dataBase64 + contentType }. | `{ revision: integer, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `set_navigation` | `storefront:write` | Replace the header and footer links of the storefront draft: { revision, navigation: { header: [{ label, href }] (max 12), footer (max 20) } }; href is a site path like /pages/about or an https URL. | `{ revision: integer, navigation: object }` |
| `set_content_pages` | `storefront:write` | Replace the storefront draft's content pages (About, FAQ, policies…): { revision, pages: [{ slug, title, bodyMarkdown, status: 'draft'\|'published' }] } (max 12, unique slugs; pages render at /pages/<slug>). | `{ revision: integer, pages: object[] }` |
| `get_custom_code` | `storefront:read` | Everything a store has added itself, and what it may add: { blocks: [{ id, where, index, sectionId, placement, pageSlug, width, visible, htmlBytes, cssBytes }], customCss, embeds: [{ provider, id, enabled }], meter: { used, limit, available, blocks,… | none |
| `set_custom_block` | `storefront:write` | Adds, changes or removes one custom HTML block in the draft: { revision, id?, html?, css?, position?, sectionId?, placement?, pageSlug?, width?, visible?, remove? }. | `{ revision: integer, id?: string, html?: string, css?: string, position?: object, sectionId?: "hero" \| "collection" \| "more-products" \| "styles", placement?: "before" \| "after", pageSlug?: string, width?: "content" \| "full", visible?: boolean, remove?: boolean }` |
| `set_theme_css` | `storefront:write` | Replaces the store's own stylesheet in the draft: { revision, css }. | `{ revision: integer, css: string }` |
| `set_script_embeds` | `storefront:write` | Replaces the store's script embeds in the draft: { revision, embeds: [{ provider, id, enabled? }] }. | `{ revision: integer, embeds: object[] }` |
| `list_storefront_versions` | `storefront:read` | What the storefront looked like before: { versions: [{ id, at, actor, kind, revision, sections, pages, changed, summary }], hasMore }. | `{ limit?: integer }` |
| `restore_storefront_version` | `storefront:write` | Put an old version back as the current DRAFT: { id } (from list_storefront_versions) → { restored, draft: { revision, document, publishedRevision }, published: false, dangling }. | `{ id: string }` |

## Catalog, images and media

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_products` | `products:read` | Products with their variants, newest first: { products, limit, nextCursor }. | `{ limit?: integer, cursor?: string }` |
| `get_product` | `products:read` | One product by id or slug → { product }. | `{ id?: string, slug?: string }` |
| `upsert_product` | `products:write` | Create a product (no id) or replace one (with id) → { success: true, id, variants: [{ id, title, values, sku }] }, the variant ids are what set_product_image { slot: { variantId } } takes, so no second get_product is needed. | `{ product: object }` |
| `list_collections` | `products:read` | All collections (manual or tag-based) with member products. | none |
| `upsert_collection` | `products:write` | Create a collection (no id) or replace one (with id). | `{ collection: object }` |
| `delete_collection` | `products:write` | Delete one collection by id. | `{ id: string }` |
| `set_product_image` | `products:write` | Give a product a photo and attach it. | `{ productId: string, slot?: "primary" \| "gallery" \| object, alt?: string, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `set_collection_image` | `products:write` | Give a collection its cover image: { collectionId, url \| dataBase64 + contentType }. | `{ collectionId: string, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `upload_image` | `products:write` | Store an image in the store's own media library without attaching it, for use anywhere an https image URL is accepted (upsert_product imageUrl/images, upsert_collection imageUrl, set-hero-image, content pages): { url \| dataBase64 + contentType, alt? }. | `{ alt?: string, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `upload_section_video` | `products:write` | Store one MP4 or WebM video in this environment's own media library: { dataBase64, contentType }. | `{ dataBase64: string, contentType: "video/mp4" \| "video/webm" }` |
| `delete_product` | `products:write` | Delete one product for good: { id \| slug } → { deleted: { id, slug, name } }. | `{ id?: string, slug?: string }` |
| `bulk_products` | `products:write` | The dashboard's bulk bar for a token: { ids: string[], action: 'archive' \| 'activate' \| 'delete' } → { updated, deleted, skipped }. | `{ ids: string[], action: "archive" \| "activate" \| "delete" }` |
| `export_products` | `products:read` | The catalog as CSV text: { format, filename, csv, rows, products, total, truncated, columns, note }. | `{ format?: "csv", status?: "all" \| "active" \| "archived" }` |
| `import_products` | `products:write` | Import a product CSV, the same parser the dashboard's migration wizard uses: { csv, dryRun?, onExistingHandle?, removeMissingVariants? } → { mode, rows, outcomes: [{ handle, rows, action: 'create' \| 'update' \| 'skip', reason?, warnings, changes?,… | `{ csv: string, dryRun?: boolean, onExistingHandle?: "skip" \| "update", removeMissingVariants?: boolean }` |
| `list_media` | `products:read` | Every image, video and licensed font in the store's own media library: { media: [{ key, url, bytes, contentType, uploadedAt, inUse, usedBy, fontValue? }], nextCursor, totalBytes }. | `{ limit?: integer, cursor?: string, orphansOnly?: boolean }` |
| `delete_media` | `products:write` | Delete one image from the store's bucket: { key, force? } → { deleted: { key, bytes, usedBy } }. | `{ key: string, force?: boolean }` |

## Orders

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_orders` | `orders:read` | Orders, newest first, with payment and fulfillment status, tags and items: { orders, limit, nextCursor }. | `{ status?: enum, fulfillment?: "all" \| "open" \| "unfulfilled" \| "partially_fulfilled" \| "fulfilled" \| "cancelled", q?: string, limit?: integer, cursor?: string }` |
| `get_order` | `orders:read` | One order with its items, shipping address, tags and timeline. | `{ id: string }` |
| `fulfill_order` | `orders:write` | Mark an order fulfilled with optional carrier and tracking details, and optionally email the shipping confirmation to the customer. | `{ id: string, carrier?: string, trackingNumber?: string, trackingUrl?: "" \| string, notifyCustomer?: boolean, lines?: object[] }` |
| `refund_order` | `orders:refund` | Refund a paid order through the processor that took the payment, in full or in part: { id, amountMinor?, reason?, restock? } → { order, refund }. | `{ id: string, amountMinor?: integer, reason?: string, restock?: boolean }` |
| `cancel_order` | `orders:write` | Cancel an order that has not shipped: { id, reason? } → the cancelled order. | `{ id: string, reason?: string }` |
| `tag_order` | `orders:write` | Replace an order's tags: { id, tags } → the order. | `{ id: string, tags: string[] }` |
| `add_order_note` | `orders:write` | Replace the merchant-facing note on an order: { id, note } → the order. | `{ id: string, note: string }` |
| `export_orders` | `orders:read` | The order book as CSV text, the same file the merchant downloads from Orders (/docs/agents "Bookkeeping exports"): { format, filename, csv, rows, orders, truncated, columns, note }. | `{ from?: string, to?: string, status?: enum }` |
| `print_packing_slip` | `orders:read` | The packing slip for one order as HTML: { id } → { orderId, orderNumber, filename, contentType, html }. | `{ id: string }` |

## Store profile

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_store_profile` | `settings:read` | The store's own identity: storeName, slug (the free address, which never changes), legalName, category, currency, phone, supportEmail, the postal address (addressLine1, addressLine2, addressCity, addressRegion, addressPostcode, addressCountry),… | none |
| `set_store_profile` | `settings:write` | Change the store's identity. | `{ storeName?: string, legalName?: string, category?: string, currency?: string, phone?: string, supportEmail?: "" \| string, addressLine1?: string, addressLine2?: string, addressCity?: string, addressRegion?: string, addressPostcode?: string, addressCountry?: string, timezone?: string, weightUnit?: "g" \| "kg" \| "lb" \| "oz" }` |

## Shipping and tracking

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_shipping` | `settings:read` | Shipping settings: allowedCountries, flat rates, the ship-from origin, parcel defaults, the shipping mode ('self', 'formahand' or a courier platform id) and the modes available on this deployment. | none |
| `set_shipping` | `settings:write` | Replace the shipping settings: allowedCountries (ISO alpha-2), rates (max 5: name, amountMinor, currency, freeOverMinor\|null, minDays\|null, maxDays\|null), origin (line1, city, state, postalCode, country) and parcel defaults. | `{ allowedCountries: string[], rates: object[], origin?: object, parcel?: object, mode?: string }` |
| `get_shipping_label_rates` | `orders:read` | Formahand Shipping: quote label rates for a paid order from the store's ship-from address to the order's address with the order's weights: { order, rates: [{ rateId, carrier, service, name, costMinor, amountMinor (what you pay: cost + markup),… | `{ orderId: string }` |
| `buy_shipping_label` | `orders:write` | Formahand Shipping: buy the label for a paid order at one of the quoted rates (rateId from get_shipping_label_rates). | `{ orderId: string, rateId: string, notifyCustomer?: boolean }` |
| `void_shipping_label` | `orders:write` | Formahand Shipping: void the order's current label. | `{ orderId: string }` |
| `get_order_tracking` | `orders:read` | Where an order's parcel is: refreshes tracking through the store's courier platform or, in any mode, through Formahand's tracking, records it on the order and returns { order (carrier, trackingNumber, trackingUrl, trackingStatus, trackingEta,… | `{ orderId: string }` |
| `set_order_tracking` | `orders:write` | Carrier and tracking number for an order you ship yourself. | `{ orderId: string, carrier?: string, trackingNumber: string, trackingUrl?: "" \| string, notifyCustomer?: boolean }` |

## SEO, redirects and posts

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_seo` | `storefront:read` | Store-wide SEO settings: { seo: { title, metaDescription, socialTitle, socialDescription, noindex, aiAssistants } } (seo is null until set; aiAssistants is 1 when AI assistants may read the store, the default). | `{ ownerType?: "product" \| "collection" \| "page" \| "post", ownerId?: string }` |
| `set_seo` | `storefront:write` | Replace the store-wide SEO settings: title (<=60), metaDescription (<=160), socialTitle (<=70), socialDescription (<=200), noindex, and aiAssistants (false refuses AI crawlers in robots.txt and hides /llms.txt; absent keeps the current value). | `{ ownerType?: "product" \| "collection" \| "page" \| "post", ownerId?: string, title?: string, metaDescription?: string, socialTitle?: string, socialDescription?: string, canonical?: "" \| string, ogImageUrl?: "" \| string, noindex?: boolean, aiAssistants?: boolean }` |
| `list_seo_overrides` | `storefront:read` | Every per-object SEO override the store has set: { overrides: [{ ownerType, ownerId, title, metaDescription, canonical, noindex, ogImageUrl, url, updatedAt }] }. | none |
| `list_redirects` | `storefront:read` | The store's path redirects: { redirects: [{ id, fromPath, toPath, status, hits, createdAt }] }. | none |
| `set_redirect` | `storefront:write` | Send one address somewhere else: { fromPath, toPath, status?: 301 \| 302 } (301 by default). | `{ fromPath: string, toPath: string, status?: 301 \| 302 }` |
| `delete_redirect` | `storefront:write` | Remove one redirect: { id } or { fromPath }. | `{ id?: string, fromPath?: string }` |
| `list_posts` | `storefront:read` | The store's posts, newest first: { posts: [{ id, slug, title, excerpt, bodyMarkdown, coverImageUrl, author, tags, status, publishedAt, url }] }, drafts included. | `{ id?: string, slug?: string }` |
| `upsert_post` | `storefront:write` | Write or rewrite a post: { id?, slug, title, excerpt?, bodyMarkdown?, coverImageUrl?, author?, tags?, status?: 'draft' \| 'published' }. | `{ id?: string, slug: string, title: string, excerpt?: string, bodyMarkdown?: string, coverImageUrl?: "" \| string, author?: string, tags?: string[], status?: "draft" \| "published" }` |
| `delete_post` | `storefront:write` | Remove a post: { id }. | `{ id: string }` |
| `set_post_image` | `storefront:write` | Give a post its cover image: { postId, url \| dataBase64 + contentType }. | `{ postId: string, url?: string, dataBase64?: string, contentType?: "image/jpeg" \| "image/png" \| "image/webp" }` |
| `get_seo_audit` | `storefront:read` | Checks the store for search problems and changes nothing: missing, duplicate, too long or too short titles and descriptions, pages with no main heading or several, products with no image, images with no description or over 500 KB, product… | `{ limit?: integer, severity?: "high" \| "medium" \| "low" }` |
| `get_seo_insights` | `orders:read` | The store's SEO insights, one part at a time: { part?: "summary" \| "topics" \| "improve" \| "add" \| "titles" \| "duplicates" \| "remove" \| "answers" \| "apply", offset?, limit? }. | `{ part?: enum, offset?: integer, limit?: integer }` |

## Discounts, integrations and checkout origins

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_discounts` | `settings:read` | Discount codes with kind, value, currency, conditions, what they apply to, whether they combine, the window, the store-wide and per-customer limits, status (active \| paused \| ended), usageCount and timesUsed, and codeLimits (the daily limits on… | `{ template?: string }` |
| `mint_discount_codes` | `discounts:issue` | Make single-use codes from a template discount (upsert_discount { template: true }): { template: id or code, count 1-100, prefix?, expiresInDays? 1-365, email? } → { template, codes: [{ id, code, expiresAt }] }. | `{ template: string, count?: integer, prefix?: string, expiresInDays?: integer, email?: string }` |
| `set_gift_exception` | `settings:write` | Let one rule give one product away below its recorded cost, or stop it: { subject: "function:<name>" \| "rule:<pricing rule id>" \| "code:<discount id or template id>", productId, allowed: true\|false } → { giftExceptions }. | `{ subject: string, productId: string, allowed?: boolean }` |
| `set_code_limits` | `settings:write` | Change the store's daily limits on codes made from a template: { perStore?, perCaller?: { function?, flow?, app?, token?, member? } }, each a whole number 1-50000, null for the default → { codeLimits: { perStore, perCaller, defaults, ceiling, changed } }. | `{ perStore?: integer, perCaller?: object }` |
| `list_gift_exceptions` | `settings:read` | Which rule, code or store function may give which product away below its recorded cost: [{ id, subject, productId, createdBy, createdAt }]. | none |
| `upsert_discount` | `settings:write` | Create or edit a discount code, a named trigger for the same rule engine as upsert_pricing_rule (/docs/checkout-rules): code (3-32 letters/digits), kind 'percent' (value 1-100), 'amount' (value in minor units, currency required), 'free_shipping' or… | `{ id?: string, code: string, kind?: "percent" \| "amount" \| "free_shipping" \| "bxgy", value?: integer, currency?: string, conditions?: object, applies?: object, combinable?: boolean, startsAt?: string, expiresAt?: string, maxRedemptions?: integer, perCustomerLimit?: integer, status?: "active" \| "paused" \| "ended", template?: boolean }` |
| `end_discount` | `settings:write` | End a discount code: { id }, it stops working immediately and stays on the list with its redemptions. | `{ id: string }` |
| `delete_discount` | `settings:write` | Delete a discount code that was never used: { id } → { deleted: true, id, code }. | `{ id: string }` |
| `list_integrations` | `settings:read` | Available integration modules (live shipping rates: the courier platforms list_integrations names) with their settings, plus this store's status, config and which secret keys are set. | none |
| `connect_integration` | `settings:write` | See… | `{ id: string, status: "active" \| "inactive", config?: object, secrets?: object }` |
| `test_shipping_rates` | `settings:read` | Quote one sample parcel through an integration module from the store's ship-from origin (set_shipping) to { country, postalCode }: { id, destination, weightGrams?, currency? }. | `{ id: string, destination: object, weightGrams?: integer, currency?: string }` |
| `get_checkout_origins` | `settings:read` | The front-end origins allowed to start checkout through POST /api/checkout from another domain (headless): { origins: [{ id, origin, createdAt }], maxOrigins }. | none |
| `set_checkout_origins` | `settings:write` | Replace the allowed front-end origins: { origins: ["https://shop.example.com", …] } (max 10; https host[:port] only, http://localhost for development). | `{ origins: string[] }` |

## Events and subscriptions

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_events` | any | What this store can emit and how to subscribe: the resource × verb vocabulary (created \| updated \| deleted) and semantic aliases (order.paid, order.delivered, …), topic pattern rules (`*` per segment), the filter grammar with examples, and the delivery envelope. | none |
| `list_events` | `events:read` | The store's event log, newest first: { events: [{ id, type, resource, resourceId, subject, occurredAt, actor, data, changed, previous }], nextCursor }. | `{ since?: string, before?: string, type?: string, resource?: string, actor?: string, limit?: integer }` |
| `get_event` | `events:read` | One event by id, with its full data, changed fields and previous values. | `{ id: string }` |
| `list_subscriptions` | `settings:read` | Event subscriptions: { subscriptions: [{ id, name, topics, filter, target, enabled, lastDeliveryAt, lastStatus, failureCount, disabledReason }], max }. | none |
| `create_subscription` | `settings:write` | Subscribe to events: { name?, topics: ["order.updated", "*.created", …], filter?: 'changed contains "trackingStatus" and data.trackingStatus == "delivered"', target: { kind: "webhook", url } \| { kind: "agent", url, bearer? }, enabled? }. | `{ name?: string, topics: string[], filter?: string, target: object, enabled?: boolean }` |
| `update_subscription` | `settings:write` | Change a subscription: { id, name?, topics?, filter?, target?, enabled? }. | `{ id: string, name?: string, topics?: string[], filter?: string, target?: object, enabled?: boolean }` |
| `delete_subscription` | `settings:write` | Delete a subscription and its delivery history. | `{ id: string }` |
| `test_subscription` | `settings:write` |  | `{ id: string }` |
| `list_deliveries` | `settings:read` | Delivery attempts of one subscription, newest first: { deliveries: [{ id, eventId, eventType, attempt, status: pending \| delivered \| failed, nextAttemptAt, lastAttemptAt, responseStatus, error }] }. | `{ subscriptionId: string, limit?: integer }` |
| `retry_delivery` | `settings:write` | Retry one pending or failed delivery now, with a fresh retry schedule. | `{ id: string }` |
| `list_webhooks` | `settings:read` | Legacy view of webhook subscriptions (use list_subscriptions): { webhooks: [{ id, url, events, status, secretSet, createdAt, lastDeliveryAt, lastStatus }] }. | none |
| `upsert_webhook` | `settings:write` | Legacy: create (no id) or update (with id) a webhook subscription from the fixed event list order.paid, order.fulfilled, order.cancelled, order.refunded, product.updated (prefer create_subscription, which takes any topic pattern and a filter). | `{ id?: string, url: string, events: "order.paid" \| "order.fulfilled" \| "order.cancelled" \| "order.refunded" \| "product.updated"[], status?: "active" \| "paused" }` |
| `delete_webhook` | `settings:write` | Delete a webhook subscription by the id list_webhooks and upsert_webhook return: { id }. | `{ id: string }` |

## Custom domains

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_domains` | `domains:read` | The store's addresses: { domains: [{ id, hostname, kind: platform \| custom, status: pending \| active \| failed, isPrimary, sslStatus, errors, lastCheckedAt }], cnameTarget, maxCustomDomains, configured }. | none |
| `describe_domain_setup` | `domains:read` | Everything needed to connect a hostname, before or after add_domain: { hostname, zone, apex, records: [{ type: CNAME, name, label, target, proxied: false, ttl }], provider: { id, name, detectedBy: nameservers \| domain-connect \| unknown,… | `{ hostname: string, www?: boolean, includeEmail?: boolean }` |
| `add_domain` | `domains:write` | Register a custom hostname for the store and get its DNS records: { hostname, www? } → { domain, www, wwwError?, setup, cnameTarget, sync }. | `{ hostname: string, www?: boolean }` |
| `check_domain` | `domains:write` | Re-check one custom domain now (ignores the 30 s throttle) → { domain, sync }. | `{ id: string }` |
| `make_primary_domain` | `domains:write` | Make an active domain the store's canonical address (links, sitemap, checkout return URLs; the other hostnames redirect to it) → { domains, sync }. | `{ id: string }` |
| `remove_domain` | `domains:write` | Remove a custom domain from the store and stop serving it (a bare domain takes its www along; the free store-on-formahand.app address cannot be removed; a removed primary hands primary back to the free address) → { ok, sync }. | `{ id: string }` |

## Flows

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_flows` | `settings:read` | How to automate this store without an external service. | `{ sections?: "templates" \| "recipes" \| "actions" \| "filters" \| "hooks" \| "all"[] }` |
| `list_flows` | `settings:read` | The store's flows: { flows: [{ id, name, enabled, topics, filter, actions, schedule, runCount, runs7d, lastRunAt, lastStatus, failureCount, disabledReason, readiness }], flowsEnabled, senderFallbackAck, max, limits }. | none |
| `get_flow` | `settings:read` | One flow by id, with its readiness ({ ready, requirements }) and its runs in the last seven days. | `{ id: string }` |
| `create_flow` | `settings:write` | Create a flow: { name, topics: ["order.delivered"], filter?: <predicate>, actions: [{ type: "send_email", template: "order-delivered", to: "buyer" }, …], enabled? }. | `{ name: string, enabled?: boolean, topics: string[], filter?: string, actions: object[], schedule?: object }` |
| `update_flow` | `settings:write` | Change a flow: { id, name?, enabled?, topics?, filter?, actions?, schedule? }. | `{ name?: string, enabled?: boolean, topics?: string[], filter?: string, actions?: object[], schedule?: object, id: string }` |
| `delete_flow` | `settings:write` | Delete a flow and its run history. | `{ id: string }` |
| `test_flow` | `settings:write` | Dry run, no side effects: renders every action of the flow against { eventId } (else the newest event matching the flow, else a sample) → { event, matched, filterMatched, actions: [{ index, type, when, status: would_run \| skipped \| error, rendered:… | `{ id?: string, flowId?: string, eventId?: string }` |
| `set_flows_enabled` | `settings:write` | The store's kill switch: { flowsEnabled: false } pauses every flow (pending runs wait), true resumes. | `{ flowsEnabled: boolean }` |
| `list_flow_runs` | `events:read` | A flow's runs, newest first: { runs: [{ id, eventId, eventType, attempt, status: pending \| delivered \| failed, error, result: { actions: [{ index, type, status: done \| failed \| skipped \| queued, detail, error, to, messageIds }] } }] }. | `{ id?: string, flowId?: string, limit?: integer }` |
| `get_flow_readiness` | `settings:read` | What a flow still needs before it can run: { readiness: { ready, requirements: [{ id, label, done, detail, href, tool, blocking }] }, flow }. | `{ flowId: string }` |
| `enable_flow` | `settings:write` | Switch a flow on once everything it needs is in place → { flow }. | `{ flowId: string, enabled?: boolean }` |
| `check_flow_url` | `settings:write` | Send one clearly marked test request to a flow's webhook or agent steps and record what answered → { checks: [{ actionIndex, url, ok, status, error, checkedAt }], readiness, flow }. | `{ flowId: string, actionIndex?: integer }` |
| `retry_flow_run` | `settings:write` | Run a flow run again → { delivery }. | `{ runId: string }` |
| `acknowledge_sender_fallback` | `settings:write` |  | `{ acknowledged?: boolean }` |

## Email templates

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_email_templates` | `settings:read` | Every email template this store has, as a list you can actually read: { templates: [{ id, key, name, description, subject, isSystem, isModified, updatedAt, bodyHtmlBytes, bodyTextBytes, variables }], count }. | `{ fields?: "bodyHtml" \| "bodyText"[] }` |
| `get_email_template` | `settings:read` | One email template, whole, by id or by key: { template: { id, key, name, description, subject, bodyHtml, bodyText, fromName, replyTo, variables, isSystem, isModified, createdAt, updatedAt } }. | `{ id: string }` |
| `upsert_email_template` | `settings:write` | Create or replace an email template by key: { key: "welcome", name, subject, bodyHtml, bodyText?, fromName?, replyTo?, variables? }. | `{ key: string, name: string, subject: string, bodyHtml: string, bodyText?: string, fromName?: string, replyTo?: "" \| string, variables?: string[] }` |
| `preview_email_template` | `settings:write` | Render a template against { eventId } or its built-in sample → { subject, bodyHtml, bodyText, errors: [{ field, error, position }], warnings, paths, event, sample, template }. | `{ id: string, eventId?: string }` |
| `reset_email_template` | `settings:write` | Restore a system template to its default copy. | `{ id: string }` |

## Plan, billing and walls

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_plan` | `settings:read` | The store's plan, effective pricing and billing: { plan: { plan: free \| growth \| pro, subscribedPlan, status, entitlements: { stores, customDomains, marketing, flows, apiTokens, teamSeats, transactionFeeBps, shippingMarkupBps,… | none |
| `get_statement` | `settings:read` | A monthly statement (/docs/billing §5): { period? } (YYYY-MM, default this month) → { statement: { id, period, currency, lines: [{ kind: plan \| email_overage \| extra_domains \| media_overage \| label, description, quantity, unitMinor, amountMinor,… | `{ period?: string }` |
| `apply_promo_code` | `settings:write` | Apply a promotion code to this store: { code } → { promotion: { code, kind: fee_bps \| markup_bps \| plan_discount_pct \| email_overage \| domain_price \| fee_holiday, value, expiresAt }, pricing } (the new effective pricing, as get_plan.pricing). | `{ code: string }` |
| `describe_walls` | `settings:read` | Every walled feature with its current state for this store: { plan, fees, pricing, usage, activity, features: { marketing: { allowed, wall, steps: [{ id, label, done, href?, tool?, detail? }], warnings, caps: { monthlyCap, monthSent, monthRemaining,… | none |
| `describe_account` | `settings:read` | The whole account in one answer, for deciding what to ask the merchant for: { plan: { id, name, monthlyUsd, status }, card: { onFile, brand?, last4? }, paymentProfile: { state: none \| ready \| expired, onFile, brand?, last4?, expires?, href,… | none |
| `start_upgrade` | `settings:write` | Start a plan upgrade: { plan: growth \| pro, interval?: monthly \| annual } → { url, sessionId, plan, interval }. | `{ plan: string, interval?: "monthly" \| "annual" }` |

## Environments

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_environment` | `settings:read` | Both environments of the store: { live: { hostname, url, runtimeStatus, runtimeVersion }, sandbox: null \| { … }, sandboxEnabledAt, selected } where selected is this token's mode. | none |
| `enable_sandbox` | `settings:write` | Create the store's sandbox (a second, complete store at <slug>-test.<domain> with payments in test mode, free test labels and owner-only email) if it does not exist. | none |
| `promote_to_live` | `settings:write` | Copy the sandbox's content to the live store: products with variants and images, collections, pages, navigation, SEO, shipping settings and rates, discounts (recreated on the live payment account), email templates, flows (disabled on arrival,… | `{ dryRun?: boolean }` |
| `refresh_sandbox` | `settings:write` | Overwrite the sandbox's content with the live store's (same resources as promote_to_live, in the other direction) and publish it in the sandbox so it mirrors what shoppers see. | `{ dryRun?: boolean }` |

## Custom fields

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_custom_fields` | `products:read` | Every custom field on one object: { fields: [{ ownerType, ownerId, namespace, key, type, value }] }, ordered namespace then key. | `{ ownerType: "product" \| "variant" \| "collection" \| "order" \| "order_line" \| "customer", ownerId: string }` |
| `set_custom_fields` | `products:write` | Write custom fields on one object. | `{ ownerType: "product" \| "variant" \| "collection" \| "order" \| "order_line" \| "customer", ownerId: string, fields: object[] }` |
| `delete_custom_field` | `products:write` | Remove one custom field by namespace and key. | `{ ownerType: "product" \| "variant" \| "collection" \| "order" \| "order_line" \| "customer", ownerId: string, namespace: string, key: string }` |
| `describe_custom_fields` | `products:read` | What custom fields are and how to use them: the six owner types, the seven value types (text, number, money, date, boolean, json, file) with their shapes and limits, the namespace and key rules, how merging works, which scopes each owner type needs,… | none |

## Customers and marketing

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_customers` | `customers:read` | The store's CRM contacts, newest first: { customers: [{ id, email, name, phone, marketingStatus, tags, totalOrders, totalSpentMinor, currency, createdAt, updatedAt, accountStatus, lastSeenAt }], nextCursor, total }. | `{ q?: string, tag?: string, minOrders?: integer, minSpentMinor?: integer, marketingStatus?: "subscribed" \| "unsubscribed" \| "pending", maxOrders?: integer, lastOrderWithinDays?: integer, lastOrderOlderThanDays?: integer, firstOrderWithinDays?: integer, customerGroup?: string, country?: string, acceptsMarketing?: boolean, limit?: integer, cursor?: string }` |
| `get_customer` | `customers:read` | One customer's whole page by id or email: { customer (tags, marketing status, order count, lifetime spend, group, country, accountStatus / lastSeenAt), orders (the last 20: number, date, totalMinor, currency, status), timeline (created, orders,… | `{ id?: string, email?: string }` |
| `upsert_customer` | `customers:write` | Create or update one CRM contact by email, the same write POST /api/crm makes: { email, name?, phone?, tags?, notes?, marketingStatus? } → { customer }. | `{ email: string, name?: string, phone?: string, tags?: string[], notes?: string, marketingStatus?: "subscribed" \| "unsubscribed" \| "pending" }` |
| `add_customer_note` | `customers:write` | Add a note to one customer's timeline: { customerId, note }. | `{ customerId: string, note: string }` |
| `tag_customer` | `customers:write` | Add or remove tags on one contact (by id or email): { id \| email, add?: string[], remove?: string[] }. | `{ id?: string, email?: string, add?: string[], remove?: string[] }` |
| `export_customers` | `customers:export` | The customer list as CSV text (the same columns the merchant downloads: email, name, phone, marketing_status, tags, total_orders, total_spent, currency, created_at, updated_at), returned as { format, filename, csv, rows, truncated, note }. | `{ format?: "csv", tag?: string, marketingStatus?: "subscribed" \| "unsubscribed" \| "pending" }` |
| `list_campaigns` | `customers:read` | Email campaigns of this environment, newest first: { campaigns: [{ id, subject, previewText, audience, status (draft \| sent \| partial \| failed), createdAt, sentAt, recipientCount, deliveredCount, suppressedCount, failedCount }] }. | `{ limit?: integer }` |
| `create_campaign` | `marketing:send` | Create a campaign DRAFT: { subject, previewText?, bodyHtml \| bodyMarkdown, audience: { tag?, segmentId?, marketingStatus: 'subscribed' } }. | `{ subject: string, previewText?: string, bodyHtml?: string, bodyMarkdown?: string, audience?: object }` |
| `preview_campaign` | `customers:read` | One campaign draft with the audience it would reach now, the recipients of the next send (at most 50), and the marketing wall as it stands: { campaign, audience, recipients, allowed, wall, steps, warnings, caps }. | `{ id: string }` |
| `send_campaign` | `marketing:send` | Send a draft to its audience (at most 50 subscribed contacts per call). | `{ id: string, scheduleAt?: string }` |
| `describe_marketing` | any | Read-only. | none |
| `list_segments` | `customers:read` | The store's customer segments: { segments: [{ id, name, filter, builtIn, createdAt, updatedAt }], counts }. | none |
| `upsert_segment` | `customers:write` | Save a segment: { id?, name, filter }. | `{ id?: string, name: string, filter: object }` |
| `delete_segment` | `customers:write` | Delete a saved segment: { id }. | `{ id: string }` |
| `preview_segment` | `customers:read` | Who a filter would reach, before saving it: { filter } or { id } for a saved or built-in segment → { count, customers } with the first 10 contacts. | `{ id?: string, filter?: object }` |

## Shopper accounts

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_shopper_accounts` | `customers:read` | How shoppers sign in to this store and what is set up: { signInEnabled, accounts, signInsThisWeek, routes, howItWorks }. | none |
| `set_shopper_accounts` | `customers:write` | Turn the store's shopper sign-in on or off: { enabled } → { signInEnabled, accounts, signInsThisWeek }. | `{ enabled: boolean }` |
| `revoke_shopper_sessions` | `customers:write` | Signs one shopper out everywhere: deletes every sign-in session of that customer, so their next visit to /account asks for a new code. | `{ customerId: string }` |

## Order editing

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `begin_order_edit` | `orders:write` | Start editing a paid order that has not shipped: { orderId } → { edit }. | `{ orderId: string }` |
| `edit_order_draft` | `orders:write` | Set what an open draft changes: { editId, lines?, shippingAddress?, shippingName?, shippingMethod?, discountCode?, manualDiscount?, customerEmail?, customerPhone?, reason? } → { edit }. | `{ editId: string, lines?: object[], shippingAddress?: object, shippingName?: string, shippingMethod?: string, discountCode?: string, manualDiscount?: object, customerEmail?: string \| "", customerPhone?: string, reason?: string }` |
| `preview_order_edit` | `orders:write` | Changes nothing: what applying this draft *would* do, { editId } → { before, after, diff, deltaMinor, consequence, priced, problems, tax }. | `{ editId: string }` |
| `apply_order_edit` | `orders:write` | Apply a previewed draft to the order: { editId, reason?, notifyCustomer? } → { edit, order, deltaMinor, consequence, paymentUrl?, refundedMinor? }. | `{ editId: string, reason?: string, notifyCustomer?: boolean }` |
| `cancel_order_edit` | `orders:write` | Throw away an open draft: { editId } → { edit }. | `{ editId: string }` |
| `waive_order_edit_balance` | `orders:write` | Settle the balance an edit asked the buyer for without collecting it here: { editId, reason? } → { edit }. | `{ editId: string, reason?: string }` |
| `list_order_edits` | `orders:read` | The store's order edits, newest first: { edits, limit }. | `{ orderId?: string, status?: "all" \| "open" \| "draft" \| "applied" \| "pending_payment" \| "cancelled" }` |

## Returns

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `create_return` | `orders:write` | Open a return on a buyer's behalf (the phone call, the email, the person at the counter): { orderId, reason, note?, lines: [{ orderLineId, quantity }] } → { return }. | `{ orderId: string, reason: enum, note?: string, lines: object[] }` |
| `list_returns` | `orders:read` | The store's returns, newest first: { returns, limit, nextCursor }. | `{ status?: enum, orderId?: string, limit?: integer, cursor?: string }` |
| `get_return` | `orders:read` | One return with its order, lines, reason, notes, refund so far and the return label or instructions the buyer was given: { id } → { return }. | `{ id: string }` |
| `approve_return` | `orders:write` | Approve a requested return and tell the buyer how to send the goods back: { id, instructions?, labelUrl?, carrier?, trackingNumber?, merchantNote?, notifyCustomer? } → { return, emailSent }. | `{ id: string, instructions?: string, labelUrl?: string, carrier?: string, trackingNumber?: string, merchantNote?: string, notifyCustomer?: boolean }` |
| `decline_return` | `orders:write` | Decline a requested return: { id, reason, notifyCustomer? } → { return, emailSent }. | `{ id: string, reason: string, notifyCustomer?: boolean }` |
| `mark_return_received` | `orders:write` | Record what actually came back, line by line: { id, lines: [{ lineId, quantity, condition, restock }], merchantNote? } → { return }. | `{ id: string, lines: object[], merchantNote?: string }` |
| `refund_return` | `orders:refund` | Refund a return through the processor that took the order's payment: { id, amountMinor?, merchantNote?, notifyCustomer? } → { order, refund, return, emailSent }. | `{ id: string, amountMinor?: integer, merchantNote?: string, notifyCustomer?: boolean }` |
| `restock_return` | `orders:write` | Put the received goods back on the shelf: { id } → { return, restocked }. | `{ id: string }` |
| `close_return` | `orders:write` | Close a return that needs nothing more: { id, merchantNote? } → { return }. | `{ id: string, merchantNote?: string }` |
| `set_return_policy` | `settings:write` | The store's return rules: { windowDays, policy?, instructions? } → { returns }. | `{ windowDays: integer, policy?: string, instructions?: string }` |

## Gift cards

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_gift_cards` | `orders:read` | The store's gift cards: what each is worth now, what it was issued for, who issued it, whether it is still good, and what is outstanding in total. | `{ status?: "active" \| "void" \| "all", limit?: integer }` |
| `get_gift_card` | `orders:read` | One gift card with its whole ledger: issued, held at a checkout, spent, released when a checkout was abandoned, refunded back onto it, adjusted by hand. | `{ id: string }` |
| `issue_gift_card` | `gift_cards:issue` | Puts new credit in the store's till and answers the code ONCE, nothing can read it back afterwards, so hand it to whoever it is for in the same breath. | `{ amountMinor: integer, currency: string, expiresAt?: string, note?: string, orderId?: string, orderLineId?: string, deliver?: object }` |
| `void_gift_card` | `gift_cards:issue` | Stops a gift card. Its balance stays on the record so the ledger still adds up, but it can no longer pay for anything. | `{ id: string, note?: string }` |
| `adjust_gift_card` | `gift_cards:issue` | Moves a card's balance by hand: a goodwill top-up, or taking credit back off one issued in error. | `{ id: string, deltaMinor: integer, currency?: string, note?: string }` |

## Orders you create yourself

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `create_order` | `orders:write` | Create an order the merchant sells themselves, by phone, at a counter, on an invoice: { customer: { email, name?, phone? }, lines: [{ productSlug, variantId?, quantity }], shipping?: { amountMinor, label }, discountMinor?, note?, shippingAddress?,… | `{ customer: object, lines: object[], shipping?: object, discountMinor?: integer, note?: string, shippingAddress?: object, sendConfirmation?: boolean }` |
| `send_payment_link` | `orders:write` | Email the customer a card checkout for an order the merchant created: { id, notifyCustomer? } → { order, paymentUrl, expiresAt, emailSent, emailError }. | `{ id: string, notifyCustomer?: boolean }` |

## Schedules, inbound hooks and the vault

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `create_flow_hook` | `settings:write` | Give a flow an inbound URL another system can POST to: { flowId, mode: "bearer" \| "hmac" }. | `{ flowId: string, mode?: "bearer" \| "hmac", enabled?: boolean }` |
| `list_flow_hooks` | `settings:read` | The store's inbound URLs: { hooks: [{ id, flowId, flowName, url, mode, enabled, secretPrefix, lastReceivedAt, receivedCount }] }. | none |
| `rotate_flow_hook` | `settings:write` | Issue a new secret for an inbound URL: { id }. | `{ id: string }` |
| `set_flow_hook_enabled` | `settings:write` | Switch an inbound URL on or off: { id, enabled }. | `{ id: string, enabled: boolean }` |
| `delete_flow_hook` | `settings:write` | Remove a flow's inbound URL: { id }. | `{ id: string }` |
| `set_secret` | `settings:write` | Store a credential the store may use when it calls another system: { name, value }. | `{ name: string, value: string }` |
| `list_secrets` | `settings:read` | The names of the store's secrets with their timestamps: { secrets: [{ name, createdAt, updatedAt, lastUsedAt }], max }. | none |
| `delete_secret` | `settings:write` | Remove a stored secret: { name }. | `{ name: string }` |
| `describe_secrets` | `settings:read` | How the store's vault works: naming rules, size and count limits, exactly where {{ secrets.name }} may be used, what run logs keep, and what promote_to_live does with secrets. | none |
| `list_flow_schedule_runs` | `events:read` | Every tick of a flow's schedule, newest first: { runs: [{ id, flowId, ranAt, status: queued \| ok \| failed \| skipped, error }] }. | `{ id: string, limit?: integer }` |

## Tracked links

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_tracked_links` | `storefront:read` | The store's tracked links, newest first: { links: [{ id, code, name, targetPath, source, medium, campaign, discountCode, active, createdByKind, createdBy, createdAt, updatedAt, clicks, clicksLast30Days, orders, revenueMinor, url }], total, offset,… | `{ id?: string, code?: string, limit?: integer, offset?: integer }` |
| `upsert_tracked_link` | `storefront:write` | Make or change one tracked link: { id?, code?, name, targetPath, channel?, source?, medium?, campaign?, discountCode?, active? } → { link }. | `{ id?: string, code?: string, name?: string, targetPath?: string, channel?: enum, source?: string, medium?: string, campaign?: string, discountCode?: string, active?: boolean }` |
| `delete_tracked_link` | `storefront:write` | Remove one tracked link: { id } or { code } → { deleted, id, code }. | `{ id?: string, code?: string }` |

## Analytics

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_analytics` | `orders:read` | Everything the store knows about a period: { from?, to?, compare? } → { period, currency, revenueMinor, taxCollectedMinor, orders, paidOrders, averageOrderMinor, refundedMinor, abandonedCheckouts, abandonedRate, newCustomers, repeatRate,… | `{ from?: string, to?: string, compare?: boolean }` |
| `get_top_products` | `orders:read` | What sold most: { period: "7d" \| "30d" \| "90d" \| "365d" } → { period, currency, bestSellers: [{ productId, name, quantity, revenueMinor }], paidOrders, revenueMinor }. | `{ period?: "7d" \| "30d" \| "90d" \| "365d" }` |
| `export_analytics` | `orders:read` | A period's numbers as CSV text: { period: "7d" \| "30d" \| "90d" \| "365d" } → { format, filename, csv, rows, period, currency }. | `{ period?: "7d" \| "30d" \| "90d" \| "365d", format?: "csv" }` |
| `import_search_performance` | `settings:write` | Upload search performance rows the owner exported themselves: { rows: [{ day, query, page, clicks, impressions, position, engine?, country?, device? }], source? } → { imported: { rows, days, from, to, source } }. | `{ rows: object[], source?: string }` |
| `get_search_performance` | `orders:read` | What search performance says for a period: { from, to, groupBy?: "query" \| "page" \| "engine" \| "country" \| "device" \| "day", engine?: "google" \| "bing" \| "other", country?, device?: "desktop" \| "mobile" \| "tablet", limit? } → { period, groupBy,… | `{ from: string, to: string, groupBy?: "query" \| "page" \| "engine" \| "country" \| "device" \| "day", engine?: "google" \| "bing" \| "other", country?: string, device?: "desktop" \| "mobile" \| "tablet", limit?: integer }` |
| `get_order_sources` | `orders:read` | Where sales came from: channel, source, campaign and tracked link, with orders and sales for a period. | `{ from?: string, to?: string }` |
| `describe_search_engines` | `settings:read` | Which search engines this store is connected to, read-only: {} → { mode, primaryHostname, engines: [{ engine, available, state, way, hostname, lastReadAt, lastError, readAccess, identities? }], requests: [{ id, engine, hostname, requestedBy,… | none |
| `connect_search_engine` | `settings:write` | Asks the owner to connect a search engine: { engine: "google" \| "bing" \| "indexnow" } → { request: { id, engine, hostname, state: "waiting_for_owner" }, approveAt, next }. | `{ engine: "google" \| "bing" \| "indexnow" }` |

## Governance

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_proposals` | any | The writes this token proposed instead of running, newest first, with their status (pending, approved, rejected, expired, failed). | `{ status?: "pending" \| "approved" \| "rejected" \| "expired" \| "failed" \| "all", limit?: integer }` |
| `get_proposal` | any | One proposal this token made, with the exact arguments it stored and, once the owner has approved it, what running it returned. | `{ id: string }` |
| `withdraw_proposal` | any | Take back a proposal this token made while it is still pending, for instance when you worked out a better change. | `{ id: string }` |
| `describe_governance` | any | The rules this token works under: act or propose mode, when it expires, the money limits, and what pauses it. | none |

## Product kinds and downloads

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_product_files` | `products:read` | The files a digital product delivers: { files: [{ id, name, bytes, contentType }] }, newest last. | `{ productId: string }` |
| `upload_product_file` | `products:write` | Attach a file to a digital product (its kind must be digital) and store it in the store's own media library: { productId, url \| dataBase64 + contentType, name? }. | `{ productId: string, url?: string, dataBase64?: string, contentType?: string, name?: string }` |
| `delete_product_file` | `products:write` | Remove one file from a digital product and delete the object from the store's bucket: { productId, fileId }. | `{ productId: string, fileId: string }` |
| `regenerate_download_links` | `orders:write` | Re-issue the download links of a paid order: { orderId } → { downloads: [{ name, url, expiresAt }] }, each valid 72 hours from now. | `{ orderId: string }` |

## Connector recipes

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_recipes` | `settings:read` | The connector recipe library: { recipes: [{ id, name, category, description, needs: { scopes, secrets, customFields, options }, flows, subscriptions, customFields }] }. | none |
| `get_recipe` | `settings:read` | One recipe in full: { recipe: { id, name, category, description, needs, flows, customFieldDefinitions, subscriptions, notes } }. | `{ id: string }` |
| `install_recipe` | `settings:write` | Install a recipe into this store: { id, secrets?: { name: value }, options?: { name: value } }. | `{ id: string, secrets?: object, options?: object }` |
| `uninstall_recipe` | `settings:write` | Remove what a recipe created: { id }. | `{ id: string }` |

## Checkout rules, pricing rules and groups

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_pricing_rules` | `settings:read` | The store's automatic discounts (live data): { rules: [{ id, name, kind, value, conditions, applies, priority, stackable, startsAt, endsAt, enabled, usageCount }] }, in the order they run. | none |
| `upsert_pricing_rule` | `settings:write` | Creates or replaces one automatic discount on the live store. | `{ id?: string, name: string, kind: "percent" \| "amount" \| "bxgy" \| "free_shipping", value?: integer, conditions?: object, applies?: object, priority?: integer, stackable?: boolean, startsAt?: string, endsAt?: string, enabled?: boolean, showOnStorefront?: boolean }` |
| `delete_pricing_rule` | `settings:write` | Removes one automatic discount from the live store. | `{ id: string }` |
| `list_checkout_rules` | `settings:read` | The store's shipping and payment rules (live data): { rules: [{ id, name, kind, conditions, target, enabled }] }. | none |
| `upsert_checkout_rule` | `settings:write` | Creates or replaces one checkout rule on the live store: hide_rate, show_rate_only, hide_payment or pickup. | `{ id?: string, name?: string, kind: "hide_rate" \| "show_rate_only" \| "hide_payment" \| "pickup" \| "local_delivery", conditions?: object, target?: object, enabled?: boolean }` |
| `delete_checkout_rule` | `settings:write` | Removes one shipping or payment rule from the live store; carts go back to seeing every rate. | `{ id: string }` |
| `list_customer_groups` | `customers:read` | The store's customer groups (live data): { groups: [{ id, name, handle, priceList, minOrderMinor, taxExempt, terms, members }] }. | none |
| `upsert_customer_group` | `settings:write` | Creates or replaces one customer group on the live store: its handle, its price list ({ "<productId or variantId>": priceMinor } or { "%": -20 }), its minimum order, tax exemption and payment terms. | `{ id?: string, name: string, handle: string, priceList?: object, minOrderMinor?: integer, taxExempt?: boolean, terms?: "card" \| "invoice" }` |
| `set_customer_group` | `customers:write` | Puts one customer in a group, or takes them out with groupHandle null. | `{ customerId: string, groupHandle: string }` |
| `simulate_checkout` | `settings:read` | Prices a hypothetical cart the way checkout would and reports what the rules do: { currency, lines, subtotalMinor, discountMinor, totalMinor, freeShipping, applied: [{ ruleId, name, amountMinor }], customer, shippingOptions, paymentProvider,… | `{ items: object[], customerEmail?: string, customerId?: string, discountCode?: string, destinationCountry?: string }` |

## Checkout settings, tax, staff notifications and the weekly summary

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_checkout_settings` | `settings:read` | What the store's payment page asks a shopper for, and where it lives (live data): { checkout: { phone, company, addressLine2, orderNote, marketingOptIn, termsLink, embedded, tipping } }. | none |
| `set_checkout_settings` | `settings:write` | Replaces what the live store's payment page asks for and where it lives: the phone, company and second-address-line fields, the order note, the marketing consent checkbox, the terms link and embedded (the payment form on the store's own page, or the… | `{ phone?: "hidden" \| "optional" \| "required", company?: "hidden" \| "optional" \| "required", addressLine2?: "optional" \| "hidden", orderNote?: boolean, marketingOptIn?: "off" \| "unchecked" \| "checked", termsLink?: boolean, embedded?: boolean, tipping?: false }` |
| `get_notification_settings` | `settings:read` | Who on the staff hears about a new order or a low stock level (live data): { notifications: { newOrderEmails, lowStockEmails } }. | none |
| `set_notification_settings` | `settings:write` | Replaces the live store's staff notification addresses: who also receives the new-order email and the low-stock email, and whether a morning summary is wanted. | `{ newOrderEmails?: string[], lowStockEmails?: string[] }` |
| `get_tax_settings` | `settings:read` | Whether the live store charges tax, and how (live data): { tax: { enabled, pricesIncludeTax, defaultTaxCode, shippingTaxCode } }. | none |
| `set_tax_settings` | `settings:write` | Replaces the live store's tax settings: whether tax is charged, whether prices already include it, and the default product and shipping tax categories. | `{ enabled?: boolean, pricesIncludeTax?: boolean, defaultTaxCode?: string, shippingTaxCode?: string }` |
| `get_weekly_summary` | `settings:read` | The weekly summary for one week, as the numbers the Monday mail is built from, plus the switch of the person this token acts for: { store: { id, name, timezone, currency, open }, week: { week, from, to, timezone }, previousWeek, currency, current: {… | `{ week?: string }` |
| `set_weekly_summary` | `settings:write` | Switches the weekly summary mail on or off for the person this token acts for, on this store: { enabled } → { enabled, default, role, canReceive, schedule, timezone, nextSend, lastSend, sandbox }. | `{ enabled: boolean }` |

## Payments account

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_payment_features` | `settings:read` | Every switch on the store's card-payments account, read live from the account itself in the token's own environment (fh_live_ the live account, fh_test_ the sandbox's): { ready, mode, features: [{ id, title, summary, cost, control, state, value,… | none |
| `set_payment_feature` | `payments:write` | Changes one switch on the store's card-payments account. | `{ feature: enum, enabled: boolean, confirm?: boolean, authentication?: "provider" \| "always" \| "over", authenticationOverMinor?: integer, method?: string, payoutInterval?: "daily" \| "weekly" \| "monthly" \| "manual" }` |
| `request_instant_payout` | `payments:write` | Sends the store's balance to its debit card now instead of waiting for the normal payout. | `{ amountMinor?: integer, currency?: string }` |
| `list_tax_registrations` | `settings:read` | Where the store is registered to collect tax, read live from its payments account, plus who told Formahand about each one. | none |
| `add_tax_registration` | `payments:write` | Tells the store's payments account that the business is registered to collect tax in one place, so orders going there are taxed from a date. | `{ country: string, state?: string, type: enum, activeFrom?: string, confirm?: boolean, attested?: boolean }` |
| `expire_tax_registration` | `payments:write` | Ends one of the store's tax registrations from a date. | `{ registrationId: string, expiresAt?: string, confirm?: boolean }` |
| `list_payment_notices` | `settings:read` | Everything the payment provider has said about the store's account rather than about one order: disputes, fraud warnings, payouts that bounced, details the account still owes, changes to the tax setup. | `{ limit?: integer }` |
| `get_payout_profile` | `settings:read` | The business details that prepare the store's payout form, live data in both modes (fh_live_ reads the live account, fh_test_ the sandbox's): { profile, defaults, warnings: [{ id, message }], applied, leftover }. | none |
| `set_payout_profile` | `settings:write` | Replaces the live store's payout profile: business type, what it sells, the website it is reviewed against, the product description, when the customer's card is charged, and the support contact. | `{ profile: object }` |

## Launch mode and preview

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_preview_link` | `storefront:read` | Two signed links that open the storefront whatever its launch mode, so you can look at a "Coming soon" or password-protected store without opening it or changing anything: { published: { url, expiresAt }, draft: { url, expiresAt }, launchMode }. | none |
| `get_launch_mode` | `settings:read` | Whether the live storefront is open or still private (live data): { access: { access: "open" \| "password" \| "coming_soon", message, hasPassword, updatedAt } }. | none |
| `set_launch_mode` | `settings:write` | Opens the live store to shoppers, or keeps it private. | `{ access: "open" \| "password" \| "coming_soon", password?: string, message?: string }` |

## Pixels and tracking choice

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_pixels` | `settings:read` | The store's advertising and analytics pixels and how shoppers are asked about tracking: {} → { pixels: [{ channel, id, label?, enabled, updatedAt }], consentMode, channels: [{ id, name, idLabel, idExample, idPattern, idHelp, labelLabel?,… | none |
| `set_pixel` | `settings:write` | Switches on a channel's pixel, or replaces the one it has: { channel, id, label?, enabled? } → the same answer as get_pixels. | `{ channel: enum, id: string, label?: string, enabled?: boolean }` |
| `remove_pixel` | `settings:write` | Removes a channel's pixel: { channel } → the same answer as get_pixels. | `{ channel: enum }` |
| `set_consent_mode` | `settings:write` | Chooses which shoppers are asked before pixels load: { consentMode } → the same answer as get_pixels. | `{ consentMode: "off" \| "ask-everywhere" \| "ask-where-required" }` |

## Sections

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_sections` | `storefront:read` | Everything needed to write a section, as data. | `{ parts?: enum[] }` |
| `list_sections` | `storefront:read` | The sections this store can place: the built-in ones every store has (the theme's hero, collection, more-products and styles, each page's own body, and custom HTML), and the store's own, add-on and catalogue sections with their version, whether you… | `{ origin?: "store" \| "catalogue" \| "app" \| "theme", cursor?: string, limit?: integer }` |
| `get_section` | `storefront:read` | One section's whole package (settings schema, template, styles, presets), whether you may change it, its kept versions and where it is placed. | `{ id: string, version?: string, part?: "template" \| "styles", offset?: integer }` |
| `save_section` | `storefront:write` | Creates one of the store's own sections, or saves a new version of it: { package }. | `{ package: object }` |
| `fork_section` | `storefront:write` | Makes the store's own copy of an add-on, catalogue or store section under a new id: { id, newId, repoint? }. | `{ id: string, newId: string, repoint?: boolean }` |
| `list_section_versions` | `storefront:read` | The kept versions of one of the store's sections, newest first, with when each was saved, by whom, and which one the draft and the live storefront render. | `{ id: string }` |
| `rollback_section` | `storefront:write` | Brings back an earlier version of one of the store's sections: { id, version }. | `{ id: string, version: string }` |
| `delete_section` | `storefront:write` | Deletes one of the store's own (or catalogue) sections with all its versions: { id, force? }. | `{ id: string, force?: boolean }` |
| `preview_section` | `storefront:read` | Renders one instance of a section with the settings you give, exactly as shoppers would see it, without placing it anywhere: { id, version?, settings?, page? } for a saved or built-in section, or { package, settings?, page? } for one you have not saved yet. | `{ id?: string, version?: string, package?: object, settings?: object, page?: string, sample?: object }` |

## Themes

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `list_themes` | `storefront:read` | Every storefront look a store can wear, one list: { themes: [{ id, name, category, line, pages, preview, phone, licence, current?, demoUrl }], current }. | none |
| `get_theme` | `storefront:read` | One theme whole: { id } → { theme, preset }: its line, pages, pictures and what it needs, then its colours, typefaces, the sections it brings (summarised), every page's instances with their settings, the placeholder images, menus, starter pages and… | `{ id: string }` |
| `list_presets` | `storefront:read` | The older name for list_themes, kept working: the same looks by their catalogue ids: { presets: [{ id, name, description, category, version, theme, ofTheme?, typefaces, colours, sections, pages, starterPages, licence }] }. | none |
| `get_preset` | `storefront:read` | The older name for get_theme, kept working. | `{ id: string }` |
| `apply_preset` | `storefront:write` | The older way to pick a theme, kept working (set-theme is the one way now; apply_preset adds dryRun and reset). | `{ id: string, dryRun?: boolean, revision?: integer, reset?: boolean }` |
| `claim_showcase_look` | `storefront:write` | Puts the look of a showcase store from the library (list_themes `library`) on this store: { code } → { claimId, applied }. | `{ code: string }` |

## Readiness

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_readiness` | `settings:read` | What each feature of this store still needs before it works, live data: { features: [{ feature, label, ready, requirements: [{ id, label, done, detail, href, tool?, severity? }], countryNote? }] }. | `{ feature?: enum }` |

## Store email sending

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_sender` | `settings:read` | The address this store sends its own mail from, live data: { sender: { fromName, email, domain, status: draft \| pending_dns \| verified \| disabled, dnsRecords: [{ record, name, type, value, status, ttl?, priority? }] } \| null, verified,… | none |
| `set_sender` | `settings:write` | Saves the address this store sends its own mail from, then returns the DNS records to publish: { email, fromName } → { sender, verified, pendingRecords, sending }. | `{ email: string, fromName: string }` |
| `verify_sender` | `settings:write` | Asks Formahand to check this store's published DNS records again and returns where they stand: { sender, verified, pendingRecords, sending }. | none |
| `send_test_email` | `settings:write` | Sends ONE real message from this store's verified sender, through the same path an order confirmation takes, so the merchant can watch it arrive: { to? } → { to, from, subject, status, sent, remainingToday, note }. | `{ to?: string, templateId?: string, eventId?: string }` |

## Shopping feeds

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `get_feeds` | `settings:read` | Where this store publishes itself for shopping engines and AI agents: { } → { hostname, feeds: { google, csv, json }, channelFeeds: { meta, pinterest, tiktok }, llmsTxt, wellKnown, mcp, docs, summary: { items, inStock, outOfStock, backorder,… | none |
| `check_feed` | `settings:read` | Fetch this store's published product feed and report on it: { channel?: "google" \| "meta" \| "pinterest" \| "tiktok" } → { ok, hostname, feeds, items, inStock, outOfStock, backorder, problems: [{ id, title, field, message }], problemCount, checkedAt, error? }. | `{ channel?: "google" \| "meta" \| "pinterest" \| "tiktok" }` |

## Store functions

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_store_functions` | `storefront:write` | The hooks a store function can run at, what each one is handed (`sampleInput`), what it may answer (`sampleOutput`, a worked example that validates against the contract), what the store will refuse to accept back (`clamps`), and what it does when… | none |
| `list_store_functions` | `storefront:write` |  | none |
| `get_store_function` | `storefront:write` | One function in full: its source, its versions (newest first, with who uploaded each and when), its counters, its last failures, and, when it is in shadow mode, what it would have done on recent traffic. | `{ name: string, version?: integer }` |
| `create_store_function` | `storefront:write` | Uploads one ES module as a new version of a store function and leaves it switched off. | `{ name: string, hook: enum, source: string, position?: integer }` |
| `test_store_function` | `storefront:write` | Runs one function once against an input you supply, or a sample input for its hook if you supply none, and returns what it asked for, what the store would actually have applied, every clamp the store had to make, and how long it took. | `{ name: string, input?: object }` |
| `enable_store_function` | `storefront:write` | Switches one function on. | `{ name: string, mode?: "shadow" \| "on", position?: integer }` |
| `disable_store_function` | `storefront:write` | Switches one function off. | `{ name: string }` |
| `rollback_store_function` | `storefront:write` | Points a function back at one of its earlier versions. | `{ name: string, version: integer }` |
| `delete_store_function` | `storefront:write` | Removes one function from this environment. | `{ name: string }` |

## Apps

| Tool | Scope | What it does | Input |
| --- | --- | --- | --- |
| `describe_apps` | `storefront:write` | What an app is, every grant it can ask for in the words the merchant reads, which of them is personal data, what an app is handed at run time, what it can never do, the allowances per plan, and `publishing.canPublish`, whether Formahand could last… | none |
| `list_apps` | `storefront:write` | Every app this store has in this environment: its name, version, whether it is a draft, waiting for the merchant, installed or switched off, what it was granted, and what it has been doing, plus `pending`, the add-on installs a token has asked for… | none |
| `get_app` | `storefront:write` | One app in full: its manifest, its source, everything it asks for, everything it was granted, its settings, and a day-by-day rollup of what it touched. | `{ name: string }` |
| `create_app` | `storefront:write` | Stores one app, a manifest and a single JavaScript file, as a draft. | `{ manifest: object, source: string }` |
| `update_app` | `storefront:write` | Uploads a new version of an app. | `{ manifest: object, source: string }` |
| `list_addons` | `storefront:write` | The Add-ons catalogue: the apps Formahand ships as source, reviews, gift cards, a wishlist, back-in-stock, a contact form, questions and answers, a size guide, what each one does, what it asks for in the words the merchant reads, and what this… | none |
| `request_install` | `storefront:write` | Asks the merchant to install an app: one this store wrote (`name`), or one of Formahand's add-ons (`catalog`, from list_addons). | `{ name?: string, catalog?: string }` |
| `withdraw_install_request` | `storefront:write` | Takes back an install request: { requestId } from request_install or from list_apps `pending`. | `{ requestId: string }` |
| `uninstall_app` | `storefront:write` | Switches an app off now: it stops answering its addresses, its access to the store is revoked and every grant it had is withdrawn. | `{ name: string, removeData?: boolean }` |
| `app_settings` | `storefront:write` | Reads or changes what the merchant configured for an installed app, the fields its manifest declares. | `{ name: string, settings?: object }` |
| `list_app_records` | `storefront:write` | Reads rows from one of an installed app's own tables, the tables it declared in its manifest and nothing else. | `{ name: string, table: string, where?: object, limit?: integer }` |
| `set_app_record` | `storefront:write` | Writes or removes one row in an installed app's own table, by its id, giving a Questions and answers add-on its first entry, approving a review, hiding one, deleting a row an app collected. | `{ name: string, table: string, id: string, idColumn?: string, set?: object, remove?: boolean }` |
| `list_app_schedules` | `storefront:write` | Every timer this store's installed apps run on: what each is called, how often it runs, when it last ran and how that went, when it runs next, and whether it is switched on. | `{ name?: string }` |
| `run_app_schedule_now` | `storefront:write` | Runs one of an app's timers immediately, without waiting for its next slot, how you check that a fix worked. | `{ name: string, schedule: string }` |
| `disable_app_schedule` | `storefront:write` | Switches one of an app's timers off. | `{ name: string, schedule: string }` |
| `enable_app_schedule` | `storefront:write` | Switches one of an app's timers back on, from its next slot. | `{ name: string, schedule: string }` |

## Read more

- https://formahand.com/docs/agents
