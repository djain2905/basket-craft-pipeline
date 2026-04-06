# Basket Craft Pipeline — Design Spec

**Date:** 2026-04-06
**Project:** basket-craft-pipeline
**Status:** Approved

---

## Overview

A Python ETL pipeline that extracts sales order data from the Basket Craft MySQL database, transforms it into a monthly aggregated summary (revenue, order count, average order value by product category), and loads the result into a local PostgreSQL database running in Docker. The pipeline is triggered manually by running `python pipeline.py`.

---

## Architecture

**Approach:** Pandas ETL — extract raw tables, transform in Python, load to PostgreSQL.

```
MySQL Source
  └─ orders, order_items, products, categories
        ↓  read_sql()
   extract.py  →  4 raw DataFrames
        ↓  in-memory
  transform.py  →  merged + aggregated DataFrame
        ↓  to_sql()
    load.py
        ↓  INSERT
PostgreSQL (Docker · port 5433)
  └─ monthly_sales_summary
```

---

## Components

### extract.py

Connects to MySQL using SQLAlchemy + pymysql. Pulls the following tables as raw DataFrames via `read_sql()`:

- `orders` — order headers (`id`, `created_at`)
- `order_items` — line items (`order_id`, `product_id`, `price`, `qty`)
- `products` — product catalog (`id`, `name`, `category_id` or `category`)
- `categories` — category lookup (`id`, `name`) — pulled if the table exists; skipped otherwise

Returns all four DataFrames to the caller (`pipeline.py`).

### transform.py

Receives the four raw DataFrames and produces a single aggregated summary DataFrame. Steps:

1. **Merge** `order_items` onto `orders` on `order_id`
2. **Merge** `products` onto the result on `product_id`
3. **Resolve category name** — if `products` has a `category` text column, use it directly; otherwise join to `categories` on `category_id`
4. **Compute line revenue** — `price × qty` per row
5. **Floor date to year-month** — truncate `created_at` to the first day of the month (e.g., `2024-03-01`)
6. **Group by** `(year_month, category)` and aggregate:
   - `revenue` = `SUM(line_revenue)`
   - `order_count` = `COUNT(DISTINCT order_id)`
   - `avg_order_value` = `revenue / order_count`

Returns a DataFrame with columns: `year_month`, `category`, `revenue`, `order_count`, `avg_order_value`.

### load.py

Connects to PostgreSQL using SQLAlchemy + psycopg2. Writes the summary DataFrame using `to_sql(if_exists="replace")`, which truncates and rebuilds the table on each run. This makes the pipeline fully idempotent — safe to re-run without producing duplicate rows.

### pipeline.py

Entry point. Calls extract → transform → load in sequence:

```python
python pipeline.py
```

---

## Data Model

**Destination table:** `monthly_sales_summary`

| Column | Type | Description |
|---|---|---|
| `year_month` | DATE | First day of the month (e.g., `2024-01-01`) |
| `category` | TEXT | Product category name |
| `revenue` | NUMERIC(12,2) | Total revenue for that month + category |
| `order_count` | INTEGER | Number of distinct orders |
| `avg_order_value` | NUMERIC(10,2) | `revenue / order_count` |

---

## Infrastructure

### Docker (docker-compose.yml)

A new PostgreSQL 16 container, separate from the MP01 campus-bites container:

| Parameter | Value |
|---|---|
| Image | `postgres:16` |
| Host port | `5433` (avoids conflict with campus-bites on 5432) |
| Database | `basket_craft` |
| User | `student` |
| Password | `student123` |

Connection string: `postgresql://student:student123@localhost:5433/basket_craft`

### Environment Variables (.env)

```
# MySQL source
MYSQL_HOST=...
MYSQL_PORT=3306
MYSQL_DB=basket_craft
MYSQL_USER=...
MYSQL_PASSWORD=...

# PostgreSQL destination
PG_HOST=localhost
PG_PORT=5433
PG_DB=basket_craft
PG_USER=student
PG_PASSWORD=student123
```

### Python Stack (requirements.txt)

| Package | Purpose |
|---|---|
| `pandas` | Extract + transform DataFrames |
| `sqlalchemy` | Database connection abstraction for both MySQL and PostgreSQL |
| `pymysql` | MySQL driver (used by SQLAlchemy) |
| `psycopg2-binary` | PostgreSQL driver |
| `python-dotenv` | Load credentials from `.env` |

---

## Project File Structure

```
basket-craft-pipeline/
├── .env                    # Credentials (gitignored)
├── .env.example            # Credential template (committed)
├── docker-compose.yml      # PostgreSQL 16, port 5433
├── requirements.txt
├── pipeline.py             # Entry point
├── extract.py              # MySQL → raw DataFrames
├── transform.py            # pandas join + aggregate
├── load.py                 # PostgreSQL writer
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-04-06-basket-craft-pipeline-design.md
```

---

## Error Handling

- Connection errors (MySQL or PostgreSQL) will raise immediately with a clear SQLAlchemy error message — no silent failures.
- If `categories` table does not exist in MySQL, `extract.py` returns an empty DataFrame and `transform.py` falls back to using `products.category` directly.
- The pipeline does not implement retry logic — for a manually triggered script this is unnecessary.

---

## Out of Scope

- Incremental loads (append-only by date range) — full replace is sufficient for this dataset size
- Scheduling / cron automation — manual trigger only
- Data quality validation / row count assertions
- A dashboard UI — the PostgreSQL table is the deliverable; querying is done separately
