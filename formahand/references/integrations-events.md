# Integrations and events

How a store talks to other systems and reacts to what happens in it: the event log, subscriptions that deliver signed webhooks, flows that act inside the store, schedules and inbound URLs, the secrets vault, integration modules and connector recipes. Prefer a recipe or a flow over a server of your own.

## In short

Every change is an event `<resource>.<verb>` (`describe_events` for the vocabulary and filter grammar, `list_events` / `get_event` to read). `create_subscription` sends matching events as signed CloudEvents to a webhook or agent endpoint (the envelope carries `mode`); `list_deliveries` / `retry_delivery` follow them.

Flows react inside the store without a server: `describe_flows` → `create_flow` (topics + filter + ordered actions: `send_email` with a store template, `notify_merchant`, `tag_order`, `webhook`, `wake_agent`), `test_flow` dry-runs, `set_flows_enabled` is the kill switch. Email templates: `list_email_templates`, `upsert_email_template`, `preview_email_template`, `reset_email_template`; editing a system key changes what the platform sends.

A flow may also run on a clock (`schedule`: `every` 15m/1h/6h/1d, optional `at`/`timezone`) or expose an inbound URL (`create_flow_hook`, bearer or HMAC). `set_secret` keeps a write-only credential, usable only as `{{ secrets.name }}` in a `webhook` url or header or a `wake_agent` bearer; logs show `***`.

Flows react to events, run on a schedule (`schedule: { every, at?, timezone? }`) or receive an inbound hook (`create_flow_hook`: bearer or HMAC, per-flow URL). `install_recipe` sets up a documented pattern (dropshipping, print on demand, nightly report, low stock, chat alerts, sheet sync), disabled until the owner enables it.

## Events

One event per write, kept in your own store:

```
id           a ULID, time-ordered, so sorting by id sorts by time and an id works as a cursor
type         <resource>.<verb>, or a semantic alias (order.paid)
resource     order, product, …
resourceId
occurredAt
actor        who wrote
data         the object after the write; for a deletion, the last known object
changed      the top-level field names that changed ([] for created and deleted)
previous     the previous values of the changed fields only
```

- **Verbs.** `created` (no object before), `updated` (both), `deleted` (no object after). `storefront.published` uses an explicit verb of its own. A write whose before and after views are equal records nothing, so a repeated idempotent call (a redelivered payment notification, a second "mark paid") is silent.
- **Diff.** Top-level fields are compared structurally (nested objects and arrays compared by value, key order ignored). `null` and a missing field are equal. `changed` is sorted; `previous[field]` holds the value before the write (`null` when the field did not exist).

## Subscriptions

A subscription is a standing request: these topics, optionally this filter, delivered to this target.

- **Limits.** 20 subscriptions per store (409), 1–20 topics, name ≤ 80, filter ≤ 1000 characters.
- **Topics.** `<resource>.<verb>` where `*` matches one whole segment and the pattern needs as many segments as the type: `order.updated`, `order.*`, `*.created`, `*`. Aliases match like any type (`order.delivered`). Pattern syntax `^[a-z0-9_*]+(\.[a-z0-9_*]+)?$`.
- **Filter.** An optional predicate, validated at create and update; a syntax error answers 400 `{ error, field: "filter", position }`.
- **Targets.** `{ kind: "webhook", url }` gets a signing secret, returned once when the subscription is created. `{ kind: "agent", url, bearer? }` posts a CloudEvents batch and, when a bearer is set, `Authorization: Bearer …`; the view shows only `bearerSet`. Both kinds are signed the same way. Your store's own echo sink is a valid target.
- **Enabled / paused.** `enabled: false` receives nothing. A subscription whose deliveries keep failing is switched off with a `disabledReason`, and `subscription.updated` is recorded.

## Predicates

A small expression language, evaluated the same way in your store and in the dashboard form, so a filter that validates in the editor behaves identically on a real event.

