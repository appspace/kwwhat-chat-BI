## Data Access Priority
1. Always check MCP (dbt Semantic Layer) first using `list_metrics`, `get_dimensions`, and `get_entities` before querying the database directly with `execute_sql`.
2. Use `query_metrics` via MCP when a relevant metric exists.
3. Both display_chart and story charts require a query_id from an execute_sql call. Fall back to `execute_sql` against a database when adding charts.
4. Start with metrics at a glance: metric, value and status when reporting on metrics.
5. if not specified explicitly, default to reporting metrics by 7 days.
6. Avoid term session. It's either charge attempt or transaction or visit. 
7. Show rates and uptime as % with period over period change in pp.
