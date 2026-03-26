# Plugin-Specific Debugging

Each eBay-WooCommerce plugin has its own architecture, meta keys, and failure
modes. Find your plugin below.

## CedCommerce (eBay Integration for WooCommerce)

The most widely used plugin. Deep WooCommerce integration via Action Scheduler.

### Key Meta Keys

```sql
-- Find all CED eBay meta keys in use
SELECT DISTINCT meta_key, COUNT(*) as cnt
FROM {prefix}postmeta
WHERE meta_key LIKE '_ced_ebay_%'
GROUP BY meta_key ORDER BY cnt DESC;
```

Common keys: `_ced_ebay_order_id`, `_ced_ebay_listing_id`, `_ced_ebay_item_id`.

### Action Scheduler Hooks

CED schedules imports and sync via AS. Key hooks to watch:

```bash
wp db query "SELECT hook, status, COUNT(*) as cnt FROM {prefix}actionscheduler_actions WHERE hook LIKE 'ced_ebay%' GROUP BY hook, status ORDER BY hook" --path=/var/www/html
```

Common hooks: `ced_ebay_import_orders`, `ced_ebay_sync_inventory`,
`ced_ebay_update_stock`, `ced_ebay_scheduled_instant_stock_sync`.

### Known CED Failure Modes

1. **Dedup check uses wrong post_status.** Default only checks `wc-completed`,
   misses `wc-processing`. Result: duplicate orders. Fix: patch the query to
   use `post_status = 'any'`.

2. **Silent import failures.** CED's import function returns without error
   when the API call succeeds but order parsing fails. No log entry.
   Detect with: count eBay API orders vs WC orders for the same period.

3. **Token refresh type error.** Older versions can throw a PHP type error
   during token refresh when the response format changes. Look for fatal
   errors in `/var/www/html/wp-content/debug.log`.

4. **AJAX endpoints exposed.** Some CED AJAX handlers lack nonce verification.
   If you've hardened these, check that legitimate sync calls aren't being
   blocked by your security patches.

### CED Credential Storage

```bash
wp db query "SELECT option_name FROM {prefix}options WHERE option_name LIKE 'ced_ebay%' AND (option_name LIKE '%token%' OR option_name LIKE '%key%' OR option_name LIKE '%secret%')" --path=/var/www/html
```

### CED Update Lock

If you've patched CED, block automatic updates or your patches get overwritten:

```bash
# Check if updates are blocked
wp plugin list --name=*ebay-integration* --fields=name,update --path=/var/www/html
```

## Codisto

Different architecture from CED. Uses its own sync tables rather than WC meta.

### Codisto-Specific Tables

```sql
-- Check if Codisto tables exist
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME LIKE '%codisto%';
```

Codisto maintains a separate sync state. If you see orders in Codisto's tables
but not in WC, the issue is in Codisto's WC write layer, not eBay API.

### Webhook Pattern

Codisto uses direct webhooks from eBay rather than polling. Check the
notification URL is registered and reachable:

```bash
grep -i "codisto" /var/www/html/wp-content/uploads/wc-logs/*.log 2>/dev/null | tail -20
```

### Known Codisto Issues

- Sync stalls after WooCommerce major updates (hooks change)
- Can conflict with other eBay plugins if both are active

## QuickSync (WP-Lister for eBay)

### Configuration

QuickSync stores settings as serialized data in options. Key option:

```bash
wp db query "SELECT option_name FROM {prefix}options WHERE option_name LIKE '%wplister%' OR option_name LIKE '%quicksync%'" --path=/var/www/html
```

### Known Limitations

- Older versions don't support eBay's Inventory API (Trading only)
- Variation products with more than 120 options can fail to sync
- No built-in Action Scheduler support (uses WP-Cron directly)

## WebKul

### Hook Priorities

WebKul registers WooCommerce hooks at high priority numbers. If you have
custom code that also hooks into `woocommerce_new_order` or
`woocommerce_order_status_changed`, priority conflicts can cause data loss.

```bash
# Search for WebKul hook registrations
grep -rn "add_action\|add_filter" /var/www/html/wp-content/plugins/*ebay*webkul*/ 2>/dev/null | grep -i "order" | head -20
```

### Meta Key Pattern

WebKul typically uses `_webkul_ebay_*` or `_wk_ebay_*` prefixes.

## Custom Integrations

If using a custom-built integration, check for these common architectural mistakes:

1. **No dedup guard.** Orders get imported twice when webhook and cron both fire.
2. **No retry limit.** Failed imports retry forever, bloating the action queue.
3. **Hardcoded table prefix.** Using `wp_` instead of detecting at runtime.
4. **No token refresh.** Works for months, then silently dies.
5. **Synchronous import.** Importing during the webhook request instead of
   scheduling via Action Scheduler. Timeouts kill the import mid-way.
