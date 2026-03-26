# Database Investigation

SQL queries for eBay-WooCommerce sync issues. All use `{prefix}` — run `wp config get table_prefix` first.
## Orders With eBay Metadata

```sql
SELECT p.ID, p.post_status, p.post_date, pm.meta_key, LEFT(pm.meta_value, 60) as meta_val
FROM {prefix}posts p JOIN {prefix}postmeta pm ON p.ID = pm.post_id
WHERE p.post_type = 'shop_order' AND pm.meta_key LIKE '%ebay%'
ORDER BY p.post_date DESC LIMIT 30;
```

## Orphaned Orders (eBay ID exists, bad WC status)

```sql
SELECT p.ID, p.post_status, p.post_date, pm.meta_value as ebay_order_id
FROM {prefix}posts p JOIN {prefix}postmeta pm ON p.ID = pm.post_id
WHERE p.post_type = 'shop_order' AND pm.meta_key LIKE '%ebay%order%id%'
  AND p.post_status IN ('wc-failed', 'wc-cancelled', 'wc-pending')
ORDER BY p.post_date DESC;
```

## Duplicate eBay Order IDs

```sql
SELECT pm.meta_value as ebay_order_id, COUNT(*) as order_count, GROUP_CONCAT(p.ID) as wc_order_ids
FROM {prefix}postmeta pm JOIN {prefix}posts p ON pm.post_id = p.ID
WHERE pm.meta_key LIKE '%ebay%order%id%' AND p.post_type = 'shop_order'
GROUP BY pm.meta_value HAVING order_count > 1;
```

## Orders Stuck in Processing (older than 24h)

```sql
SELECT ID, post_status, post_date, post_modified FROM {prefix}posts
WHERE post_type = 'shop_order' AND post_status = 'wc-processing'
  AND post_date < DATE_SUB(NOW(), INTERVAL 24 HOUR)
ORDER BY post_date DESC LIMIT 20;
```

## Order Count by Status (last 7 days)

```sql
SELECT post_status, COUNT(*) as cnt FROM {prefix}posts
WHERE post_type = 'shop_order' AND post_date > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY post_status ORDER BY cnt DESC;
```

## Orders With Missing Customer Data

```sql
SELECT p.ID, p.post_date,
  MAX(CASE WHEN pm.meta_key = '_billing_email' THEN pm.meta_value END) as email,
  MAX(CASE WHEN pm.meta_key = '_shipping_first_name' THEN pm.meta_value END) as ship_name,
  MAX(CASE WHEN pm.meta_key = '_shipping_address_1' THEN pm.meta_value END) as ship_addr
FROM {prefix}posts p JOIN {prefix}postmeta pm ON p.ID = pm.post_id
WHERE p.post_type = 'shop_order' AND p.post_date > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY p.ID, p.post_date
HAVING email IS NULL OR email = '' OR ship_name IS NULL OR ship_name = ''
ORDER BY p.post_date DESC LIMIT 20;
```

## Check Integration Plugin Meta Keys

```sql
SELECT DISTINCT pm.meta_key, COUNT(*) as cnt
FROM {prefix}postmeta pm JOIN {prefix}posts p ON pm.post_id = p.ID
WHERE p.post_type = 'shop_order'
  AND (pm.meta_key LIKE '%ebay%' OR pm.meta_key LIKE '%ced%' OR pm.meta_key LIKE '%codisto%')
GROUP BY pm.meta_key ORDER BY cnt DESC;
```

## Verify Line Item Integrity

```sql
SELECT p.ID, p.post_date, p.post_status, COUNT(oi.order_item_id) as item_count
FROM {prefix}posts p
LEFT JOIN {prefix}woocommerce_order_items oi ON p.ID = oi.order_id AND oi.order_item_type = 'line_item'
WHERE p.post_type = 'shop_order' AND p.post_date > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY p.ID, p.post_date, p.post_status HAVING item_count = 0
ORDER BY p.post_date DESC;
```

## Action Scheduler History for eBay Hooks

```sql
SELECT action_id, hook, status, scheduled_date_gmt, last_attempt_gmt
FROM {prefix}actionscheduler_actions WHERE hook LIKE '%ebay%'
ORDER BY scheduled_date_gmt DESC LIMIT 20;
```

## WP-CLI Shortcuts

```bash
# Quick order search by eBay order ID
wp db query "SELECT post_id FROM {prefix}postmeta WHERE meta_value = 'EBAY_ORDER_ID_HERE'" --path=/var/www/html
# Get full eBay meta for a specific WC order
wp db query "SELECT meta_key, LEFT(meta_value, 80) FROM {prefix}postmeta WHERE post_id = ORDER_ID_HERE AND meta_key LIKE '%ebay%'" --path=/var/www/html
```
