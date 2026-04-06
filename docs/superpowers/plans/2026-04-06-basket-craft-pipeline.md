# Basket Craft Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Python ETL pipeline that pulls Basket Craft sales data from MySQL, aggregates it into a monthly summary by product category, and loads it into a local PostgreSQL database running in Docker.

**Architecture:** Four focused modules — `extract.py` (MySQL → DataFrames), `transform.py` (pandas join + aggregate), `load.py` (DataFrame → PostgreSQL), orchestrated by `pipeline.py`. Tests are co-located in `tests/`. The transform module is pure pandas (no DB), making it the easiest to test first. Extract and load modules accept an engine parameter for clean test injection.

**Tech Stack:** Python 3, pandas, SQLAlchemy 2.x, pymysql, psycopg2-binary, python-dotenv, pytest, PostgreSQL 16 in Docker.

---

## File Map

| File | Responsibility |
|---|---|
| `requirements.txt` | Python dependencies |
| `.env.example` | Credential template (committed; real `.env` is gitignored) |
| `docker-compose.yml` | PostgreSQL 16 container on host port 5433 |
| `extract.py` | `get_mysql_engine()` + `extract(engine)` → 4 raw DataFrames |
| `transform.py` | `build_summary(df_orders, df_items, df_products, df_categories)` → summary DataFrame |
| `load.py` | `get_pg_engine()` + `load(df_summary, engine)` → writes to PostgreSQL |
| `pipeline.py` | Entry point: wires extract → transform → load |
| `tests/test_transform.py` | Unit tests for transform logic (pure pandas, no DB) |
| `tests/test_extract.py` | Unit tests for extract (mocked engine) |
| `tests/test_load.py` | Unit tests for load (mocked engine) |

---

## Task 1: Project Scaffolding

**Files:**
- Create: `requirements.txt`
- Create: `.env.example`
- Create: `docker-compose.yml`

- [ ] **Step 1: Create and activate virtual environment**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Expected: shell prompt changes to show `(.venv)`.

- [ ] **Step 2: Write `requirements.txt`**

```
pandas>=2.0
sqlalchemy>=2.0
pymysql>=1.1
psycopg2-binary>=2.9
python-dotenv>=1.0
pytest>=8.0
```

- [ ] **Step 3: Install dependencies**

```bash
pip install -r requirements.txt
```

Expected: all five packages install without errors.

- [ ] **Step 4: Write `.env.example`**

```
# MySQL source (Basket Craft)
MYSQL_HOST=
MYSQL_PORT=3306
MYSQL_DB=basket_craft
MYSQL_USER=
MYSQL_PASSWORD=

# PostgreSQL destination (Docker)
PG_HOST=localhost
PG_PORT=5433
PG_DB=basket_craft
PG_USER=student
PG_PASSWORD=student123
```

- [ ] **Step 5: Copy `.env.example` to `.env` and fill in your MySQL credentials**

```bash
cp .env.example .env
```

Open `.env` and fill in `MYSQL_HOST`, `MYSQL_USER`, and `MYSQL_PASSWORD` with the actual Basket Craft connection details.

- [ ] **Step 6: Write `docker-compose.yml`**

Docker Compose automatically reads the `.env` file in the same directory, so use variable substitution to keep credentials out of this file:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: basket_craft_db
    environment:
      POSTGRES_DB: ${PG_DB}
      POSTGRES_USER: ${PG_USER}
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    ports:
      - "${PG_PORT}:5432"
    volumes:
      - basket_craft_data:/var/lib/postgresql/data

volumes:
  basket_craft_data:
```

Docker Compose reads `PG_DB`, `PG_USER`, `PG_PASSWORD`, and `PG_PORT` from `.env` at startup — no credentials appear in the committed file.

- [ ] **Step 7: Start the PostgreSQL container**

```bash
docker compose up -d
```

Expected output:
```
✔ Container basket_craft_db  Started
```

- [ ] **Step 8: Verify the container is running and accepting connections**

```bash
docker compose ps
```

Expected: `basket_craft_db` shows status `running`.

```bash
docker exec basket_craft_db psql -U student -d basket_craft -c "SELECT 1;"
```

Expected:
```
 ?column?
