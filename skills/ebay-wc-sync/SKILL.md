---
name: ebay-wc-sync
description: >
  Diagnostic knowledge for debugging eBay-WooCommerce order sync failures.
  Covers CedCommerce, Codisto, QuickSync, WebKul, and custom integrations.
license: Apache-2.0
compatibility:
  - claude-code
  - openskills
  - any agent with shell access
metadata:
  category: diagnostics
  domain: ecommerce
  platforms: [wordpress, woocommerce, ebay]
  version: 1.0.0
---

# eBay-WooCommerce Sync Diagnostics

Troubleshoot broken order sync between eBay and WooCommerce. This skill gives
you the queries, commands, and decision trees to find the failure point fast.

## Prerequisites

This skill requires **shell access to the WordPress server** (SSH MCP, local
terminal, or similar). All WP-CLI and SQL commands must run where `wp` and
`mysql` are available.

## Table Prefix Detection

Before running any SQL query from the reference docs, detect the table prefix:

```bash
wp config get table_prefix --path=/var/www/html
```

Every query below uses `{prefix}` as a placeholder. Substitute the real value.

## Routing Table

| You're dealing with... | Start here |
|------------------------|------------|
| Orders not syncing at all | [diagnostic-workflow.md](references/diagnostic-workflow.md) |
| A specific known error or symptom | [common-failures.md](references/common-failures.md) |
| Need to query the database directly | [db-investigation.md](references/db-investigation.md) |
| Confused about which eBay API is in play | [api-guide.md](references/api-guide.md) |
| Using CedCommerce, Codisto, or another plugin | [plugin-specifics.md](references/plugin-specifics.md) |
| WP-Cron or Action Scheduler seems broken | [wp-cron-health.md](references/wp-cron-health.md) |
| Want the visual decision tree | [diagnostic-flowchart.md](assets/diagnostic-flowchart.md) |

## How This Skill Works

This is a pure knowledge skill. It contains no executable code or hooks.
You read the reference docs, run the commands on the server, and interpret
the results. The diagnostic workflow doc is the best starting point when
you don't know what's wrong yet.

## Quick Checks

These three commands tell you a lot in 30 seconds:

```bash
# Is WP-Cron disabled?
wp config get DISABLE_WP_CRON --path=/var/www/html

# How many pending actions in the scheduler?
wp action-scheduler list --status=pending --per-page=1 --format=count --path=/var/www/html

# Recent failed orders?
wp db query "SELECT ID, post_status, post_date FROM {prefix}posts WHERE post_type='shop_order' AND post_status='wc-failed' ORDER BY post_date DESC LIMIT 5" --path=/var/www/html
```
