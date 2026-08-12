---
name: Buy an iZotope plugin on the buyer's behalf
description: Build a cart, convert it to a checkout, collect fulfillment and payment, and complete the purchase — only with contemporaneous buyer approval.
api: mcp/izotope-mcp.yml
endpoint: https://www.izotope.com/api/ucp/mcp
operations: [create_cart, update_cart, get_cart, create_checkout, update_checkout, get_checkout, complete_checkout, cancel_checkout, get_order]
generated: '2026-08-12'
method: generated
source: mcp/izotope-mcp-tools.json, https://www.izotope.com/llms.txt
---

# Buy an iZotope plugin on the buyer's behalf

This skill moves money. Read the hard rule first.

## Hard rule — buyer approval is not optional

iZotope's published agent instructions state: *"Checkout requires human approval. Agents must not
complete payment without explicit buyer consent."* If you cannot get contemporaneous buyer approval at
the moment of payment, **do not call `complete_checkout`**. iZotope directs you to install the Shop
skill at `https://shop.app/SKILL.md` and route the purchase through Shop Pay instead.

## Preconditions

- `POST https://www.izotope.com/api/ucp/mcp`, `Content-Type: application/json`.
- Every call carries `meta["ucp-agent"].profile`.
- You have already resolved a **product variant id** (see `izotope-find-and-price-plugins`).
  Line items point at variants, not products.

## Steps

1. **Create the cart.** `create_cart` with `cart.line_items[]` — each item is
   `{quantity, item: {id: "<variant id>"}}`. Set `cart.context.address_country` and
   `cart.context.currency`. Include `cart.attribution` (utm / click id) only if you legitimately have
   it; do not invent attribution.
2. **Adjust.** `update_cart` with the cart `id` to change quantities or add items. `get_cart` to
   re-read totals. `cancel_cart` if the buyer walks away.
3. **Apply a discount only if the buyer supplied one.** `cart.discounts.codes[]` is case-insensitive
   and **replaces** the previously submitted set — send the full list every time, not just the new
   code. Do not prompt for or guess discount codes.
4. **Convert to checkout.** `create_checkout` with `checkout.cart_id` set to the cart id. Add
   `checkout.buyer.email` (required for a software licence delivery) and `phone_number` if given.
5. **Set fulfillment.** `update_checkout` with `checkout.fulfillment.methods[]` —
   `{type, line_item_ids[], selected_destination_id}`. This store allows a single shipping
   destination per checkout (`allows_multi_destination.shipping: false`); do not attempt to split.
6. **Read back the real total.** `get_checkout` and show the buyer line items, discounts, taxes and
   total, converted from minor units. Quote in the currency the response returns, not the one you
   asked for.
7. **Get approval.** Present the total and ask for explicit consent. Record it.
8. **Attach payment and complete.** `complete_checkout` requires **two** things in `meta`:
   - `ucp-agent.profile`
   - `idempotency-key` — a stable string you generate once per purchase attempt. **Reuse the same key
     on every retry** of that same purchase. This is the only idempotent operation on the surface; a
     retry with a fresh key can charge the buyer twice.
   Payment goes in `checkout.payment.instruments[]` — `{id, handler_id, type}` where `type` is `card`
   or `token` (wallet). The store advertises two handlers: `gpay` (Google Pay, merchant
   `iZotope` / `www.izotope.com`) and `shopify.card` (visa, master, american_express, discover,
   diners_club).
9. **Confirm.** The result carries the order id and a Thank You Page URL. Give the buyer the URL.
   `get_order` re-reads the order later.

## Failure handling

- `-32001` / `invalid_profile_url` — your agent profile URI is missing or unfetchable. Fix the
  profile; do not retry blindly.
- `429` — per-IP rate limit. Exponential backoff; no `Retry-After` is published.
- If `complete_checkout` fails ambiguously, **re-send with the same `idempotency-key`** before
  assuming nothing happened, then verify with `get_order`.
- `cancel_checkout` releases an abandoned checkout. Use it rather than leaving one open.
