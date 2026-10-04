# Storefront: draft, design and publish

How to change what shoppers see: the draft and its revision, the editor commands, the modules a section is built from, custom HTML and CSS, and publishing. Nothing here reaches shoppers until `publish_storefront`.

## The loop

Storefront appearance and copy (theme, sections, menus, pages) live in a revision-checked draft: `get_storefront_draft` → `apply_storefront_commands` (listed by `list_modules`) / `set_hero_image` / `set_navigation` / `set_content_pages` with that `revision` → `publish_storefront { revision }`. Shoppers see nothing before publish; a stale revision conflicts: re-read, retry. Pick a look: `list_themes`, then `set-theme { presetId }`.

Sections (`describe_sections` → `save_section`): put `data-fh-el="<setting key>"` on every element printing a text, image or link setting and `data-fh-group` on reorderable flex or grid containers, or saving is refused (`element_unmarked`); merchants then edit each element in the builder. One element: `update-section` for text, `set-element-style { instanceId, key, align, size, visible, spacing, order, offsetX, offsetY, width, height, layer, mobile, tablet }`, `set-element-order { instanceId, group, order }`; theme sections take `areaId`. Custom blocks are not editable inside.

Rebuilding a site ("make it look like this"): read `get_custom_code` (`allowed.recreate`, `allowed.motion`), start from `set-theme { presetId: "blank-canvas" }`, build each band as a section, `remove-section` replaced theme sections.

Everything else (catalog, images, SEO, shipping, discounts, flows) is live at once.

Before changing a storefront, inspect the reference across desktop/mobile and relevant interaction/visitor states; record what was observed, inferred or remains unknown. Check rendered dependencies when possible and do not claim hidden states from one visit. Then call get_custom_code and read `allowed`, plus describe_sections, to verify the current save/render contract. Rebuild the visual intent with authorized merchant assets and truthful store data; make gaps and substitutions visible. Use set-theme { presetId: "blank-canvas" } and one editable section per band where appropriate, marking settings with data-fh-el and reorderable groups with data-fh-group. CSS handles presentation; agent interactions use separate `behavior`, subject to the owner/admin Scripts in sections switch and its inert builder preview. Use the current media and font upload paths from the skill references. Compare the draft at matching viewports/states, test real store flows in scope, and report captured, implemented, locally verified, store-verified and published as distinct statuses. Preview does not authorize publish; honor prefers-reduced-motion.

## Start from a theme

One word: a look is a **theme**, and there is one way to pick one. A theme's `id` is an identifier to pass back, not a name to show: tell the merchant the theme's `name`.

1. `list_themes`: every theme, `{ id, name, category, line, pages, preview, phone, licence, current?, demoUrl }`, with `current` naming the one the draft wears. `preview` and `phone` are pictures of its home page. `demoUrl`: a hosted demo store of the theme, open to visitors, or null.
2. `get_theme { id }`: one theme whole: its line, pages and pictures, then its sections (summarised), every page's instances with their settings, the pictures, menus, starter pages and licence.
3. `apply_storefront_commands { revision, commands: [{ type: "set-theme", presetId: "<id>" }] }`, with the `id` that `list_themes` returns: picks it, in one write with the rest of the batch. The answer's `theme` is `{ id, name, preset, sections, summary }`, and `theme.summary` is also the first of `notes`. For a design theme or a starter the rest of the batch is applied right after, on the new revision; if one of those commands is refused, the theme is already in the draft and the refusal says so (`themeApplied`, `revision`).
4. Adjust: `update-section` for words and pictures, `set-colors`, `set-fonts`, `set-section-visibility`; replace the placeholder pictures with the merchant's own (`upload_image`, then `update-section`).
5. `get_preview_link`, then `publish_storefront { revision }`.

`get_store` answers `theme: { id, name }`, the theme the draft wears as `list_themes` names it (`themePresetId` is the older field and names the original theme a design builds on).

**Older names.** `list_presets`, `get_preset { id }` and `apply_preset { id, dryRun?, revision?, reset? }` still work and read the same catalogue; `apply_preset` is how to ask for a dry run or a reset. Every id works in all of them: the `id` from `list_themes` or the one `list_presets` returns. `apply_preset` answers `{ applied, revision, summary?, replaced, kept, replacedInstances, notKept, unplaced?, modules?, contentPages: { added, skipped }, stylesheetReplaced, sections: { ids, reused, restored }, imagesUploaded, notes, next }`. A read-only theme section is named `<theme>--<name>`, which your own sections cannot be, so it never takes one of your ids. A section in the way that came from an add-on is refused (`preset_conflict`) and nothing changes: make a copy of it under another id first.

