# Storefront: draft, design and publish

How to change what shoppers see: the draft and its revision, the editor commands, the modules a section is built from, custom HTML and CSS, and publishing. Nothing here reaches shoppers until `publish_storefront`.

## The loop

Storefront appearance and copy (theme settings, sections, modules, hero image, navigation, pages) live in a revision-checked draft: `get_storefront_draft` → `apply_storefront_commands` / `set_hero_image` / `set_navigation` / `set_content_pages` with that `revision` (discover commands with `list_modules`) → `publish_storefront { revision }`. Nothing shoppers see changes until publish; a stale revision is a conflict, re-read and retry.

Everything else, products, collections, images, SEO, shipping, discounts, integrations, subscriptions, flows, templates, changes the store the moment the tool returns.

Before you change how the storefront looks, call get_custom_code and read `allowed`: the HTML and CSS the store keeps, the theme's colour and font tokens and keyframes, and the editor rules. If you have a design skill, load it first (in Claude Code: /frontend-design:frontend-design; in Codex, the design skill you have installed); otherwise follow https://formahand.com/docs/custom-code. Preview with get_preview_link before you publish, check the page at phone width (about 390px) as well as desktop, and honour prefers-reduced-motion.

## Draft, versions and media

1. `get_storefront_draft` → note `draft.revision`.
2. `list_modules` to discover commands (`set-copy`, `set-colors`, `set-theme`, `set-section-content`, `set-module-settings`, …) and module settings schemas.
3. `apply_storefront_commands { revision, commands }`. The store applies them, gives the draft a new revision and returns it. A stale revision is a 409 → `isError`; re-read and retry.
4. The merchant can see the draft in the builder at any time.
5. `publish_storefront { revision }`, the only way anything goes live. Requires `storefront:publish`, which the default token scopes do not include.

Storefront appearance and copy are the **only** thing that waits for a publish. Products, collections, images, custom fields, per-page SEO, redirects, posts, shipping, discounts, pricing and checkout rules, customer groups, integrations, subscriptions, flows, templates and orders all change the store the moment the tool returns.

### Version history and restore

Every save and every publish snapshots the whole editor document, and has done since the first editor. `list_storefront_versions { limit? }` (`storefront:read`) is the reader: newest first, each entry is `{ id, at, actor, kind: "save" | "publish" | "restore", revision, sections, pages, changed, summary }`, where `sections` and `pages` name what differs from the version before it and `changed` names which of `settings`, `areas`, `navigation` and `appearance` moved. `actor` is the token, the person or the assistant that made it.

`restore_storefront_version { id }` (`storefront:write`) writes that snapshot back over the **draft** with a new revision, exactly as a save would, and records itself in the history as a `restore` so the restore can itself be undone. **It never publishes.** What shoppers see is untouched until `publish_storefront { revision }` runs with the revision it returned, so an agent can restore, read `get_storefront_draft`, and change its mind. A snapshot an older editor wrote that no longer validates is refused with a 409 rather than half-applied. The answer also carries `dangling: [{ kind, reference }]`, the products, pages, collections and images the old version still points at that are no longer in the store. It never blocks the restore (the merchant asked for that version, and the draft is not live), but it is the list to clear before `publish_storefront`. Merchants use the same thing under **Online Store → Storefront → History**, with a confirm that says the change is not live yet.

Routes: `GET /api/storefront-versions` and `POST /api/storefront-versions { id }`.

### Media

Every image an agent or a merchant uploads lands in the store's own media library under `products/`, the prefix is a folder, not a promise: the same object may be a product photo, a collection cover, a hero or a picture inside a page. `list_media { limit?, cursor?, orphansOnly? }` (`products:read`) lists it with `{ key, url, bytes, contentType, uploadedAt, inUse, usedBy }`, where `usedBy` names each place that still points at the object (`product`, `variant`, `collection`, `storefront`, `page`, `post`) and `inUse: false` means nothing does. Media is billed by total size ([billing](https://formahand.com/docs/billing)), so orphans cost the merchant money for nothing; `orphanBytes` in the answer is what cleaning up would free.

"In use" means the **live** publication and the draft: the published theme version (which is where the logo and the favicon live, so neither is ever reported as an orphan), the live home page, the draft document, every product, variant, collection, content page and post, and every per-object `ogImageUrl`, a social card is referenced from nowhere else, so a scan that skipped it called every one an orphan. A superseded publication does not count, otherwise nothing a store had ever shown could be cleaned up.

