# WP-Cron and Action Scheduler Health

eBay sync plugins depend on scheduled tasks. If the scheduler is broken,
orders pile up on eBay and never reach WooCommerce.

## Quick Health Check

```bash
# Is WP-Cron disabled?
wp config get DISABLE_WP_CRON --path=/var/www/html

# If disabled, system cron must exist
crontab -l 2>/dev/null | grep wp-cron

# Expected system cron entry (runs every minute):
# * * * * * cd /var/www/html && wp cron event run --due-now --quiet
```

## WP-Cron vs Action Scheduler

| | WP-Cron | Action Scheduler |
|---|---------|-----------------|
| Trigger | Page load (or system cron) | WP-Cron or system cron |
| Retry | No built-in retry | Automatic retry with backoff |
| Visibility | `wp cron event list` | `wp action-scheduler list` |
| Scale | Fine for <50 events | Handles thousands of actions |
| Used by | QuickSync, older plugins | CedCommerce, modern plugins |

Most eBay plugins now use Action Scheduler. Check which one yours uses:

```bash
# WP-Cron events related to eBay
wp cron event list --path=/var/www/html 2>/dev/null | grep -i ebay

# Action Scheduler actions related to eBay
wp action-scheduler list --status=pending --per-page=10 --path=/var/www/html 2>/dev/null | grep -i ebay
```

## Diagnosing WP-Cron Issues

### Symptom: Cron events never fire

```bash
# Check if WP-Cron is reachable
curl -sI "https://yoursite.com/wp-cron.php?doing_wp_cron" | head -3
```

If this returns 403 or times out, WP-Cron is blocked. Common causes:
- Cloudflare or security plugin blocking wp-cron.php
- Server firewall blocking loopback requests
- PHP max_execution_time too low

### Symptom: Events fire but nothing happens

The event fires but the callback crashes silently. Check:

```bash
# PHP error log
tail -50 /var/log/php-fpm/error.log 2>/dev/null || tail -50 /var/log/php*/error.log 2>/dev/null
# WP debug log
tail -50 /var/www/html/wp-content/debug.log 2>/dev/null
```

## Diagnosing Action Scheduler Issues

### Queue status overview

```bash
wp db query "SELECT status, COUNT(*) as cnt FROM {prefix}actionscheduler_actions GROUP BY status ORDER BY cnt DESC" --path=/var/www/html
```

Healthy output: `complete` (majority), `pending` (small number), `failed` (near zero).

### Red flags

- `pending` > 1000: queue is backing up
- `failed` > 100: something is crashing repeatedly
- `in-progress` > 10: actions are timing out mid-execution

### Stuck in-progress actions

```bash
wp db query "SELECT action_id, hook, scheduled_date_gmt, last_attempt_gmt FROM {prefix}actionscheduler_actions WHERE status='in-progress' AND last_attempt_gmt < DATE_SUB(NOW(), INTERVAL 10 MINUTE)" --path=/var/www/html
```

These are zombies. The process died mid-run. Force them back to pending:

```bash
wp db query "UPDATE {prefix}actionscheduler_actions SET status='pending', last_attempt_gmt=NULL, attempts=attempts+1 WHERE status='in-progress' AND last_attempt_gmt < DATE_SUB(NOW(), INTERVAL 10 MINUTE)" --path=/var/www/html
```

### Cleaning up completed actions

If the table has millions of rows, queries slow down site-wide:

```bash
# Count completed actions
wp db query "SELECT COUNT(*) FROM {prefix}actionscheduler_actions WHERE status='complete'" --path=/var/www/html

# Clean up old completed actions (older than 30 days)
wp db query "DELETE FROM {prefix}actionscheduler_actions WHERE status='complete' AND last_attempt_gmt < DATE_SUB(NOW(), INTERVAL 30 DAY) LIMIT 10000" --path=/var/www/html
```

Run the DELETE in batches of 10,000 to avoid table locks on production.

### Manually running the queue

```bash
# Process pending actions now (useful for testing)
wp action-scheduler run --path=/var/www/html
```

## System Cron Setup

If `DISABLE_WP_CRON` is `true`, you need a system cron:

```bash
# Add via crontab -e
* * * * * cd /var/www/html && /usr/local/bin/wp cron event run --due-now --quiet 2>/dev/null
```

For Action Scheduler specifically:

```bash
# Run AS queue via system cron (more reliable than WP-Cron for high volume)
*/2 * * * * cd /var/www/html && /usr/local/bin/wp action-scheduler run --quiet 2>/dev/null
```