----------
        1
(1 row)
```

- [ ] **Step 9: Commit**

```bash
git add requirements.txt .env.example docker-compose.yml
git commit -m "feat: add project scaffolding, Docker config, and dependencies"
```

---

## Task 2: Transform Module (TDD)

**Files:**
- Create: `tests/__init__.py`
- Create: `tests/test_transform.py`
- Create: `transform.py`

- [ ] **Step 1: Create the tests directory and empty `__init__.py`**

```bash
mkdir tests
touch tests/__init__.py
```

- [ ] **Step 2: Write the failing tests in `tests/test_transform.py`**

```python
import pandas as pd
import pytest
from transform import build_summary


def _make_inputs(with_category_col=True):
    """Helper: returns (df_orders, df_items, df_products, df_categories)."""
    df_orders = pd.DataFrame({
        'id': [1, 2, 3],
        'created_at': pd.to_datetime(['2024-01-15', '2024-01-20', '2024-02-10']),
    })
    df_items = pd.DataFrame({
        'order_id': [1, 2, 3],
        'product_id': [10, 10, 20],
        'price': [25.00, 50.00, 30.00],
        'qty': [2, 1, 1],
    })
    if with_category_col:
        df_products = pd.DataFrame({
            'id': [10, 20],
            'name': ['Basket A', 'Basket B'],
            'category': ['Woven', 'Wicker'],
        })
        df_categories = pd.DataFrame()
    else:
        df_products = pd.DataFrame({
            'id': [10, 20],
            'name': ['Basket A', 'Basket B'],
            'category_id': [5, 6],
        })
        df_categories = pd.DataFrame({'id': [5, 6], 'name': ['Woven', 'Wicker']})
    return df_orders, df_items, df_products, df_categories


def test_output_columns():
    result = build_summary(*_make_inputs())
    assert list(result.columns) == [
        'year_month', 'category', 'revenue', 'order_count', 'avg_order_value'
    ]


def test_revenue_is_price_times_qty():
    # order 1: 25*2=50, order 2: 50*1=50 → Jan/Woven revenue = 100
    result = build_summary(*_make_inputs())
    jan_woven = result[(result['year_month'] == '2024-01-01') & (result['category'] == 'Woven')]
    assert jan_woven.iloc[0]['revenue'] == 100.00


def test_order_count_is_distinct_orders():
    # orders 1 and 2 both fall in Jan/Woven → count = 2
    result = build_summary(*_make_inputs())
    jan_woven = result[(result['year_month'] == '2024-01-01') & (result['category'] == 'Woven')]
    assert jan_woven.iloc[0]['order_count'] == 2


def test_avg_order_value():
    # Jan/Woven: revenue=100, order_count=2 → AOV=50
    result = build_summary(*_make_inputs())
    jan_woven = result[(result['year_month'] == '2024-01-01') & (result['category'] == 'Woven')]
    assert jan_woven.iloc[0]['avg_order_value'] == 50.00


def test_groups_by_month_and_category():
    # Jan/Woven (orders 1+2) and Feb/Wicker (order 3) → 2 rows
    result = build_summary(*_make_inputs())
    assert len(result) == 2


def test_year_month_is_first_of_month():
    result = build_summary(*_make_inputs())
    assert result['year_month'].iloc[0] == pd.Timestamp('2024-01-01')


def test_category_resolved_from_categories_table():
    # products has category_id; categories table maps it to a name
    result = build_summary(*_make_inputs(with_category_col=False))
    assert set(result['category']) == {'Woven', 'Wicker'}
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
pytest tests/test_transform.py -v
```

Expected: all 7 tests FAIL with `ModuleNotFoundError: No module named 'transform'`.

- [ ] **Step 4: Implement `transform.py`**

```python
import pandas as pd