**Starting a design from nothing.** `set-theme { presetId: "blank-canvas" }` is where an agent starts when it rebuilds a storefront. It takes the home sections off the page and places plain starter sections instead: a hero (heading, text, button and picture in a group the merchant can reorder), a product grid (choose its collection with `update-section { instanceId: "shop", settings: { collection: "<handle>" } }`; until then it shows its heading and a link to every product), a text and picture band, questions and answers, a newsletter sign-up, and a header and footer. Every heading, text, button and picture in them is marked, so the merchant can select and change it in the builder. They are ordinary sections of your store (`canvas-hero`, `canvas-products` and so on): change them with `save_section`, or replace them with your own.

The `apply-preset` editor command exists so the draft's history is honest, but it only runs through `apply_preset` and `set-theme`, which add the sections and pictures first. Sent on its own to `apply_storefront_commands` it is refused with that sentence.

## The kinds of theme

- Picking one keeps what is yours: your colours, typefaces and copy follow the theme rules, your stylesheet is kept with the theme's spacing and corner tokens in front of it, a menu that has links is kept, and your hero image goes onto the theme's hero section.
- **The design themes** (Rounded Edit, Sage Apothecary, Pale Studio, Quiet Wardrobe, Fitting room, Checkerboard Aperitif, Garden Room) are whole looks: their own colours, typefaces, stylesheet and menus, and most lay out the product, catalog and cart pages too.
- **The starters**: Clean grid and Editorial promo are simple whole looks; **Blank canvas** is a blank start, plain sections you fill in yourself.

**Your code is never overwritten.** If your store already has sections with a theme's ids (you picked it before, and may have changed them), they are used as they are, at their own versions, and the answer says so (`sections.reused`). To get the originals back, apply it with `apply_preset { id, reset: true }`: each original is saved as a new version of its section, so your version stays in the section's history. A store that is on one of the original themes today keeps its home page until you pick a theme: nothing changes on its own.

## What a theme changes

| Part of your draft | What happens |
| --- | --- |
| Colours, typefaces | A design theme or starter brings its own; an original theme keeps yours |
| Your stylesheet | Replaced by a design theme's (its radius and spacing tokens, then its own CSS); an original theme keeps it |
| Every page's layout and the header and footer slots | Replaced by the theme's, with its sections at the versions it was made with |
| Header and footer menus | Replaced by the theme's (an original theme keeps a menu that has links) |
| Store modules | Only the ones the theme names (`modules`), such as the "Shop by collection" grid a design leaves out: switched on or off as the theme says. The answer lists them (`modules`); `set-module` changes them back. Custom HTML and add-on blocks are never touched |
| Starter pages | Added only where you have no page with that address; your own pages are never touched |
| Your own sections | **Kept.** Each instance moves to the end of the page it was on, hidden, and the answer lists them. Show one again with `set-section-visibility`. A page body section of your own (product, catalog or cart) is the exception: a page has one body, and the theme's page brings its own, so yours comes off that page and stays in your store (`unplaced`); put it back in place of the page's body whenever you like |
| Custom HTML blocks | Kept the same way, hidden at the end of their page |
| Store name, heading, intro, logo, tab icon, social links, hero image, page copy, embeds | Kept |
| Products, collections, orders, customers | Never touched |

The theme's sections are added to your store and listed by `list_sections`, like any section. Some themes bring read-only sections (`fork_section` makes your own copy); the original themes and **Blank canvas** bring sections that are your store's own from the moment you apply them, so you (or your agent) change them with `save_section` and Formahand never updates them. Its pictures and drawings are added to your media library, so every image setting points at your own `/media/` path; replace them with your photographs whenever you like. A picture is either a neutral shape in the theme's colours or one that comes with the design. Applying the same theme again keeps one copy of each picture in your library, not two.

**Typefaces.** A theme either picks one of the store's font pairings (the same list as Brand in Storefront settings) or brings the design's own heading and body typefaces, chosen from the section font list. With its own, your store's typefaces become **Your own typefaces**: every page loads the two families once, and the headings and text follow them, catalog, product and cart pages included. Choosing another pairing in Brand, or `set-fonts` with another pairing, puts them aside; `set-fonts { pairing: "custom", headingFamily, bodyFamily }` changes them, each an exact name from the font list (`describe_sections`, part `fonts`). Each family loads 400 to 700; a theme that names other weights (`weights: [300, 400]`) or `italic: true` for a family loads those too, at most six weights, and `set-fonts` takes `headingWeights`, `bodyWeights` (any weights the family has, at most six; `[]` goes back to 400 to 700), `headingItalic` and `bodyItalic`.

