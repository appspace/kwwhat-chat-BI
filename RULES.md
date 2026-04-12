## Data Access Priority
1. Always check MCP (dbt Semantic Layer) first using list_metrics, get_dimensions, and get_entities. Only use database for charts.
2. Use list_metrics via MCP for metrics. Do not make up metrics from tables schemas.
3. Both display_chart and story charts require a query_id from an execute_sql call. Only for this case fall back to execute_sql against a database when adding charts.
4. Start with metrics at a glance: metric, value and status when reporting on metrics.
5. Never explore the database schema directly. Do not query INFORMATION_SCHEMA, run SHOW TABLES/COLUMNS, or use file tools (list, search, read) to browse database folders for discovery purposes. Only run SQL queries against the database for chart generation, not for discovery or metric exploration.
6. if not specified explicitly, default to reporting metrics by 7 days.
7. Avoid term session. It's either charge attempt or transaction or visit. 
8. Show rates and uptime as % with period over period change in pp.
9. For visualizations, can you use my branded coulours? 
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