def build_summary(df_orders, df_items, df_products, df_categories):
    # Step 1: merge line items onto order headers
    df = df_items.merge(df_orders, left_on='order_id', right_on='id', suffixes=('_item', '_order'))

    # Step 2: merge products
    df = df.merge(df_products, left_on='product_id', right_on='id', suffixes=('', '_product'))

    # Step 3: resolve category name
    if 'category' in df_products.columns:
        df['category_name'] = df['category']
    else:
        df = df.merge(df_categories, left_on='category_id', right_on='id', suffixes=('', '_cat'))
        df['category_name'] = df['name_cat']

    # Step 4: compute line revenue
    df['line_revenue'] = df['price'] * df['qty']

    # Step 5: floor created_at to first of month
    df['created_at'] = pd.to_datetime(df['created_at'])
    df['year_month'] = df['created_at'].dt.to_period('M').dt.to_timestamp()

    # Step 6: group by month + category, aggregate
    summary = (
        df.groupby(['year_month', 'category_name'])
        .agg(
            revenue=('line_revenue', 'sum'),
            order_count=('order_id', 'nunique'),
        )
        .reset_index()
        .rename(columns={'category_name': 'category'})
    )
    summary['revenue'] = summary['revenue'].round(2)
    summary['avg_order_value'] = (summary['revenue'] / summary['order_count']).round(2)

    return summary[['year_month', 'category', 'revenue', 'order_count', 'avg_order_value']]
```

- [ ] **Step 5: Run tests to verify they all pass**

```bash
pytest tests/test_transform.py -v
```

Expected:
```
PASSED tests/test_transform.py::test_output_columns
PASSED tests/test_transform.py::test_revenue_is_price_times_qty
PASSED tests/test_transform.py::test_order_count_is_distinct_orders
PASSED tests/test_transform.py::test_avg_order_value
PASSED tests/test_transform.py::test_groups_by_month_and_category
PASSED tests/test_transform.py::test_year_month_is_first_of_month
PASSED tests/test_transform.py::test_category_resolved_from_categories_table
7 passed in ...
```

- [ ] **Step 6: Commit**

```bash
git add transform.py tests/
git commit -m "feat: add transform module with unit tests"
```

---

## Task 3: Extract Module (TDD)

**Files:**
- Create: `tests/test_extract.py`
- Create: `extract.py`

- [ ] **Step 1: Write the failing tests in `tests/test_extract.py`**

```python
import pandas as pd
from unittest.mock import patch, MagicMock
from extract import extract


def _mock_read_sql_side_effects():
    """Returns the four DataFrames read_sql will be called with, in order."""
    return [
        pd.DataFrame({'id': [1], 'created_at': ['2024-01-15']}),                              # orders
        pd.DataFrame({'order_id': [1], 'product_id': [10], 'price': [25.0], 'qty': [2]}),     # order_items
        pd.DataFrame({'id': [10], 'name': ['Basket A'], 'category': ['Woven']}),              # products
        pd.DataFrame({'id': [1], 'name': ['Woven']}),                                          # categories
    ]


def test_extract_returns_four_dataframes():
    mock_engine = MagicMock()
    mock_inspector = MagicMock()
    mock_inspector.get_table_names.return_value = [
        'orders', 'order_items', 'products', 'categories'
    ]

    with patch('extract.inspect', return_value=mock_inspector), \
         patch('extract.pd.read_sql', side_effect=_mock_read_sql_side_effects()):
        df_orders, df_items, df_products, df_categories = extract(mock_engine)

    assert list(df_orders.columns) == ['id', 'created_at']
    assert list(df_items.columns) == ['order_id', 'product_id', 'price', 'qty']