**Undo.** Before anything changes, the draft as it was is saved in the version history (the dashboard shows it as **"Before theme <name>"**; `list_storefront_versions` names it "Before preset <name>"). Restoring it (`restore_storefront_version`, or Version history in Storefront settings) brings the draft back exactly, sections, pins and all.

**Picking a second theme** replaces the first one's look and sections on your pages (the first one's sections stay in your store), and keeps your own sections as above. Picking the same theme again starts its pages over, with the sections your store already has.

**A dry run** (`apply_preset { id, dryRun: true }`, or the first step of **Use this theme** in the dashboard) answers with everything above for your store (what would be replaced, which of your sections would be kept and where, which starter pages would be added or skipped) and writes nothing. It needs only the read scope.

## Draft, versions and media

1. `get_storefront_draft` → note `draft.revision`.
2. `list_modules` to discover commands (`set-copy`, `set-colors`, `set-theme`, `set-section-content`, `set-module-settings`, …) and module settings schemas.
3. `apply_storefront_commands { revision, commands }`. The store applies them, gives the draft a new revision and returns it. A stale revision is a 409 → `isError`; re-read and retry.
4. The merchant can see the draft in the builder at any time.
5. `publish_storefront { revision }`, the only way anything goes live. Requires `storefront:publish`, which the default token scopes do not include.

Storefront appearance and copy are the **only** thing that waits for a publish. Products, collections, images, custom fields, per-page SEO, redirects, posts, shipping, discounts, pricing and checkout rules, customer groups, integrations, subscriptions, flows, templates and orders all change the store the moment the tool returns.

### Version history and restore

Every save and every publish snapshots the whole editor document, and has done since the first editor. `list_storefront_versions { limit? }` (`storefront:read`) is the reader: newest first, each entry is `{ id, at, actor, kind: "save" | "publish" | "restore", revision, sections, pages, changed, summary }`, where `sections` (section instance ids, including one whose section now renders at another version) and `pages` name what differs from the version before it and `changed` names which of `settings`, `areas`, `navigation` and `appearance` moved. `actor` is the token, the person or the assistant that made it.

`restore_storefront_version { id }` (`storefront:write`) writes that snapshot back over the **draft** with a new revision, exactly as a save would, and records itself in the history as a `restore` so the restore can itself be undone. **It never publishes.** What shoppers see is untouched until `publish_storefront { revision }` runs with the revision it returned, so an agent can restore, read `get_storefront_draft`, and change its mind. A snapshot comes back as schema version 3 with every section instance and the version it was pinned at; one saved before sections is moved to version 3 on the way, as a stored draft is. A snapshot an older editor wrote that no longer validates is refused with a 409 rather than half-applied. The answer also carries `dangling: [{ kind, reference }]`: the products, pages, collections and images the old version still points at that are no longer in the store, and any section version it pins that the store no longer keeps (`kind: "section"`). It never blocks the restore (the merchant asked for that version, and the draft is not live), but it is the list to clear before `publish_storefront`. Merchants use the same thing under **Online Store → Storefront → History**, with a confirm that says the change is not live yet.

Routes: `GET /api/storefront-versions` and `POST /api/storefront-versions { id }`.

### Media

Every image, video or licensed font an agent or a merchant uploads lands in the store's own media library under `products/`, the prefix is a folder, not a promise: the same object may be a product photo, a collection cover, a hero, a video in a section or a picture inside a page. `list_media { limit?, cursor?, orphansOnly? }` (`products:read`) lists it with `{ key, url, bytes, contentType, uploadedAt, inUse, usedBy }`, where `usedBy` names each place that still points at the object (`product`, `variant`, `collection`, `storefront`, `page`, `post`) and `inUse: false` means nothing does. Media is billed by total size ([billing](https://formahand.com/docs/billing)), so orphans cost the merchant money for nothing; `orphanBytes` in the answer is what cleaning up would free. Upload a video with `upload_section_video`; a WOFF2 font still needs the owner's licence attestation in the dashboard.

"In use" means the **live** publication and the draft: the published theme version (which is where the logo and the favicon live, so neither is ever reported as an orphan), the live home page, the draft document, every product, variant, collection, content page and post, and every per-object `ogImageUrl`, a social card is referenced from nowhere else, so a scan that skipped it called every one an orphan. A superseded publication does not count, otherwise nothing a store had ever shown could be cleaned up.

`delete_media { key, force? }` (`products:write`) removes one object. An image something still points at is **refused** with the list of places that use it, because the alternative is a product page with a hole in it; clear the references first, or pass `force: true` when you mean it.