`delete_media { key, force? }` (`products:write`) removes one object. An image something still points at is **refused** with the list of places that use it, because the alternative is a product page with a hole in it; clear the references first, or pass `force: true` when you mean it.

### Element editing and custom blocks

The draft also holds one entry per **element**, a single heading, paragraph, button or image inside a section, so an agent is never limited to whole sections:

- `set-element { areaId, key, value?, href?, src?, alt? }` writes one element's text, its button destination or its image. `areaId` is a home section (`hero`, `collection`, `more-products`, `styles`) or a page area (`chrome`, `catalog`, `product`, `cart`, …); `key` is one the theme declares. `list_modules` returns the command schema, and `get_storefront_draft` shows what is set today.
- `set-element-style { areaId, key, align?, size?, visible?, spacing? }` moves it on the page: `align` is `left | center | right`, `size` is `s | m | l | xl` on the theme's own type scale, `spacing` is `0`–`4`, and `visible: false` takes the element off the page. Pass `null` for `align`, `size` or `spacing` to hand it back to the theme. Nothing here is CSS: the values map onto each theme's own tokens, so the page stays responsive and looks like the theme.
- `add-custom-block { html?, css?, position?, placement?, sectionId?, pageSlug?, width?, visible? }` puts the merchant's own markup anywhere on the storefront (see "Page layout" below), with `update-custom-block`, `move-custom-block` and `remove-custom-block { id }` after it. See [Storefront modules v2](https://formahand.com/docs/modules-v2#custom-blocks).

Every one of these is an ordinary editor command: same revision check, same draft, live only after `publish_storefront`.

### Page layout

Every page is one ordered list of sections and custom blocks, in the order shoppers see it. The draft shows it as `document.layout`:

```json
{
  "pages": {
    "home": [
      { "kind": "block", "id": "intro" },
      { "kind": "section", "id": "hero" },
      { "kind": "section", "id": "collection" },
      { "kind": "block", "id": "faq" },
      { "kind": "section", "id": "more-products" },
      { "kind": "section", "id": "styles" }
    ],
    "catalog": [{ "kind": "block", "id": "sale" }, { "kind": "section", "id": "main" }],
    "page:about": [{ "kind": "section", "id": "main" }, { "kind": "block", "id": "team" }]
  },
  "slots": { "header.top": [{ "kind": "block", "id": "notice" }] }
}
```

`home` lists its four sections exactly once each. `catalog`, `product` and `page:<content page slug>` have one section, `main`, which is the page's own body; a page with no blocks is simply left out. `slots["header.top"]` sits under the announcement bar and above the navigation, `slots["footer.top"]` just above the footer, on every page but the payment page, and they hold blocks only. A hidden section draws nothing; the blocks beside it stay where they are.

Where a block goes:

- `position { placement?, target, page?, slot? }` on `add-custom-block`, `move-custom-block` and `update-custom-block`. `placement` is `before` or `after` (default `after`); `target` is a section id, a block id, `start` or `end`; `page` is `home` (default), `catalog`, `product` or `page:<slug>`; `slot` is `header.top` or `footer.top`. A block target brings its own page. Example: `{ "type": "move-custom-block", "id": "faq", "position": { "placement": "before", "target": "hero" } }`, or `{ "type": "add-custom-block", "html": "<p>Free delivery this week</p>", "position": { "target": "start", "slot": "header.top" } }`.
- The older `sectionId` (with `placement`) still works and means right after that home section and after the blocks already there, so blocks added one by one come out in the order they were sent; `pageSlug` means above or below that content page's body. Send either `position` or these, not both.
- `set-layout { page, order }` (or `{ slot, order }`) sets a whole page at once: `order` lists every section of that page once and every block that should be on it, as `{ kind, id }` or a bare id (a bare id is a section when the page has one by that name). A block from another page or slot moves in; a block already on the page and left out is refused rather than lost. Example: `{ "type": "set-layout", "page": "home", "order": ["styles", { "kind": "block", "id": "intro" }, "hero", "collection", "more-products"] }`.
- `move-section { sectionId, index }` moves a home section alone; the blocks keep their places.

A target that is not there is refused with the usual command error, for instance `Command 2 (move-custom-block, faq): position.target "nope" is not a section or a block on the storefront.` Drafts written before layouts existed are read with the layout their blocks' anchors describe, which renders exactly as before.

## Editor commands

`apply_storefront_commands { revision, commands }` takes these, generated from the editor's own schema (`?` marks optional). A batch applies whole or not at all; `list_modules` answers the same schema live.

