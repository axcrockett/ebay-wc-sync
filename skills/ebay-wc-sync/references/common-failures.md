# Common Failures

Every failure here has been seen in production. Each entry: what it looks like,
why it happens, how to confirm, and how to fix it.

## 1. Webhook Delivery Failures

**Symptoms:** Orders stop syncing suddenly. No errors in WooCommerce admin.
Plugin settings look fine.

**Cause:** eBay can't reach your webhook URL. Firewall rule change, SSL cert
expired, site URL changed, or Cloudflare challenge blocking eBay's IPs.

**Confirm:**
```bash
# Check if eBay notification URL is reachable from outside
curl -sI "https://yoursite.com/wp-json/your-plugin/webhook" | head -5
# Check recent firewall blocks (if using fail2ban)
grep "Ban" /var/log/fail2ban.log 2>/dev/null | tail -10
```

**Fix:** Whitelist eBay IP ranges. Renew SSL cert. Verify webhook URL in
plugin settings matches your current site URL.

## 2. OAuth Token Expiration

**Symptoms:** Sync worked last week, nothing changed, now orders don't come in.
No error messages anywhere obvious.

**Cause:** eBay OAuth tokens expire (typically 2 hours for user tokens, 18
months for refresh tokens). Some plugins don't auto-refresh reliably.

**Confirm:**
```bash
wp db query "SELECT option_name, LEFT(option_value, 80) FROM {prefix}options WHERE option_name LIKE '%ebay%token%' OR option_name LIKE '%ebay%expir%' OR option_name LIKE '%ebay%auth%'" --path=/var/www/html
```

**Fix:** Re-authorize through the plugin's eBay connection page. If refresh
tokens are also expired, you need a full re-auth from eBay Developer portal.

## 3. SKU Mismatch Between Platforms

**Symptoms:** Some orders import, others don't. The ones that fail have
"product not found" errors in logs.

**Cause:** eBay listing SKU doesn't match WooCommerce product SKU. Common
after bulk product updates, CSV imports, or variation changes.

**Confirm:**
```bash
# Find WC products by SKU
wp db query "SELECT post_id, meta_value FROM {prefix}postmeta WHERE meta_key='_sku' AND meta_value='YOUR_EBAY_SKU'" --path=/var/www/html
```

**Fix:** Align SKUs on both platforms. Most plugins match on `_sku` meta.

## 4. Category Mapping Drift

**Symptoms:** New listings don't appear in the right WooCommerce categories.
Old listings are fine.

**Cause:** eBay periodically changes their category tree. Your mapping config
points to deprecated category IDs.

**Fix:** Re-run category mapping in the plugin settings. Export and review
the mapping table if the plugin provides one.

## 5. Orders Stuck in "Processing"

**Symptoms:** Orders appear in WooCommerce but never move to "completed"
even after eBay shows them as shipped/delivered.

**Cause:** Status sync is one-directional (eBay-to-WC import only) or the
fulfillment update webhook is broken.

**Confirm:**
```bash
wp db query "SELECT ID, post_status, post_date FROM {prefix}posts WHERE post_type='shop_order' AND post_status='wc-processing' AND post_date < DATE_SUB(NOW(), INTERVAL 7 DAY) ORDER BY post_date LIMIT 20" --path=/var/www/html
```

**Fix:** Check if the plugin supports bi-directional status sync. If not,
build a scheduled action to poll eBay Fulfillment API for status updates.

## 6. Duplicate Order Creation

**Symptoms:** Same eBay order appears twice (or more) in WooCommerce.
Customer gets charged correctly on eBay but WC shows duplicates.

**Cause:** Race condition between webhook and scheduled import. Or the dedup
check queries the wrong post_status (e.g., only checks `wc-completed`, misses
`wc-processing`).

**Confirm:**
```bash
wp db query "SELECT meta_value as ebay_order_id, COUNT(*) as cnt FROM {prefix}postmeta WHERE meta_key LIKE '%ebay%order%id%' GROUP BY meta_value HAVING cnt > 1" --path=/var/www/html
```

**Fix:** Fix dedup query to check `post_status = 'any'`. Add a transient lock
around import to prevent concurrent processing of the same order.

## 7. Missing Customer Data

**Symptoms:** Orders import but shipping address, billing email, or buyer
name is blank in WooCommerce.

**Cause:** eBay API returns customer data in a nested structure that the
plugin doesn't fully parse. Or eBay's buyer privacy settings hide the data.

**Confirm:**
```bash
wp db query "SELECT p.ID, pm.meta_key, pm.meta_value FROM {prefix}posts p JOIN {prefix}postmeta pm ON p.ID = pm.post_id WHERE p.post_type='shop_order' AND pm.meta_key IN ('_billing_email','_shipping_first_name','_shipping_address_1') AND (pm.meta_value IS NULL OR pm.meta_value = '') ORDER BY p.post_date DESC LIMIT 10" --path=/var/www/html
```

**Fix:** Check if the plugin maps eBay's `ShippingAddress` fields correctly.
Guest checkout orders on eBay may have limited data by design.

## 8. Action Scheduler Queue Overflow

**Symptoms:** Sync is delayed by hours. The site is slow. `wp_actionscheduler_actions`
table has millions of rows.

**Cause:** Failed actions keep retrying, new actions pile up, the table grows
unbounded. Common with high-volume stores or aggressive retry settings.

**Confirm:**
```bash
wp db query "SELECT status, COUNT(*) as cnt FROM {prefix}actionscheduler_actions GROUP BY status" --path=/var/www/html
# If complete > 100k, cleanup is needed
```

**Fix:** Clean completed actions, increase AS batch size, consider external
cron runner. See [wp-cron-health.md](wp-cron-health.md).
