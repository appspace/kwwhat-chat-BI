---
name: reliability
description: Analyze charging network reliability using the primary reliability metrics defined in semantic_models.yml - first attempt success rate, troubled success rate, failed rate, and average port uptime. Works with Snowflake DBT_PROD fact tables. Expects tables 'fact_visits', 'fact_uptime' and 'fact_downtime_daily'. Triggers when user asks about reliability, network reliability, uptime, downtime, offline or faulted chargers, error codes, success rate, failed charges, or charger/port performance.
---

# Reliability Analysis

Report on the charging network's primary reliability metrics for a given time window, compared to the prior period of equal length.

## Requirements

- Database: Snowflake
- Tables: `DBT_PROD.fact_visits`, `DBT_PROD.fact_uptime`, `DBT_PROD.fact_downtime_daily`, `DBT_PROD.dim_error_codes` (for decoding error codes)
- Metric definitions: `semantic_models.yml` (source of truth — do not invent metrics not defined there)

## Metrics

Per `semantic_models.yml`, the primary reliability metrics are:

| Metric | Definition |
|---|---|
| `average_uptime` | Average fraction of commissioned port-minutes not lost to downtime (`fact_uptime`) |
| `first_attempt_success_rate` | Share of visits where the first charge attempt succeeded |
| `troubled_success_rate` | Share of visits that succeeded but needed more than one attempt |
| `failed_rate` | Share of visits that did not end in a successful charge |

## How Uptime Is Tracked

- **Lifespan only**: uptime is tracked only between a charger's commissioning and decommissioning. Time before commissioning or after decommissioning is neither uptime nor downtime — never count it, and never describe a decommissioned charger as "down".
- **Port grain**: uptime and downtime are tracked per port (the unit that serves one EV at a time).
- **Two downtime types** (`fact_downtime_daily.reason`):
  - `OFFLINE` — the charger stopped communicating (no heartbeat). Every port on the charger is down.
  - `FAULTED` — all connectors on the port reported a `Faulted` status.
- **What counts as downtime**:
  - One connector faulted on a multi-connector port while other connectors are still available → **not** downtime.
  - All connectors on the port are faulted → downtime.
  - Connector 0 (the charger-level connector in OCPP 1.6) reports `Faulted` → downtime for the whole charger.
- Use `fact_uptime.uptime` for uptime. Do not derive uptime by subtracting downtime minutes yourself.

## Attributing Faulted Downtime to Errors

FAULTED rows in `fact_downtime_daily` carry the error from the downtime on that day/port that ended most recently:

| Column | Meaning |
|---|---|
| `latest_error_code` | OCPP 1.6 `ChargePointErrorCode` (e.g. `HighTemperature`). Decode via `latest_error_code_key` → `dim_error_codes` |
| `latest_vendor_error_code` | Vendor-specific fault code (e.g. ChargeX `CX003`). Only meaningful together with `latest_taxonomy`. Decode via `latest_vendor_error_code_key` → `dim_error_codes` |
| `latest_taxonomy` | Vendor fault-code scheme that `latest_vendor_error_code` belongs to (e.g. `https://chargex.inl.gov`) |

- All three are null for OFFLINE rows — there is no error source when the charger isn't communicating. Report OFFLINE downtime as "offline (no heartbeat)", not as an unknown error.
- A FAULTED row may still have null codes if the charger didn't report one. Label it "no error code reported".
- Prefer the vendor code when present (it is more specific), and fall back to the OCPP code.
- Never decode a vendor code by `fault_code` alone — join on the key (taxonomy + code).
- Only the *latest* error per day/port is kept, so attribution is approximate when a port had several different faults on the same day. Mention this if it matters for the answer.

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

When uptime is below target or the user asks why chargers were down, break downtime down by reason and error:

```sql
SELECT
  d.reason,
  COALESCE(e.error_code_name,
           CASE WHEN d.reason = 'OFFLINE' THEN 'Offline (no heartbeat)' ELSE 'No error code reported' END) AS error_name,
  d.latest_taxonomy,
  d.latest_vendor_error_code,
  d.latest_error_code,
  COUNT(DISTINCT d.port_key) AS affected_ports,
  SUM(d.duration_minutes) AS downtime_minutes
FROM DBT_PROD.fact_downtime_daily d
LEFT JOIN DBT_PROD.dim_error_codes e ON d.latest_error_code_key = e.error_code_key
WHERE d.date_id >= DATEADD(day, -7, CURRENT_DATE())
GROUP BY 1, 2, 3, 4, 5
ORDER BY downtime_minutes DESC;
```

For a row-level drill-down into uptime (specific chargers, ports, or days), select `date_id, charger_id, port_id, reason, duration_minutes, latest_error_code, latest_vendor_error_code, latest_taxonomy` from `DBT_PROD.fact_downtime_daily`, ordered by `date_id DESC, duration_minutes DESC`.

To break metrics down by charger, port, or location, slice `fact_visits` by `first_charger_id`/`last_charger_id` or `fact_uptime` by `charger_id`/`port_id`, per the dimensions on the `visits`, `charge_attempts`, and `uptime` semantic models.

## Process

1. **Execute the SQL query** using the available database connection
2. **Compute rates** for the current and previous period:
   - `average_uptime = avg_uptime` (already a fraction)
   - `first_attempt_success_rate = first_attempt_success / total_visits`
   - `troubled_success_rate = troubled_success / total_visits`
   - `failed_rate = failed_visits / total_visits`
3. **Compute period-over-period change** in percentage points (current % − previous %) for each metric
4. **If uptime is below 98.0%** (or the user asks about downtime), run the downtime breakdown query and name the top reasons/errors after the summary table
5. **Format** all rates as percentages with one decimal place (e.g. `94.2%`), and pp deltas with a sign (e.g. `+1.3 pp`, `-0.8 pp`)

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

Status thresholds:

| Metric | 🟢 |
|---|---|
| Average uptime | ≥ 98.0% |

If visualizing, use the brand palette from `RULES.md` (e.g. Dark Turquoise `#165255` / Turquoise `#6AD8D6` for uptime, Purple `#C357AA` family for success/failure rates).