| Command | Fields |
| --- | --- |
| `set-copy` | `heading?: string, intro?: string` |
| `set-colors` | `background?: string, accent?: string, foreground?: string, muted?: string, border?: string` |
| `set-logo` | `url: string, alt?: string, width?: integer` |
| `set-favicon` | `url: string` |
| `set-fonts` | `pairing: enum` |
| `set-social` | `links: object` |
| `set-badges` | `newDays: integer` |
| `set-theme` | `presetId: "shopco" \| "atelier" \| "market" \| "studio" \| "orebi" \| "catalog"` |
| `set-section-visibility` | `sectionId: "hero" \| "collection" \| "more-products" \| "styles", visible: boolean` |
| `move-section` | `sectionId: "hero" \| "collection" \| "more-products" \| "styles", index: integer` |
| `set-section-content` | `sectionId: "hero" \| "collection" \| "more-products" \| "styles", content: object` |
| `set-hero-image` | `url: string` |
| `set-area-content` | `areaId: enum, content: object` |
| `set-navigation` | `navigation: object` |
| `set-content-pages` | `pages: object[]` |
| `set-module-settings` | `moduleId: enum, settings: object` |
| `set-element` | `areaId: enum, key: string, value?: string, href?: string, src?: string, alt?: string` |
| `set-element-style` | `areaId: enum, key: string, align?: "left" \| "center" \| "right", size?: "s" \| "m" \| "l" \| "xl", visible?: boolean, spacing?: integer` |
| `add-custom-block` | `id?: string, html?: string, css?: string, position?: object, placement?: "before" \| "after", sectionId?: "hero" \| "collection" \| "more-products" \| "styles", pageSlug?: string, width?: "content" \| "full", visible?: boolean` |
| `update-custom-block` | `id: string, html?: string, css?: string, position?: object, placement?: "before" \| "after", sectionId?: "hero" \| "collection" \| "more-products" \| "styles", pageSlug?: string, width?: "content" \| "full", visible?: boolean` |
| `remove-custom-block` | `id: string` |
| `set-theme-css` | `css: string` |
| `set-script-embeds` | `embeds: object[]` |
| `move-custom-block` | `id: string, position?: object, placement?: "before" \| "after", sectionId?: "hero" \| "collection" \| "more-products" \| "styles", pageSlug?: string` |
| `set-layout` | `page?: string, slot?: "header.top" \| "footer.top", order: object \| string[]` |

## Modules

A section is built from modules. `list_modules` gives one line per module; `list_modules { id }` gives that module's full settings schema.

| Module | What it is for | Settings |
| --- | --- | --- |
| `product-variants` Product options | Size, colour and other option pickers on products that have variants. | `enabled`, `style`, `label`, `cornerRadius`, `gap` |
| `announcement-bar` Announcement bar | A one-line message above the navigation, with an optional link. | `enabled`, `text`, `linkLabel`, `linkHref`, `tone` |
| `catalog-search` Search | A search box in the header; results use the catalog page. | `enabled`, `placeholder`, `buttonLabel` |
| `catalog-filters` Filters & sorting | Collection chips, faceted filters with counts, sorting, the result count and the pager on the catalog, collection and search pages. | `enabled`, `showCollections`, `showSort`, `showCount`, `showFacets`, `showPrice`, `allLabel`, `sortLabel`, `clearLabel` |
| `collection-grid` Collection grid | Cards for your collections on the home page. | `enabled`, `heading`, `limit`, `showCount` |
| `product-recommendations` Recommendations | “Add to your order” under the cart, from the same picks the product page recommends. Product pages already carry their theme's own recommendation row. | `enabled`, `title`, `cartTitle`, `limit` |
| `trust-badges` Trust badges | A reassurance row under the cart and payment page: secure checkout, returns and dispatch time. | `enabled`, `secureTitle`, `secureText`, `returnsTitle`, `returnsText`, `shippingTitle`, `shippingText` |
| `custom-block` Custom block | Your own HTML and CSS anywhere on the home, catalog, product or a content page, or in the header or footer. Up to 40 blocks; scripts, inline styles and tracking pixels are removed, and each block's CSS only applies inside that block. | `enabled`, `blocks` |
| `app-block` App block | Show what one of your installed apps has to say, reviews on a product page, a return request form, a badge, in a place you choose. The app supplies the data and the layout; the store draws it and fills it in, so nothing an app sends ever runs on a shopper's page. Up to 8 blocks. | `enabled`, `blocks` |

## Custom HTML and CSS: what you can use

Everything below is also returned by `get_custom_code` under `allowed`, so an agent can read the rules before it writes anything.

### In a block's HTML

