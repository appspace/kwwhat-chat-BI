# kwwhat-chat-BI

Chat BI for [kwwhat](https://github.com/appspace/kwwhat), the open-source EV charging context layer. Ask plain-English questions about charger reliability and get answers grounded in the kwwhat metrics, not made-up ones.

This repo is the Chat BI agent configuration: the rules, semantic models and skills that turn a general-purpose chat agent into an EV charging analyst. It runs on [nao](https://github.com/getnao/nao) and queries the kwwhat marts in Snowflake.

## What's in here

| Path | What it does |
|---|---|
| `RULES.md` | House rules the Chat BI agent follows: Snowflake SQL only, query `fact_*` and `dim_*` tables only, no invented metrics, results start with a "metrics at a glance" table, percentages with period-over-period change in pp, brand palette for charts |
| `docs/semantic_models.yml` | Authoritative metric, dimension and entity definitions (uptime, visits, first attempt success rate, and so on) |
| `docs/marts.yml` | Column-level documentation of the kwwhat marts (`fact_visits`, `fact_charge_attempts`, `fact_uptime`, `dim_ports`, and others) |
| `docs/project-overview.md` | Domain background: hardware hierarchy, charge attempts, visits, success criteria |
| `agent/skills/reliability.md` | Skill for the network reliability report: first attempt success, troubled success, failed rate and average uptime, compared with the prior period |
| `nao_config.yaml` | nao project config: the Snowflake connection and which tables are exposed to the Chat BI agent |

## Prerequisites

- The kwwhat dbt project built into a Snowflake schema (the rules assume `DBT_PROD`), which gives you the `fact_*` and `dim_*` tables
- Python 3 and [`nao-core`](https://pypi.org/project/nao-core/)
- A Snowflake user with key-pair authentication and read access to those tables
- An API key for the LLM provider you configure in nao

## Setup

```bash
python3 -m venv nao_venv
source nao_venv/bin/activate
pip install nao-core
```

Create a `.env` in the repo root. It is gitignored, so it stays local:

```bash
SNOWFLAKE_USER=
SNOWFLAKE_ACCOUNT_ID=
SNOWFLAKE_DATABASE=
SNOWFLAKE_WAREHOUSE=
SNOWFLAKE_SCHEMA_NAME=
SNOWFLAKE_PRIVATE_KEY=       # PEM contents of the encrypted private key
SNOWFLAKE_KEY_PASSPHRASE=
NAO_DEPLOY_API_KEY=          # your nao cloud API key, only needed to deploy
```

`nao_config.yaml` reads these through `env(...)`, so no credentials live in the repo.

## Run

Either run the Chat BI agent locally or deploy it to nao cloud at [app.getnao.io](https://app.getnao.io).

### Locally

```bash
source nao_venv/bin/activate
set -a && source .env && set +a   # export the variables from .env
nao chat
```

### Deploy to nao cloud

```bash
source nao_venv/bin/activate
set -a && source .env && set +a
nao deploy https://app.getnao.io --api_key $NAO_DEPLOY_API_KEY
```

## Try it

- "How reliable was the network last week?"
- "Which chargers had the lowest uptime this month?"
- "What's the first attempt success rate by location?"

## Notes

- The Chat BI agent is told not to use the word "session". It says "charge attempt", "transaction" or "visit", matching the kwwhat definitions.
- To add a metric, define it in `docs/semantic_models.yml` first. The Chat BI agent won't report metrics that aren't defined there.

## License

[MIT](LICENSE)
