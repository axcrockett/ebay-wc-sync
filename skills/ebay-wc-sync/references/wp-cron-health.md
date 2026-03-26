# WP-Cron and Action Scheduler Health

eBay sync plugins depend on scheduled tasks. If the scheduler is broken, orders pile up on eBay.

## Quick Health Check

```bash
wp config get DISABLE_WP_CRON --path=/var/www/html
crontab -l 2>/dev/null | grep wp-cron
# Expected: * * * * * cd /var/www/html && wp cron event run --due-now --quiet
```

## WP-Cron vs Action Scheduler

| | WP-Cron | Action Scheduler |
|---|---------|-----------------|
| Trigger | Page load or system cron | WP-Cron or system cron |
| Retry | No built-in retry | Automatic retry with backoff |
| Visibility | `wp cron event list` | `wp action-scheduler list` |
| Used by | QuickSync, older plugins | CedCommerce, modern plugins |

```bash
# Which scheduler does your plugin use?
wp cron event list --path=/var/www/html 2>/dev/null | grep -i ebay
wp action-scheduler list --status=pending --per-page=10 --path=/var/www/html 2>/dev/null | grep -i ebay
```

## WP-Cron Issues

**Events never fire:** `curl -sI "https://yoursite.com/wp-cron.php?doing_wp_cron" | head -3`
Causes: Cloudflare/security plugin blocking, firewall blocking loopback, low `max_execution_time`.

**Events fire but nothing happens:** Callback crashes silently.
```bash
tail -50 /var/log/php-fpm/error.log 2>/dev/null || tail -50 /var/log/php*/error.log 2>/dev/null
tail -50 /var/www/html/wp-content/debug.log 2>/dev/null
```

## Action Scheduler Issues

**Queue status:**
```bash
wp db query "SELECT status, COUNT(*) as cnt FROM {prefix}actionscheduler_actions GROUP BY status ORDER BY cnt DESC" --path=/var/www/html
```
Red flags: `pending` > 1000, `failed` > 100, `in-progress` > 10.

**Stuck in-progress (zombies):**
```bash
wp db query "SELECT action_id, hook, scheduled_date_gmt FROM {prefix}actionscheduler_actions WHERE status='in-progress' AND last_attempt_gmt < DATE_SUB(NOW(), INTERVAL 10 MINUTE)" --path=/var/www/html
# Force back to pending:
wp db query "UPDATE {prefix}actionscheduler_actions SET status='pending', last_attempt_gmt=NULL, attempts=attempts+1 WHERE status='in-progress' AND last_attempt_gmt < DATE_SUB(NOW(), INTERVAL 10 MINUTE)" --path=/var/www/html
```

**Cleanup (if table has millions of rows):**
```bash
wp db query "DELETE FROM {prefix}actionscheduler_actions WHERE status='complete' AND last_attempt_gmt < DATE_SUB(NOW(), INTERVAL 30 DAY) LIMIT 10000" --path=/var/www/html
```
Run DELETE in batches of 10,000 to avoid table locks.

**Manually run queue:** `wp action-scheduler run --path=/var/www/html`

## System Cron Setup

If `DISABLE_WP_CRON` is true, add system cron entries:
```bash
# WP-Cron events
* * * * * cd /var/www/html && /usr/local/bin/wp cron event run --due-now --quiet 2>/dev/null
# Action Scheduler queue (more reliable for high volume)
*/2 * * * * cd /var/www/html && /usr/local/bin/wp action-scheduler run --quiet 2>/dev/null
```