Kept: `h2`, `h3`, `h4`, `h5`, `h6`, `p`, `a`, `img`, `ul`, `ol`, `li`, `strong`, `em`, `br`, `hr`, `blockquote`, `figure`, `figcaption`, `details`, `summary`, `dl`, `dt`, `dd`, `b`, `i`, `u`, `s`, `small`, `mark`, `sub`, `sup`, `abbr`, `time`, `code`, `pre`, `section`, `article`, `table`, `thead`, `tbody`, `tfoot`, `tr`, `th`, `td`, `caption`, `div`, `span`, `iframe`.

**No `h1`.** Not allowed in a block: the page already has its one h1 (the store or page title), and a second one confuses screen readers and search engines. Start a block's headings at h2; an h1 you send keeps its text and loses the tag.

Any other element loses its tags and keeps its text. Comments, on* handlers and the style attribute are removed.

| Attribute | Where |
| --- | --- |
| `class`, `id`, `title`, `lang`, `dir`, up to 4 data-* attributes, values up to 64 characters | every element |
| `href`, `target` | `a` |
| `src`, `alt`, `width`, `height` | `img` |
| `src`, `width`, `height`, `allowfullscreen` | `iframe` |
| `colspan`, `rowspan` | `td` |
| `colspan`, `rowspan`, `scope` | `th` |
| `datetime` | `time` |
| `open` | `details` |

- Links: href is a site path, #anchor, https:, mailto: or tel:. target="_blank" is the only target, and every link gets rel="noopener noreferrer nofollow".
- Images: src is https only; loading="lazy" and an empty alt are added when missing.
- Video: iframe src is an https player address only: www.youtube.com/embed/…, youtube.com/embed/…, www.youtube-nocookie.com/embed/…, player.vimeo.com/video/….
- Ids: id="offer" is published as id="fhc-offer", and href="#offer" in the same block follows it.
- Scripts: A block runs no JavaScript. For analytics or chat, switch on a provider with set_script_embeds.

### In CSS (the theme stylesheet and a block's CSS)

| At-rule | Kept |
| --- | --- |
| `@media` | any media query, including prefers-reduced-motion |
| `@supports` | feature queries |
| `@starting-style` | entry styles for transitions (top level) |
| `@keyframes` | from, to and percentage frames; page-wide names, so pick distinctive ones |
| `@property` | syntax, inherits and initial-value; lets a custom property animate |
| `@font-face` | every src is local(), an https file on fonts.gstatic.com, or a path under your store's own /media/ |

Removed: `@import`, `@charset`, `@namespace`, `@layer`, `@container`, `@page`, `@font-feature-values`, `@counter-style` and any other at-rule, with everything inside them. Inside rules: position: fixed (a fixed layer can cover the checkout button); url() that is neither https nor a path under your store's own /media/.

- Theme stylesheet: Every selector in the theme stylesheet is published under [data-fh-storefront], the storefront's own <body>. :root, html and body mean that element, so page-wide colours and fonts go there.
- Block CSS: Every selector in a block's CSS is prefixed with that block's own class, so it only reaches inside the block. :scope or & means the block itself.
- Keyframes, @property and @font-face are page-wide and not scoped.
- Honour reduced motion: put animations inside @media (prefers-reduced-motion: no-preference), or switch them off under prefers-reduced-motion: reduce.

### What every theme already gives you

Colour tokens `--fh-background`, `--fh-foreground`, `--fh-accent`, `--fh-on-accent`, `--fh-muted`, `--fh-border`: The store's brand colours as chosen in the builder (set-colors). Every theme defines them, so CSS built on them follows the merchant's palette.

```css
.promo{background:var(--fh-accent);color:var(--fh-on-accent)}
.promo small{color:var(--fh-muted);border-top:1px solid var(--fh-border)}
```

Font tokens `--fh-font-heading`, `--fh-font-body`: Defined when the merchant picks a font pairing (set-fonts); give a fallback so the theme's own font applies otherwise.

```css
.promo h2{font-family:var(--fh-font-heading,inherit)}
.promo p{font-family:var(--fh-font-body,inherit)}
```

Keyframes each theme ships, usable by name in `animation`:

| Theme | Keyframes |
| --- | --- |
| Shopco (`shopco`) | `pulse`, `spin` |
| Luma (`atelier`) | `enter`, `exit`, `pulse` |
| Munchies (`market`) | `marquee` |
| Noir (`studio`) | `fadeIn`, `marquee` |
| Gridline (`orebi`) | `enter`, `exit` |
| Barrio (`catalog`) | `pulse` |