```
expression  := or
or          := and ("or" and)*
and         := not ("and" not)*
not         := "not" not | comparison
comparison  := primary (compareOp primary)?        -- no chaining: a == b == c is an error
compareOp   := "==" | "!=" | "<" | "<=" | ">" | ">=" | "in" | "contains" | "startsWith" | "endsWith"
primary     := literal | path | list | "(" expression ")"
list        := "[" (primary ("," primary)*)? "]"
literal     := string | number | "true" | "false" | "null"
string      := "..." or '...' with escapes \" \' \\ \n \t \r
number      := integer or decimal, optionally negative (-3, 2.5)
path        := root ("." segment | "[" integer "]")*
root        := type | actor | resource | changed | data | previous
```

Fields: `type` (`"order.paid"`), `actor`, `resource`, `changed` (array of field names), `data.<path>` (the object after the write; `data.items[0].sku` and `data.items.0.sku` are the same), `previous.<path>` (previous values of changed fields). Unknown roots are a parse error.

Semantics: keywords are case-sensitive and there is no `=`, `&&`, `||` or `!` (each gets a hint: `Unknown operator "="; use == instead of =`). Missing paths are `undefined` and never throw; `data.x == null` is true when `x` is missing or null. `==`/`!=` compare structurally (JSON-style deep equality). `<` `<=` `>` `>=` are true only when both sides are numbers or both are strings (lexicographic). `x in list` needs an array on the right (a literal or a path such as `data.tags`). `a contains b` is array membership or substring; `startsWith`/`endsWith` need two strings. `not`, `and`, `or` and the result use JavaScript truthiness (`false`, `null`, `undefined`, `0`, `NaN`, `""` are false; empty arrays and objects are true); `and`/`or` short-circuit. The segments `__proto__`, `constructor` and `prototype` always resolve to `undefined`; arrays accept only integer segments. Limits: 1000 characters, 400 tokens, 32 levels of nesting. An empty filter matches everything. Errors carry a 0-based `position`: `Unexpected token ")" at position 8.`, `Unterminated string`, `Chained comparisons are not allowed`, `Missing closing parenthesis`, `Unknown field \`foo\`; use type, actor, resource, changed, data.<path> or previous.<path>`.

Examples (all returned by `describe_events`):

| Filter | Meaning |
| --- | --- |
| `type == "order.paid"` | a paid order |
| `changed contains "trackingStatus" and data.trackingStatus == "delivered"` | parcel delivered |
| `resource == "product" and data.inventory <= 3 and changed contains "inventory"` | low stock |
| `type == "customer.created"` | new customer |
| `type == "product.updated" and previous.status == "archived" and data.status == "active"` | product published again |
| `type == "order.paid" and data.totalMinor >= 10000` | a big order |
| `data.shippingAddress.country in ["US", "CA"] and not (actor startsWith "system:")` | human- or agent-made, North America |
| `type == "order.refunded" or type == "order.cancelled"` | money went back |

"Parcel delivered → send the buyer an email" as one subscription, pointing at the agent or function that sends it:

```json
POST /api/subscriptions
{
  "name": "Delivered → thank-you email",
  "topics": ["order.updated"],
  "filter": "changed contains \"trackingStatus\" and data.trackingStatus == \"delivered\"",
  "target": { "kind": "agent", "url": "https://automations.example.com/formahand", "bearer": "…" },
  "enabled": true
}
```

The same as an MCP call: `create_subscription { topics: ["order.updated"], filter: 'changed contains "trackingStatus" and data.trackingStatus == "delivered"', target: { kind: "webhook", url } }`. Because the filter reads `changed`, a later re-check that only refreshes `trackingCheckedAt` does not match, and subscribing to `order.delivered` instead needs no filter at all.

## Receiving a delivery

Each matching (subscription, event) pair becomes one **delivery**: `{ id, subscriptionId, eventId, attempt, status: "pending" | "delivered" | "failed", nextAttemptAt, lastAttemptAt, responseStatus, error }`.

**Envelope.** CloudEvents 1.0, structured JSON:

```json
{
  "specversion": "1.0",
  "id": "01K5…",                                  // the event id
  "source": "https://shop.example.com",           // the store's primary hostname
  "type": "order.updated",
  "time": "2026-09-14T10:12:03.412Z",
  "subject": "order/2d767824-…",                  // <resource>/<id>
  "datacontenttype": "application/json",
  "mode": "live",                                 // extension attribute: "live" or "test" (a sandbox's events)
  "data": {
    "object": { …the order… },
    "changed": ["trackingCheckedAt", "trackingEvents", "trackingStatus"],
    "previous": { "trackingStatus": "in_transit", … },
    "actor": "system:tracking",
    "store": { "id": "[redacted]", "name": "Formahand Supply" }
  }
}
```

