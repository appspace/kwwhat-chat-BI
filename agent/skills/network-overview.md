---
name: network-overview
description: Summarize the size of the charging network - number of commissioned ports and number of locations with commissioned ports. Works with Snowflake DBT_PROD dimension tables. Expects tables 'dim_ports' and 'dim_chargers'. Triggers when user asks for a network overview, network size, how many ports/chargers/locations there are, or the current charger inventory.
---

# Network Overview

Report the current size of the charging network, counting only commissioned equipment.

## Requirements

- Database: Snowflake
- Tables: `DBT_PROD.dim_ports`, `DBT_PROD.dim_chargers`
- Column definitions: `marts.yml`

## Rules

- **Only count commissioned chargers**: a charger is commissioned when `dim_chargers.decommissioned_ts IS NULL`. Ports inherit commissioning status from their charger.
- `dim_chargers` is SCD Type 1 (current state only), so these counts describe the network **as of now**. Do not use them to answer "how big was the network on date X", and do not report a period-over-period change for them.

## Metrics

Report in this priority order:

| Priority | Metric | Definition |
|---|---|---|
| 1 | `commissioned_ports` | Distinct ports (`port_key`) whose charger is commissioned |
| 2 | `locations_with_commissioned_ports` | Distinct `location_id`s that have at least one commissioned port |

## SQL Query

```sql
SELECT
  COUNT(DISTINCT p.port_key) AS commissioned_ports,
  COUNT(DISTINCT c.location_id) AS locations_with_commissioned_ports
FROM DBT_PROD.dim_ports p
JOIN DBT_PROD.dim_chargers c ON p.charger_id = c.charger_id
WHERE c.decommissioned_ts IS NULL;
```

## Process

1. **Execute the SQL query** using the available database connection
2. **Format** counts as whole numbers with thousands separators (e.g. `1,284`)

## Output Format

Start with a metrics-at-a-glance table, per house convention:

```markdown
## Network Overview — Metrics at a Glance (as of today)

| Metric | Value |
|---|---|
| Commissioned ports | X,XXX |
| Locations with commissioned ports | XXX |
```

If visualizing, use the brand palette from `RULES.md` (e.g. Dark Turquoise `#165255` for ports, Purple `#C357AA` for locations).