def test_extract_returns_empty_categories_when_table_missing():
    mock_engine = MagicMock()
    mock_inspector = MagicMock()
    mock_inspector.get_table_names.return_value = ['orders', 'order_items', 'products']

    side_effects = _mock_read_sql_side_effects()[:3]  # no categories read

    with patch('extract.inspect', return_value=mock_inspector), \
         patch('extract.pd.read_sql', side_effect=side_effects):
        _, _, _, df_categories = extract(mock_engine)

    assert df_categories.empty
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest tests/test_extract.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'extract'`.

- [ ] **Step 3: Implement `extract.py`**

```python
import os
import pandas as pd
from sqlalchemy import create_engine, inspect
from dotenv import load_dotenv

load_dotenv()


def get_mysql_engine():
    url = (
        f"mysql+pymysql://{os.getenv('MYSQL_USER')}:{os.getenv('MYSQL_PASSWORD')}"
        f"@{os.getenv('MYSQL_HOST')}:{os.getenv('MYSQL_PORT', 3306)}/{os.getenv('MYSQL_DB')}"
    )
    return create_engine(url)


def extract(engine):
    df_orders = pd.read_sql("SELECT id, created_at FROM orders", engine)
    df_items = pd.read_sql(
        "SELECT order_id, product_id, price, qty FROM order_items", engine
    )
    df_products = pd.read_sql("SELECT * FROM products", engine)

    inspector = inspect(engine)
    if 'categories' in inspector.get_table_names():
        df_categories = pd.read_sql("SELECT id, name FROM categories", engine)
    else:
        df_categories = pd.DataFrame()

    return df_orders, df_items, df_products, df_categories
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
pytest tests/test_extract.py -v
```

Expected:
```
PASSED tests/test_extract.py::test_extract_returns_four_dataframes
PASSED tests/test_extract.py::test_extract_returns_empty_categories_when_table_missing
2 passed in ...
```

- [ ] **Step 5: Commit**

```bash
git add extract.py tests/test_extract.py
git commit -m "feat: add extract module with unit tests"
```

---

## Task 4: Load Module (TDD)

**Files:**
- Create: `tests/test_load.py`
- Create: `load.py`

- [ ] **Step 1: Write the failing tests in `tests/test_load.py`**

```python
import pandas as pd
from unittest.mock import patch, MagicMock
from load import load


def _sample_summary():
    return pd.DataFrame({
        'year_month': pd.to_datetime(['2024-01-01', '2024-02-01']),
        'category': ['Woven', 'Wicker'],
        'revenue': [100.00, 60.00],
        'order_count': [2, 1],
        'avg_order_value': [50.00, 60.00],
    })


def test_load_calls_to_sql_with_replace():
    mock_engine = MagicMock()

    with patch('pandas.DataFrame.to_sql') as mock_to_sql:
        load(_sample_summary(), mock_engine)

    mock_to_sql.assert_called_once()
    _, kwargs = mock_to_sql.call_args
    assert kwargs['if_exists'] == 'replace'


def test_load_targets_correct_table():
    mock_engine = MagicMock()

    with patch('pandas.DataFrame.to_sql') as mock_to_sql:
        load(_sample_summary(), mock_engine)

    args, _ = mock_to_sql.call_args
    assert args[0] == 'monthly_sales_summary'


def test_load_does_not_write_index():
    mock_engine = MagicMock()

    with patch('pandas.DataFrame.to_sql') as mock_to_sql:
        load(_sample_summary(), mock_engine)

    _, kwargs = mock_to_sql.call_args
    assert kwargs['index'] is False
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest tests/test_load.py -v
```

Expected: FAIL with `ModuleNotFoundError: No module named 'load'`.

- [ ] **Step 3: Implement `load.py`**

```python
import os
from sqlalchemy import create_engine, Date, Text, Numeric, Integer
from dotenv import load_dotenv

load_dotenv()


def get_pg_engine():
    url = (
        f"postgresql+psycopg2://{os.getenv('PG_USER')}:{os.getenv('PG_PASSWORD')}"
        f"@{os.getenv('PG_HOST')}:{os.getenv('PG_PORT', 5433)}/{os.getenv('PG_DB')}"
    )
    return create_engine(url)


