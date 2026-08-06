---
generated: '2026-08-06'
method: generated
name: Buy from the AGIBOT store as an agent
description: Search the AGIBOT catalog, build a cart, and take a checkout to buyer-approved completion over the store's live UCP/MCP endpoint.
api: mcp/agibot-mcp.yml
operations: [search_catalog, lookup_catalog, get_product, create_cart, update_cart, get_cart, create_checkout, update_checkout, complete_checkout, get_order]
source: >-
  Every tool name is verified verbatim in mcp/agibot-mcp-tools.json, captured
  from a live tools/list call against https://store.agibot.com/api/ucp/mcp on
  2026-08-06. Guardrails are quoted from https://store.agibot.com/llms.txt.
---

# Buy from the AGIBOT store as an agent

AgiBot's online store implements the Universal Commerce Protocol and exposes an anonymous
hosted MCP endpoint at `https://store.agibot.com/api/ucp/mcp` (JSON-RPC 2.0). Thirteen tools
are published with full JSON Schema inputs.

## Discovery
- `GET https://store.agibot.com/.well-known/ucp` — confirm the protocol version the store
  accepts (`2026-04-08` current, `2026-01-23` also supported), the service endpoints, the
  capabilities, and the payment handlers before you call anything.
- `GET https://store.agibot.com/llms.txt` (or `/agents.md`) — the store's own agent instructions.

## Auth
- Catalog, cart and checkout tools are anonymous.
- Customer-scoped calls use OAuth 2.0 / OIDC with PKCE `S256` against
  `https://store-account.agibot.com`; scopes `openid`, `email`, `customer-account-api:full`,
  `customer-account-mcp-api:full`. See `authentication/agibot-authentication.yml` and
  `scopes/agibot-scopes.yml`.
- Every tool requires `meta.ucp-agent.profile` — a URI identifying you as the calling agent.

## Steps
1. **Find the product** — `search_catalog` with the buyer's intent. Use `lookup_catalog` to
   resolve several known identifiers at once, or `get_product` for the full detail of one.
   Pass `context.address_country` and `context.currency` so pricing and availability are real.
2. **Build the cart** — `create_cart` with the chosen variants, then `update_cart` to adjust
   quantities and `get_cart` to read back totals. `cancel_cart` abandons it.
3. **Open the checkout** — `create_checkout`. Keep the returned id
   (`gid://shopify/Checkout/…`); every later call addresses it.
4. **Fulfil** — `update_checkout` to set the shipping address and method. Re-read with
   `get_checkout` to see line items, totals, discounts and taxes as the store computed them.
5. **Complete — only with the buyer in the loop** — `complete_checkout`. `cancel_checkout`
   backs out.
6. **Track** — `get_order` for status after completion.

## Guardrails (the store states these, not us)
- **Checkout requires contemporaneous human approval.** An agent must not complete payment
  without explicit buyer consent. If you cannot get approval at the moment of payment, route
  the purchase through Shop Pay instead of calling `complete_checkout`.
- **Respect rate limits.** The MCP endpoint is rate-limited per IP; back off on `429`. No
  numeric quota is published — see `rate-limits/agibot-rate-limits.yml`.

## Errors
- JSON-RPC 2.0 error objects. `429` is the only status the store documents by name. See
  `errors/agibot-problem-types.yml`.

## Notes
- Read-only browsing needs no MCP at all: `GET /products/{handle}.json`,
  `GET /collections/{handle}/products.json` and `GET /search?q=…&type=product`.
- This surface sells robots. It is unrelated to the AimDK robot control protocol — see
  `skills/agibot-robot-motion-control.md` for that.
