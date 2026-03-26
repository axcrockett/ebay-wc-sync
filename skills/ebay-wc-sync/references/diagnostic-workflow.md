# Diagnostic Workflow

Step-by-step triage when eBay orders aren't reaching WooCommerce. Work through
these in order — each step either finds the problem or rules out a layer.

## Step 1: Verify eBay API Credentials

Tokens expire. This is the most common silent failure.

```bash
# Check if the plugin stores token expiry in options
wp db query "SELECT option_name, option_value FROM {prefix}options WHERE option_name LIKE '%ebay%token%' OR option_name LIKE '%ebay%expir%'" --path=/var/www/html
```

If the token expired, the plugin silently stops pulling orders. No error in
WooCommerce logs — it just stops. Refresh via the plugin's settings page or
re-authorize with eBay.

## Step 2: Check WP-Cron / Action Scheduler Health

Most sync plugins schedule imports via WP-Cron or Action Scheduler.

```bash
# Is WP-Cron disabled?
wp config get DISABLE_WP_CRON --path=/var/www/html

# If DISABLE_WP_CRON is true, verify system cron exists
crontab -l 2>/dev/null | grep wp-cron

# Action Scheduler queue health
wp action-scheduler list --status=pending --per-page=5 --path=/var/www/html
wp action-scheduler list --status=failed --per-page=5 --path=/var/www/html
```

See [wp-cron-health.md](wp-cron-health.md) for deeper cron diagnostics.

## Step 3: Check for Stuck or Failed Actions

```bash
# Actions that have been pending for over an hour (should not happen)
wp db query "SELECT action_id, hook, status, scheduled_date_gmt, last_attempt_gmt FROM {prefix}actionscheduler_actions WHERE status='pending' AND scheduled_date_gmt < DATE_SUB(NOW(), INTERVAL 1 HOUR) AND hook LIKE '%ebay%' ORDER BY scheduled_date_gmt LIMIT 10" --path=/var/www/html

# Failed actions with error messages
wp db query "SELECT a.action_id, a.hook, a.status, l.message FROM {prefix}actionscheduler_actions a LEFT JOIN {prefix}actionscheduler_logs l ON a.action_id = l.action_id WHERE a.status='failed' AND a.hook LIKE '%ebay%' ORDER BY a.last_attempt_gmt DESC LIMIT 10" --path=/var/www/html
```

## Step 4: Compare Order Counts

```bash
# WooCommerce orders in the last 7 days
wp db query "SELECT post_status, COUNT(*) as cnt FROM {prefix}posts WHERE post_type='shop_order' AND post_date > DATE_SUB(NOW(), INTERVAL 7 DAY) GROUP BY post_status" --path=/var/www/html
```

Compare this against eBay's Seller Hub order count for the same period.
A mismatch confirms orders are being lost somewhere in the pipeline.

## Step 5: Check Webhook Delivery

If the plugin uses eBay push notifications (webhooks):

```bash
# Check WooCommerce logs for webhook errors
ls -lt /var/www/html/wp-content/uploads/wc-logs/ | head -20
grep -i "ebay\|webhook\|notification" /var/www/html/wp-content/uploads/wc-logs/*.log 2>/dev/null | tail -30
```

Common webhook failures: firewall blocking eBay IPs, SSL certificate expired,
site URL changed after plugin setup. See [common-failures.md](common-failures.md).

## Step 6: Check for Orphaned or Duplicate Orders

```bash
# Orders with eBay metadata but stuck in unexpected status
wp db query "SELECT p.ID, p.post_status, p.post_date, pm.meta_value as ebay_order_id FROM {prefix}posts p JOIN {prefix}postmeta pm ON p.ID = pm.post_id WHERE p.post_type='shop_order' AND pm.meta_key LIKE '%ebay%order%id%' AND p.post_status IN ('wc-pending','wc-on-hold','wc-failed') ORDER BY p.post_date DESC LIMIT 20" --path=/var/www/html
```

See [db-investigation.md](db-investigation.md) for more targeted queries.

## Step 7: Identify the Plugin's Import Path

Different plugins import orders differently. Check which one is active:

```bash
wp plugin list --status=active --path=/var/www/html | grep -i "ebay\|ced\|codisto\|quicksync\|webkul"
```

Then go to [plugin-specifics.md](plugin-specifics.md) for that plugin's
known failure modes and debugging hooks.

## Decision Summary

| Finding | Likely cause | Next step |
|---------|-------------|-----------|
| Token expired | OAuth expiry | Re-authorize in plugin settings |
| WP-Cron disabled, no system cron | Cron not running | Set up system cron |
| Actions stuck pending | AS queue overflow | See wp-cron-health.md |
| Order count mismatch | Import failing silently | Check plugin logs |
| Webhook errors in logs | Delivery failure | Check firewall/SSL/URL |
| Duplicate eBay order IDs | Dedup logic broken | See plugin-specifics.md |
