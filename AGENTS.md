# AGENTS.md — Universal Agent Discovery

<available_skills>
  <skill name="ebay-wc-sync"
         keywords="ebay woocommerce sync order import webhook CedCommerce Codisto QuickSync WebKul eBay-WooCommerce order-sync debugging diagnostics WordPress WP-Cron Action-Scheduler OAuth token fulfillment trading-api inventory-api SKU mismatch duplicate-orders missing-orders stuck-processing category-mapping webhook-failure"
         description="Diagnostic knowledge for debugging eBay-WooCommerce order sync failures. Covers CedCommerce, Codisto, QuickSync, WebKul, and custom integrations. Includes SQL query library, API compatibility guide, plugin-specific debugging patterns, and step-by-step triage workflows."
         path="skills/ebay-wc-sync/SKILL.md" />
</available_skills>

## What This Skill Does

When eBay orders aren't reaching WooCommerce — or arrive mangled, duplicated,
or incomplete — this skill tells you exactly where to look and what commands
to run. It covers the five most popular integration plugins and works with
custom-built sync systems too.

This is a pure knowledge skill. No executable code, no hooks. Your agent reads
the reference docs and runs the commands on the server.

## Installation

### Claude Code

```bash
claude plugin add axcrockett/ebay-wc-sync
```

### OpenSkills

```bash
npx openskills install axcrockett/ebay-wc-sync
```

Or invoke directly:

```
<invoke>openskills read ebay-wc-sync</invoke>
```

### Manual

Clone the repo into your agent's skill directory:

```bash
git clone https://github.com/axcrockett/ebay-wc-sync.git
```

Then point your agent to `skills/ebay-wc-sync/SKILL.md` as the entry point.

## Prerequisites

- Shell access to the WordPress server (SSH, local terminal, or SSH MCP)
- WP-CLI installed on the server
- MySQL/MariaDB query access (via `wp db query` or direct)

## Covered Plugins

| Plugin | Coverage |
|--------|----------|
| CedCommerce (eBay Integration for WooCommerce) | Full — meta keys, AS hooks, known bugs, patches |
| Codisto | Moderate — sync tables, webhook patterns, known issues |
| QuickSync (WP-Lister for eBay) | Moderate — config, limitations |
| WebKul | Basic — hook priorities, meta keys |
| Custom integrations | Architectural anti-patterns checklist |

## Reference Docs

| Doc | What it covers |
|-----|---------------|
| `diagnostic-workflow.md` | Step-by-step triage from "orders not syncing" to root cause |
| `common-failures.md` | 8 failure types with symptoms, cause, confirm, fix |
| `db-investigation.md` | SQL query library (all use `{prefix}` placeholder) |
| `api-guide.md` | Trading vs Inventory vs Fulfillment API — the incompatibility trap |
| `plugin-specifics.md` | Per-plugin debugging: CedCommerce, Codisto, QuickSync, WebKul |
| `wp-cron-health.md` | WP-Cron and Action Scheduler diagnostics |
| `diagnostic-flowchart.md` | Visual decision tree (Mermaid + text fallback) |
