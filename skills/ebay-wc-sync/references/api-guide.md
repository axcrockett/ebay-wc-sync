# eBay API Guide

eBay has three API families that matter for WooCommerce sync. They were built
at different times, overlap in confusing ways, and are partially incompatible
with each other. Understanding which one your plugin uses is the first step
in debugging.

## The Three APIs

### Trading API (Legacy)

The original eBay API. SOAP/XML based. Still widely used by older plugins.

- **Order retrieval:** `GetOrders`, `GetOrderTransactions`
- **Listing management:** `AddItem`, `ReviseItem`, `EndItem`
- **Auth:** Auth'n'Auth tokens (long-lived, configured in plugin settings)
- **Identification:** Look for XML payloads, SOAP headers, or `X-EBAY-API-CALL-NAME` in logs

Most CedCommerce and WebKul installations use Trading API.

### Inventory API (Modern REST)

eBay's newer REST API for listing management. Introduced to replace Trading
API's listing calls, but order retrieval was moved to a separate API.

- **Listing management:** `/sell/inventory/v1/offer`, `/sell/inventory/v1/inventory_item`
- **Cannot retrieve orders** — that's the Fulfillment API's job
- **Auth:** OAuth 2.0 tokens (short-lived access + long-lived refresh)
- **Identification:** REST endpoints, `Authorization: Bearer` headers

### Fulfillment API (Post-Order REST)

Handles everything after a sale: order retrieval, shipping, refunds.

- **Order retrieval:** `/sell/fulfillment/v1/order`
- **Shipping:** `/sell/fulfillment/v1/order/{orderId}/shipping_fulfillment`
- **Auth:** OAuth 2.0 (same tokens as Inventory API)
- **Identification:** `/sell/fulfillment/` in URL paths

## The Incompatibility Problem

Listings created via **Trading API** (`AddItem`) can be updated with Trading
API (`ReviseItem`) but have limited compatibility with Inventory API updates.

Listings created via **Inventory API** (`/offer`) cannot be managed by Trading
API calls like `ReviseItem`.

This means: if your plugin switches from Trading to Inventory API (or if two
plugins manage the same listings), you get silent failures — update calls
return success but nothing changes on eBay.

## How to Identify Which API Your Plugin Uses

```bash
# Check plugin logs for API indicators
grep -i "GetOrders\|GetOrderTransactions\|SOAP\|X-EBAY-API" /var/www/html/wp-content/uploads/wc-logs/*.log 2>/dev/null | tail -5

# REST/Fulfillment API indicators
grep -i "sell/fulfillment\|sell/inventory\|Bearer" /var/www/html/wp-content/uploads/wc-logs/*.log 2>/dev/null | tail -5
```

```bash
# Check what auth data the plugin stores
wp db query "SELECT option_name FROM {prefix}options WHERE option_name LIKE '%ebay%' AND (option_name LIKE '%token%' OR option_name LIKE '%auth%' OR option_name LIKE '%api%')" --path=/var/www/html
```

- If you see `auth_token` or `auth_n_auth`: **Trading API**
- If you see `oauth_token`, `refresh_token`, `access_token`: **REST APIs** (Inventory + Fulfillment)

## Order Lifecycle Gotchas

eBay orders are not as simple as "placed -> paid -> shipped":

- **Combined payments:** Buyer purchases 3 items, eBay creates one combined order. If your plugin imports per-transaction, you get 3 WC orders for 1 payment.
- **Order splits:** Large orders can be split by eBay for fulfillment. One eBay order becomes 2+ shipments.
- **Defunct orders:** eBay can mark orders as `DEFUNCT` (cancelled before payment). Some plugins still try to import these.
- **Unpaid items:** Orders may exist for days without payment. Importing them creates WC orders with no revenue.

## Token Refresh Patterns

| Token type | Lifetime | What happens when it expires |
|-----------|----------|------------------------------|
| Auth'n'Auth (Trading) | 18 months | Plugin stops working silently. Re-auth required. |
| OAuth access token | 2 hours | Should auto-refresh. If refresh fails, re-auth required. |
| OAuth refresh token | 18 months | Full re-authorization from eBay Developer portal. |

```bash
# Check token expiry stored in DB
wp db query "SELECT option_name, option_value FROM {prefix}options WHERE option_name LIKE '%ebay%expir%'" --path=/var/www/html
```
