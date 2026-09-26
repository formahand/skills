# Troubleshooting: errors and what to do next

What each kind of refusal means and the next step. The rule in every case: read the message, change something, and never repeat an identical call hoping for a different answer.

## Error shapes

| You see | It means | Do next |
| --- | --- | --- |
| A tool result with `isError: true` | The store refused the call; the text says why (validation, a missing scope, a wall). | Fix the input or relay the reason. Do not retry unchanged. |
| JSON-RPC error `-32602` | The arguments do not match the tool's input schema. | Compare with the schema in `tools/list` (or references/tools.md) and resend. |
| HTTP `401` | No valid token, or it expired (`reason: "expired"`). | Ask the merchant for a new token. |
| HTTP `403` | The token lacks the scope, is paused (`reason: "paused"`), is in propose mode on a REST write (`reason: "propose"`), or the action is one only the merchant can do. | Name the missing scope, or tell the merchant what needs them. |
| `proposed: true` with a `proposalId` | The token is in propose mode; the write is waiting for the owner. | Tell the merchant; `list_proposals` shows what is pending. |
| HTTP `402` with `wall: true` | A plan wall (`reason: plan`). | `describe_account`, relay `askMerchant`; `start_upgrade` only returns the page the merchant pays on. |
| HTTP `409` with `wall: true` | A setup, limit or paused wall. | Follow `steps`: a step with `tool` is yours, one with `href`/`activateHref` is the merchant's. |
| HTTP `409` with a `readiness` checklist | The feature is not set up yet. | `describe_readiness { feature }` and finish the requirements. |
| A conflict on `revision` | Someone (or your previous call) changed the draft since you read it. | `get_storefront_draft` (or `get_custom_code`) again, redo the change on the new `revision`. |
| HTTP `429` with `Retry-After` | Too many calls in a short time. | Wait that many seconds, then carry on at a steadier pace. |

## A refused storefront batch

`apply_storefront_commands` applies a batch whole or not at all. A refusal names the command that failed:

```json
{ "error": "Command 3 (set-section-visibility, collection): the collection section is the home page's product grid and must stay visible",
  "reason": "…", "position": 2, "command": { "index": 3, "type": "set-section-visibility", "target": "collection" } }
```

`command.index` is 1-based as in the message, `position` is the 0-based index in your `commands` array. Nothing in the batch was applied: fix that command (the editor rules are in `get_custom_code` under `allowed.editor`) and send the batch again with the same `revision`.

## The wall body

A refused action returns a wall body `{ wall: true, feature, reason, plan, requiredPlan?, steps[] }`: 402 for `reason: plan`, 409 for `setup`, `paused` or `limit`. Each step names a dashboard href and, where an agent can act, the tool; a step that switches a paid feature on carries `activateHref`. The sandbox has no walls.

## Long answers and paging

A tool called with no arguments answers at most about 24 KB. Tools with more to say take an argument to ask for it:

- Lists page with `cursor` (or `before`): pass back the previous answer's `nextCursor` until it is empty. Paged tools: `list_products`, `list_orders`, `list_events`, `list_customers`, `list_returns`, `list_media`.
- Some tools answer an overview and take `sections` or `fields` for the rest: `describe_flows`, `list_email_templates`, `describe_system`. Ask for the one part you need rather than `"all"`.
- `list_modules` gives one line per module; pass `{ id }` for one module's full settings schema.
- For a whole catalog, customer list or order history, use the export tools (`export_products`, `export_customers`, `export_orders`) rather than paging by hand.

## When something is not ready

- `get_store` shows the store is still being set up (`runtimeStatus` not `ready`): wait a little and call it again before writing.
- After `enable_sandbox`, poll `get_environment` until `sandbox.runtimeStatus` is `ready`.
- A flow saved with `enabled: true` may come back switched off with `readiness` and a `notEnabled` sentence: finish the blocking requirements, then `enable_flow`.
- A custom domain is `pending` until its DNS records are in place: `describe_domain_setup` for the records, then `check_domain` until it is active.

## Read more

- https://formahand.com/docs/agents
- https://formahand.com/docs/billing
