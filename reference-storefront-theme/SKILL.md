---
name: reference-storefront-theme
description: Use a live website, screenshot, or URL as inspiration to design and build a merchant-editable Formahand storefront theme. Covers reference discovery, capability checks, safe asset use, Formahand sections and behaviors, and evidence-based verification.
---

# Reference-inspired Formahand storefronts

Use this skill when a merchant wants a Formahand storefront shaped by another
site, screenshot, or visual direction. The goal is a polished store that
captures the requested presentation while using the merchant's real catalog,
authorized assets, and Formahand's supported theme architecture. Do not claim
an exact reproduction when behavior or states were not observed and verified.

## Read the platform contract first

For work through a Formahand store connection, also read the `formahand` skill
and make its required first five calls. Read the current sections, behavior,
media, theme and custom-code references before implementation; prefer the live
tool descriptions and `describe_sections` / `get_custom_code.allowed` response
over remembered examples. The source of truth is the current store contract,
not an old skill example or a reference site's technology.

Formahand's theme is validated data, editable sections, settings and assets;
it is not an imported arbitrary frontend runtime. Build reusable bands as
sections, put merchant-editable copy and media in settings, mark printed values
with `data-fh-el`, and mark reorderable flex/grid children with
`data-fh-group`. Use custom blocks only for one-off content. Server-rendered
product and collection cards should remain useful even when interaction code
is unavailable.

Use section `behavior` for agent-authored interactions, following its
bounded UI and commerce bridge. Behaviors have no general DOM, network,
session-data, or cross-store access. They run only when the owner/admin enables **Scripts in
sections**; builder editing previews are intentionally inert. Provide a
reasonable no-behavior fallback. Never substitute legacy same-origin `script`
for an agent behavior or imply that the two have the same trust model.

Use the platform's self-hosted media path for video and responsive pictures,
following its autoplay, poster, and reduced-motion rules. For a font not in the
open family list, the signed-in owner must provide a licensed WOFF2 upload and
attest its licence. Do not assume Adobe/Typekit, arbitrary remote font files,
Lottie JSON, or external widgets are supported.

## 1. Explore before building

Inspect the reference at desktop and phone widths, if browser access is
available. Start with the main page, then inspect relevant pages and states:

- hover, focus and keyboard navigation menus;
- search open, suggestions, no-results and errors;
- product variants, unavailable options and quick add;
- cart open, quantity changes, removal, discounts and empty state;
- newsletter success and validation/error states;
- delayed overlays, campaign prompts and returning-visitor behavior;
- responsive layout, media crop, scrolling and motion.

For every capture or note, retain the URL, viewport, locale, elapsed time and
relevant consent/visitor conditions. Label each finding `observed`, `inferred`,
or `unknown`. One visit cannot reveal every campaign, experiment or visitor
segment: leave unobserved behavior `unknown`. If browser access or a state is
unavailable, say so instead of fabricating evidence. Create a small
task-specific audit or screenshots only when useful; do not impose permanent
store requirement files on every project.

## 2. Follow dependencies and protect data

When available, inspect rendered structure and computed styles, font and media
resources, and relevant script/app configuration requests. Determine which
behavior appears to belong to the theme, an external app, or the commerce
backend; visual appearance alone does not prove the implementation. Keep the
investigation limited to public storefront data. Do not capture or retain
credentials, session tokens, personal buyer data, checkout secrets, or unrelated
network payloads. Never attempt to defeat access controls, bot protections,
consent, or paywalls.

Use the reference for visual direction, not as permission to copy. Reuse its
code, brand marks, product photography, copy, reviews, fonts, or other assets
only when the merchant confirms they have rights and provides/authorizes them.
Otherwise recreate the composition and interaction intent with merchant-owned
assets, licensed media, and truthful copy. Do not invent reviews, ratings,
discounts, scarcity, or product claims.

## 3. Verify platform support before promising

Map each desired feature to the current Formahand contract and evidence. Keep
these states distinct:

- `verified`: exercised through the relevant save/render or store-backed flow;
- `documented`: documented contract, but full flow not exercised;
- `gap`: no supported capability for the requirement;
- `unknown`: not inspected or not enough evidence;
- `not in scope`: intentionally excluded.

Documentation, types, fixtures, and unit tests alone do not prove an end-
to-end shopper flow. For important features, use representative merchant data
through save, preview/render, and real store-backed behavior where safe. Record
the gap and the visible substitute before silently changing the design.

Particular boundaries to check:

- **Interactions:** section `behavior` is opt-in and builder preview is inert.
  Autoplay, reveal games, campaign targeting, search suggestions, cart drawers,
  and external widgets need explicit supported behavior/integrations; don't
  imply a CSS mock is a working commerce feature.
- **App data:** the behavior bridge has no general app-route API.
  Private app data such as a loyalty balance cannot be fetched by an agent
  behavior unless Formahand adds a documented bounded capability. Do not route
  around this by adding a legacy same-origin script.
- **Commerce:** use Formahand widgets and bounded commerce APIs. Never expose
  buyer/order data in public section settings or markup. Test with a sandbox or
  safe test data when the feature changes cart/checkout behavior.
- **Media and type:** use documented self-hosted video/picture paths and
  owner-attested licensed fonts. If an asset or font is unavailable, identify
  the substitution.
- **Scripts:** do not ship a reference site's code or agent-authored arbitrary
  same-origin JavaScript. The platform's legacy `script` field is a different,
  more powerful owner-trust boundary.

## 4. Build an editable theme, not a screenshot

1. Read the store draft and current theme; call `get_custom_code` and read its
   `allowed` contract. Read `describe_sections` for package, widget, behavior,
   template and style rules.
2. Choose a close starting theme or `blank-canvas` when a full redesign is
   intended. Preserve page structure and real product/collection data.
3. Build repeated or merchant-editable visual bands as sections. Add schema
   settings for copy, media, links and product/collection sources. Mark every
   printed editable value and reorderable group. Preview each section, then
   place it in the draft.
4. Use section-scoped CSS for visual design and motion. Honor responsive
   breakpoints, keyboard access, semantic labels, focus visibility and
   `prefers-reduced-motion`. Use behavior only for interactions that need it,
   with a no-script fallback.
5. Keep the header, product, cart and content-page experience coherent; don't
   treat the homepage as the whole store. Do not hide/replace theme sections
  with CSS, remove or hide them through the supported editor commands.

## 5. Prove completion accurately

Compare reference and draft at matching viewports and observed states. Check
layout, typography, crop, overflow, menus, keyboard operation, reduced motion,
real product options, cart totals and newsletter outcomes when in scope. Test
the actual save-and-render path, not just standalone section output.

Report the stages separately: `captured`, `implemented`, `locally verified`,
`store-verified`, and `published`. State the evidence and remaining unknowns
or substitutions. A successful save or preview is not publication. Publishing
requires the user's authorization and the normal Formahand governance rules;
this skill alone does not authorize publish, paid choices, or merchant-bound
decisions.
