# Sections: reusable parts of a page

A section is a reusable part of a storefront page: typed settings the builder turns into fields, a template, its own styles and presets. Build one instead of a raw HTML block when a layout repeats or the merchant should edit it later. Saving one changes the draft; nothing reaches shoppers until `publish_storefront`. Start from a theme (list_themes, set-theme), then adjust. The loop: describe_sections → save_section → preview_section → apply_storefront_commands → get_preview_link → publish_storefront.

## The loop

1. `describe_sections`: Read the rules: the package format, the setting types, the template language, the limits and the refusal codes.
2. `save_section`: Save your package { package }, with data-fh-el on every element that prints a text, image or link setting (describe_sections { parts: ["editable"] }). It is checked whole; a refusal names the section, setting, preset or template line to fix, and the answer lists what the merchant can select (editable). Saving again with changes makes a new version.
3. `preview_section`: Render one instance { id, settings } exactly as shoppers would see it, before it is on any page.
4. `apply_storefront_commands`: Place it on the draft: { revision, commands: [{ type: "add-section", sectionId, position, settings }] }; update-section, apply-section-preset, duplicate-section, set-section-visibility and remove-section change it later.
5. `get_preview_link`: Open the draft at desktop and phone width (about 390px).
6. `publish_storefront`: Publish { revision }. Nothing a shopper sees changes before this.

## Make it editable

In the builder a merchant selects one element inside a section and types into its text, or aligns, resizes, hides, reorders or nudges it, on desktop and on phones. They can only select what the template marks. Define the marks correctly and the merchant can change your section's words and positions without you; leave them out and the section can only be moved as a whole.

