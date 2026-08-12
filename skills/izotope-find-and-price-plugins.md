---
name: Find and price iZotope plugins
description: Search iZotope's store for audio plugins matching a buyer's need and return accurate localized prices, without transacting.
api: mcp/izotope-mcp.yml
endpoint: https://www.izotope.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-08-12'
method: generated
source: mcp/izotope-mcp-tools.json
---

# Find and price iZotope plugins

Read-only. This skill never creates a cart, a checkout, or an order.

## Before you call anything

- Endpoint: `POST https://www.izotope.com/api/ucp/mcp`
- Headers: `Content-Type: application/json`, `Accept: application/json, text/event-stream`
- No API key. No account. The catalog tools are anonymous.
- **Every** call must carry `meta["ucp-agent"].profile` — a reachable URI identifying you as the
  calling agent. Omit it and the server answers JSON-RPC error `-32001`
  (`invalid_profile_url`, "Missing profile uri").

## Steps

1. **Confirm the surface is live.** `GET https://www.izotope.com/.well-known/ucp` and check
   `ucp.version` is a version you support (`2026-04-08` at time of writing; `2026-01-23` also served).
2. **Set buyer context.** Build `catalog.context` with at least `address_country` (ISO 3166-1 alpha-2)
   and `currency` (ISO 4217). iZotope's own agent instructions require these for correct pricing and
   availability. Add `language` (BCP 47) when you know it.
3. **Search.** Call `search_catalog` with `catalog.query` set to the buyer's intent in plain language
   ("repair noisy dialogue", "mastering suite", "vocal processing"). Narrow with `catalog.filters`:
   `categories[]` (OR logic), `price.min` / `price.max` (minor units), `available` (defaults true —
   sale-ready items only).
4. **Resolve specifics.** If the buyer names products, call `lookup_catalog` to resolve several
   identifiers at once rather than searching repeatedly.
5. **Get detail before quoting.** Call `get_product` with `catalog.id` for the chosen product. Pass
   `catalog.selected[]` (`name` / `label`) when the product has variant options — iZotope sells
   tiered editions (Elements / Standard / Advanced) as variants, and the price differs by variant.
6. **Convert money before you speak it.** Prices come back as integers in ISO 4217 **minor** units
   paired with a currency code: `{"amount": 2500, "currency": "USD"}` is **$25.00**. Divide by 100 for
   two-decimal currencies; zero-decimal currencies such as JPY are already whole units. Never quote
   the raw integer to a buyer.

## Rules

- Back off on `429`. The endpoint is rate-limited per IP and publishes no rate-limit headers and no
  `Retry-After`, so use exponential backoff rather than reading a reset value that does not exist.
- iZotope products are downloadable software licences. Do not promise shipping dates or physical
  delivery.
- If the buyer wants to buy, hand off to `izotope-buy-a-plugin` — do not complete a purchase from
  this skill.
