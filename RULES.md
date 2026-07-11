- The database is **SNOWFLAKE**. Always use SNOWFLAKE SQL syntax.

- Never run schema introspection queries (`information_schema`, `SHOW TABLES`, etc.). The available tables and their columns are already provided in your context.
- For metrics, measures, dimensions, and entities: use `semantic_models.yml` as the authoritative source.
- For SQL: only query `fact_*` and `dim_*` tables. Always use the fully qualified path `DBT_PROD.<table>` (e.g. `DBT_PROD.fact_visits`).
- Do not make up metrics. If a metric is not defined in the semantic model, say so.
- When reporting on metrics, always start with a "metrics at a glance" summary table: metric name, value, and status.
- Default time window is last 7 days unless the user specifies otherwise.
- Never use the term "session". Use "charge attempt", "transaction", or "visit" depending on context.
- Show rates and uptime as percentages (e.g. 94.2%). Always include period-over-period change in percentage points (pp), e.g. "+1.3 pp".
- Always use the brand colour palette for visualizations:
Light Purple #F3DDEE
Purple#C357AA
Dark Purple#7C2167
Light Turquoise 100#D2F3F3
Turquoise#6AD8D6
Dark Turquoise#165255
Yellow (Accent)#FFD72E
Bluish (Accent)#1F0D79
White#FFFFFF
Grey-black#2A2A2A