- Put data-fh-el="<setting key>" on the element that prints each text, textarea, rich text, image, link or url setting: the h2 that prints settings.heading, the img that shows settings.image, the a that uses settings.ctaLink. The mark's own attributes count as inside it, so alt="{{ settings.imageAlt }}" on a marked img and href on a marked a need nothing more.
- A button or link whose label is a setting of another name, or a card that moves as one, is declared in schema.elements { key, label, type: text | richtext | image | button | link | group, setting?, style?, group? } and marked with data-fh-el="<key>".
- Put data-fh-group="<key>" on a container whose marked direct children the merchant may reorder, and make that container display:flex or display:grid in the styles; order then moves them. A group that is neither gets a note, because reordering would move nothing.
- A mark is a fixed key, never an expression, and names a setting or a declared element; anything else is refused with its template line (element_unknown, element_invalid). At most 40 elements per section.
- save_section refuses a new package that prints a text, image or link setting only outside marked elements: elements_unmarked when nothing is marked, and element_unmarked for each setting, with the line of its first print. Sections already saved keep working, and rollback_section, fork_section and copies between environments are not held to it.
- The one way out, written on purpose: list the setting in schema.fieldOnly (for example fieldOnly: ["imageAlt"] when the text is only ever an attribute of an unmarked element). The merchant still edits it in the section's fields; it just cannot be selected on the page.
- List items ({% for item in settings.questions %}) print item fields, not settings: they are edited in the section's fields and are outside the rule. Settings a tag only tests ({% if settings.image %}) or hands to a widget are not prints.
- Styles for a marked element should not depend on its position (no :first-child on marked children, no absolute placement of a group's members): the merchant's order and nudges must still read well. Test at phone width (390px).

What `save_section` and `preview_section` answer about it:

| Field | Means |
| --- | --- |
| `editable` | Every element the builder can select: { key, type, label, setting?, group?, style }. style lists the knobs offered (align, size, visible, spacing); order, offset, width, height and layer are offered on every element. |
| `builder` | One sentence to relay to the merchant: what they can select and change in this section. |
| `notes` | Advice, never a refusal: a declared element no template line marks, a group that is not flex or grid, a setting printed outside its mark as well. |

A complete package that passes, a hero whose four elements sit in one group the merchant can reorder:

```json
{
  "schemaVersion": 1,
  "id": "hero-stack",
  "name": "Hero",
  "description": "A heading, a line of text, a button and a picture, stacked.",
  "category": "hero",
  "version": "1.0.0",
  "origin": "store",
  "schema": {
    "settings": [
      {
        "key": "heading",
        "type": "text",
        "label": "Heading",
        "default": "Made by hand, made to last",
        "maxLength": 120,
        "inlineEditable": true
      },
      {
        "key": "body",
        "type": "textarea",
        "label": "Text",
        "default": "Small batches from our workshop.",
        "maxLength": 400,
        "inlineEditable": true
      },
      {
        "key": "ctaLabel",
        "type": "text",
        "label": "Button label",
        "default": "Shop now",
        "maxLength": 40
      },
      {
        "key": "ctaLink",
        "type": "link",
        "label": "Button link"
      },
      {
        "key": "image",
        "type": "image",
        "label": "Picture"
      },
      {
        "key": "imageAlt",
        "type": "text",
        "label": "Picture description",
        "maxLength": 160
      },
      {
        "key": "background",
        "type": "color",
        "label": "Background"
      }
    ],
    "elements": [
      {
        "key": "cta",
        "label": "Button",
        "type": "button",
        "setting": "ctaLabel"
      }
    ],
    "maxInstancesPerPage": 2,
    "pages": [
      "home",
      "page"
    ]
  },
  "template": "<div class=\"hero\" data-fh-group=\"stack\">\n  <h2 data-fh-el=\"heading\">{{ settings.heading }}</h2>\n  <p data-fh-el=\"body\">{{ settings.body }}</p>\n  {% if settings.ctaLabel %}<a class=\"hero__button\" data-fh-el=\"cta\" href=\"{{ settings.ctaLink | default: routes.catalog }}\">{{ settings.ctaLabel }}</a>{% endif %}\n  {% if settings.image %}<img class=\"hero__image\" data-fh-el=\"image\" src=\"{{ settings.image | image_url: 1200 }}\" alt=\"{{ settings.imageAlt }}\">{% endif %}\n</div>\n",
  "styles": ":scope{background:var(--fh-s-background,var(--fh-background,transparent));color:var(--fh-foreground,inherit)}\n.hero{display:flex;flex-direction:column;align-items:center;gap:1rem;max-width:760px;margin-inline:auto;padding:clamp(2.5rem,8vw,5rem) 1.25rem;text-align:center}\n.hero>*{margin:0}\n.hero h2{font-family:var(--fh-font-heading,inherit);font-size:clamp(2rem,6vw,3.5rem);line-height:1.1}\n.hero__button{display:inline-block;padding:.8rem 1.6rem;border-radius:999px;background:var(--fh-accent,currentColor);color:var(--fh-on-accent,#fff);text-decoration:none}\n.hero__image{width:100%;height:auto;border-radius:12px}\n"
}
```

Changing one element of a placed instance:

- Text, images and links live in the instance's settings: { type: "update-section", instanceId, settings: { heading: "New words" } }.
- { type: "set-element-style", instanceId, key, align?: "left" | "center" | "right", size?: "s" | "m" | "l" | "xl", visible?, spacing?: 0-4, order?: -20 to 20, offsetX?: -400 to 400 px or { value, unit: "cqw" } -100 to 100 (a share of the section width, two decimals), offsetY?: -400 to 400 px, width?, height?: { value, unit: "px" } 8 to 2000, { value, unit: "%" } 1 to 100 of the parent or { value, unit: "cqw" } 0.5 to 100 of the section width, layer?: -20 to 20 (in front of or behind the other elements of the section), mobile?: { order?, offsetX?, offsetY?, align?, visible?, width?, height?, layer? }, tablet?: the same } changes one element of a placed instance; null takes a knob off. Geometry (order, offsets, width, height, layer) is per device: the top level applies from 1024px, tablet from 641px to 1023px, mobile at 640px and below, and a device without its own shows the section layout; align and visible a device leaves out come from desktop. Shares of the section width scale with it. A moved element may cover its neighbours; it never rises above another section or the buy and consent controls. Width never exceeds the parent; text, buttons and groups use height as a minimum; an image keeps its proportions unless both are set (then it is cropped to fill).
- { type: "set-element-order", instanceId, group, order: ["cta", "heading", "body"], mobile? } numbers a group's elements in one step.
- The theme's own sections are edited through their areas: set-element and set-element-style with areaId.

```json
[
  {
    "type": "update-section",
    "instanceId": "hero-stack",
    "settings": {
      "heading": "New season"
    }
  },
  {
    "type": "set-element-style",
    "instanceId": "hero-stack",
    "key": "heading",
    "size": "xl",
    "align": "left",
    "mobile": {
      "align": "center"
    }
  },
  {
    "type": "set-element-order",
    "instanceId": "hero-stack",
    "group": "stack",
    "order": [
      "image",
      "heading",
      "body",
      "cta"
    ]
  },
  {
    "type": "set-element-style",
    "instanceId": "hero-stack",
    "key": "cta",
    "offsetY": 12
  }
]
```

## The package

| Field | What it is |
| --- | --- |
| `schemaVersion` | Always 1: the version of the section format. |
| `id` | 2 to 40 lower-case letters, digits and single hyphens, starting with a letter. Built-in ids and layout words (hero, main, start, page and the like) are taken. |
| `name` | What the builder calls it, up to 60 characters. |
| `description` | One or two sentences for the builder's section picker, up to 400 characters. |
| `category` | One of hero, text, products, collections, media, social-proof, promotion, layout, custom. |
| `version` | major.minor.patch, such as 1.2.0. Leave it out when saving and the next version is chosen from what changed. |
| `author` | { kind, name }: kind is "owner" for the store's own sections. Optional. |
| `origin` | "store" for the store's own; add-on sections are read-only. Optional. |
| `app` | Only on an add-on's sections: the add-on that ships it. |
| `forkedFrom` | Set for you on a copy made with fork_section: { id, version, origin }. |
| `rendering` | Always "template" for your own sections. Optional. |
| `schema` | { settings, groups, pages, slots, maxInstancesPerPage, elements, fieldOnly, sticky, body }: the settings the builder and agents fill in, where the section may go and how often; elements and fieldOnly: part editable; sticky: true keeps a header section (slots include header) at the top of the window; body: a page kind makes it that page's body (the *-page parts). Up to 40 settings in up to 8 groups; pages from home, catalog, product, page, cart, blog, post (default home and page); slots from header.top, footer.top, header, footer (default none); maxInstancesPerPage 1 to 60 (default 10). |
| `template` | The section's markup with {{ }} output, {% if %} and {% for %}, up to 32 KB, in the template language described below. |
| `styles` | CSS for this section only, up to 16 KB. Every rule is confined to the section; :scope or & means the section itself. Colour, number, range, select, on/off and image settings arrive as var(--fh-s-<setting in kebab case>). |
| `script` | Optional legacy same-origin module; new or changed source requires a signed-in person. Agents and apps use separate `behavior` instead. |
| `behavior` | Optional separate section behaviour worker; `export default function mount(api)`. Bounded UI, overlay, commerce and section storage bridge. See `describe_sections` part `behaviors`. |
| `artwork` | Optional: { name: svg text }, up to 32 of 32 KB, for the icon tag. |
| `presets` | Up to 12 named starting points: { id, name, description?, settings } with some of the settings filled in. |

## Setting types

| Type | What it holds | Its own fields |
| --- | --- | --- |
| `text` | One line of text, up to 600 characters. | `default`, `maxLength`, `placeholder`, `required`, `inlineEditable` |
| `textarea` | Plain text over several lines, up to 4,000 characters. | `default`, `maxLength`, `placeholder`, `required`, `inlineEditable` |
| `richtext` | Formatted text: paragraphs, bold, italic, links, lists and small headings, up to 8,000 characters. | `default`, `maxLength`, `required`, `inlineEditable` |
| `number` | A number, with an optional minimum, maximum, step and unit. | `default`, `min`, `max`, `step`, `integer`, `unit` |
| `range` | A number picked on a slider between a minimum and a maximum. | `min` (required), `max` (required), `step` (required), `default` (required), `unit` |
| `boolean` | On or off. | `default` |
| `select` | One choice from a list of 2 to 24 options. | `options` (required), `default` (required), `display` |
| `color` | A colour as #rrggbb, or #rrggbbaa with an opacity pair, or empty for the theme's own. | `default` |
| `font` | A family from the font list (part fonts) and the weights used, or empty for the theme's own. | `default`, `weights`, `italic` |
| `image` | An image from your store's media library. | `default`, `required` |
| `link` | Where a button or link goes: a page on your store, an https:// address, mailto: or tel:. | `default` |
| `product` | One of your products, by its handle. | `default` |
| `collection` | One of your collections, by its handle. | `default` |
| `url` | An outside https:// address. | `default` |
| `list` | Repeated items of the same shape, such as questions or testimonials, up to 24. | `fields` (required), `itemLabel`, `minItems`, `maxItems`, `default` |
| `html` | Your own HTML (built-in custom HTML section only). | `default` |
| `css` | Your own CSS for this section (built-in custom HTML section only). | `default` |

Every setting also has `key` (letters, digits and `_`, starting with a letter; the template reads it as `settings.<key>`), `label`, and optionally `help` and `group`. `inlineEditable` lets people type a text, textarea or rich text setting on the page itself in the builder.

## Templates

Ordinary HTML plus `{{ }}`, `{% if %}` and `{% for %}` over the objects below; `describe_sections` answers the same reference as data.

### Tags

| Tag | Syntax | What it does | Example | Renders |
| --- | --- | --- | --- | --- |
| output | `{{ value \| filter: argument }}` | Prints a value, escaped. Chain up to 6 filters with \|. | `<p>{{ settings.tagline }}</p>` | `<p>Cups &amp; &lt;saucers&gt;</p>` |
| if | `{% if condition %}…{% elsif condition %}…{% else %}…{% endif %}` | Shows a part only when a condition holds. elsif and else are optional; conditions may use filters, the operators, and, or and not. | `{% if settings.showBadge %}<span>Sale</span>{% else %}<span>New</span>{% endif %}` | `<span>Sale</span>` |
| for | `{% for item in list \| reverse limit: 8 offset: 0 %}…{% else %}…{% endfor %}` | Repeats a part for each item of a list; \| reverse (optional) turns it round first. limit (a whole number or a number setting) is at most 50, and a loop without one stops there; offset skips items first. The else part shows when the list is empty. Inside, loop tells you where you are. | `{% for product in settings.shelf.products limit: 8 %}{{ product.title }};{% endfor %}` | `Blue mug;Side plate;` |
| for-else | `{% for item in list %}…{% else %}…{% endfor %}` | What to show when the list has nothing in it. | `{% for question in settings.questions %}{{ question.question }}{% else %}Nothing yet{% endfor %}` | `Nothing yet` |
| comment | `{# a note #}` | A note for whoever edits the template; never printed. | `A{# not shown #}B` | `AB` |
| widget | `{% widget "newsletter" label: "…" placeholder: "…" button: "…" note: "…" %}` | Places a store control: newsletter, search, cart-button, account, quick-add (part widgets). Arguments are quoted text or a setting (empty takes the default). Between elements, outside loops, at most 4. | `{% widget "account" label: "Sign in" %}` | `<a class="fh-widget fh-widget-account" data-fh-widget="account" href="/account"><span class="fh-widget-label">Sign in</span></a>` |
| icon | `{% icon "star" size: 20 label: "…" %}` | Draws an icon (part icons) in the text colour; the name is quoted text or a setting. size 12 to 96 (24). A label makes it announced. Between elements only. "art:<name>" draws its artwork; size is the width, 8 to 1600. | `{% icon "check" size: 16 %}` | `<svg class="fh-icon fh-icon-check" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" focusable="false"><path d="M5 12l5 5l10 -10"/></svg>` |

Operators: `==` (equal), `!=` (not equal), `>` (greater than (numbers)), `<` (less than (numbers)), `>=` (greater than or equal (numbers)), `<=` (less than or equal (numbers)), `contains` (a list holds a value, or a text holds another text), `and` (both are true), `or` (either is true), `not` (the opposite, written before a value).

Everything printed with {{ }} is HTML-escaped (& < > " and ' become entities), so text from settings or products never becomes markup. | raw prints a value as HTML; the section's markup, raw parts included, keeps a custom HTML block's rules (no scripts, event handlers or inline styles; links only to your store, https, mailto: and tel:).

In a condition, false, empty (null), an empty text, 0 and an empty list count as false; everything else counts as true. Printing empty prints nothing; printing true or false prints the word; printing an image, product or collection prints nothing (print one of its fields instead).

Text outside {{ }} and {% %} is printed exactly as written, spaces and new lines included.

### Objects

**settings** (every section): This instance's settings by key. Product, collection and image settings are objects (empty when none is chosen); a list setting is a list of items.

| Field | Type | Meaning |
| --- | --- | --- |
| <key> | setting | One setting's value, typed by its definition. |

Example: `<h2>{{ settings.heading }}</h2>` renders `<h2>Winter sale</h2>`.

**section** (every section): This instance and where it sits.

| Field | Type | Meaning |
| --- | --- | --- |
| id | text | The instance id, unique on the page; an id attribute shows as fhc-<id>. |
| type | text | The section's id. |
| version | text | The section version being shown. |
| index | number | Its position among the page's sections, from 1. |
| first | boolean | True for the page's first section. |
| last | boolean | True for the page's last section. |

Example: `<div id="{{ section.id }}">` renders `<div id="promo">`.

**store** (every section): Your store.

| Field | Type | Meaning |
| --- | --- | --- |
| name | text | The store's name. |
| headline | text | The store headline from the theme settings. |
| intro | text | The store introduction from the theme settings. |
| currency | text | The store currency, such as GBP. |
| locale | text | The store language and region, such as en-GB. |
| logo | image? | The store logo, when set. |
| socialProfiles | list<socialProfile> | The store's social profiles. |
| contact | contact | How to reach the store, as its footer shows it. |

Example: `{{ store.name }}` renders `Hallo Ceramics`.

**page** (every section): The page being shown.

| Field | Type | Meaning |
| --- | --- | --- |
| kind | text | home, catalog, product, cart or page (a content page). |
| title | text | The page's title. |
| slug | text | A content page's slug; empty elsewhere. |
| url | url | The page's address. |
| product | product? | The product, on a product page. |
| collection | collection? | The collection, on a collection's listing. |

Example: `{% if page.kind == "home" %}Welcome{% endif %}` renders `Welcome`.

**cart** (every section): A summary of the shopper's cart.

| Field | Type | Meaning |
| --- | --- | --- |
| itemCount | number | Units in the cart. |
| subtotal | money | The cart subtotal in minor units. |
| currency | text | The cart currency. |

Example: `{{ cart.itemCount }} items` renders `2 items`.

**request** (every section): The address being shown.

| Field | Type | Meaning |
| --- | --- | --- |
| path | text | The path of the page, such as /collections/mugs. |

Example: `{{ request.path }}` renders `/`.

**routes** (every section): Addresses of your store's own pages, so links keep working on any domain.

| Field | Type | Meaning |
| --- | --- | --- |
| home | url | The home page. |
| catalog | url | Every product. |
| cart | url | The cart. |
| search | url | Search results; a form sends q. |
| account | url | The shopper's account: sign-in and orders. |

Example: `<a href="{{ routes.catalog }}">Shop</a>` renders `<a href="/collections/all">Shop</a>`.

**navigation** (every section): The store's menus: the merchant's links, or the theme's when none are set.

| Field | Type | Meaning |
| --- | --- | --- |
| header | list<menuItem> | The header menu. |
| footer | list<menuItem> | The footer menu. |
| tracking | trackingLink? | Track your order, for a footer to print where it likes; a template that reads it gets footer without it. Empty when footer links there. |

Example: `{% for link in navigation.header %}<a href="{{ link.href }}">{{ link.label }}</a>{% endfor %}` renders `<a href="/collections/all">Shop</a><a href="/pages/about">About</a>`.

**blog** (every section): The store's latest published posts; empty without posts.

| Field | Type | Meaning |
| --- | --- | --- |
| url | url | The blog page. |
| posts | list<post> | Up to 12, newest first. |

Example: `{% for post in blog.posts limit: 3 %}<a href="{{ post.url }}">{{ post.title }}</a>{% endfor %}` renders `<a href="/blog/first-firing">The first firing</a>`.

**now** (every section): Today, in UTC.

| Field | Type | Meaning |
| --- | --- | --- |
| year | number | The year, such as 2026. |
| date | date | Today, such as 2026-09-27. |

Example: `&copy; {{ now.year }} {{ store.name }}` renders `&copy; 2026 Hallo Ceramics`.

**builder** (every section): Whether the page is drawn in the builder.

| Field | Type | Meaning |
| --- | --- | --- |
| editing | boolean | True only in the builder: draw a placeholder for an empty state. False for shoppers and preview links. |

Example: `{% if builder.editing %}Pick products{% else %}Shop{% endif %}` renders `Shop`.

**loop** (inside {% for %}): Where the innermost for loop is.

| Field | Type | Meaning |
| --- | --- | --- |
| index | number | The item's position, from 1. |
| index0 | number | The item's position, from 0. |
| first | boolean | True for the first item. |
| last | boolean | True for the last item. |
| length | number | How many items the loop shows. |
| itemId | text | Over a list setting, the item's id (tiles-2) for data-fh-item; styles read its fields as var(--fh-i-<field>). |

Example: `{% for product in settings.shelf.products %}{{ loop.index }}.{{ product.title }} {% endfor %}` renders `1.Blue mug 2.Side plate `.

**image** (image settings, store.logo, product.image, collection.image): An image from your store's media library.

| Field | Type | Meaning |
| --- | --- | --- |
| src | url | The image's address on your store. |
| alt | text | Its description for screen readers; empty when none was given. |
| width | number? | Its width in pixels, when known. |
| height | number? | Its height in pixels, when known. |

Example: `<img src="{{ settings.image \| image_url: 800 }}" alt="{{ settings.image.alt }}">` renders `<img src="/media/products/mug.jpg?width=800" alt="A blue mug">`.

**product** (product settings, page.product, collection.products): One of your products.

| Field | Type | Meaning |
| --- | --- | --- |
| title | text | The product's name. |
| handle | text | Its handle, the last part of its address. |
| url | url | Its page on your store. |
| price | money | Its price in minor units; print it with \| money. |
| compareAtPrice | money? | The earlier price shown struck through, when set. |
| priceFrom | money? | The lowest option price, when the product has options. |
| hasVariants | boolean | True when shoppers pick options such as size. |
| available | boolean | True when it can be bought now. |
| onSale | boolean | True when compareAtPrice is higher than price. |
| isNew | boolean | True while the store's New badge applies to it. |
| image | image? | Its main image. |
| images | list<image> | Every image, main image first. |
| description | text | Its description as plain text. |
| tags | list<text> | Its tags. |
| createdAt | date | When it was added; print it with \| date. |
| discountPercent | number | The sale as a whole percentage, as the theme's badge shows it; 0 when not on sale. |
| inventory | number? | Units in stock; empty when unknown. |
| lowStock | boolean | True when in stock with 10 or fewer left. |

Example: `{{ settings.featured.title }}: {{ settings.featured.price \| money }}` renders `Blue mug: £24.00`.

**collection** (collection settings, page.collection): One of your collections. A collection setting set to "all" is every product, newest first, and its url is routes.catalog.

| Field | Type | Meaning |
| --- | --- | --- |
| title | text | The collection's name. |
| handle | text | Its handle. |
| url | url | Its page on your store. |
| description | text | Its description as plain text. |
| image | image? | Its image, or its first product's. |
| productCount | number | How many products it holds. |
| products | list<product> | Its products in the collection's order, at most 50. |

Example: `{{ settings.shelf.title }} ({{ settings.shelf.productCount }})` renders `Mugs (2)`.

**menuItem** (navigation.header, navigation.footer): One link of a menu.

| Field | Type | Meaning |
| --- | --- | --- |
| label | text | The link's text. |
| href | url | Where it goes. |
| current | boolean | True when it points at the page being shown. |

Example: `{% for link in navigation.header %}{% if link.current %}<span aria-current="page">{{ link.label }}</span>{% endif %}{% endfor %}` renders `<span aria-current="page">Shop</span>`.

**trackingLink** (navigation.tracking): The order tracking link; print it as a marked element.

| Field | Type | Meaning |
| --- | --- | --- |
| label | text | Its text. |
| url | url | The tracking page. |
| current | boolean | On that page. |

Example: `{% if navigation.tracking %}<a data-fh-el="tracking" href="{{ navigation.tracking.url }}">{{ navigation.tracking.label }}</a>{% endif %}` renders `<a data-fh-el="tracking" href="/orders/track">Track your order</a>`.

**post** (blog.posts): A published post.

| Field | Type | Meaning |
| --- | --- | --- |
| title | text | Its title. |
| url | url | Its page. |
| image | image? | Its cover. |
| excerpt | text | Its summary. |
| date | date | Published on. |

Example: `{% for post in blog.posts %}<p>{{ post.date \| date: "long" }}: {{ post.excerpt }}</p>{% endfor %}` renders `<p>2 September 2026: How the new glaze came out.</p>`.

**contact** (store.contact): The store's contact details from its store profile; each part is empty until the merchant gives it.

| Field | Type | Meaning |
| --- | --- | --- |
| legalName | text | The business's legal name. |
| email | text | The support email address. |
| phone | text | The phone number. |
| address | address | The postal address. |

Example: `{% if store.contact.email %}<a href="mailto:{{ store.contact.email }}">{{ store.contact.email }}</a>{% endif %}` renders `<a href="mailto:hello@hallo.example">hello@hallo.example</a>`.

**address** (store.contact.address): A postal address; all of it is empty until a first line and a country are given.

| Field | Type | Meaning |
| --- | --- | --- |
| line1 | text | The first line. |
| line2 | text | The second line. |
| city | text | The town or city. |
| region | text | The county, state or region. |
| postalCode | text | The postcode. |
| country | text | The country, as the store profile holds it. |

Example: `{{ store.contact.address.city }} {{ store.contact.address.postalCode }}` renders `Leeds LS1 1AA`.

**socialProfile** (store.socialProfiles): One of the store's social profiles.

| Field | Type | Meaning |
| --- | --- | --- |
| platform | text | instagram, tiktok, x and so on: also its icon's name. |
| label | text | Its name, such as Instagram. |
| url | url | The profile's https address. |

Example: `{% for profile in store.socialProfiles %}<a href="{{ profile.url }}">{{ profile.label }}</a>{% endfor %}` renders `<a href="https://instagram.com/hallo">Instagram</a>`.

### Filters

| Filter | Takes | Arguments | What it does | Example | Renders |
| --- | --- | --- | --- | --- | --- |
| upper | text | none | Capital letters. | `{{ settings.heading \| upper }}` | `WINTER SALE` |
| lower | text | none | Small letters. | `{{ settings.heading \| lower }}` | `winter sale` |
| truncate | text | length (optional) | Shortens text to a length, ending with … when it was cut. | `{{ store.headline \| truncate: 10 }}` | `Stoneware…` |
| money | money | format (optional) | A price in minor units in the store currency, written exactly as the theme writes prices ($42.00, £24.00). | `{{ cart.subtotal \| money }}` | `£48.00` |
| date | date | format (optional) | A date written out. | `{{ settings.featured.createdAt \| date: "long" }}` | `20 September 2026` |
| image_url | image, product or collection | width | The address of an image (a product's or collection's main image) for a given width. Empty when there is no image. | `{{ settings.featured \| image_url: 400 }}` | `/media/products/mug.jpg?width=400` |
| url | product, collection or text | none | The page address of a product or collection; a link text passes through when it is a safe link. | `{{ settings.featured \| url }}` | `/products/blue-mug` |
| default | anything | fallback | A fallback for an empty value. | `{{ store.logo.alt \| default: store.name }}` | `Hallo Ceramics` |
| size | list or text | none | How many items a list holds, or characters a text holds. | `{{ settings.shelf.products \| size }}` | `2` |
| join | list of text | separator (optional) | One text from a list. | `{{ settings.featured.tags \| join: ", " }}` | `mugs, blue` |
| first | list | none | The first item. Without a loop, a path takes one: list.first, list.last, list.0. | `{{ settings.featured.tags \| first }}` | `mugs` |
| last | list | none | The last item of a list. | `{{ settings.featured.tags \| last }}` | `blue` |
| reverse | list | none | The list backwards. | `{{ settings.featured.tags \| reverse \| join: ", " }}` | `blue, mugs` |
| fill | text | value | Puts a value where the text says {n}. | `{{ "Only {n} left" \| fill: settings.discount }}` | `Only 20 left` |
| initials | text | none | A name's initials (first two words), in capitals. | `{{ store.name \| initials }}` | `HC` |
| plus | number | amount | Adds a number. | `{{ settings.discount \| plus: 5 }}` | `25` |
| minus | number | amount | Takes a number away. | `{{ settings.discount \| minus: 5 }}` | `15` |
| times | number | factor | Multiplies. | `{{ settings.discount \| times: 2 }}` | `40` |
| divided_by | number | divisor | Divides, keeping decimals; add \| round for a whole number. | `{{ settings.featured.price \| divided_by: 100 }}` | `24` |
| round | number | places (optional) | Rounds to the nearest. | `{{ settings.discount \| divided_by: 3 \| round: 1 }}` | `6.7` |
| floor | number | none | Rounds down to a whole number. | `{{ settings.discount \| divided_by: 3 \| floor }}` | `6` |
| ceil | number | none | Rounds up to a whole number. | `{{ settings.discount \| divided_by: 3 \| ceil }}` | `7` |
| abs | number | none | The number without its sign. | `{{ settings.discount \| minus: 25 \| abs }}` | `5` |
| raw | text | none | Prints the value as HTML instead of escaping it. It still follows the same rules as a custom HTML block, so it can format text but never run code. | `{{ settings.note \| raw }}` | `<strong>Free</strong> delivery` |

### Widgets

| Widget | What it is | Arguments |
| --- | --- | --- |
| newsletter | The store's newsletter sign-up: an email field and a button. Addresses are confirmed by email before anything is sent. | `label` What screen readers call the form. (Newsletter); `placeholder` The hint inside the email field. (Email address); `button` The button's text. (Subscribe); `note` The consent line under the form. (Marketing email only. Unsubscribe any time.); `fieldLabel` A visible label for the email field (fh-widget-field-label, after the field); none when empty.; `noteLinkLabel` A link in the note, such as your privacy page: its text; it goes where the note says {link}, or after the note.; `noteLinkHref` Where that link goes: a page of your store (/pages/privacy).; `noteLink2Label` A second link in the note, at {link2}, or after the first.; `noteLink2Href` Where the second link goes: a page of your store. |
| search | A search box that opens the store's search results; it shows what the shopper searched for. | `label` What screen readers call the search. (Search); `placeholder` The hint inside the field. (Search products); `button` The button's text. (Search) |
| cart-button | A link to the cart with the number of items in it (shown when there are any). | `label` The link's text. (Cart); `icon` An icon name to show before the text; none when empty. |
| account | A link to the shopper's account: sign-in, orders and addresses. | `label` The link's text. (Account); `icon` An icon name to show before the text; none when empty. |
| quick-add | One click adds a named product (or one of its options) to the cart, with the store's own add form: a product setting or a handle, never a price. The shopper stays on the page and the cart count updates; on the cart page the page reloads with the new totals; without a script the form still posts. A product with options needs variant:, else it links to its page. | `product` The product: a product setting (settings.gift), the page's product, or a handle in quotes.; `variant` The option to add, by id, for a product with options (such as product.selectedVariant.id).; `quantity` How many, 1 to 10. (1); `label` The button's text. (Add to cart); `addedLabel` What the button's status says once it is added. (Added); `chooseLabel` The link's text for a product with options and no variant. (Choose options); `soldOutLabel` What it says when nothing is left. (Sold out); `el` An element key: the merchant can select, move and hide it in the builder. |

### Recipes

**search**: The search widget with its own hint; on the search results page it shows what was searched for.

Example: `{% widget "search" placeholder: "Find a mug" %}` renders `<form class="fh-widget fh-widget-search" data-fh-widget="search" method="get" action="/search" role="search" aria-label="Search"><input class="fh-widget-input" type="search" name="q" maxlength="80" autocomplete="off" placeholder="Find a mug" aria-label="Find a mug"><button class="fh-widget-button" type="submit">Search</button></form>`.

**social-icons**: Each social profile as its platform's icon, announced by its name.

Example: `{% for profile in store.socialProfiles %}<a href="{{ profile.url }}">{% icon profile.platform size: 20 label: profile.label %}</a>{% endfor %}` renders `<a href="https://instagram.com/hallo"><svg class="fh-icon fh-icon-instagram" xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" role="img" aria-label="Instagram" focusable="false"><path d="M4 8a4 4 0 0 1 4 -4h8a4 4 0 0 1 4 4v8a4 4 0 0 1 -4 4h-8a4 4 0 0 1 -4 -4l0 -8"/><path d="M9 12a3 3 0 1 0 6 0a3 3 0 0 0 -6 0"/><path d="M16.5 7.5v.01"/></svg></a>`.

**item-colours**: A list whose items each have their own colour: give the list a colour field, put data-fh-item on each item's element, and read the field in styles as var(--fh-i-<field>). No style attribute needed.

Example: `{% for tile in settings.tiles %}<li data-fh-item="{{ loop.itemId }}">{{ tile.title }}</li>{% endfor %}` renders `<li data-fh-item="tiles-1">Mugs</li><li data-fh-item="tiles-2">Plates</li>`.

**fonts**: A font setting names a family from the font list; the section loads only the weights the setting declares and exposes the family as var(--fh-s-<setting>), so styles write font-family:var(--fh-s-heading-font,inherit). Printing it gives the family's name.

Example: `<p>Set in {{ settings.headingFont }}</p>` renders `<p>Set in Fraunces</p>`.

**header**: A section with slots: ["header"] (or "footer") placed in that slot replaces the theme's own header (or footer) on every page but the payment page. Build it from store.logo, navigation, the cart-button, search and account widgets, and details and summary for a menu that opens on small screens. schema.sticky: true keeps a header at the top of the window while the page scrolls.

Example: `<a href="{{ routes.home }}">{{ store.name }}</a>{% widget "cart-button" %}` renders `<a href="/">Hallo Ceramics</a><a class="fh-widget fh-widget-cart-button" data-fh-widget="cart-button" href="/cart" aria-label="Cart (2)"><span class="fh-widget-label">Cart</span><span class="fh-widget-count" aria-hidden="true">2</span></a>`.

**main-heading**: A section may hold one h1, outside loops, for the page's main heading (a hero's). The page keeps the first h1 it meets and shows any later one as an h2 with the same attributes; on catalog, product and content pages the theme's own title is the h1, so a section's h1 shows as an h2 there.

Example: `<h1 data-fh-el="heading">{{ settings.heading }}</h1>` renders `<h1 data-fh-el="heading">Winter sale</h1>`.

**one-item**: One item of a list without a loop, such as the first product of a collection as the page's h1: .first, .last or a position from 0 (up to 99) as a step of the path. An item that is not there prints nothing.

Example: `<h1>{{ settings.shelf.products.first.title }}</h1><p>{{ settings.shelf.products.1.title }}</p>` renders `<h1>Blue mug</h1><p>Side plate</p>`.

**all-products**: A collection setting set to "all" (a default, a preset or the merchant's choice) is the whole catalogue, newest first, capped like any collection; loop over its products with limit: and offset:.

Example: `<a href="{{ settings.everything \| url }}">{{ settings.everything.title }}</a>` renders `<a href="/collections/all">All products</a>`.

**sale-badge**: The theme's sale pill, from the product's own numbers.

Example: `{% if settings.featured.onSale %}<span class="pill">-{{ settings.featured.discountPercent }}%</span>{% endif %}` renders `<span class="pill">-20%</span>`.

**field-label**: fieldLabel gives the newsletter's email field a visible label (fh-widget-field-label, after the field inside fh-widget-field). Without a placeholder of its own the field's placeholder is then a space, so styles can float the label: .fh-widget-input:not(:placeholder-shown)+.fh-widget-field-label.

Example: `{% widget "newsletter" fieldLabel: "Email" %}` renders `<form class="fh-widget fh-widget-newsletter" data-fh-widget="newsletter" method="post" action="/api/newsletter" aria-label="Newsletter"><span class="fh-widget-field"><input class="fh-widget-input" id="fh-promo-w1" type="email" name="email" required maxlength="254" autocomplete="email" placeholder=" "><label class="fh-widget-field-label" for="fh-promo-w1">Email</label></span><input class="fh-widget-trap" name="company" tabindex="-1" autocomplete="off" aria-hidden="true"><button class="fh-widget-button" type="submit">Subscribe</button><p class="fh-widget-note">Marketing email only. Unsubscribe any time.</p></form>`.

### Not available

- Other files: no include, render, layout or import.
- Code: no scripts, event handlers or javascript: links. Use details and summary, or a section script.
- Forms, inputs and svg: use {% widget %} and {% icon %}. Section templates may write buttons for interaction controls: type=button, class, aria and inert data markers survive; submit behaviour and form attributes are refused.
- Inline styles: removed. Use the section's styles and CSS variables.
- Network: a template fetches nothing; it sees only the objects here.
- Variables: no assign or capture. Use filters in place, and list.first for one item.
- Other stores' data, the shopper's personal details, orders and customer records.

## Styles

A section's styles are CSS under the same rules as a block's CSS in [custom code](https://formahand.com/docs/custom-code): the same at-rules are kept, the same things are removed. Every rule is confined to the section, and `:scope` or `&` means the section itself:

```css
:scope{background:var(--fh-s-background);color:var(--fh-s-foreground)}
.promo a{color:inherit;text-decoration:underline}
```

`:scope` and `&` at the start of a selector stand for the section itself, and what follows keeps its meaning: `:scope > .row` (its own rows), `:scope *` and `:scope a` (anything, or any link, inside it), `&.is-dark` and `:scope:hover` (the section in that state). A list inside `:is()`, `:where()`, `:not()` or `:has()` keeps its own commas. Two things stay out of section styles: `@container` queries (use `@media`), and arrows that move a carousel.

**Fixed layers.** A section whose schema says `fixed: true` may use `position: fixed` in its styles: a full-screen menu on a phone, a purchase bar along the bottom. The store keeps shoppers safe around it:

- Every `z-index` in that section's styles is at most 100 (a higher one is written as 100, and one that is not a whole number or a keyword, such as `var()`, is left out), so a section's layer sits above a sticky header section (50) and below the store's own layers: the skip link, a script's overlay, the builder's outlines. A `position` other than a plain keyword is left out too.
- On the cart page `position: fixed` is left out of every section's styles, header and footer included, so nothing can cover the checkout button or the totals. Write the fallback first and it applies there: `position:absolute;position:fixed`.
- In the builder a fixed layer is written as `position: absolute`: it scrolls with the page, and the builder's outlines and selection still reach everything under it.

Custom HTML blocks, the store stylesheet and every section without `fixed: true` never use `position: fixed`. Everywhere, with or without `fixed: true`, a `position` has to be a plain word (`static`, `relative`, `absolute`, `sticky`, and `fixed` where allowed): a value built with `var()`, `env()`, `attr()` or any other function, or written with a backslash escape, is left out, since it could still come out as `fixed` (`--p: fixed; position: var(--p)` keeps only `--p`). A property name written with a backslash escape is left out too, and nested rules follow the same rules ([custom code](https://formahand.com/docs/custom-code)).

Styles never contain `{{ }}`. Instead, every colour, font, number, range, select, on/off and image setting arrives as a CSS variable on the section: `--fh-s-` followed by the setting's key in kebab case. `minHeight` becomes `var(--fh-s-min-height)`, with its unit when the setting has one; an on/off setting is `1` or `0`; an empty colour is left out, so give a fallback (`var(--fh-s-accent,inherit)`) and the theme's own colour shows. The theme's colour and font tokens (`--fh-accent`, `--fh-font-heading` and the rest) work inside a section as they do anywhere else. For a button on the brand colour use `background:var(--fh-accent);color:var(--fh-on-accent)`: the store computes a readable text colour for whatever accent the owner picks. Never paint a button `background:currentColor` without a colour of its own.

A colour can be see-through: `#rrggbbaa`, where the last pair is the opacity from `00` (clear) to `ff` (solid), such as `#00000080` for a half-dark overlay on a photo.

**Fonts.** A `font` setting names a family from the font list (`describe_sections { parts: ["fonts"] }`: about three hundred of the most used open font families; some 1,400 more open families are accepted by their exact name without being listed, so a design's own face can be used) and declares the weights your styles use, at most four (`weights: [400, 700]`), and `italic: true` to load the family's italics in those weights too when it has them. The section loads that family in those weights only, and the variable holds the family with a fallback, so `h2{font-family:var(--fh-s-heading-font,inherit)}` uses it and falls back to the theme's font while the setting is empty. The store's own font pairing is unchanged; a section's font applies inside that section only. The store's own typefaces (the custom pairing) load 400 to 700 unless their weights are named: `set-fonts { pairing: "custom", headingFamily: "Playfair", headingWeights: [300, 400, 800], headingItalic: true }` loads any weights the family has, at most six, so a section can set `var(--fh-font-heading)` at 300 or 800 without a font setting of its own.

**A picture in the styles.** An image setting arrives as `url("<its path in your media library>")`, at its original size, so styles can lay it as a tiled background (`background-image:var(--fh-s-grain,none)`) or draw a one-colour drawing in the text colour (`background:currentColor;mask:var(--fh-s-mark,none) center/contain no-repeat`, with the `-webkit-mask` twin). An empty setting writes no variable, so give the fallback `none`. An image field of a list item arrives the same way as `--fh-i-<field>`. Only a path in the store's own media library is ever written.

**List items with their own colours.** Colour, number, range, select and on/off fields of a list's items become CSS variables on each item's own element: put `data-fh-item="{{ loop.itemId }}"` on it inside `{% for %}`, and the item's `background` field is `var(--fh-i-background)` there. No style attribute is needed, and each tile of a row can have its own colour:

```css
.tile{background:var(--fh-i-background,var(--fh-accent))}
```

# Section behaviours

A section can include a `behavior` field, a small module that runs in a dedicated Web Worker. It can change its own section only by asking Formahand's page bridge to perform a bounded operation. The bridge checks every selector and value against that section's root.

Behaviours are saved and versioned with the section. They run only on pages where section scripts are allowed, and only while the owner or an admin has turned on **Scripts in sections**. A page creates at most 16 behaviour workers; each worker is rate and size limited. Editing previews stay inert. The legacy `script` field keeps its existing same-origin permissions and review requirement. Moving a package to `behavior` is a deliberate code change; it does not silently contain or rewrite an older script.

## Entry point and bridge

Write one named default function. Imports and additional exports are refused. The function may be async and may return a cleanup callback.

```js
export default function mount(api) {
  // The page is reached through api.ui and the other listed capabilities.
}
```

Its policy also refuses network connections, scripts, nested workers, images, frames and forms. `import`, `importScripts`, `eval` and `new Function` are refused. Use only the frozen API passed to `mount`.

| API | What it can do |
| --- | --- |
| `api.ui.measure(selector)` | Read position, size and scroll measurements for elements inside this section. |
| `api.ui.scroll(selector, { left?, top?, behavior? })` | Scroll this section's own element to bounded numeric coordinates. Reduced motion forces an instant scroll. |
| `api.ui.set(selector, changes)` | Change text, ordinary classes, a small ARIA/visibility attribute set and a safe subset of styles. Anchor `href` is restricted to local product paths; image `src` to local media-library JPG/PNG/WebP paths. It cannot write HTML, arbitrary URLs, forms, scripts or stylesheets. |
| `api.ui.animate(selector, frames, options)` | Animate the same safe style subset; duration is capped and reduced motion skips the animation. |
| `api.ui.on(selector, type, callback)` | Subscribe to a small list of trusted shopper events. Scroll events include bounded element scroll offsets. Only safe key and pointer details are relayed; only search input values are included. |
| `api.ui.onPlatform(callback)` | Hear whether the tab is hidden, reduced motion is on, and the viewport size changed. |
| `api.overlay.open(name, { label? })` / `close()` / `isOpen()` | Ask the platform overlay to open a named element from this section. Focus trapping and scroll lock remain platform owned. |
| `api.cart` | Read, add, update, remove, apply/remove a code, read prices and open this section's authored cart overlay (or navigate to the cart after a shopper gesture). Prices always come from the store. |
| `api.search.suggest(query)` | Ask the store for bounded product suggestions. |
| `api.offers.get()` / `api.shopper.get()` | Read public offers and whether the shopper is signed in. |
| `api.storage.get/set/remove(key)` | Keep small values in the section's namespaced browser storage. |
| `api.onCleanup(callback)` | Register worker-local cleanup. The platform also removes page listeners, animations and overlays when the worker stops. |

There is no general fetch, app route or arbitrary URL operation. The page bridge caps calls per second and in flight, bounds selector length, result count, event rate, text, attributes, styles and animation duration, and stops a worker that exceeds its budget or stops responding. Its responses expose only the requested measurements and public cart fields. Selectors can use classes, tags and exact `data-fh-*` markers, not private attribute selectors such as `[value]`; CSS escapes are not accepted.

Commerce reads and writes share a limit of 20 requests per 10 seconds per instance. Debounce search input and reuse results. Cart changes require a recent trusted click, pointer press or activation key event subscribed through this section's UI bridge; code cannot quietly change the cart on mount. Do not put credentials or private buyer data in section settings: settings and published markup are public storefront content.

## Search overlays and cart drawers

Agents own the markup and presentation, not the network route. A search overlay can contain `{% widget "search" %}` and a fixed set of initially hidden result slots. Subscribe to the widget's `.fh-widget-input` `input` event, debounce `event.value`, then call `api.search.suggest(query)`. Each result contains `name`, `slug`, `url`, `imageUrl`, `imageAlt` and shopper-priced amount/currency fields. Fill existing title/price nodes with `ui.set({ text })`; set the result anchor's local product `href` and its image's local media `src` through the bounded attribute bridge. Hide unused slots. There is no HTML insertion or arbitrary remote image operation.

For a cart drawer, author a named `data-fh-overlay="cart"` element in the same section and bind its button to `api.cart.open()`. Use `api.cart.get()` and `onChange` to fill preallocated line slots, with subscribed quantity buttons calling `update`/`remove`. The platform supplies focus trapping, Escape, backdrop and scroll locking; your section supplies the layout. Cart responses deliberately omit names, addresses, email, notes and custom line attributes. Checkout and account pages never run these behaviours.

## A product slider

The section template owns the cards and controls. CSS provides responsive slide widths and touch scroll snapping; the behaviour handles arrows, dots, keyboard, announcements, optional autoplay and reduced motion. Products are real server-rendered cards, so they remain useful if behaviour is switched off.

Example template:

```html
<section class="shelf" aria-label="New arrivals">
  <div class="shelf__track" data-fh-slider-track role="region" aria-roledescription="carousel" aria-label="New arrivals" tabindex="0">
    {% for product in settings.shelf.products limit: 8 %}
      <article class="shelf__slide" data-fh-slider-slide role="group" aria-roledescription="slide" aria-label="Slide {{ loop.index }}">
        <a href="{{ product.url }}">
          <img src="{{ product.image | image_url: 900 }}" alt="{{ product.image.alt }}" loading="lazy">
          <h2>{{ product.title }}</h2>
          <p>{{ product.price | money }}</p>
        </a>
      </article>
    {% endfor %}
  </div>
  <div class="shelf__controls">
    <button type="button" data-fh-slider-prev aria-label="Previous slide">Previous</button>
    <button type="button" data-fh-slider-next aria-label="Next slide">Next</button>
    <div class="shelf__dots" data-fh-slider-dots role="group" aria-label="Choose a slide">
      {% for product in settings.shelf.products limit: 8 %}
        <button type="button" data-fh-slider-dot="{{ loop.index }}" aria-label="Show slide {{ loop.index }}"></button>
      {% endfor %}
    </div>
  </div>
  <p class="shelf__status" data-fh-slider-status role="status" aria-live="polite" aria-atomic="true">Slide 1</p>
</section>
```

The section stylesheet draws the look. These defaults show one slide on phones, two above 640px and three above 960px:

```css
.shelf__track{display:flex;gap:1rem;overflow-x:auto;scroll-snap-type:x mandatory;scrollbar-width:none}
.shelf__slide{flex:0 0 100%;scroll-snap-align:start}
.shelf__slide img{display:block;width:100%;height:auto}
.shelf__controls{display:flex;align-items:center;gap:.75rem}
.shelf__dots{display:flex;gap:.5rem}
.shelf__dots button[aria-current="true"]{background:var(--fh-accent)}
.shelf__status{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0,0,0,0);white-space:nowrap}
@media(min-width:640px){.shelf__slide{flex-basis:calc((100% - 1rem)/2)}}
@media(min-width:960px){.shelf__slide{flex-basis:calc((100% - 2rem)/3)}}
@media(prefers-reduced-motion:reduce){.shelf__track{scroll-behavior:auto}}
```

For optional autoplay, add a numeric `autoplay` setting (milliseconds, default `0`) to the section schema. Zero disables it; positive values are clamped to 2.5–60 seconds. Attach this as the package's `behavior`:

```js
export default function mount(api) {
  const track = ".shelf__track";
  const slides = ".shelf__slide";
  const dots = ".shelf__dots [data-fh-slider-dot]";
  const status = ".shelf__status";
  const previous = "[data-fh-slider-prev]";
  const next = "[data-fh-slider-next]";
  let current = 0;
  let timer = 0;
  let hovering = false;
  let focused = false;
  let hidden = api.hidden;
  let reducedMotion = api.reducedMotion;
  const configuredInterval = Number(api.settings.autoplay) || 0;
  const interval = configuredInterval > 0
    ? Math.min(60000, Math.max(2500, configuredInterval))
    : 0;

  async function go(to, smooth = true) {
    const [viewport, positions] = await Promise.all([
      api.ui.measure(track),
      api.ui.measure(slides),
    ]);
    if (!viewport.length || !positions.length) return;
    const count = positions.length;
    current = ((to % count) + count) % count;
    const target = positions[current];
    await api.ui.scroll(track, {
      left: viewport[0].scrollLeft + target.x - viewport[0].x,
      behavior: smooth && !reducedMotion ? "smooth" : "instant",
    });
    await api.ui.set(dots, { attributes: { "aria-current": null } });
    await api.ui.set(`${dots}[data-fh-slider-dot="${current + 1}"]`, {
      attributes: { "aria-current": "true" },
    });
    await api.ui.set(status, { text: `Slide ${current + 1} of ${count}` });
  }

  function stop() {
    if (timer) clearInterval(timer);
    timer = 0;
  }
  function start() {
    stop();
    if (interval && !hidden && !reducedMotion && !hovering && !focused)
      timer = setInterval(() => go(current + 1), interval);
  }

  api.ui.on(previous, "click", () => { stop(); go(current - 1).then(start); });
  api.ui.on(next, "click", () => { stop(); go(current + 1).then(start); });
  api.ui.measure(slides).then((items) => {
    items.forEach((_, index) => {
      api.ui.on(`${dots}[data-fh-slider-dot="${index + 1}"]`, "click", () => {
        stop(); go(index).then(start);
      });
    });
    go(current, false);
  });
  api.ui.on(track, "keydown", (event) => {
    if (event.key === "ArrowLeft") { stop(); go(current - 1).then(start); }
    if (event.key === "ArrowRight") { stop(); go(current + 1).then(start); }
    if (event.key === "Home") { stop(); go(0).then(start); }
    if (event.key === "End") { stop(); api.ui.measure(slides).then((items) => go(items.length - 1)).then(start); }
  });
  api.ui.on(track, "scroll", async () => {
    const [viewport, positions] = await Promise.all([
      api.ui.measure(track),
      api.ui.measure(slides),
    ]);
    if (!viewport.length) return;
    let closest = 0;
    let distance = Infinity;
    positions.forEach((item, index) => {
      const value = Math.abs(item.x - viewport[0].x);
      if (value < distance) { distance = value; closest = index; }
    });
    if (closest !== current) {
      current = closest;
      await api.ui.set(dots, { attributes: { "aria-current": null } });
      await api.ui.set(`${dots}[data-fh-slider-dot="${current + 1}"]`, {
        attributes: { "aria-current": "true" },
      });
      await api.ui.set(status, { text: `Slide ${current + 1} of ${positions.length}` });
    }
  });
  api.ui.on(track, "mouseenter", () => { hovering = true; stop(); });
  api.ui.on(track, "mouseleave", () => { hovering = false; start(); });
  api.ui.on(track, "focusin", () => { focused = true; stop(); });
  api.ui.on(track, "focusout", () => { focused = false; start(); });
  api.ui.onPlatform((state) => {
    hidden = state.hidden;
    reducedMotion = state.reducedMotion;
    if (hidden || reducedMotion) stop();
    else start();
  });

  start();
  api.onCleanup(stop);
}
```

This example uses touch scroll snapping for swipe and browser-provided scroll momentum. Autoplay is an optional behaviour setting; keep it off by default, and use `api.ui.onPlatform` to pause it when the tab is hidden or reduced motion is enabled. Ask `describe_sections { parts: ["behaviors"] }` for this recipe in agent-readable form.

# Section media and licensed fonts

## Controls and self-hosted media

Section templates may author `<button type="button" class="…" aria-label="…">` controls. Merchant custom HTML blocks continue to refuse buttons. Bind controls through a section behavior's UI bridge.

Section templates support `<video>`, `<picture>` and void `<source>` elements. Video URLs and picture source sets use this store's `/media/products/<uuid>` paths. Upload MP4/WebM with `POST /api/section-media?storeId=…` using multipart `file` or JSON `{ "dataBase64": "…", "contentType": "video/mp4" | "video/webm" }`, up to 50 MB; or use the `upload_section_video` MCP tool with the same base64 fields. The MIME type is checked against the file signature, and uploads go to the selected environment's own media library. `list_media` discovers uploaded objects. Licensed WOFF2 uploads stay on the signed-in owner's Fonts and video screen, with licence attestation.

```html
<video src="/media/products/UUID.mp4" poster="/media/products/UUID.png" muted loop autoplay playsinline></video>
<picture><source srcset="/media/products/UUID.webp 600w, /media/products/UUID.webp 1200w" media="(min-width:600px)"><img src="/media/products/UUID.png" alt="Your product"></picture>
```

Autoplay requires muted, loop and a media-library poster. The platform starts playback only when reduced motion is off; reduced motion or no JavaScript leaves the poster visible. Controls are available for manual playback. Supported preload values are `none` and `metadata`. No remote player JSON or Lottie is supported.

## Licensed fonts

The signed-in store owner can upload WOFF2 (5 MB maximum) in the same dashboard panel after confirming that the licence permits web embedding on this store. The multipart field `licenseAttestation` must be `true`. Tokens cannot attest a licence. The store records who confirmed the licence and when alongside the file in its media library.

The upload returns `fontValue: "upload:<uuid>"`. Use that value in a section `font` setting, or a custom brand pairing's `headingFamily` / `bodyFamily`. `list_media` returns `fontValue` on licensed fonts for agent discovery. Paste it into the section font picker and select **Use uploaded font**. Rendering writes a local `@font-face` with `font-display:swap`; font setting CSS variables work as usual. Each upload is one face; use a separate upload for an italic face. Upload paths do not change and belong to this store, and fonts count toward media usage.

## Limits

| What | Limit |
| --- | --- |
| Template | 32 KB |
| Styles | 16 KB |
| Artwork drawings per section | 32, each 32 KB, 160 KB together |
| Settings per section | 40 |
| Groups per section | 8 |
| Presets per section | 12 |
| Options in one select setting | 24 |
| Fields in one list item | 8 |
| Items in one list setting | 24 |
| Instances on one page | 60 |
| Instances in the header or footer slot | 40 |
| The most maxInstancesPerPage may be | 60 |
| Sections per store | 100 |
| Versions kept per section | 20 |
| Nested if and for tags | 8 |
| Expressions and tags in one template | 1,500 |
| Characters in one expression | 300 |
| Filters in one expression | 6 |
| Items one loop shows | 50 |
| Loop steps in one render | 1,000 |
| Products one collection offers a template | 50 |
| Product and collection settings one instance may use | 24 |
| Render time for one instance | 10 ms |
| Render time for every section on a page | 100 ms |
| Markup one instance may produce | 64 KB |
| Markup every section on a page may produce | 512 KB |
| Image widths image_url accepts | 64 to 2,400 pixels |

A section over a render limit is left out of the page rather than slowing it down, and the store's event log records why.

## Examples

Complete packages that pass every check; start from one.

**A promo band, with presets and styles**

```json
{
  "schemaVersion": 1,
  "id": "promo-band",
  "name": "Promo band",
  "description": "A coloured strip with an offer, an optional code and a link.",
  "category": "promotion",
  "version": "1.2.0",
  "author": {
    "kind": "owner",
    "name": ""
  },
  "origin": "store",
  "schema": {
    "settings": [
      {
        "key": "heading",
        "type": "text",
        "label": "Offer",
        "default": "Free delivery over £50",
        "maxLength": 100,
        "inlineEditable": true
      },
      {
        "key": "text",
        "type": "text",
        "label": "Small print",
        "maxLength": 160,
        "inlineEditable": true
      },
      {
        "key": "code",
        "type": "text",
        "label": "Discount code",
        "help": "Shown in a box shoppers can copy by hand.",
        "maxLength": 32
      },
      {
        "key": "linkLabel",
        "type": "text",
        "label": "Link label",
        "default": "Shop the sale",
        "maxLength": 40
      },
      {
        "key": "link",
        "type": "link",
        "label": "Link",
        "default": "/collections/all"
      },
      {
        "key": "showBadge",
        "type": "boolean",
        "label": "Show a Sale badge",
        "default": true
      },
      {
        "key": "background",
        "type": "color",
        "label": "Background",
        "default": "#111111"
      },
      {
        "key": "foreground",
        "type": "color",
        "label": "Text colour",
        "default": "#ffffff"
      }
    ],
    "elements": [
      {
        "key": "cta",
        "label": "Link",
        "type": "link",
        "setting": "linkLabel"
      }
    ],
    "maxInstancesPerPage": 3,
    "pages": [
      "home",
      "catalog",
      "product",
      "page"
    ],
    "slots": [
      "header.top",
      "footer.top"
    ]
  },
  "template": "<div class=\"promo\" data-fh-group=\"band\">\n  {% if settings.showBadge %}<span class=\"promo__badge\">Sale</span>{% endif %}\n  <strong data-fh-el=\"heading\">{{ settings.heading }}</strong>\n  {% if settings.text %}<span data-fh-el=\"text\">{{ settings.text }}</span>{% endif %}\n  {% if settings.code %}<code data-fh-el=\"code\">{{ settings.code | upper }}</code>{% endif %}\n  {% if settings.link %}<a data-fh-el=\"cta\" href=\"{{ settings.link }}\">{{ settings.linkLabel }}</a>{% endif %}\n</div>\n",
  "styles": ":scope{background:var(--fh-s-background);color:var(--fh-s-foreground)}\n.promo{display:flex;flex-wrap:wrap;gap:.75rem;align-items:center;justify-content:center;padding:.75rem 1rem;font-size:.95rem}\n.promo a{color:inherit;text-decoration:underline}\n.promo code{border:1px dashed currentColor;padding:.1rem .4rem;border-radius:4px}\n.promo__badge{text-transform:uppercase;font-size:.75rem;letter-spacing:.08em;border:1px solid currentColor;padding:.1rem .5rem;border-radius:999px}\n",
  "presets": [
    {
      "id": "announcement",
      "name": "Announcement",
      "settings": {
        "showBadge": false,
        "background": "#f4f1ea",
        "foreground": "#1a1a1a"
      }
    },
    {
      "id": "code",
      "name": "With a code",
      "settings": {
        "heading": "20% off everything",
        "code": "WINTER20",
        "text": "Until Sunday."
      }
    }
  ]
}
```

**Questions and answers, with a list setting**

```json
{
  "schemaVersion": 1,
  "id": "faq-accordion",
  "name": "Questions and answers",
  "description": "Questions shoppers ask, each opening to its answer.",
  "category": "text",
  "version": "1.0.0",
  "origin": "store",
  "schema": {
    "settings": [
      {
        "key": "heading",
        "type": "text",
        "label": "Heading",
        "default": "Questions",
        "maxLength": 120,
        "inlineEditable": true
      },
      {
        "key": "questions",
        "type": "list",
        "label": "Questions",
        "itemLabel": "Question",
        "minItems": 1,
        "maxItems": 24,
        "fields": [
          {
            "key": "question",
            "type": "text",
            "label": "Question",
            "maxLength": 200,
            "required": true,
            "inlineEditable": true
          },
          {
            "key": "answer",
            "type": "richtext",
            "label": "Answer",
            "maxLength": 2000,
            "inlineEditable": true
          }
        ],
        "default": [
          {
            "question": "How long does delivery take?",
            "answer": "<p>Two to four working days in the UK.</p>"
          },
          {
            "question": "Can I return something?",
            "answer": "<p>Yes, within 30 days. See our <a href=\"/pages/returns\">returns page</a>.</p>"
          }
        ]
      },
      {
        "key": "openFirst",
        "type": "boolean",
        "label": "Open the first answer",
        "default": false
      }
    ],
    "maxInstancesPerPage": 2,
    "pages": [
      "home",
      "product",
      "page"
    ]
  },
  "template": "<div class=\"faq\">\n  <h2 data-fh-el=\"heading\">{{ settings.heading }}</h2>\n  {% for item in settings.questions %}\n  <details class=\"faq__item\"{% if loop.first and settings.openFirst %} open{% endif %}>\n    <summary>{{ item.question }}</summary>\n    <div class=\"faq__answer\">{{ item.answer | raw }}</div>\n  </details>\n  {% endfor %}\n</div>\n",
  "styles": ".faq{max-width:760px;margin-inline:auto;padding:2rem 1rem}\n.faq__item{border-bottom:1px solid rgba(127,127,127,.3);padding:1rem 0}\n.faq__item summary{cursor:pointer;font-weight:600;list-style:none}\n.faq__item[open] summary{margin-bottom:.5rem}\n",
  "presets": []
}
```

## Versions, drafts and copies

**Every save is a version, and versions never change.** Saving a section again writes a new version. Leave `version` out and `save_section` chooses it from what changed:

| What changed | The new version |
| --- | --- |
| A setting removed or its type changed, a select option or list field removed, a kind of page or slot no longer allowed, or fewer instances allowed per page | A new major version (2.0.0) |
| A setting, option, list field, preset, kind of page or slot added | A new minor version (1.1.0) |
| The template, the styles, labels, help text or defaults | A patch (1.0.1) |

If you give a version yourself, it has to be at least that step; a smaller one is refused with the version to use. The answer to every save says the version, the step and what changed.

**Saving changes the draft, publishing changes the storefront.** A new version is what the draft renders at once; the live storefront keeps the version it was published with until `publish_storefront`. Restoring an earlier storefront version (`restore_storefront_version`) brings back the section versions it used as well.

**Nothing on a page breaks when a setting goes.** An instance written against an older version keeps rendering: a value for a setting the new version dropped is ignored, and a value that no longer fits shows the default instead.

**Rollback is a new version.** `rollback_section { id, version }` saves that version's content again as the next version, so nothing is lost and you can roll forward the same way. The 20 most recent versions of each section are kept.

**Make it mine.** `fork_section { id, newId }` makes your own copy of an add-on's section (or of one of yours) under a new id, starting at 1.0.0 and remembering where it came from. The copy is yours to change. Pass `repoint: true` to move every instance in the draft onto the copy; otherwise they stay on the original. Built-in sections cannot be copied yet.

**Deleting.** `delete_section` is refused while the draft or the live storefront uses the section, and the refusal says where. Remove those instances first, or pass `force: true`: each instance then stays where it is and shows nothing, the builder marks it, and saving a section with the same id brings it back.

## Sections from add-ons

An [add-on](https://formahand.com/docs/apps) can ship up to six sections. They are named after the add-on (`faq--accordion`, `promo--band`), checked exactly as `save_section` checks yours, and placeable as soon as the add-on is installed. They are read-only in your store: to change one, `fork_section` it. They need no grant, because a section only reads what every template reads.

Two add-ons in the catalogue ship sections today:

| Add-on | Sections |
| --- | --- |
| Questions and answers | `faq--accordion`: a heading and questions that open to their answers |
| Promo bands | `promo--band`: an offer band with a code and a link, for any page and the header or footer; `promo--stats`: two to six big numbers with a line each |

When you uninstall an add-on, every instance of its sections stays where you put it and shows nothing; the builder marks those places. Install the add-on again and they come back exactly as they were, settings and all.

If you write an app yourself, its manifest takes `sections`: an array of whole packages, each with `origin: "app"`, `app` set to the app's name and an id of `<app>--<name>`. `describe_apps` and `describe_sections` have the details.

## Refusals

One sentence per problem, naming the section and then the page or slot, instance, preset, setting, list item, field, template line or styles. Fix every line and send the package again.

| Code | Means |
| --- | --- |
| `section_invalid` | The section package does not match the section format. |
| `section_id_invalid` | A section id is 2 to 40 lower-case letters, digits and single hyphens, starting with a letter. |
| `section_id_reserved` | That id belongs to a built-in section or a layout keyword. |
| `section_id_taken` | Your store already has a section with that id. |
| `section_not_found` | No section with that id is in your store. |
| `section_read_only` | Built-in, add-on and catalogue sections cannot be edited; make your own copy first. |
| `section_in_use` | The section is still placed on a page; remove its instances first. |
| `section_limit` | Your store already holds the most sections it can. |
| `fork_not_allowed` | This section cannot be copied. |
| `version_invalid` | A version looks like 1.4.0. |
| `version_not_newer` | A new version has to be higher than the current one. |
| `version_bump_required` | The change needs a bigger version step (for example a removed setting needs a new major version). |
| `setting_invalid` | A setting value does not fit its definition. |
| `setting_unknown` | The section does not declare that setting. |
| `setting_definition_invalid` | A setting definition is not valid. |
| `setting_type_reserved` | That setting type is only available to built-in sections. |
| `setting_limit` | A section declares at most 40 settings. |
| `group_unknown` | A setting names a group the section does not declare. |
| `preset_invalid` | A preset holds a value its setting does not accept. |
| `preset_limit` | A section has at most 12 presets. |
| `template_too_large` | A section template is at most 32 KB. |
| `template_syntax` | The template has a syntax error. |
| `template_unknown_tag` | The template uses a tag sections do not have. |
| `template_unknown_object` | The template names something sections cannot read. |
| `template_unknown_setting` | The template reads a setting the section does not declare. |
| `template_unknown_filter` | The template uses a filter sections do not have. |
| `template_forbidden` | The template contains something sections never allow (scripts, event handlers, inline styles, other files). |
| `template_nesting` | Tags are nested more than 8 deep. |
| `template_loop_limit` | A loop asks for more than 50 items. |
| `behavior_invalid` | A section behaviour must be a self-contained default mount(api) function using only the worker bridge. |
| `behavior_too_large` | A section behaviour is at most 32 KB. |
| `template_expression_limit` | The template has too many expressions, or one that is too long. |
| `template_missing` | The section has no template; only built-in sections render without one. |
| `element_unknown` | The template marks an element (data-fh-el) that is neither a setting of the section nor declared in schema.elements. |
| `element_invalid` | An element declaration or a data-fh-el or data-fh-group mark is not valid. |
| `elements_unmarked` | The section prints text, image or link settings but marks no element with data-fh-el, so the builder cannot select anything inside it. |
| `element_unmarked` | A text, image or link setting is printed only outside elements marked with data-fh-el; mark the element that prints it, or list the setting in schema.fieldOnly. |
| `styles_too_large` | Section styles are at most 16 KB. |
| `styles_template_syntax` | Styles cannot contain template expressions; read settings through the section's CSS variables. |
| `artwork_invalid` | A drawing in the section's artwork has a name or markup sections do not accept; the message says what to remove. |
| `artwork_limit` | A section carries at most 32 drawings, each at most 32 KB and 160 KB together. |
| `instance_invalid` | The instance does not match the instance format. |
| `instance_id_invalid` | An instance id is 1 to 32 lower-case letters, digits and hyphens, starting with a letter or digit. |
| `instance_id_reserved` | That instance id is a built-in section id or a layout keyword. |
| `instance_id_taken` | Another instance already uses that id. |
| `instance_not_found` | No instance with that id is on the storefront. |
| `instance_limit` | The page already holds as many instances of this section as it allows. |
| `page_instance_limit` | A page holds at most 60 section instances. |
| `page_not_allowed` | The section cannot be placed on that page. |
| `slot_not_allowed` | The section cannot be placed in that slot. |
| `home_empty` | The home page has to show at least one visible section or block. |
| `collection_required` | The older name of home_empty: the home page has to show at least one visible section or block. |
| `main_required` | Every page other than home has its own body exactly once, and home has none. |
| `render_timeout` | The section took too long to render and was left out of the page. |
| `render_too_large` | The section produced too much markup and was left out of the page. |
| `render_loop_limit` | The section looped over too many items and was left out of the page. |

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `describe_sections` | `storefront:read` | Everything needed to write a section, as data. |
| `list_sections` | `storefront:read` | The sections this store can place: the built-in ones every store has (the theme's hero, collection, more-products and styles, each page's own body, and custom HTML), and the store's own, add-on and catalogue sections with their version, whether you… |
| `get_section` | `storefront:read` | One section's whole package (settings schema, template, styles, presets), whether you may change it, its kept versions and where it is placed. |
| `save_section` | `storefront:write` | Creates one of the store's own sections, or saves a new version of it: { package }. |
| `fork_section` | `storefront:write` | Makes the store's own copy of an add-on, catalogue or store section under a new id: { id, newId, repoint? }. |
| `list_section_versions` | `storefront:read` | The kept versions of one of the store's sections, newest first, with when each was saved, by whom, and which one the draft and the live storefront render. |
| `rollback_section` | `storefront:write` | Brings back an earlier version of one of the store's sections: { id, version }. |
| `delete_section` | `storefront:write` | Deletes one of the store's own (or catalogue) sections with all its versions: { id, force? }. |
| `preview_section` | `storefront:read` | Renders one instance of a section with the settings you give, exactly as shoppers would see it, without placing it anywhere: { id, version?, settings?, page? } for a saved or built-in section, or { package, settings?, page? } for one you have not saved yet. |

## Read more

- https://formahand.com/docs/sections
- https://formahand.com/docs/custom-code
- https://formahand.com/docs/apps
