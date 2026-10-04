# Apps

An app is a feature of the merchant's own on their store: one JavaScript file and a manifest, served at `/apps/<slug>/` on the store's own address, with only the grants the merchant ticks on an install screen. Build it on the test store, ask the merchant to install it, then promote.

## What an app is

One JavaScript file and a manifest.

```js
export default {
	async fetch(request, env) {
		const url = new URL(request.url);
		if (url.pathname === "/hello")
			return Response.json({ message: `Hello from ${env.APP}` });
		return Response.json({ error: "Not found" }, { status: 404 });
	},
};
```

That file is served at `https://<your store>/apps/<name>/`, and the path it sees is the part after the app's own name: the code above answers `/apps/hello/hello`.

The manifest says everything else:

```jsonc
{
	"manifest": 1,
	"slug": "reviews",                 // owns /apps/reviews/, its tables and its files
	"name": "Reviews",
	"version": "1.0.0",
	"description": "Product reviews, asked for a week after delivery.",
	"scopes": ["orders:read", "products:read", "storage:db"],
	"tables": ["reviews"],
	"migrations": ["CREATE TABLE IF NOT EXISTS app_reviews_reviews (…)"],
	"outbound": [],                    // exact hostnames; none means it can call nothing
	"secrets": [],                     // by name, each one its own question
	"emailTemplates": [],              // templates the app itself sends
	"functions": [ /* … */ ],        // store functions it brings, switched off
	"schedules": [],                   // when it wakes on its own
	"blocks": [ /* … */ ],
	"settings": [ /* … */ ]
}
```

An app is a **single file**. Formahand never installs a dependency, never runs a package manager and never builds anything for you: if your app needs a library, bundle it into the file before you upload it, exactly as a store function does.

## What an app is handed

```js
export default {
	async fetch(request, env) {
		const orders = await env.FORMAHAND.json("/api/orders?status=paid&limit=10");
		const rows = await env.APP_DB.prepare("SELECT * FROM app_x_things").all();
		return Response.json({ orders: orders.body, rows: rows.results });
	},
};
```

| On `env` | What it is | When |
| --- | --- | --- |
| `FORMAHAND` | Your store's own API, with the grants you ticked: `/api/…` and nothing else on Formahand. `fetch(path, init)` and `json(path, init)`. | always |
| `SETTINGS` | What you typed on the install screen. | always |
| `STORE`, `MODE`, `APP`, `VERSION` | Your store's id, `live` or `test`, the app's name and version. | always |
| `INVOCATION` | `request` when one of its addresses was called, `schedule` when a timer woke it. | always |
| `APP_DB` | The app's own records. | *Keep its own records* granted |
| `APP_FILES` | The app's own files. | *Keep its own files* granted |
| `SECRETS` | Only the secrets you granted, by name, lower-cased. | per name granted |

**`env.FORMAHAND` takes a path, never an address.** `"/api/orders?limit=10"` works; anything with a host in it is refused.

```
x-formahand-shopper: v_9bK2qf1TxsA7Lm0dQzR4Yw
```

That is a **visitor id**.

If one of those is ever something an app can ask for, it will be a line on the install screen with a sentence beside it, ticked by you.

`x-formahand-app-base` is the other header: `/apps/<name>`, so an app can write links back into its own addresses without guessing which of your domains a shopper arrived on.

## The manifest

Top-level fields, generated from the manifest schema (`manifest`, `slug`, `name`, `version` and `description` are required; the rest default to empty):

| Field | Type | Required |
| --- | --- | --- |
| `manifest` | 1 | yes |
| `slug` | string | yes |
| `name` | string | yes |
| `version` | string | yes |
| `description` | string | yes |
| `author` | object | no |
| `scopes` | enum[] | no |
| `tables` | string[] | no |
| `migrations` | string[] | no |
| `outbound` | string[] | no |
| `secrets` | string[] | no |
| `emailTemplates` | string[] | no |
| `functions` | object[] | no |
| `blocks` | object[] | no |
| `sections` | any[] | no |
| `settings` | object[] | no |
| `schedules` | object[] | no |
| `limits` | object | no |