```css
/* Luma and Gridline: fade and rise in; set the start values with --tw-enter-* */
.promo{--tw-enter-opacity:0;--tw-enter-translate-y:1rem;animation:enter .5s ease-out both}
```

```css
/* Luma and Gridline: the reverse of enter, with --tw-exit-* */
.promo.is-leaving{--tw-exit-opacity:0;animation:exit .3s ease-in both}
```

```css
/* Shopco, Luma and Barrio: a soft opacity pulse */
.badge{animation:pulse 2s ease-in-out infinite}
```

```css
/* Every theme: define your own and it works anywhere */
@keyframes promo-rise{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}
@media (prefers-reduced-motion:no-preference){.promo{animation:promo-rise .5s ease-out both}}
```

### Limits

| What | Limit |
| --- | --- |
| HTML in one block | 32 KB |
| CSS in one block | 8 KB |
| The theme stylesheet | 64 KB |
| Blocks per store | 40 |
| Script embeds | 4, one per provider |
| Everything together | 256 KB of blocks and CSS |

### Editor rules

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

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `list_modules` | `storefront:read` | Discover what the storefront editor can do: the editor commands `apply_storefront_commands` accepts, as a JSON schema, and the storefront modules a section can be built from. |
| `get_storefront_draft` | `storefront:read` | The current storefront draft: { draft: { revision, document, publishedRevision }, liveVersion, modules }. |
| `apply_storefront_commands` | `storefront:write` | Apply editor commands (see list_modules) to the draft, at most 30 per call. |
| `publish_storefront` | `storefront:publish` | Publish the draft at this revision so shoppers see it. |
| `set_hero_image` | `storefront:write` | Set the home page hero image of the storefront draft: { revision, url \| dataBase64 + contentType }. |
| `set_logo` | `storefront:write` | Set the storefront's header logo in the draft: { revision, url \| dataBase64 + contentType, alt?, width? }. |
| `set_favicon` | `storefront:write` | Set the browser-tab icon in the storefront draft: { revision, url \| dataBase64 + contentType }. |
| `set_navigation` | `storefront:write` | Replace the header and footer links of the storefront draft: { revision, navigation: { header: [{ label, href }] (max 12), footer (max 20) } }; href is a site path like /pages/about or an https URL. |
| `set_content_pages` | `storefront:write` | Replace the storefront draft's content pages (About, FAQ, policies…): { revision, pages: [{ slug, title, bodyMarkdown, status: 'draft'\|'published' }] } (max 12, unique slugs; pages render at /pages/<slug>). |
| `get_custom_code` | `storefront:read` | Everything a store has added itself, and what it may add: { blocks: [{ id, where, index, sectionId, placement, pageSlug, width, visible, htmlBytes, cssBytes }], customCss, embeds: [{ provider, id, enabled }], meter: { used, limit, available, blocks,… |
| `set_custom_block` | `storefront:write` | Adds, changes or removes one custom HTML block in the draft: { revision, id?, html?, css?, position?, sectionId?, placement?, pageSlug?, width?, visible?, remove? }. |
| `set_theme_css` | `storefront:write` | Replaces the store's own stylesheet in the draft: { revision, css }. |
| `set_script_embeds` | `storefront:write` | Replaces the store's script embeds in the draft: { revision, embeds: [{ provider, id, enabled? }] }. |
| `list_themes` | `storefront:read` | The storefront themes a store can wear: { themes: [{ id, name, description, categories, cover, settings, content }] }. |
| `list_storefront_versions` | `storefront:read` | What the storefront looked like before: { versions: [{ id, at, actor, kind, revision, sections, pages, changed, summary }], hasMore }. |
| `restore_storefront_version` | `storefront:write` | Put an old version back as the current DRAFT: { id } (from list_storefront_versions) → { restored, draft: { revision, document, publishedRevision }, published: false, dangling }. |
| `get_preview_link` | `storefront:read` | Two signed links that open the storefront whatever its launch mode, so you can look at a "Coming soon" or password-protected store without opening it or changing anything: { published: { url, expiresAt }, draft: { url, expiresAt }, launchMode }. |
| `get_launch_mode` | `settings:read` | Whether the live storefront is open or still private (live data): { access: { access: "open" \| "password" \| "coming_soon", message, hasPassword, updatedAt } }. |
| `set_launch_mode` | `settings:write` | Opens the live store to shoppers, or keeps it private. |

## Read more

- https://formahand.com/docs/custom-code
- https://formahand.com/docs/modules-v2
- https://formahand.com/docs/agents