Webhook targets receive that object with `Content-Type: application/json`; agent targets receive a one-element array with `Content-Type: application/cloudevents-batch+json` and, when set, `Authorization: Bearer <bearer>`.

**Headers** on every attempt: `webhook-id` (the delivery id, stable across retries: use it as the idempotency key), `webhook-timestamp` (unix seconds), `webhook-signature: v1,<base64 HMAC-SHA256(secret, "<id>.<timestamp>.<body>")>` per [Standard Webhooks](https://www.standardwebhooks.com/), the legacy `X-Formahand-Signature: sha256=<hex HMAC-SHA256(secret, body)>`, `X-Formahand-Event` (the type), `X-Formahand-Delivery` (the delivery id) and `User-Agent: Formahand-Events/1`. The key is the secret's UTF-8 bytes as given (64 hex characters).

Verification in Node:

```js
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyFormahand(secret, headers, rawBody) {
  const id = headers["webhook-id"], ts = headers["webhook-timestamp"];
  if (!id || !ts || Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false; // 5-minute tolerance
  const expected = `v1,${createHmac("sha256", secret).update(`${id}.${ts}.${rawBody}`).digest("base64")}`;
  const ok = (headers["webhook-signature"] ?? "").split(" ").some((candidate) =>
    candidate.length === expected.length && timingSafeEqual(Buffer.from(candidate), Buffer.from(expected)));
  // Legacy header, if you still verify the old way:
  const legacy = `sha256=${createHmac("sha256", secret).update(rawBody).digest("hex")}`;
  const legacyOk = headers["x-formahand-signature"] === legacy;
  return ok || legacyOk;
}
// Then: if (seen.has(headers["webhook-id"])) return 200; seen.add(...); handle(JSON.parse(rawBody));
```

Receivers answer any 2xx; anything else is a failed attempt.

## Flows

A flow:

```json
{ "name": "Delivered → thank-you email", "enabled": true,
  "topics": ["order.delivered"], "filter": "",
  "actions": [{ "type": "send_email", "template": "order-delivered", "to": "buyer" }] }
```

- **Limits.** 20 flows per store (409), 1–20 topics, name 1–80, filter ≤ 1000 characters, 1–10 actions.
- **Topics and filter** are the subscription vocabulary and predicate language of [events](https://formahand.com/docs/events#4-subscriptions), validated the same way (400 `{ error, field: "filter", position }`).
- **Actions** run in order. Every string field is a template rendered in text mode against the context of §4; `when` is an optional per-action predicate (same grammar as `filter`; false → the action is `skipped` with `when: false`).

| Action | Fields | Does | Skipped when |
| --- | --- | --- | --- |
| `send_email` | `template` (key, must exist at save time), `to`, `cc?` | Renders the template and hands one email per recipient to Formahand to send | no recipient resolves |
| `notify_merchant` | `subject`, `text` (plain text; the HTML is the escaped text in a `<div>`) | Emails the store owner | Formahand has no owner address for the store |
| `add_order_note` | `text` | Appends a line to the order's `note` (cut at 2000) and records `order.updated` | not an order event; order missing |
| `tag_order` | `tags[]` (1–10) | Merges the tags into the order's tags, records `order.updated` | not an order event |
| `tag_customer` | `tags[]` | Merges the tags into the customer named by `data.email` (customer events) or `data.customerEmail` (order events), records `customer.updated` | no email on the event; no customer record for it |
| `webhook` | `url` (a template; must pass the outbound URL rules of [events](https://formahand.com/docs/events#12-outbound-policy) after rendering), `secret?` | POSTs the CloudEvents envelope; with a secret, signed exactly like a subscription (`webhook-signature`, `X-Formahand-Signature`); without one, no signature headers |, (a non-2xx answer fails the action) |
| `wake_agent` | `url`, `bearer?` | POSTs a CloudEvents batch of one with `Authorization: Bearer` like an agent subscription | none |
| `set_order_status` | `status: "cancelled"` | Cancels the order exactly as the dashboard does (inventory released, `order.cancelled` recorded) | the order is not `pending` + `unfulfilled` (that 409 is a skip, not a failure) |
| `wait` | `minutes` (1–10 080) | Holds the run and lets the steps after it carry on later; an order's `data` is re-read when it resumes | `when` is false |
| `issue_code` | `template` (a discount code kept as a template), `expiresInDays?` (1–365), `bindToCustomer?` | Makes one single-use code from the template ([checkout rules](https://formahand.com/docs/checkout-rules) "Codes made from a template"). Every action after it reads it as `{{ issued.code }}`, with `{{ issued.link }}` a link that applies it and `{{ issued.expiresAt }}`; the code is kept on the step's result, so it survives a wait or a retry. With `bindToCustomer`, only the event's customer can use it | `bindToCustomer` and the event names no customer email; a day's code limit reached is a failure |

A personal code after purchase is two steps: `{ "type": "issue_code", "template": "COMEBACK", "expiresInDays": 30, "bindToCustomer": true }` then `{ "type": "send_email", "template": "come-back", "to": "buyer" }`, with `{{ issued.code }}` in the template. The same on `customer.created` gives every new customer a welcome code of their own.

`to` and `cc` accept, comma-separated: `buyer` (`data.customerEmail`, else `data.email`), `merchant` (the store owner's address), a literal address, or a template such as `{{ data.customerEmail }}`; anything that does not render to an address is reported in the skip detail. Secrets (`secret`, `bearer`) are stored with the flow like a subscription's target, shown as `secretSet` / `bearerSet` in every view and event, and kept when an update omits them for an action of the same type at the same position.

## Scheduled triggers

A flow can run on a clock as well as on events. The flow keeps everything it already has, filter, actions, run log, kill switch, and gains a `schedule`:

```json
{ "every": "15m" | "1h" | "6h" | "1d", "at": "08:00", "timezone": "Europe/Berlin" }
```

- `every` is the interval. Without `at` the runs sit on interval boundaries from the epoch (`15m` → :00, :15, :30, :45). With `at` they are anchored to that wall-clock time in `timezone` (UTC when none is given), so `{ every: "6h", at: "02:30", timezone: "Europe/Berlin" }` runs at 02:30, 08:30, 14:30 and 20:30 local. The offset is recomputed at each scheduling, so a daylight-saving change moves the next run, not the stored schedule.
- Saving a schedule adds the topic `schedule.tick` to the flow for you. Sending `schedule: null` removes the schedule and leaves the topic list alone.

A tick belongs to the flow it was raised for: no other flow, and no subscription filtering on `schedule.tick`, can pick up someone else's tick.

**Budgets.** Your plan decides how many schedules and scheduled runs you get.

| | Free | Growth | Pro |
| --- | --- | --- | --- |
| Schedules per store | 3 | 20 | 50 |
| Scheduled runs per day | 200 | 2,000 | 10,000 |

Schedules beyond the allowance (the oldest ones stay inside it) simply never tick, and the day's allowance stops further runs until midnight UTC; nothing is deleted and no data is lost. Your sandbox is never budgeted.

Every tick is kept: `GET /api/flows/:id/schedule-runs?limit=` and `list_flow_schedule_runs { id }` show `{ id, flowId, ranAt, status: queued | ok | failed | skipped, error }`, keyed by the tick's event id.

`test_flow` accepts `schedule.tick` as a sample, so a scheduled flow can be dry-run before its first slot.

## Inbound hooks

A flow can also expose a URL another system posts to, a courier's status push, a supplier's stock feed, a script on your laptop, with no helper server in between.

```
POST https://formahand.com/hooks/<storeId>/<flowId>
```

`create_flow_hook { flowId, mode }` (or Automations → Inbound URLs) returns the URL and, **once**, a secret. The flow gains the topic `hook.received` and runs against

```json
{ "type": "hook.received", "resource": "hook", "resourceId": "<flowId>", "actor": "system:hook",
  "data": { "flowId": "…", "body": { … }, "headers": { "content-type": "application/json" }, "receivedAt": "…" } }
```

so actions read the payload as `{{ data.body.reference }}` and so on.

**Bearer**, the caller sends the secret in a header:

```bash
curl -X POST https://formahand.com/hooks/STORE/FLOW \
  -H "Authorization: Bearer whsec_abc123…" \
  -H "Content-Type: application/json" \
  -d '{"reference":"ABC-123","status":"ready"}'
```

**HMAC**, the caller signs the body, so the secret is never sent:

```bash
BODY='{"reference":"ABC-123","status":"ready"}'
T=$(date +%s)
SIG=$(printf '%s.%s' "$T" "$BODY" | openssl dgst -sha256 -hmac "whsec_abc123…" -hex | sed 's/^.* //')
curl -X POST https://formahand.com/hooks/STORE/FLOW \
  -H "X-Formahand-Signature: t=$T,v1=$SIG" \
  -H "Content-Type: application/json" \
  -d "$BODY"
```

One hook per flow. `GET /api/flow-hooks` (`list_flow_hooks`) lists them; `rotate_flow_hook` issues a new secret at the same URL and the old one stops working at once; `set_flow_hook_enabled` switches receiving without losing the secret; `delete_flow_hook` removes it. Only the first characters of a secret are ever listed again (`secretPrefix`).

**In your sandbox a hook is created switched off**, so a promoted or copied integration cannot start receiving before you say so; turn it on with `set_flow_hook_enabled { enabled: true }` or the Inbound URLs tab.

## The secrets vault

Your store keeps credentials for other people's systems in a write-only vault.

| | |
| --- | --- |
| Name | `^[a-z][a-z0-9_]{1,40}$`, lowercase letters, digits and underscores, e.g. `warehouse_api_key` |
| Value | up to 4 KB |
| Count | 50 per store, per environment |
| Read back | never, not by a route, not by a tool, not in a run log |

`PUT /api/secrets { name, value }` / `set_secret` stores or replaces one; `GET /api/secrets` / `list_secrets` returns names and timestamps only (`createdAt`, `updatedAt`, `lastUsedAt`); `DELETE /api/secrets?name=` / `delete_secret` removes one. `describe_secrets` explains the rules to an agent. If the vault is not available for your store yet the route answers 503 with the reason.

**Where a secret may be used.** Only where it leaves the store for a system you chose:

- a `webhook` action's `url`
- a `webhook` action's `headers`, e.g. `{ "Authorization": "Bearer {{ secrets.warehouse_api_key }}" }` (up to 5 headers; the envelope's own headers cannot be overwritten)
- a `wake_agent` action's `bearer`

`{{ secrets.* }}` anywhere else, an email, a note, a tag, a filter, is refused when the flow is saved, with the field named. An action that refers to a secret your store does not have fails its run with `missing_secret`.

**Redaction.** A run log and a dry run keep the rendered request, with every stored value replaced by `***`. `lastUsedAt` is the only trace a use leaves.

`promote_to_live` and `refresh_sandbox` never copy secrets: set them again on the other side.

## Readiness: the gate on switching a flow on

A flow that cannot work should never be switched on to fail quietly at three in the morning. Every flow carries a **readiness** checklist, returned by `GET /api/flows`, `GET /api/flows/:id`, `POST /api/flows/:id/readiness`, `list_flows`, `get_flow` and `get_flow_readiness`:

```json
{ "ready": false,
  "requirements": [
    { "id": "email-sender", "label": "An address the store's email is sent from", "done": false,
      "detail": "Verify your own sending address under Email → Sending settings; until then this store sends no email at all.",
      "href": "/dashboard#settings/email/domain", "blocking": true },
    { "id": "template:order-shipped", "label": "The email template \"order-shipped\"", "done": true, … },
    { "id": "shipping-mode", "label": "A shipping setup that reports parcels", "done": false, "blocking": false, … } ] }
```

`ready` is true when every **blocking** requirement is done; the rest is advice and never stands in the way.

| Requirement | Blocking | Done when | Fixed by |
| --- | --- | --- | --- |
| `email-sender` (any `send_email` / `notify_merchant`) | yes | your store has a verified sending address of its own | Email → Sending settings → Email domain, `#settings/email/domain` (yours to do; no tool) |
| `merchant-address` (`notify_merchant`, `to: merchant`) | yes | Formahand knows the owner's address | none |
| `template:<key>` (per `send_email`) | yes | a template with that key exists | Automations → Email templates, `#automations/templates`; `upsert_email_template` |
| `secret:<name>` (any `{{ secrets.* }}`) | yes | the vault holds that name | Settings → Agents & API → Vault, `#settings/agents/vault`; `set_secret` |
| `webhook-url:<i>` (`webhook`, `wake_agent`) | yes | the address answered a check, or is built from the event | `check_flow_url` |
| `inbound-url` (a `hook.received` flow) | yes | the flow has an inbound URL and it is on | Automations → Inbound URLs, `#automations/hooks`; `create_flow_hook` |
| `schedule-missing` (a `schedule.tick` flow with no schedule) | yes | the flow carries a schedule | `update_flow { schedule }` |
| `customer-target:<i>` (`tag_customer`) | no | the store has customer records | none |
| `shipping-mode` (`order.fulfilled` / `.delivered` / `.tracking_updated`) | no | a shipping module is active or flat rates exist | Checkout → Shipping; `set_shipping` |
| `schedule-paused` | no | the schedule is not paused | save the schedule again |
| `flows-on` | no | your store's kill switch is on | `set_flows_enabled` |

**The gate.** `POST /api/flows/:id/enable { enabled }` (`enable_flow`) switches a flow on and answers **409 with the readiness** when something blocking is missing; switching off is never gated. An ordinary write that asks for `enabled: true` is not refused, the flow is **saved switched off** with `readiness` and a `notEnabled` sentence in the answer, so nothing you or your agent wrote is lost and nothing half-built starts running.

## Integration modules

`list_integrations` names each module the store can use (couriers, feeds and the like) and the settings it takes. `connect_integration { id, status, config, secrets }` switches one on with the merchant's own credentials: put credentials in `secrets` only, never in `config`. Reads report whether a secret is set, never its value. A credential is the merchant's: ask them for it, and never paste one into chat output or a log.

## Tools

| Tool | Scope | What it does |
| --- | --- | --- |
| `describe_events` | any | What this store can emit and how to subscribe: the resource × verb vocabulary (created \| updated \| deleted) and semantic aliases (order.paid, order.delivered, …), topic pattern rules (`*` per segment), the filter grammar with examples, and the delivery envelope. |
| `list_events` | `events:read` | The store's event log, newest first: { events: [{ id, type, resource, resourceId, subject, occurredAt, actor, data, changed, previous }], nextCursor }. |
| `get_event` | `events:read` | One event by id, with its full data, changed fields and previous values. |
| `list_subscriptions` | `settings:read` | Event subscriptions: { subscriptions: [{ id, name, topics, filter, target, enabled, lastDeliveryAt, lastStatus, failureCount, disabledReason }], max }. |
| `create_subscription` | `settings:write` | Subscribe to events: { name?, topics: ["order.updated", "*.created", …], filter?: 'changed contains "trackingStatus" and data.trackingStatus == "delivered"', target: { kind: "webhook", url } \| { kind: "agent", url, bearer? }, enabled? }. |
| `update_subscription` | `settings:write` | Change a subscription: { id, name?, topics?, filter?, target?, enabled? }. |
| `delete_subscription` | `settings:write` | Delete a subscription and its delivery history. |
| `test_subscription` | `settings:write` |  |
| `list_deliveries` | `settings:read` | Delivery attempts of one subscription, newest first: { deliveries: [{ id, eventId, eventType, attempt, status: pending \| delivered \| failed, nextAttemptAt, lastAttemptAt, responseStatus, error }] }. |
| `retry_delivery` | `settings:write` | Retry one pending or failed delivery now, with a fresh retry schedule. |
| `list_webhooks` | `settings:read` | Legacy view of webhook subscriptions (use list_subscriptions): { webhooks: [{ id, url, events, status, secretSet, createdAt, lastDeliveryAt, lastStatus }] }. |
| `upsert_webhook` | `settings:write` | Legacy: create (no id) or update (with id) a webhook subscription from the fixed event list order.paid, order.fulfilled, order.cancelled, order.refunded, product.updated (prefer create_subscription, which takes any topic pattern and a filter). |
| `delete_webhook` | `settings:write` | Delete a webhook subscription by the id list_webhooks and upsert_webhook return: { id }. |
| `describe_flows` | `settings:read` | How to automate this store without an external service. |
| `list_flows` | `settings:read` | The store's flows: { flows: [{ id, name, enabled, topics, filter, actions, schedule, runCount, runs7d, lastRunAt, lastStatus, failureCount, disabledReason, readiness }], flowsEnabled, senderFallbackAck, max, limits }. |
| `get_flow` | `settings:read` | One flow by id, with its readiness ({ ready, requirements }) and its runs in the last seven days. |
| `create_flow` | `settings:write` | Create a flow: { name, topics: ["order.delivered"], filter?: <predicate>, actions: [{ type: "send_email", template: "order-delivered", to: "buyer" }, …], enabled? }. |
| `update_flow` | `settings:write` | Change a flow: { id, name?, enabled?, topics?, filter?, actions?, schedule? }. |
| `delete_flow` | `settings:write` | Delete a flow and its run history. |
| `test_flow` | `settings:write` | Dry run, no side effects: renders every action of the flow against { eventId } (else the newest event matching the flow, else a sample) → { event, matched, filterMatched, actions: [{ index, type, when, status: would_run \| skipped \| error, rendered:… |
| `set_flows_enabled` | `settings:write` | The store's kill switch: { flowsEnabled: false } pauses every flow (pending runs wait), true resumes. |
| `list_flow_runs` | `events:read` | A flow's runs, newest first: { runs: [{ id, eventId, eventType, attempt, status: pending \| delivered \| failed, error, result: { actions: [{ index, type, status: done \| failed \| skipped \| queued, detail, error, to, messageIds }] } }] }. |
| `get_flow_readiness` | `settings:read` | What a flow still needs before it can run: { readiness: { ready, requirements: [{ id, label, done, detail, href, tool, blocking }] }, flow }. |
| `enable_flow` | `settings:write` | Switch a flow on once everything it needs is in place → { flow }. |
| `check_flow_url` | `settings:write` | Send one clearly marked test request to a flow's webhook or agent steps and record what answered → { checks: [{ actionIndex, url, ok, status, error, checkedAt }], readiness, flow }. |
| `retry_flow_run` | `settings:write` | Run a flow run again → { delivery }. |
| `acknowledge_sender_fallback` | `settings:write` |  |
| `create_flow_hook` | `settings:write` | Give a flow an inbound URL another system can POST to: { flowId, mode: "bearer" \| "hmac" }. |
| `list_flow_hooks` | `settings:read` | The store's inbound URLs: { hooks: [{ id, flowId, flowName, url, mode, enabled, secretPrefix, lastReceivedAt, receivedCount }] }. |
| `rotate_flow_hook` | `settings:write` | Issue a new secret for an inbound URL: { id }. |
| `set_flow_hook_enabled` | `settings:write` | Switch an inbound URL on or off: { id, enabled }. |
| `delete_flow_hook` | `settings:write` | Remove a flow's inbound URL: { id }. |
| `set_secret` | `settings:write` | Store a credential the store may use when it calls another system: { name, value }. |
| `list_secrets` | `settings:read` | The names of the store's secrets with their timestamps: { secrets: [{ name, createdAt, updatedAt, lastUsedAt }], max }. |
| `delete_secret` | `settings:write` | Remove a stored secret: { name }. |
| `describe_secrets` | `settings:read` | How the store's vault works: naming rules, size and count limits, exactly where {{ secrets.name }} may be used, what run logs keep, and what promote_to_live does with secrets. |
| `list_flow_schedule_runs` | `events:read` | Every tick of a flow's schedule, newest first: { runs: [{ id, flowId, ranAt, status: queued \| ok \| failed \| skipped, error }] }. |
| `list_email_templates` | `settings:read` | Every email template this store has, as a list you can actually read: { templates: [{ id, key, name, description, subject, isSystem, isModified, updatedAt, bodyHtmlBytes, bodyTextBytes, variables }], count }. |
| `get_email_template` | `settings:read` | One email template, whole, by id or by key: { template: { id, key, name, description, subject, bodyHtml, bodyText, fromName, replyTo, variables, isSystem, isModified, createdAt, updatedAt } }. |
| `upsert_email_template` | `settings:write` | Create or replace an email template by key: { key: "welcome", name, subject, bodyHtml, bodyText?, fromName?, replyTo?, variables? }. |
| `preview_email_template` | `settings:write` | Render a template against { eventId } or its built-in sample → { subject, bodyHtml, bodyText, errors: [{ field, error, position }], warnings, paths, event, sample, template }. |
| `reset_email_template` | `settings:write` | Restore a system template to its default copy. |
| `list_recipes` | `settings:read` | The connector recipe library: { recipes: [{ id, name, category, description, needs: { scopes, secrets, customFields, options }, flows, subscriptions, customFields }] }. |
| `get_recipe` | `settings:read` | One recipe in full: { recipe: { id, name, category, description, needs, flows, customFieldDefinitions, subscriptions, notes } }. |
| `install_recipe` | `settings:write` | Install a recipe into this store: { id, secrets?: { name: value }, options?: { name: value } }. |
| `uninstall_recipe` | `settings:write` | Remove what a recipe created: { id }. |
| `list_discounts` | `settings:read` | Discount codes with kind, value, currency, conditions, what they apply to, whether they combine, the window, the store-wide and per-customer limits, status (active \| paused \| ended), usageCount and timesUsed, and codeLimits (the daily limits on… |
| `mint_discount_codes` | `discounts:issue` | Make single-use codes from a template discount (upsert_discount { template: true }): { template: id or code, count 1-100, prefix?, expiresInDays? 1-365, email? } → { template, codes: [{ id, code, expiresAt }] }. |
| `set_gift_exception` | `settings:write` | Let one rule give one product away below its recorded cost, or stop it: { subject: "function:<name>" \| "rule:<pricing rule id>" \| "code:<discount id or template id>", productId, allowed: true\|false } → { giftExceptions }. |
| `set_code_limits` | `settings:write` | Change the store's daily limits on codes made from a template: { perStore?, perCaller?: { function?, flow?, app?, token?, member? } }, each a whole number 1-50000, null for the default → { codeLimits: { perStore, perCaller, defaults, ceiling, changed } }. |
| `list_gift_exceptions` | `settings:read` | Which rule, code or store function may give which product away below its recorded cost: [{ id, subject, productId, createdBy, createdAt }]. |
| `upsert_discount` | `settings:write` | Create or edit a discount code, a named trigger for the same rule engine as upsert_pricing_rule (/docs/checkout-rules): code (3-32 letters/digits), kind 'percent' (value 1-100), 'amount' (value in minor units, currency required), 'free_shipping' or… |
| `end_discount` | `settings:write` | End a discount code: { id }, it stops working immediately and stays on the list with its redemptions. |
| `delete_discount` | `settings:write` | Delete a discount code that was never used: { id } → { deleted: true, id, code }. |
| `list_integrations` | `settings:read` | Available integration modules (live shipping rates: the courier platforms list_integrations names) with their settings, plus this store's status, config and which secret keys are set. |
| `connect_integration` | `settings:write` | See… |
| `test_shipping_rates` | `settings:read` | Quote one sample parcel through an integration module from the store's ship-from origin (set_shipping) to { country, postalCode }: { id, destination, weightGrams?, currency? }. |
| `get_checkout_origins` | `settings:read` | The front-end origins allowed to start checkout through POST /api/checkout from another domain (headless): { origins: [{ id, origin, createdAt }], maxOrigins }. |
| `set_checkout_origins` | `settings:write` | Replace the allowed front-end origins: { origins: ["https://shop.example.com", …] } (max 10; https host[:port] only, http://localhost for development). |

## Read more

- https://formahand.com/docs/events
- https://formahand.com/docs/automations
- https://formahand.com/docs/recipes
- https://formahand.com/docs/headless
