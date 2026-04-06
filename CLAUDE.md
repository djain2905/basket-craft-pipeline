# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ETL pipeline: pulls sales data from a remote Basket Craft MySQL database, aggregates it into a monthly summary by product category, and loads it into a local PostgreSQL database running in Docker. Entry point is `python pipeline.py`.

## Environment

Use a Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Database

PostgreSQL runs in Docker on port **5433** (separate from any other local containers):

```bash
docker compose up -d      # start
docker compose down       # stop
docker compose ps         # check status
```

Verify connection:
```bash
docker exec basket_craft_db psql -U student -d basket_craft -c "SELECT 1;"
```

## Running the Pipeline

```bash
python pipeline.py
```

## Running Tests

```bash
pytest tests/ -v               # all tests
pytest tests/test_transform.py # transform only (no DB required)
```

## Architecture

Four modules with single responsibilities, called in sequence by `pipeline.py`:

| Module | Role |
|---|---|
| `extract.py` | `get_mysql_engine()` + `extract(engine)` — pulls `orders`, `order_items`, `products`, `categories` from MySQL as DataFrames |
| `transform.py` | `build_summary(df_orders, df_items, df_products, df_categories)` — merges and aggregates into monthly summary |
| `load.py` | `get_pg_engine()` + `load(df_summary, engine)` — writes to PostgreSQL with `if_exists="replace"` (idempotent) |
| `pipeline.py` | Wires the three modules together |

The transform is pure pandas (no DB) — test it without any running services. Extract and load accept an `engine` parameter for clean test injection via mocks.

**Output table:** `monthly_sales_summary` — columns: `year_month` (DATE, first of month), `category` (TEXT), `revenue` (NUMERIC), `order_count` (INTEGER), `avg_order_value` (NUMERIC).

## Environment Variables (.env)

```
MYSQL_HOST=db.isba.co
MYSQL_PORT=3306
MYSQL_DATABASE=basket_craft
MYSQL_USER=analyst
MYSQL_PASSWORD=

PG_HOST=localhost
PG_PORT=5433
PG_DB=basket_craft
PG_USER=student
PG_PASSWORD=student123
```

Docker Compose reads these same variables for container startup — no credentials are hardcoded anywhere.