### Element editing and custom blocks

The draft also holds one entry per **element**, a single heading, paragraph, button or image inside a section, so an agent is never limited to whole sections:

- `set-element { areaId, key, value?, href?, src?, alt? }` writes one element's text, its button destination or its image. `areaId` is a home section (`hero`, `collection`, `more-products`, `styles`) or a page area (`chrome`, `catalog`, `product`, `cart`, …); `key` is one the theme declares. `list_modules` returns the command schema, and `get_storefront_draft` shows what is set today.
- `set-element-style { areaId | instanceId, key, align?, size?, visible?, spacing?, order?, offsetX?, offsetY?, mobile? }` changes its look and place. On a store section it takes `instanceId` and one of the elements its template marks (`data-fh-el`); `order` (-20 to 20) moves it inside its group, `offsetX` and `offsetY` (-400 to 400 px) nudge it, and `mobile { order?, offsetX?, offsetY?, align?, visible? }` applies at phone width only. The rest works on both kinds of section: `align` is `left | center | right`, `size` is `s | m | l | xl` on the theme's own type scale, `spacing` is `0`–`4`, and `visible: false` takes the element off the page. Pass `null` for `align`, `size` or `spacing` to hand it back to the theme. Nothing here is CSS: the values map onto each theme's own tokens, so the page stays responsive and looks like the theme.
- `set-element-order { instanceId, group, order: [keys] , mobile? }` numbers the elements of one reorderable group (`data-fh-group`) of a store section in one step. A store section's text, images and links are its settings: `update-section { instanceId, settings }`. See [sections](https://formahand.com/docs/sections), "Editable elements".
- `remove-section { instanceId }` takes a theme home section (`hero`, `collection`, `more-products`, `styles`) off the page for good, not just hidden; `add-section { sectionId: "hero", position? }` puts it back once, on home.
- `add-custom-block { html?, css?, position?, placement?, sectionId?, pageSlug?, width?, visible? }` puts the merchant's own markup anywhere on the storefront (see "Page layout" below), with `update-custom-block`, `move-custom-block` and `remove-custom-block { id }` after it. See [Storefront modules v2](https://formahand.com/docs/modules-v2#custom-blocks). Nothing inside a block can be selected or edited in the builder; for anything the merchant should change later, build a section.

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

### Starting from a theme

