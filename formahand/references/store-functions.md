# Store functions

A store function is one small JavaScript file the store runs at a named point in the cart or checkout: adding a line to a cart, pricing a cart, allowing a discount, narrowing shipping rates or payment methods, checking a checkout, reacting to an order. It is handed JSON and answers JSON. Build and test it on the test store; on a live store a token can only switch it to shadow, and the merchant switches it on.

## The hooks

Generated from the hook registry. Export an object with the named method (or `run`, which every hook accepts). `describe_store_functions` answers the input and output shape of each, with a sample.

| Hook | Method | What it does |
| --- | --- | --- |
| `cart.transform` | `transform` | Add lines to the cart (a free gift, a bundle's parts) and set line attributes. |
| `cart.price` | `price` | Set a lower price for lines in the cart. |
| `discount.eligibility` | `eligible` | Allow or veto a discount code or automatic rule for a cart. |
| `shipping.options` | `options` | Hide, rename or reorder the shipping rates the store offers. |
| `payment.methods` | `methods` | Hide payment methods for a cart. |
| `checkout.validate` | `validate` | Block a checkout with a message the shopper reads. |
| `order.event` | `event` | React to a paid, fulfilled, delivered, refunded or cancelled order. |
| `thankyou.content` | `content` | Show offers and a short message on the order confirmation page: a product for a new cart, or a code for next time. |

Several functions may sit on one hook; they run in `position` order (set on create_store_function or enable_store_function), each handed what the previous one left. A hook with no function behaves as if functions did not exist. For anything that must always block a checkout (age checks, export rules, places you cannot ship), use a checkout rule (`upsert_checkout_rule`) rather than a function: a rule is data the store evaluates itself.

## When the platform has no feature for it

Build it as store code. The pieces work together through data: a function decides, an app keeps records and routes, and a storefront interaction presents it. New agent-authored interactions belong in section `behavior` and use only its bounded APIs. Existing owner-authored legacy section scripts are a separate, more powerful same-origin option; they are not the safe agent behavior API. The store clamps every function answer (store code names products and quantities, never prices).

- **Change what is in the cart**: `cart.transform` answers add_line actions ({ productId or variantId, quantity, gift?, required?, label? }) and line attributes. A free gift below its recorded cost needs `set_gift_exception { subject: "function:<name>", productId }` first.
- **Price it**: `cart.price` lowers line prices, adds `orderDiscounts [{ label, amountMinor }]` and a `shippingDiscount`, and answers `offers` the storefront shows as progress bars.
- **Personal codes**: mark a code `template: true`, then `mint_discount_codes`, a flow's issue_code action, a function's mint_code action, or an app granted *Make one-off discount codes* makes single-use codes that are never worth more than the template. `https://<store>/discount/<CODE>` applies one by link.
- **Show it to the shopper**: section `behavior` can use its documented bounded cart and offers APIs. It has no general app-route API, so private app data such as loyalty balances are not available to agent behaviors unless Formahand adds a documented capability. Existing trusted legacy scripts can read `api.cart.get()`, `api.offers.get()` and `api.shopper.get()`, add with `api.cart.add({ productSlug, quantity })` and call an app with `api.app(name).fetch(path)`; with the app granted *Know which signed-in shopper is using it*, `api.shopper.fromApp(name, path)` answers what the app keeps for this shopper. Without code, a cart page body places `{% widget "offers" %}`, `{% widget "progress" %}` and `{% widget "quick-add" product: settings.gift %}`, or reads `offers` and `cart.progress` (describe_sections part cart-page).
- **After the order**: `thankyou.content` shows up to three offers on the order confirmation page, a product for a new cart or a code for next time; never a charge.
- **Share data**: custom fields (a product's `pricing.breaks`, a customer's `functions.points`) are read by functions and templates alike; the event log triggers flows.

A free gift with purchase, whole:

```js
const TOTE = "tote"; // the store's own product id (list_products shows it)
const THRESHOLD_MINOR = 5000; // $50.00
const LABEL = "Free tote";

export default {
	transform(input) {
		const chosen = input.cart.lines.filter((line) => line.productId !== TOTE);
		const spent = chosen.reduce(
			(total, line) => total + line.unitPriceMinor * line.quantity,
			0,
		);
		const offer = {
			id: "free-tote",
			kind: "gift",
			label: "Free tote over $50",
			thresholdMinor: THRESHOLD_MINOR,
			productIds: [TOTE],
		};
		if (spent < THRESHOLD_MINOR) return { version: 1, offers: [offer] };
		return {
			version: 1,
			actions: [
				{ type: "add_line", productId: TOTE, quantity: 1, gift: true, label: LABEL },
			],
			offers: [offer],
		};
	},
};
```

## What the store will not accept back

Every answer is checked against what the function was handed, and anything outside these rules is dropped and reported in `clamps`:

- `cart.transform`: add_line names a catalogue id and a quantity, never a price: the line is priced from the catalogue and then only ever down. An id that is not an active item of this store is dropped (unknown-product), and so is one out of stock (unavailable). Every function together adds at most 3 lines and 10 units to one cart (over-lines, over-units). An item already in the cart is covered, not doubled, and an item the shopper removed is never added back (declined). gift prices the covered units at zero; below a recorded cost only where the merchant set a gift exception for this function and product. required keeps a line the shopper cannot remove, and is honoured only on a free gift (required-not-gift). At the checkout quote the lines are already fixed, so a function there can only cover lines that exist (not-at-quote). Attributes are plain text, five per line at most, and never overwrite one the shopper set.
- `cart.price`: A lineId the cart does not have is dropped (unknown-line): a function reprices what it was handed, it never adds merchandise. The same lineId twice: the second is dropped (duplicate-line). A price above the one the cart already had is refused (raise). This hook only ever takes money off; there is no way to add a surcharge. Below zero becomes zero (floor). Below a recorded supplier cost becomes that cost (cost), and a line already at or under cost cannot be discounted at all (at-cost). Fractional minor units round up, in the shopper's favour. orderDiscounts (at most 3, each with a label) are spread over the lines in whole minor units and never take a line below its floor (order-floor), so the payment session charges what the cart shows. shippingDiscount lowers the payment page's rates by a percent or an amount, never below zero and never raising one. A gift exception the merchant set for this function and a product lifts that product's cost floor; zero stays the floor. offers are shown, never charged: their progress is the platform's arithmetic on the discounted subtotal. Every clamp comes back in the verdict, so the merchant sees it happened.
- `discount.eligibility`: `reason` is rewritten as plain text before a shopper reads it: no markup, no links.
- `shipping.options`: A rate id the store did not offer is dropped (unknown-rate): this hook narrows an offer, it cannot make one. The same id twice: the second is dropped (duplicate-rate). Hiding every rate is refused outright (all-hidden) and the store's own list stands: a checkout with no shipping option is a checkout that cannot complete. A new label is rewritten as plain text, and a price can never be changed here, only hidden, renamed or reordered.
- `payment.methods`: A method id the store did not offer is dropped (unknown-method). Hiding every method is refused (all-hidden) and the full list stands: a checkout with nothing to pay with cannot complete.
- `checkout.validate`: `ok: true` drops any message: a checkout that passed says nothing. A refusal with no message gets 'This order cannot be placed.', and every message is rewritten as plain text before a shopper reads it.
- `order.event`: send_email may only name a template in the `templates` list you were handed; anything else is dropped (unknown-template). call_webhook may only name a subscription in the `subscriptions` list you were handed (unknown-subscription). A tag the order already carries is dropped (duplicate-tag); matching is case-insensitive. Actions past the per-event ceiling are dropped (over-budget). mint_code may only name a template code in the `codeTemplates` list you were handed (unknown-code-template); at most one per answer, within the store's daily code caps, and the code reaches this answer's send_email. tag_customer tags this order's own customer and nobody else (no-customer when there is none). No action here moves money, changes a price or cancels an order.
- `thankyou.content`: At most 3 offers on the page from every function together, in position order; the first message wins. add names a catalogue id and a quantity (1 to 10), never a price: the shopper puts it in a new cart with one click and pays for it on a new checkout. Nothing is charged and no saved card is used. An id that is not an active, in-stock item of this store drops the offer (unknown-product). code must be a code of this store a shopper can type now; any other drops the offer (unknown-code). The page shows it with the store's own apply link. Titles, texts and the message are plain text, escaped; the platform draws the block, never the function.

## Writing one

```js
export default {
	price(input) {
		return { version: 1, lines: [] };
	},
};
```

One file, no imports: bundle anything it needs into the file. Everything the function needs must be in `input` or in the file itself; it has no network, storage or clock of its own, and the same input always gives the same answer. Keep it stateless.

## The loop: build it with an agent

Nine tools, all on the `storefront:write` scope. Write scope even to read, because a listing carries your source, and there is no read-only door into somebody's code.

| Tool | Does |
| --- | --- |
| `describe_store_functions {}` | The hooks, what each is handed, what it may answer, what happens when it does not. Read this first. |
| `create_store_function { name, hook, source, position? }` | Uploads the file as a **new version** and leaves the function switched off. Returns the static checks, including hints that are not errors. `position` orders it among the hook's functions. |
| `test_store_function { name, input? }` | Runs it once against an input you supply, or a sample one, and returns what it asked for, what the store would apply, every clamp, and how long it took. Nothing is charged and the function need not be switched on. |
| `enable_store_function { name, mode?, position? }` | `on` for a test store; on a live store a token may only choose `shadow`. |
| `disable_store_function { name }` | Switches it off. Always allowed. |
| `list_store_functions {}` | Every function, with counters and the last failure. |
| `get_store_function { name, version? }` | One function in full: source, versions, counters, and its shadow results. |
| `rollback_store_function { name, version }` | Points it back at an earlier version. Keeps the current status. |
| `delete_store_function { name }` | Removes it. The versions are kept so the store can still answer "what was running on the twelfth". |

The loop is: read `describe_store_functions` → write → `create_store_function` on the **test** store → `test_store_function` until the numbers are right → `enable_store_function` → place a test order → promote → shadow on live → the merchant switches it on.

Uploading the same name again adds a version and switches the function **off**, so a fix is never live before you have run it. Rollback is the exception: it keeps the status, because rollback is the lever you pull when something is wrong right now.

### Reading a `test_store_function` answer

```json
{
  "hook": "cart.price",
  "outcome": "ok",
  "wallMs": 4,
  "applied": [
    { "lineId": "cart-0", "fromUnitPriceMinor": 1200, "toUnitPriceMinor": 1080,
      "unitDiscountMinor": 120, "reason": "10% off drinkware (3 in the cart)" }
  ],
  "clamps": [],
  "fellBack": false
}
```

`clamps` is the field to read first. An empty list means the store took your answer as given. An entry means it did not, and why, `cost` and `at-cost` mean you went under a recorded cost, `raise` means you asked for more than the line already cost, `unknown-line` and `unknown-rate` mean you named something that is not there, `all-hidden` means your answer would have left the checkout with nothing to offer. A function whose clamps list is never empty is a function that does not yet understand its own store.

## Versions, shadow mode and rollback

**Every upload is a new version**, and a function is a pointer at one of them. `get_store_function` lists the last ten with who uploaded each and when. Rolling back moves the pointer: it takes effect immediately.

**Shadow mode** is how a function earns a live store. In shadow, a function is invoked on real traffic, with the same input and the same budget it would have live, and its answer is **recorded and thrown away**: every hook takes its fallback exactly as if the function were not there. Then `get_store_function` tells you what it would have done, how many carts it would have changed, by how much on average, how many checkouts it would have refused.

For anything that touches a price, shadow mode should be where a function lands and stays for a week. The dashboard says so, and an agent that cannot yet quote those numbers should not be asking for the switch.

## Test stores, live stores, and who may switch what

A function exists **per environment**. A version on your test store and a version on your live store are separate things, and `promote_to_live` copies a function across **switched off**.

On a test store: as many functions as you like, none of them counted, walled or billed, and no shopper ever sees them.

On a live store, three gates:

1. **Readiness**, your storefront is serving, you have something to price, and payments are set up. `describe_readiness` lists the steps; the dashboard shows the same checklist.
2. **The allowance**, your plan includes a number of live functions and a number of invocations a month. Past the included invocations, the overflow is billed on your statement and needs a payment method; without one, nothing new is switched on.
3. **A person.** A token may put a function into **shadow** mode on a live store and may never take it out. Moving a function from shadow to live is done in the dashboard, under **Online Store → Apps & functions**, by someone signed in.

That third gate is the point of the whole design, and it is what lets everything else be permissive. An agent can write a function, test it, promote it, run it on real traffic and show you exactly what it would have done, and then you click.

## Rules of thumb

- **Read `describe_store_functions` before you write anything.** The fallback is a property of the hook, and it changes how you write the function.
- **Compute the answer, not the difference.** The store does the subtraction and the clamping.
- **Never trust the clock.** It does not move. Anything time-dependent should come from `input.at`, and a rule that needs a real calendar belongs in a scheduled discount, not a function.
- **Keep it pure.** Same input, same output, every time. It is the only way to explain a price to a shopper.
- **Fail small.** A function that returns the empty answer when it is unsure is a function that never surprises anybody.
- **Shadow first, for anything that touches money.** A week of real traffic costs three sentences of reading and buys you the whole argument.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `describe_store_functions` | `storefront:write` | The hooks a store function can run at, what each one is handed (`sampleInput`), what it may answer (`sampleOutput`, a worked example that validates against the contract), what the store will refuse to accept back (`clamps`), and what it does when… |
| `list_store_functions` | `storefront:write` |  |
| `get_store_function` | `storefront:write` | One function in full: its source, its versions (newest first, with who uploaded each and when), its counters, its last failures, and, when it is in shadow mode, what it would have done on recent traffic. |
| `create_store_function` | `storefront:write` | Uploads one ES module as a new version of a store function and leaves it switched off. |
| `test_store_function` | `storefront:write` | Runs one function once against an input you supply, or a sample input for its hook if you supply none, and returns what it asked for, what the store would actually have applied, every clamp the store had to make, and how long it took. |
| `enable_store_function` | `storefront:write` | Switches one function on. |
| `disable_store_function` | `storefront:write` | Switches one function off. |
| `rollback_store_function` | `storefront:write` | Points a function back at one of its earlier versions. |
| `delete_store_function` | `storefront:write` | Removes one function from this environment. |

## Read more

- https://formahand.com/docs/store-functions
- https://formahand.com/docs/checkout-rules