## Grants

The complete list of what an app can ask for, with the words the merchant reads on the install screen. Ask for as little as you can: nothing is ticked in advance.

| Grant | The merchant reads | What it means |
| --- | --- | --- |
| `products:read` | Read your products | Titles, prices, images, stock and custom fields, including your costs and the drafts shoppers cannot see. |
| `products:write` | Change your products | Create and edit products, variants and custom fields, with every change recorded against the app. |
| `orders:read` | Read your orders (personal data) | Order lines, totals, status and tags, with the buyer as an id and a country; tick this one deliberately, as an order is about a person. |
| `orders:write` | Change your orders | Add notes and tags and move an order through its status, never changing money or stock directly. |
| `orders:refund` | Refund an order | Ask your store to refund an order, which moves money back to the buyer; tick this one deliberately. |
| `gift_cards:issue` | Issue gift cards | Make gift cards your shop owes until they are spent, within your agents' money limits; tick this one deliberately. |
| `discounts:issue` | Make one-off discount codes | Make single-use codes from a template discount, never worth more than it and at most 500 a day; tick this one deliberately. |
| `settings:read` | Read your store profile | Your store's name, legal name, addresses and timezone, and not your plan, automations, templates or keys. |
| `settings:read_all` | Read all your settings | Everything an agent may read in settings, from shipping to email templates and your plan, but never a secret's value. |
| `settings:write` | Change your settings | Change shipping, discounts, rules, integrations, automations, webhooks and email templates; tick this one deliberately. |
| `customers:read` | Read your customers, personal data | Names, emails, phones, addresses and order history, the one grant that hands an app real personal data; tick this one deliberately. |
| `customers:write` | Change your customers | Add customers and change their tags, notes, custom fields and groups, without reading who they are. |
| `customers:export` | Download your customer list, personal data | Download every customer in one file, with names, emails, phones and addresses; tick this one deliberately. |
| `email:send_template` | Send one of your own emails | Send one of your own email templates, with your wording, to an order's buyer, to you or to a confirmed sign-up. |
| `marketing:send` | Send email campaigns | Send campaigns to your newsletter subscribers from your own address, in your name; tick this one deliberately. |
| `events:read` | Read your store's event log, personal data | Every change in your store, where order events carry the buyer's email and address; tick this one deliberately. |
| `storefront:read` | Read your storefront draft | Your storefront draft, with theme settings, sections, pages and menus, including what you have not published. |
| `storefront:write` | Edit your storefront draft | Change your storefront draft and save sections and functions of its own, which reach shoppers only when you publish. |
| `storefront:publish` | Publish your storefront | Put the draft live so shoppers see it, whoever made the changes; tick this one deliberately. |
| `domains:read` | Read your domains | Your store's addresses and the DNS records each one needs, and nothing about your email or account. |
| `domains:write` | Change your domains | Add, check and remove domains and pick the primary one shoppers are sent to; tick this one deliberately. |
| `payments:write` | Change your payment features and move your balance | Switch payment features and send your balance to your bank, within your agents' limits; tick this one deliberately. |
| `webhooks:call` | Trigger one of your own event subscriptions | Not available yet. |
| `storage:db` | Keep its own records | Its own tables under your app allowance, holding the app's records, not your store's, so never an order. |
| `storage:files` | Keep its own files | Its own place for files, such as review photos or return labels, under your app file allowance. |
| `schedule:cron` | Run on a timer you set | Wake the app at most every 15 minutes with no shopper waiting, each run counting as one function visit. |
| `shopper:identify` | Know which signed-in shopper is using it (personal data) | Tell the app a fixed id for each signed-in shopper, so it recognises them on every visit; tick this one deliberately. |
| `shopper:email` | Know the signed-in shopper's email address, personal data | Tell the app the signed-in shopper's email address, which needs the id grant too; tick this one deliberately. |

## Store functions an app brings with it