If the merchant wants a ready-made look, start from a **theme** before designing from scratch ([themes](https://formahand.com/docs/storefront-themes)): `list_themes` (each with pictures of its home page and `current` on the one the draft wears), `get_theme { id }` for one whole, then `apply_storefront_commands { revision, commands: [{ type: "set-theme", presetId: "<id>" }] }`. The six original themes put their own header, footer and home page in the store and keep the merchant's colours, typefaces, copy and menus; every other theme brings its whole look. The merchant's own sections are kept, hidden at the end of their page, and the answer's `theme.summary` names what changed. Then adjust with `apply_storefront_commands` and publish. The draft as it was is in the version history as "Before preset <name>". `apply_preset { id, dryRun: true }` says what a theme would change before anything is written, and `apply_preset { id, reset: true }` puts a theme's original sections back over the store's own copies as new versions; `list_presets`, `get_preset` and `apply_preset` are the older names for the same catalogue.

### Sections of your own

When a layout repeats, or the merchant should edit it later, build a **section** rather than a raw HTML block: a package of typed settings the builder turns into fields, a template, its own styles and presets, saved once and placed as often as needed ([sections](https://formahand.com/docs/sections)). The loop:

1. `describe_sections` for the package format, the setting types, the template language and the limits.
2. `save_section { package }`. The package is checked whole, and a refusal names every problem (`Section promo-band, template line 4: …`). Saving again makes a new version; leave `version` out and it is chosen for you.
3. `preview_section { id, settings }` to see one instance before it is on any page.
4. `apply_storefront_commands` with `{ type: "add-section", sectionId, position, settings }` places it on the draft; `update-section`, `apply-section-preset`, `duplicate-section`, `set-section-visibility` and `remove-section` change it later.
5. `get_preview_link`, then `publish_storefront { revision }`.

Saving a section moves the draft to its new version; the live storefront keeps the version it was published with until you publish. Add-on and catalogue sections are read-only: `fork_section` makes your own copy.

## Editor commands

`apply_storefront_commands { revision, commands }` takes these, generated from the editor's own schema (`?` marks optional). A batch applies whole or not at all; `list_modules` answers the same schema live.

| Command | Fields |
| --- | --- |
| `set-copy` | `heading?: string, intro?: string` |
| `set-colors` | `background?: string, accent?: string, foreground?: string, muted?: string, border?: string` |
| `set-logo` | `url: string, alt?: string, width?: integer` |
| `set-favicon` | `url: string` |
| `set-fonts` | `pairing: enum, headingFamily?: string, bodyFamily?: string, headingWeights?: integer[], bodyWeights?: integer[], headingItalic?: boolean, bodyItalic?: boolean` |
| `set-social` | `links: object` |
| `set-badges` | `newDays: integer` |
| `set-theme` | `presetId: "shopco" \| "atelier" \| "market" \| "studio" \| "orebi" \| "catalog"` |
| `set-section-visibility` | `instanceId?: any, sectionId?: any, page?: any, visible: boolean` |
| `move-section` | `instanceId?: any, sectionId?: any, page?: any, index?: integer, position?: any` |
| `add-section` | `sectionId: string, instanceId?: string, position?: any, settings?: object, presetId?: string` |
| `update-section` | `instanceId: any, page?: any, settings: object` |
| `apply-section-preset` | `instanceId: any, page?: any, presetId: string` |
| `duplicate-section` | `instanceId: any, page?: any, newInstanceId?: string, position?: any` |
| `remove-section` | `instanceId: any, page?: any` |
| `set-section-version` | `sectionId: string, version: string` |
| `apply-preset` | `presetId: string, version?: string` |
| `set-section-content` | `sectionId: any, content: any` |
| `set-hero-image` | `url: string` |
| `set-area-content` | `areaId: enum, content: any` |
| `set-navigation` | `navigation: object` |
| `set-content-pages` | `pages: object[]` |
| `set-module-settings` | `moduleId: enum, settings: object` |
| `set-element` | `areaId: any, key: any, value?: string, href?: string, src?: string, alt?: string` |
| `set-element-style` | `areaId?: any, instanceId?: any, page?: any, key: string, align?: "left" \| "center" \| "right", size?: "s" \| "m" \| "l" \| "xl", visible?: boolean, spacing?: integer, order?: any, offsetX?: any, offsetY?: any, width?: any, height?: any, layer?: any, mobile?: any, tablet?: any` |
| `set-element-order` | `instanceId: any, page?: any, group: any, order: any[], mobile?: boolean, tablet?: boolean` |
| `add-custom-block` | `id?: any, html?: string, css?: string, position?: any, placement?: "before" \| "after", sectionId?: any, pageSlug?: string, width?: "content" \| "full", visible?: boolean` |
| `update-custom-block` | `id: any, html?: string, css?: string, position?: any, placement?: "before" \| "after", sectionId?: any, pageSlug?: string, width?: "content" \| "full", visible?: boolean` |
| `remove-custom-block` | `id: any` |
| `set-theme-css` | `css: string` |
| `set-script-embeds` | `embeds: object[]` |
| `move-custom-block` | `id: any, position?: any, placement?: "before" \| "after", sectionId?: any, pageSlug?: string` |
| `set-layout` | `page?: any, slot?: "header.top" \| "footer.top" \| "header" \| "footer", order: object \| string[]` |

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

Kept: `h2`, `h3`, `h4`, `h5`, `h6`, `p`, `a`, `img`, `ul`, `ol`, `li`, `strong`, `em`, `br`, `hr`, `blockquote`, `figure`, `figcaption`, `details`, `summary`, `dl`, `dt`, `dd`, `b`, `i`, `u`, `s`, `small`, `mark`, `sub`, `sup`, `abbr`, `time`, `code`, `pre`, `section`, `article`, `nav`, `table`, `thead`, `tbody`, `tfoot`, `tr`, `th`, `td`, `caption`, `div`, `span`, `iframe`.

**No `h1`.** Not allowed in a block: the page already has its one h1 (the store or page title), and a second one confuses screen readers and search engines. Start a block's headings at h2; an h1 you send keeps its text and loses the tag.

Any other element loses its tags and keeps its text. Comments, on* handlers and the style attribute are removed.

| Attribute | Where |
| --- | --- |
| `class`, `id`, `title`, `lang`, `dir`, `hidden`, up to 4 data-* attributes, values up to 64 characters, aria-label (up to 200 characters), aria-labelledby, aria-describedby and aria-controls (ids, namespaced like id), aria-hidden and aria-expanded (true or false), aria-current (page, step, location, date, time, true, false), tabindex (0 or -1), role (banner, complementary, contentinfo, definition, dialog, figure, group, img, list, listitem, navigation, none, note, presentation, region, search, separator, status, term) | every element |
| `src`, `poster`, `controls`, `muted`, `loop`, `autoplay`, `playsinline`, `preload`, `width`, `height` | `video` |
| `src`, `srcset`, `type`, `media`, `sizes` | `source` |
| `href`, `target` | `a` |
| `srcset`, `sizes`, `src`, `alt`, `width`, `height`, `fetchpriority`, `decoding` | `img` |
| `src`, `width`, `height`, `allowfullscreen` | `iframe` |
| `colspan`, `rowspan` | `td` |
| `colspan`, `rowspan`, `scope` | `th` |
| `datetime` | `time` |
| `open` | `details` |

- Links: href is a site path, #anchor, https:, mailto: or tel:. target="_blank" is the only target. A link to another site gets rel="noopener noreferrer nofollow"; a site path or #anchor gets none, or rel="noopener noreferrer" with target="_blank".
- Images: src is https only; an empty alt is added when missing, and loading="lazy" unless you wrote loading="eager". fetchpriority (high, low, auto) and decoding (sync, async, auto) are kept.
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

Removed: `@import`, `@charset`, `@namespace`, `@layer`, `@container`, `@page`, `@font-feature-values`, `@counter-style` and any other at-rule, with everything inside them. Inside rules: position: fixed, and any position that is not a plain word such as var() (a fixed layer can cover the checkout button); url() that is neither https nor a path under your store's own /media/.

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
| Bold Street (`shopco`) | `pulse`, `spin` |
| Luma (`atelier`) | `enter`, `exit`, `pulse` |
| Corner Pantry (`market`) | `marquee` |
| Noir (`studio`) | `fadeIn`, `marquee` |
| Gridline (`orebi`) | `enter`, `exit` |

```css
/* Luma and Gridline: fade and rise in; start values in --tw-enter-* */
.promo{--tw-enter-opacity:0;--tw-enter-translate-y:1rem;animation:enter .5s ease-out both}
```

```css
/* Luma and Gridline: the reverse of enter, with --tw-exit-* */
.promo.is-leaving{--tw-exit-opacity:0;animation:exit .3s ease-in both}
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

### Motion without JavaScript

Animation is allowed.

**Scrolling marquee or ticker.** Repeat the row's content twice inside a flex track and slide the track by half its width.

```css
@keyframes vt-marquee{to{transform:translateX(-50%)}}
.ticker{overflow:hidden}
.ticker-track{display:flex;width:max-content;gap:3rem}
@media (prefers-reduced-motion:no-preference){.ticker-track{animation:vt-marquee 30s linear infinite}.ticker:hover .ticker-track{animation-play-state:paused}}
```

**Reveal as the shopper scrolls.** A scroll-driven animation, inside @supports so other browsers show the content at rest.

```css
@keyframes vt-rise{from{opacity:0;transform:translateY(40px)}to{opacity:1;transform:none}}
@media (prefers-reduced-motion:no-preference){@supports (animation-timeline:view()){.reveal{animation:vt-rise linear both;animation-timeline:view();animation-range:entry 0% cover 35%}}}
```

**Entrance on page load, one item after another.** One keyframe, a different animation-delay per item.

```css
@keyframes vt-in{from{opacity:0;transform:translateY(16px)}to{opacity:1;transform:none}}
@media (prefers-reduced-motion:no-preference){.hero-line{animation:vt-in .6s cubic-bezier(.2,.7,.2,1) both}.hero-line:nth-child(2){animation-delay:.12s}.hero-line:nth-child(3){animation-delay:.24s}}
```

**Hover and press feedback.** Transitions on transform, box-shadow and colour.

```css
.btn{transition:transform .2s,box-shadow .2s}
.btn:hover{transform:translate(-2px,-2px);box-shadow:4px 4px 0 var(--fh-foreground)}
.card img{transition:transform .4s}
.card:hover img{transform:scale(1.05)}
```

**Accordion or FAQ that opens and closes.** details and summary need no script; style the open state.

```css
details summary{cursor:pointer;list-style:none}
details summary::after{content:'+';float:right;transition:transform .2s}
details[open] summary::after{transform:rotate(45deg)}
```

**Carousel the shopper swipes.** A scroll-snap row; it scrolls by touch, trackpad or keyboard.

```css
.rail{display:flex;gap:1rem;overflow-x:auto;scroll-snap-type:x mandatory}
.rail>*{flex:0 0 min(80%,320px);scroll-snap-align:start}
```

**Floating or spinning decoration.** An infinite keyframe on a span or image; position: absolute inside a position: relative parent.

```css
@keyframes vt-float{50%{transform:translateY(-12px) rotate(4deg)}}
@media (prefers-reduced-motion:no-preference){.chip{animation:vt-float 6s ease-in-out infinite}}
```

**Sticky bar or heading.** position: sticky works everywhere. position: fixed works in a section that declares schema.fixed: true, not in a custom block.

```css
.buy-bar{position:sticky;top:0;z-index:5}
```

### Editor rules

- The theme's home sections (hero, collection, more-products, styles) appear at most once each. Hide one with set-section-visibility, take it off the page with remove-section, and put a removed one back with add-section { sectionId }. The home page has to show at least one visible section or block. Reorder with move-section (index among the page's sections; blocks keep their places) or set-layout.
- Every page is one ordered list of sections and custom blocks (the draft's document.layout): "home", "catalog", "product" and "page:<content page slug>", whose one section is its own body, "main". Place a block with position { placement: "before" | "after", target: a section id, a block id, "start" or "end", page?, slot? } on add-custom-block, move-custom-block or update-custom-block; reorder a whole page with set-layout { page, order }, which lists every section once and every block that should be there.
- The header.top slot (under the announcement bar, above the navigation) and the footer.top slot (above the footer) hold blocks too, on every page but the payment page: position { target: "start", slot: "header.top" }.
- sectionId without position still means right after that section and after the blocks already there, so blocks added one by one stay in the order they were sent. A block beside a hidden section keeps rendering.
- A block target that is not on the storefront, a section on the wrong page, or a set-layout that leaves out a block on that page is refused, naming the command.
- Take a theme section you are replacing off the page with remove-section (or hide it with set-section-visibility { sectionId, visible: false }), never with display:none in CSS: the builder does not apply the theme stylesheet, so a section hidden only by CSS shows up in the merchant's editor next to your design.
- One element of a placed section: update-section { instanceId, settings } for its text, set-element-style { instanceId, key, align, size, visible, spacing, order, offsetX, offsetY, width, height, layer, mobile, tablet } for its look, place and layer (any marked element; place per device), set-element-order { instanceId, group, order } for a group. Theme sections take areaId instead of instanceId.
- Hiding a hidden section, or showing a shown one, changes nothing and is not an error.
- At most 30 commands per apply_storefront_commands call. A batch applies whole or not at all, and a refusal names the command's position, type and target.
- At most 40 custom blocks per store.
- Text colours must stay readable on the background; set-colors refuses a pair that is too close.
- A logo or favicon is one of the store's own uploads; set_logo and set_favicon upload for you.

### Recreating a reference site

When a merchant says "make it look like this site", an agent works in this order:

1. Explore before building. Inspect desktop and phone layouts plus relevant states: delayed overlays, hover and keyboard menus, search, product options, quick add, cart, newsletter success/error and returning visits. Record URL, viewport, locale, elapsed time and visitor/consent conditions for each observation. Mark findings observed, inferred or unknown; one visit cannot establish hidden campaigns or A/B variants.
2. Follow dependencies when browser access permits: inspect rendered structure, computed styles, fonts, media and relevant script/app requests to distinguish theme behavior from app and commerce-backend behavior. Never capture or retain credentials, personal buyer data or unrelated network payloads. If access is unavailable, say so rather than inventing evidence.
3. Verify representative platform features through the actual save-and-render path before promising them. Documentation establishes a contract, not a complete shopper flow.
4. Preserve presentation, not someone else's protected implementation: reproduce layout, rhythm, type scale, colors and interaction intent using assets the merchant owns or is licensed to use. Do not copy source code, logos, product names, photography, reviews, ratings or claims without authorization. Use real merchant products and truthful copy; make every substitution explicit.
5. Start from nothing: set-theme { presetId: "blank-canvas" } takes the theme's home sections off the page and places starter sections that become the store's own (hero, product grid, text and image, questions, newsletter, header, footer). Change or replace them; they are ordinary sections.
6. Pick the closest theme and font pairing (list_themes, set-theme, set-fonts), then set-colors, the merchant's logo and navigation. Use an available open font by exact family name, or ask the signed-in owner to upload a properly licensed WOFF2 face; do not assume arbitrary remote font hosting is allowed.
7. Build every visual band as a section with save_section, never as a raw custom block: read describe_sections { parts: ["editable"] }, then mark each element that prints a text, image or link setting with data-fh-el and each reorderable flex or grid container with data-fh-group. The save refuses a section whose words the merchant could not select, and its answer lists what the builder can select. Put words and pictures in settings, never in the template.
8. Take theme sections you replace off the page with remove-section (add-section brings one back); set-section-visibility only hides one for now. Never hide or move a theme section with CSS: the builder does not apply the theme stylesheet to theme sections, so it would show the old section beside the new design.
9. Read get_custom_code `allowed` and describe_sections before implementation. Section templates support self-hosted video and responsive picture sources; video autoplay must follow the documented muted/loop/poster and reduced-motion rules. CSS handles presentation; agent-authored interactions belong in separate `behavior`, not legacy same-origin `script`.
10. Put each section's layout and motion in its own styles, built on the theme tokens (--fh-accent, --fh-foreground, --fh-font-heading), so it follows the merchant's colours; keep the theme stylesheet for page-wide things. A section that shows products reads a collection setting, so it follows the catalog.
11. For each advanced interaction, check the current `behavior` bridge contract and the store's Scripts in sections setting. Behaviors run only after the owner/admin enables it; builder editing previews are inert. Provide a useful no-behavior fallback, and never promise scratch reveals, campaign targeting or external widgets without a real implementation/integration.
12. Compare local preview and reference at identical viewports and observed states. Test keyboard and screen-reader semantics, reduced motion, errors, real product options, search, cart totals and newsletter outcomes as relevant. Track captured, implemented, locally verified, store-verified and published separately; list unknowns and gaps. Preview is not publication, and this playbook does not authorize publishing.

### Sections instead of blocks

When a layout repeats, or the merchant should edit it later in the builder, build a section instead of a raw HTML block: typed settings the builder turns into fields, a template and its own styles, saved once and placed as often as needed. Nothing inside a custom block can be selected or edited in the builder. The rules are in [sections](sections.md) and in `describe_sections`. Start from a theme (list_themes, set-theme), then adjust. The loop: describe_sections → save_section → preview_section → apply_storefront_commands → get_preview_link → publish_storefront.

Mark every element that prints a text, image or link setting with data-fh-el="<setting key>" and every reorderable flex or grid container with data-fh-group="<key>": the merchant selects, types into, aligns, resizes, hides, reorders and nudges each one. save_section refuses a new section that prints such a setting only outside marked elements. describe_sections { parts: ["editable"] } has the rules and an example.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `list_modules` | `storefront:read` | Discover what the storefront editor can do: the editor commands `apply_storefront_commands` accepts, as a JSON schema, and the storefront modules a section can be built from. |
| `get_storefront_draft` | `storefront:read` | The current storefront draft: { schemaVersion: 3, draft: { revision, document, publishedRevision, updatedAt, updatedBy }, liveVersion, publishedAt, modules }. |
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
| `list_storefront_versions` | `storefront:read` | What the storefront looked like before: { versions: [{ id, at, actor, kind, revision, sections, pages, changed, summary }], hasMore }. |
| `restore_storefront_version` | `storefront:write` | Put an old version back as the current DRAFT: { id } (from list_storefront_versions) → { restored, draft: { revision, document, publishedRevision }, published: false, dangling }. |
| `list_themes` | `storefront:read` | Every storefront look a store can wear, one list: { themes: [{ id, name, category, line, pages, preview, phone, licence, current?, demoUrl }], current }. |
| `get_theme` | `storefront:read` | One theme whole: { id } → { theme, preset }: its line, pages, pictures and what it needs, then its colours, typefaces, the sections it brings (summarised), every page's instances with their settings, the placeholder images, menus, starter pages and… |
| `list_presets` | `storefront:read` | The older name for list_themes, kept working: the same looks by their catalogue ids: { presets: [{ id, name, description, category, version, theme, ofTheme?, typefaces, colours, sections, pages, starterPages, licence }] }. |
| `get_preset` | `storefront:read` | The older name for get_theme, kept working. |
| `apply_preset` | `storefront:write` | The older way to pick a theme, kept working (set-theme is the one way now; apply_preset adds dryRun and reset). |
| `claim_showcase_look` | `storefront:write` | Puts the look of a showcase store from the library (list_themes `library`) on this store: { code } → { claimId, applied }. |
| `get_preview_link` | `storefront:read` | Two signed links that open the storefront whatever its launch mode, so you can look at a "Coming soon" or password-protected store without opening it or changing anything: { published: { url, expiresAt }, draft: { url, expiresAt }, launchMode }. |
| `get_launch_mode` | `settings:read` | Whether the live storefront is open or still private (live data): { access: { access: "open" \| "password" \| "coming_soon", message, hasPassword, updatedAt } }. |
| `set_launch_mode` | `settings:write` | Opens the live store to shoppers, or keeps it private. |

## Read more

- https://formahand.com/docs/storefront-themes
- https://formahand.com/docs/custom-code
- https://formahand.com/docs/modules-v2
- https://formahand.com/docs/agents
