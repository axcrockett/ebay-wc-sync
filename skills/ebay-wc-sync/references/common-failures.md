# Common Failures

Every failure here has been seen in production. Each entry: symptom, cause, confirm, fix.

## 1. Webhook Delivery Failures

**Symptoms:** Orders stop syncing suddenly. No errors in WooCommerce admin.
**Cause:** eBay can't reach your webhook URL — firewall change, SSL expired, site URL changed, Cloudflare challenge.
**Confirm:** `curl -sI "https://yoursite.com/wp-json/your-plugin/webhook" | head -5`
**Fix:** Whitelist eBay IP ranges. Renew SSL. Verify webhook URL in plugin settings.

## 2. OAuth Token Expiration

**Symptoms:** Sync worked last week, nothing changed, now orders don't come in.
**Cause:** eBay OAuth tokens expire (2h access, 18mo refresh). Some plugins don't auto-refresh.
**Confirm:**
```bash
wp db query "SELECT option_name, LEFT(option_value, 80) FROM {prefix}options WHERE option_name LIKE '%ebay%token%' OR option_name LIKE '%ebay%expir%'" --path=/var/www/html
```
**Fix:** Re-authorize through the plugin's eBay connection page.

## 3. SKU Mismatch Between Platforms

**Symptoms:** Some orders import, others don't. "Product not found" in logs.
**Cause:** eBay listing SKU doesn't match WooCommerce `_sku` meta. Common after bulk updates or CSV imports.
**Confirm:** `wp db query "SELECT post_id, meta_value FROM {prefix}postmeta WHERE meta_key='_sku' AND meta_value='YOUR_EBAY_SKU'" --path=/var/www/html`
**Fix:** Align SKUs on both platforms.

## 4. Category Mapping Drift

**Symptoms:** New listings land in wrong WooCommerce categories. Old listings fine.
**Cause:** eBay periodically changes their category tree. Mapping config points to deprecated IDs.
**Fix:** Re-run category mapping in plugin settings.

## 5. Orders Stuck in "Processing"

**Symptoms:** Orders appear but never move to "completed" even after eBay shows shipped.
**Cause:** Status sync is one-directional or fulfillment webhook is broken.
**Confirm:**
```bash
wp db query "SELECT ID, post_status, post_date FROM {prefix}posts WHERE post_type='shop_order' AND post_status='wc-processing' AND post_date < DATE_SUB(NOW(), INTERVAL 7 DAY) ORDER BY post_date LIMIT 20" --path=/var/www/html
```
**Fix:** Check if plugin supports bi-directional status sync. If not, poll eBay Fulfillment API.

## 6. Duplicate Order Creation

**Symptoms:** Same eBay order appears twice or more in WooCommerce.
**Cause:** Race condition between webhook and scheduled import, or dedup check queries wrong status.
**Confirm:**
```bash
wp db query "SELECT meta_value as ebay_order_id, COUNT(*) as cnt FROM {prefix}postmeta WHERE meta_key LIKE '%ebay%order%id%' GROUP BY meta_value HAVING cnt > 1" --path=/var/www/html
```
**Fix:** Fix dedup query to check all statuses. Add transient lock around import.

## 7. Missing Customer Data

**Symptoms:** Orders import but shipping address, billing email, or buyer name is blank.
**Cause:** Plugin doesn't fully parse eBay's nested customer data, or eBay buyer privacy hides it.
**Confirm:**
```bash
wp db query "SELECT p.ID, pm.meta_key, pm.meta_value FROM {prefix}posts p JOIN {prefix}postmeta pm ON p.ID = pm.post_id WHERE p.post_type='shop_order' AND pm.meta_key IN ('_billing_email','_shipping_first_name','_shipping_address_1') AND (pm.meta_value IS NULL OR pm.meta_value = '') ORDER BY p.post_date DESC LIMIT 10" --path=/var/www/html
```
**Fix:** Check plugin's eBay `ShippingAddress` field mapping. Guest checkout orders may have limited data.

## 8. Action Scheduler Queue Overflow

**Symptoms:** Sync delayed by hours. Site is slow. `actionscheduler_actions` table has millions of rows.
**Cause:** Failed actions retry endlessly, new actions pile up.
**Confirm:**
```bash
wp db query "SELECT status, COUNT(*) as cnt FROM {prefix}actionscheduler_actions GROUP BY status" --path=/var/www/html
```
**Fix:** Clean completed actions, increase AS batch size. See [wp-cron-health.md](wp-cron-health.md).