An app is a feature with its own addresses; a [store function](https://formahand.com/docs/store-functions) is one answer to one question inside a cart, a checkout or an order. Some features need both, the Reviews add-on is the example, because "ask for a review a week after the parcel lands" is a question about an order and nothing to do with a web address.

So an app may carry store functions in its manifest, source and all, and installing it registers them the same way your agent would.

```jsonc
"functions": [
	{ "name": "review-request", "hook": "order.event", "source": "export default { event(input) { … } }" }
]
```

Three things about that, and the first is the one that matters:

**They arrive switched off.** Installing an app never puts code on your cart, your checkout or your orders. Several functions may sit on one hook: an app's run after your own by default (position 100), and you can reorder them. The function appears on **Online Store → Apps & functions** beside the ones you wrote yourself, off, and you switch it on there, and on your live store only you or an admin can, never a token. That is the same line every function is held to, and it is why bringing one costs no line on the install screen: an app that brings a function has not been granted anything, it has offered you something to turn on.

**They run as your store, not as the app.** A function is handed its input by Formahand and answers with actions, a tag, a note, one of your own email templates, one of your own event subscriptions. It never holds the app's token, never sees the app's records, and can name nothing you do not already own. An app with no grants at all can still bring one, and a grant you untick changes nothing about it.

**They go when the app goes.** Uninstalling an app deletes the functions it registered, so switching an app off is not a thing that leaves code behind on your orders.

## Showing something on your storefront

An app never puts JavaScript on a shopper's page. That is the same rule custom code follows and apps do not get an exception to it. What an app provides instead is a **block**, and the store draws it.

There are two kinds.

**A list block** gives a template, ordinary markup with `{{field}}` placeholders, and an address of the app's own to read JSON from. Your store fetches the JSON, fills the template in with every value escaped as text, and shows the result.

```jsonc
{
	"id": "list", "title": "Reviews", "kind": "list",
	"slots": ["product.below"], "path": "list", "itemsKey": "reviews",
	"template": "<blockquote><p><strong>{{stars}}</strong> {{author}}</p><p>{{body}}</p></blockquote>",
	"empty": "<p>No reviews yet.</p>"
}
```

**A form block** gives a list of fields, and your store draws the form itself and posts it to the app as JSON. An app never ships a form, because a form is the one piece of markup that can send a shopper's typing somewhere. A form block can also wait on a link: `requireParam` means it stays hidden until the address carries that query parameter, which is how a link in an email opens a form only the person who was sent it can use.

```jsonc
{
	"id": "write", "title": "Write a review", "kind": "form",
	"slots": ["home.below"], "path": "submit", "requireParam": "review",
	"fields": [
		{ "name": "rating", "label": "How many stars?", "type": "rating", "required": true },
		{ "name": "body", "label": "What did you think?", "type": "textarea", "required": true }
	]
}
```

A block on a product page is told which product it is on. A block whose app is uninstalled shows nothing, rather than a stale copy of what the app used to say.

## Scheduled runs

Most of what an app does happens because somebody asked for it: a shopper opened a page, a block fetched some JSON, a form was posted. A **timer** is the other kind of work, the weekly digest, the nightly tidy-up, the hourly check on somebody else's system, and it is the thing an app cannot arrange for itself.

An app declares its timers in its manifest, you grant them on the install screen, and Formahand wakes the app at those times.

```jsonc
"scopes": ["orders:read", "storage:db", "schedule:cron"],
"schedules": [
	{ "name": "weekly-digest", "every": "1d", "handler": "digest" },
	{ "name": "poll-supplier", "every": "15m" }
]
```

`every` is one of `15m`, `1h`, `6h` and `1d`.

**What the app writes.** A timer calls an export of the app's own, beside its `fetch`:

```js
export default {
	async fetch(request, env) { /* its pages */ },
	async scheduled(invocation, env, ctx) {
		// invocation = { kind: "schedule", name: "poll-supplier",
		//                at: "2026-09-24T09:15:00.000Z", runId: "…:2026-09-24T09:15:00.000Z" }
		const orders = await env.FORMAHAND.json("/api/orders?status=paid&limit=50");
		// …
	},
};
```

`handler` in the manifest names another export instead of `scheduled`, which is how one app keeps a fifteen-minute poll and a daily digest apart. Whatever the handler returns is ignored unless it is a `Response`; what is recorded is whether it threw.

**A run can happen twice, and the app is told which run it is.** A timer runs **at least once** per slot, not exactly once.

So every run carries an id. `invocation.runId` names the *slot* being served, not the moment the run started, which means the first attempt and the second attempt at the same slot carry the **same** id, and the next slot carries a different one. An app doing anything that must not happen twice, sending a message, taking a payment, posting to somebody else's system, writes the id down before it does the work and does nothing at all when it sees that id again:

```js
async scheduled(invocation, env) {
	const done = await env.APP_DB.prepare(
		"INSERT OR IGNORE INTO digest_runs (run_id) VALUES (?)",
	).bind(invocation.runId).run();
	if (!done.meta.changes) return; // this slot has already been done
	// … the work that must not happen twice
},
```

`invocation.at` is not that key. It is the clock of the moment your app was woken, so two attempts at one slot have different `at`s. *Run it now* is not a repeat either: it gets an id of its own, so testing a fix by hand never makes the real run look like something already done.

**A timer gets the same everything a page does.** The same grants, the same records and files, the same secrets, the same addresses it may call, the same 256 KB and the same thirty seconds.

**What you see, and what you can do.** Every installed app's card lists its timers: how often each runs, when it last ran and how that went, when it runs next, and two buttons. *Switch it off* stops one timer and leaves the app's pages working.

Your test store runs its timers too, and they are never billed and never walled, like everything else in a sandbox. And because an app on your test store and an app on your live store are separate things, **promoting to live does not bring a timer with it**: apps are installed per environment, and so are the timers they came with.

## The loop: build one with an agent

| Tool | Does |
| --- | --- |
| `describe_apps {}` | Every grant in plain words, what an app is handed, what it can never do, the limits. Read this first. |
| `list_addons {}` | The catalogue: what each add-on does, what it asks for, and what your store has already done with it. Read this **before writing an app that already exists**, and read its source before writing one that nearly does. |
| `create_app { manifest, source }` | Stores it as a draft. Nothing is granted and nothing is served. |
| `update_app { manifest, source }` | A new version. Asks for nothing new → live now. Asks for more → back to waiting for you. |
| `list_apps {}` | Every app, its state, what it was granted and how full the storage is. |
| `get_app { name }` | One app in full, with the history of every grant and a day-by-day rollup of what it touched. |
| `request_install { name }` or `{ catalog }` | Asks you to install one of your own apps, or one of ours from the catalogue. As far as a token goes, for either. |
| `uninstall_app { name, removeData? }` | Switches it off now. Always allowed. |
| `app_settings { name, settings? }` | Reads or changes what you configured. |
| `list_app_records { name, table, where?, limit? }` | Reads rows from one of the app's own tables. |
| `set_app_record { name, table, id, set?, remove? }` | Changes or removes one row, how a review is approved or hidden. |
| `list_app_schedules { name? }` | Every timer the store's apps run on: how often, last run and how it went, next run, on or off, and the runs spent today. |
| `run_app_schedule_now { name, schedule }` | Runs one timer immediately without moving its next slot. |
| `disable_app_schedule { name, schedule }` / `enable_app_schedule { name, schedule }` | Stops one timer, or starts it again from its next slot. |

Nothing on that list installs an add-on or takes one over, and nothing ever will: both are decisions about your store's data and your store's code, and they are made by a person on a screen.

Nothing on that list **creates** a timer, and nothing ever will: a timer exists because a manifest declared it and you ticked *Run on a timer you set*. A tool that could add one would be a tool that widened a grant after you gave it.

The loop is: `describe_apps` → write → `create_app` on your **test** store → open its addresses → `request_install` → you approve → place its blocks → `promote_to_live` → you approve it there too.

## Test store, live store

An app exists **per environment**. The one on your test store and the one on your live store are separate things, with separate records, separate grants and separate settings. **App data never travels between them**, in either direction: an app's records are like your orders, not like your products.

On a test store: as many apps as you like, nothing counted, nothing walled, no shopper ever sees them.

## Limits

| Budget | Value |
| --- | --- |
| The file | 1 MB |
| Time to answer | 30 seconds |
| Processing time | 200 ms |
| Calls out | 20 per request |
| Answer | 256 KB |
| Blocks a store may place | 8 |
| Timers an app may declare | 5 |
| Shortest a timer may be | 15 minutes |
| Processing time, an app that runs on a timer | 500 ms, if its manifest asks |
| Calls out, an app that runs on a timer | 50 per run, if its manifest asks |

Going over any of them is an error the app's caller sees, never something the store hides. An answer over 256 KB is **refused with a message that says so** rather than quietly cut short, an app that returns half a page and no explanation is the hardest kind of bug to find. If an app has more than that to say, it pages.

(A custom block you write yourself stays at 32 KB of markup. That is a field you type into; an app's answer is generated, and a list of fifty reviews is legitimately larger.)

## Rules of thumb

- **Ask for as little as you can.** The install screen is a list somebody reads, with nothing ticked in advance. An app that wants to read your customers had better be able to say why in one sentence, and most apps that think they need it need *Read your orders*, which tells them an order exists, what was on it and where it got to, without telling them who bought it.
- **Do the slow thing off the request.** A shopper is waiting for 30 seconds at most; a call to somebody else's server belongs behind a page that is already loaded, not in front of one.
- **Check the order, not the link.** A link proves who was sent it. Whether they bought the thing is a question for your store's API.
- **Handle the allowance.** Writing can fail with a sentence about the allowance. Show it; do not swallow it.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `describe_apps` | `storefront:write` | What an app is, every grant it can ask for in the words the merchant reads, which of them is personal data, what an app is handed at run time, what it can never do, the allowances per plan, and `publishing.canPublish`, whether Formahand could last… |
| `list_apps` | `storefront:write` | Every app this store has in this environment: its name, version, whether it is a draft, waiting for the merchant, installed or switched off, what it was granted, and what it has been doing, plus `pending`, the add-on installs a token has asked for… |
| `get_app` | `storefront:write` | One app in full: its manifest, its source, everything it asks for, everything it was granted, its settings, and a day-by-day rollup of what it touched. |
| `create_app` | `storefront:write` | Stores one app, a manifest and a single JavaScript file, as a draft. |
| `update_app` | `storefront:write` | Uploads a new version of an app. |
| `list_addons` | `storefront:write` | The Add-ons catalogue: the apps Formahand ships as source, reviews, gift cards, a wishlist, back-in-stock, a contact form, questions and answers, a size guide, what each one does, what it asks for in the words the merchant reads, and what this… |
| `request_install` | `storefront:write` | Asks the merchant to install an app: one this store wrote (`name`), or one of Formahand's add-ons (`catalog`, from list_addons). |
| `withdraw_install_request` | `storefront:write` | Takes back an install request: { requestId } from request_install or from list_apps `pending`. |
| `uninstall_app` | `storefront:write` | Switches an app off now: it stops answering its addresses, its access to the store is revoked and every grant it had is withdrawn. |
| `app_settings` | `storefront:write` | Reads or changes what the merchant configured for an installed app, the fields its manifest declares. |
| `list_app_records` | `storefront:write` | Reads rows from one of an installed app's own tables, the tables it declared in its manifest and nothing else. |
| `set_app_record` | `storefront:write` | Writes or removes one row in an installed app's own table, by its id, giving a Questions and answers add-on its first entry, approving a review, hiding one, deleting a row an app collected. |
| `list_app_schedules` | `storefront:write` | Every timer this store's installed apps run on: what each is called, how often it runs, when it last ran and how that went, when it runs next, and whether it is switched on. |
| `run_app_schedule_now` | `storefront:write` | Runs one of an app's timers immediately, without waiting for its next slot, how you check that a fix worked. |
| `disable_app_schedule` | `storefront:write` | Switches one of an app's timers off. |
| `enable_app_schedule` | `storefront:write` | Switches one of an app's timers back on, from its next slot. |

## Read more

- https://formahand.com/docs/apps
