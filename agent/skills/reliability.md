---
name: reliability
description: Analyze charging network reliability using the primary reliability metrics defined in semantic_models.yml - first attempt success rate, troubled success rate, failed rate, and average port uptime. Works with Snowflake DBT_PROD fact tables. Expects tables 'fact_visits' and 'fact_uptime'. Triggers when user asks about reliability, network reliability, uptime, downtime, outages, success rate, failed charges, or charger/port performance.
---

# Reliability Analysis

Report on the charging network's primary reliability metrics for a given time window, compared to the prior period of equal length.

## Requirements

- Database: Snowflake
- Tables: `DBT_PROD.fact_visits`, `DBT_PROD.fact_uptime`
- Metric definitions: `semantic_models.yml` (source of truth — do not invent metrics not defined there)

## Metrics

Per `semantic_models.yml`, the primary reliability metrics are:

| Metric | Definition |
|---|---|
| `average_uptime` | Average fraction of commissioned port-minutes not lost to outages (`fact_uptime`) |
| `first_attempt_success_rate` | Share of visits where the first charge attempt succeeded |
| `troubled_success_rate` | Share of visits that succeeded but needed more than one attempt |
| `failed_rate` | Share of visits that did not end in a successful charge |

## SQL Query

Default window is the last 7 days unless the user specifies otherwise. Compare against the immediately preceding period of the same length.

```sql
WITH current_visits AS (
  SELECT
    COUNT(*) AS total_visits,
    SUM(CASE WHEN is_successful AND charge_attempt_count = 1 THEN 1 ELSE 0 END) AS first_attempt_success,
    SUM(CASE WHEN is_successful AND charge_attempt_count > 1 THEN 1 ELSE 0 END) AS troubled_success,
    SUM(CASE WHEN NOT is_successful THEN 1 ELSE 0 END) AS failed_visits
  FROM DBT_PROD.fact_visits
  WHERE visit_end_ts >= DATEADD(day, -7, CURRENT_DATE())
),
previous_visits AS (
  SELECT
    COUNT(*) AS total_visits,
    SUM(CASE WHEN is_successful AND charge_attempt_count = 1 THEN 1 ELSE 0 END) AS first_attempt_success,
    SUM(CASE WHEN is_successful AND charge_attempt_count > 1 THEN 1 ELSE 0 END) AS troubled_success,
    SUM(CASE WHEN NOT is_successful THEN 1 ELSE 0 END) AS failed_visits
  FROM DBT_PROD.fact_visits
  WHERE visit_end_ts >= DATEADD(day, -14, CURRENT_DATE())
    AND visit_end_ts < DATEADD(day, -7, CURRENT_DATE())
),
current_uptime AS (
  SELECT AVG(uptime) AS avg_uptime
  FROM DBT_PROD.fact_uptime
  WHERE date_id >= DATEADD(day, -7, CURRENT_DATE())
),
previous_uptime AS (
  SELECT AVG(uptime) AS avg_uptime
  FROM DBT_PROD.fact_uptime
  WHERE date_id >= DATEADD(day, -14, CURRENT_DATE())
    AND date_id < DATEADD(day, -7, CURRENT_DATE())
)
SELECT
  cv.total_visits,
  cv.first_attempt_success,
  cv.troubled_success,
  cv.failed_visits,
  pv.total_visits AS prev_total_visits,
  pv.first_attempt_success AS prev_first_attempt_success,
  pv.troubled_success AS prev_troubled_success,
  pv.failed_visits AS prev_failed_visits,
  cu.avg_uptime,
  pu.avg_uptime AS prev_avg_uptime
FROM current_visits cv
CROSS JOIN previous_visits pv
CROSS JOIN current_uptime cu
CROSS JOIN previous_uptime pu;
```

To break metrics down by charger, port, or location, slice `fact_visits` by `first_charger_id`/`last_charger_id` or `fact_uptime` by `charger_id`/`port_id`, per the dimensions on the `visits`, `charge_attempts`, and `uptime` semantic models.

## Process

1. **Execute the SQL query** using the available database connection
2. **Compute rates** for the current and previous period:
   - `average_uptime = avg_uptime` (already a fraction)
   - `first_attempt_success_rate = first_attempt_success / total_visits`
   - `troubled_success_rate = troubled_success / total_visits`
   - `failed_rate = failed_visits / total_visits`
3. **Compute period-over-period change** in percentage points (current % − previous %) for each metric
4. **Format** all rates as percentages with one decimal place (e.g. `94.2%`), and pp deltas with a sign (e.g. `+1.3 pp`, `-0.8 pp`)

## Output Format

Start with a metrics-at-a-glance table, per house convention:

```markdown
## Reliability — Metrics at a Glance (last 7 days)

| Metric | Value | vs. prior 7 days | Status |
|---|---|---|---|
| Average uptime | XX.X% | +X.X pp | 🟢/🟡/🔴 |
| First attempt success rate | XX.X% | +X.X pp | 🟢/🟡/🔴 |
| Troubled success rate | XX.X% | +X.X pp | 🟢/🟡/🔴 |
| Failed rate | XX.X% | +X.X pp | 🟢/🟡/🔴 |
```

If visualizing, use the brand palette from `RULES.md` (e.g. Dark Turquoise `#165255` / Turquoise `#6AD8D6` for uptime, Purple `#C357AA` family for success/failure rates).