def load(df_summary, engine):
    df_summary.to_sql(
        'monthly_sales_summary',
        engine,
        if_exists='replace',
        index=False,
        dtype={
            'year_month': Date(),
            'category': Text(),
            'revenue': Numeric(12, 2),
            'order_count': Integer(),
            'avg_order_value': Numeric(10, 2),
        },
    )
    print(f"Loaded {len(df_summary)} rows into monthly_sales_summary.")
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
pytest tests/test_load.py -v
```

Expected:
```
PASSED tests/test_load.py::test_load_calls_to_sql_with_replace
PASSED tests/test_load.py::test_load_targets_correct_table
PASSED tests/test_load.py::test_load_does_not_write_index
3 passed in ...
```

- [ ] **Step 5: Commit**

```bash
git add load.py tests/test_load.py
git commit -m "feat: add load module with unit tests"
```

---

## Task 5: Pipeline Entry Point + End-to-End Run

**Files:**
- Create: `pipeline.py`

- [ ] **Step 1: Write `pipeline.py`**

```python
from extract import get_mysql_engine, extract
from transform import build_summary
from load import get_pg_engine, load


def main():
    print("Extracting from MySQL...")
    mysql_engine = get_mysql_engine()
    df_orders, df_items, df_products, df_categories = extract(mysql_engine)
    print(f"  orders: {len(df_orders)} rows")
    print(f"  order_items: {len(df_items)} rows")
    print(f"  products: {len(df_products)} rows")
    print(f"  categories: {len(df_categories)} rows")

    print("Transforming...")
    df_summary = build_summary(df_orders, df_items, df_products, df_categories)
    print(f"  summary: {len(df_summary)} rows")

    print("Loading to PostgreSQL...")
    pg_engine = get_pg_engine()
    load(df_summary, pg_engine)

    print("Pipeline complete.")


if __name__ == '__main__':
    main()
```

- [ ] **Step 2: Run the full test suite to confirm nothing is broken**

```bash
pytest tests/ -v
```

Expected: all 12 tests pass.

- [ ] **Step 3: Run the pipeline end-to-end**

Make sure the Docker container is running (`docker compose up -d`) and your `.env` has real MySQL credentials, then:

```bash
python pipeline.py
```

Expected output (row counts will vary):
```
Extracting from MySQL...
  orders: 1234 rows
  order_items: 5678 rows
  products: 42 rows
  categories: 8 rows
Transforming...
  summary: 24 rows
Loading to PostgreSQL...
Loaded 24 rows into monthly_sales_summary.
Pipeline complete.
```

- [ ] **Step 4: Verify the data landed in PostgreSQL**

```bash
docker exec basket_craft_db psql -U student -d basket_craft \
  -c "SELECT year_month, category, revenue, order_count, avg_order_value FROM monthly_sales_summary ORDER BY year_month, category LIMIT 10;"
```

Expected: rows with a `year_month` on the first of each month, a category name, and numeric values for revenue, order count, and AOV.

- [ ] **Step 5: Commit**

```bash
git add pipeline.py
git commit -m "feat: add pipeline entry point and verify end-to-end run"
```

- [ ] **Step 6: Push to GitHub**

```bash
git push origin main
```

---

## Self-Review Notes

- **Spec coverage:** All five spec sections covered — infrastructure (Task 1), extract (Task 3), transform (Task 2), load (Task 4), pipeline.py (Task 5). Output table schema enforced via SQLAlchemy `dtype` in `load.py`.
- **Category fallback:** `test_category_resolved_from_categories_table` covers the case where `products` has `category_id` + a `categories` table, and `test_extract_returns_empty_categories_when_table_missing` covers the case where `categories` doesn't exist.
- **Idempotency:** `if_exists='replace'` confirmed by `test_load_calls_to_sql_with_replace`.
- **No placeholders:** All steps have exact code and expected output.
- **Type consistency:** `build_summary` signature is consistent across Tasks 2–5. `extract(engine)` and `load(df_summary, engine)` signatures are consistent across Tasks 3–5.
