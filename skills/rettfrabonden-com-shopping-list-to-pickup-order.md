---
generated: '2026-09-19'
method: generated
name: Turn a shopping list into pickup orders
description: Find who sells each item on a shopping list nearby, build an anonymous cart, review it, submit pickup orders (one per producer, no payment), and track their status — keeping the buyer_ref token safe because it cannot be recovered.
api: mcp/rettfrabonden-com-mcp.yml
operations: []
mcp_tools: [lokal_find_offers, lokal_cart_create, lokal_cart_add_item, lokal_cart_view, lokal_cart_submit, lokal_order_status]
source: >-
  MCP-only flow: none of these six tools has a published REST operation (mcp/rettfrabonden-com-tool-crosswalk.yml
  mcp_only[]). Tool names, required inputs and annotations verified in mcp/rettfrabonden-com-mcp-tools.json
  (live tools/list 2026-09-19); reversibility facts from conventions/rettfrabonden-com-conventions.yml.
---

# Turn a shopping list into pickup orders

This flow exists **only on the MCP server** at `https://rettfrabonden.com/mcp` (Streamable HTTP). It places
real pickup orders that e-mail opted-in sellers, so it carries consequences a search does not.

## Session
- Send `initialize` first and keep the `Mcp-Session-Id` response header on every later request; `tools/list`
  or `tools/call` without it returns HTTP 404 / JSON-RPC `-32001 Session not found`.
- No credential is required. See `authentication/rettfrabonden-com-authentication.yml`.

## Before you act — what can and cannot be undone
- Carts expire after **7 days**. A line's quantity can be changed by calling `lokal_cart_add_item` again with
  the same `product_id` (it upserts). No tool removes a line or deletes a cart.
- **A submitted order has no buyer-side cancel.** `lokal_order_status` can show `declined`, `cancelled` or
  `cancel_reason: no_show`, but those are seller/system states. No payment is charged, which bounds the
  consequence, but the seller may be e-mailed the moment you submit. Confirm with the user before step 5.
- There is no dry run and no idempotency key (`conventions/rettfrabonden-com-conventions.yml`): a retried
  `lokal_cart_submit` after a timeout may create duplicate orders — check `lokal_order_status` / `lokal_cart_view`
  before retrying.

## Steps
1. **Find who sells each item** — `lokal_find_offers` with `items` (e.g. `["poteter","honning","egg"]`) and
   either `near` (a Norwegian place name) or `lat`+`lng`; `radius_km` defaults to 50. Read-only. For each item
   you get up to 5 nearby producers with contact info and `can_order`; only `can_order: true` producers can be
   ordered from through the platform — for the rest, give the user the contact details instead.
2. **Create a cart** — `lokal_cart_create` (no inputs). Store `cart_id` **and** `buyer_ref` immediately; the
   tool warns the `buyer_ref` capability token "cannot be recovered". Treat it as a secret.
3. **Add products** — `lokal_cart_add_item` with `cart_id`, `buyer_ref`, `product_id`, `qty` and an optional
   `note` (e.g. "please keep cold"). The product must be `in_stock` and from a verified non-umbrella producer;
   product ids come from the offers/search results or the ACP feed (`getAcpProductFeed`, `item_id`).
4. **Review** — `lokal_cart_view` with `cart_id` and `buyer_ref`. Show the user the contents grouped by
   producer with subtotals and the total, and state plainly that submitting creates one pickup order per
   producer and cannot be cancelled through the API.
5. **Submit** — `lokal_cart_submit` with `cart_id` and `buyer_ref`, only after explicit user confirmation.
   Stock is re-checked at submit; if any item is no longer in stock the whole submit is rejected with a
   per-item message — fix the cart (step 3) and review again. Capture the returned order ids per producer.
6. **Track** — `lokal_order_status` with `order_id` and `buyer_ref`. Status moves
   `pending → confirmed → ready → completed`, or `declined` / `cancelled`; the `timeline[]` array carries
   each transition with a timestamp. Tell the user pickup is in person and payment happens with the producer.

## Rate limits
- MCP `tools/call` is billed to the 150-per-15-minutes search bucket per IP (`rate-limits/rettfrabonden-com-rate-limits.yml`).
  One `lokal_find_offers` call covers a whole list — prefer it over one search per item.

## Errors
- JSON-RPC errors: `-32601` for an unknown method, `-32001` for a missing session. Tool-level rejections come
  back as tool results with a message, not HTTP errors. See `errors/rettfrabonden-com-problem-types.yml`.
