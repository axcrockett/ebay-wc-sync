# Plugin-Specific Debugging

Each plugin has its own meta keys, scheduler hooks, and failure modes.

## CedCommerce (eBay Integration for WooCommerce)

Most widely used. Deep WooCommerce integration via Action Scheduler.

**Meta keys:** `_ced_ebay_order_id`, `_ced_ebay_listing_id`, `_ced_ebay_item_id`
```bash
wp db query "SELECT DISTINCT meta_key, COUNT(*) as cnt FROM {prefix}postmeta WHERE meta_key LIKE '_ced_ebay_%' GROUP BY meta_key ORDER BY cnt DESC" --path=/var/www/html
```

**AS hooks:** `ced_ebay_import_orders`, `ced_ebay_sync_inventory`, `ced_ebay_update_stock`
```bash
wp db query "SELECT hook, status, COUNT(*) as cnt FROM {prefix}actionscheduler_actions WHERE hook LIKE 'ced_ebay%' GROUP BY hook, status ORDER BY hook" --path=/var/www/html
```

**Credential storage:**
```bash
wp db query "SELECT option_name FROM {prefix}options WHERE option_name LIKE 'ced_ebay%' AND (option_name LIKE '%token%' OR option_name LIKE '%key%' OR option_name LIKE '%secret%')" --path=/var/www/html
```

**Known failure modes:**
1. Dedup check uses wrong `post_status` — only checks `wc-completed`, misses `wc-processing`. Creates duplicates.
2. Silent import failures — API call succeeds but order parsing fails with no log entry.
3. Token refresh type error — older versions throw PHP fatal on response format change. Check `debug.log`.
4. AJAX endpoints lack nonce verification — if hardened, legitimate sync calls may be blocked.

**Update lock:** If you've patched CED, block auto-updates or patches get overwritten.

## Codisto

Uses its own sync tables rather than WC postmeta.

```sql
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME LIKE '%codisto%';
```

Uses direct webhooks from eBay (not polling). If orders exist in Codisto tables but not WC, the issue is Codisto's WC write layer. Known to stall after WooCommerce major updates. Conflicts with other eBay plugins if both active.

## QuickSync (WP-Lister for eBay)

Stores settings as serialized data in options:
```bash
wp db query "SELECT option_name FROM {prefix}options WHERE option_name LIKE '%wplister%' OR option_name LIKE '%quicksync%'" --path=/var/www/html
```

Limitations: older versions Trading API only, variation products >120 options fail to sync, no Action Scheduler support (WP-Cron only).

## WebKul

Registers WooCommerce hooks at high priority numbers. Priority conflicts with custom `woocommerce_new_order` or `woocommerce_order_status_changed` hooks can cause data loss.

Meta key prefixes: `_webkul_ebay_*` or `_wk_ebay_*`.

## Custom Integrations

Common architectural mistakes:
1. **No dedup guard** — webhook and cron both import the same order
2. **No retry limit** — failed imports retry forever, bloating the action queue
3. **Hardcoded table prefix** — using `wp_` instead of detecting at runtime
4. **No token refresh** — works for months, then silently dies
5. **Synchronous import** — importing during webhook request instead of scheduling via AS; timeouts kill it
