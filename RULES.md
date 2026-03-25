## Data Access Priority
1. Always check MCP (dbt Semantic Layer) first using `list_metrics`, `get_dimensions`, and `get_entities` before querying the database directly with `execute_sql`.
2. Use `query_metrics` via MCP when a relevant metric exists.
3. Fall back to `execute_sql` only when the required data is not available through the Semantic Layer.
