# ANARIX AI Agent – Full Technical Documentation

This document explains the ANARIX AI Agent business intelligence platform in full detail: what the project does, how it is architected, how data flows end-to-end, and how to extend or build a similar system.

---

## 1. Problem Statement & Vision

Modern business teams want to ask natural-language questions like:

- “Which products have the highest ROAS this month?”
- “Show me products with CPC below $2 and at least 1000 impressions.”
- “What is the total revenue and ad spend by product?”

They do not want to:

- Learn SQL or database schemas.
- Manually build charts and dashboards.

**ANARIX AI Agent** solves this by:

- Converting **natural language** questions into **PostgreSQL** queries using **LLaMA 3.1**.
- Executing those queries against a curated business database.
- Returning **tabular results, explanations, and ready-to-use visualizations**.

The system is designed so that another engineer can:

- Plug in new business datasets (especially Excel-based exports).
- Reuse the same architecture to build a new NL→SQL BI agent.

---

## 2. High-Level Architecture

At a high level, the system consists of:

1. **FastAPI backend(s)** for:
   - A **pure LLaMA API** (no pattern matching) that directly answers questions ([app/main.py]).
   - An **enhanced BI web interface** with validation logic, visualizations, and business rules ([app/query_interface.py]).
2. **PostgreSQL database** that holds three core analytical tables ([database/models.py]):
   - `ad_sales` – ad performance metrics.
   - `total_sales` – consolidated product revenue and units.
   - `eligibility` – product eligibility/compliance information.
3. **LLM service layer** ([services/llm_service.py]):
   - Talks to a local **Ollama** server hosting LLaMA 3.1:8b.
   - Generates SQL based on a strict schema and business rules.
4. **Visualization & formatting services**:
   - [services/visualization_service.py] – Plotly-based charts.
   - [services/response_formatter.py] – Uniform responses and legacy helpers.
5. **Data acquisition / ETL from Excel**:
   - [real_data_integration/excel_to_database.py] – Reads Excel exports and loads them into the 3-table schema in PostgreSQL.
6. **Utility scripts** for debugging the database and schema:
   - `check_*.py`, `show_tables.py`, `view_data.py`, etc.

The system is intentionally modular:

- **LLM is isolated** in its own service (easy to swap or upgrade model).
- **Database schema is explicit and strongly typed** via SQLAlchemy models.
- **Visualization is encapsulated** so you can change chart types or libraries later.

---

## 3. Core Data Model (Business Tables)

Defined in [database/models.py]. These tables represent your "business truth".

### 3.1. ad_sales

**Table:** `ad_sales`  
**Purpose:** Ad performance metrics per product and date (3,696+ records).

Key columns:

- `item_id` (TEXT) – primary logical key used for joins.
- `date` (DATE) – when the record was captured.
- `ad_sales` (DECIMAL) – revenue attributable to ads.
- `impressions` (INTEGER) – number of times the ad was shown.
- `ad_spend` (DECIMAL) – how much was spent on the ads.
- `clicks` (INTEGER) – number of ad clicks.
- `units_sold` (INTEGER) – units sold via ads.

Indexes in `AdSales` optimize typical BI queries:

- By item and date (time series, per-product trends).
- By revenue and spend (ROI / ROAS calculations).
- By clicks and impressions (CPC, CTR analysis).

### 3.2. total_sales

**Table:** `total_sales`  
**Purpose:** Overall product revenue and unit sales (702+ records).

Columns:

- `item_id` – foreign key-typed join field (string).
- `date` – date of the transaction.
- `total_sales` – total revenue for that item/date.
- `total_units_ordered` – total units ordered.

Indexes support revenue rankings and time-based analysis.

### 3.3. eligibility

**Table:** `eligibility`  
**Purpose:** Product compliance/eligibility status (4,381+ records).

Columns:

- `item_id` – join key across tables.
- `eligibility_datetime_utc` – timestamp when eligibility was evaluated.
- `eligibility` – status label (e.g., "eligible" / "ineligible").
- `message` – human-readable reason for eligibility.

Indexes let you query by item, status, and timestamp.

### 3.4. BusinessIntelligenceView helper

`BusinessIntelligenceView` in [database/models.py] defines canonical join SQL:

- Joins `ad_sales`, `total_sales`, and `eligibility` on `item_id`.
- Provides table-level metadata used for reasoning about queries.

This is a **virtual view** pattern: you don’t create a physical view, but you centralize how multi-table queries should be structured.

---

## 4. Data Ingestion from Excel

Real data comes from Excel exports and is loaded into PostgreSQL via [real_data_integration/excel_to_database.py].

### 4.1. Excel Files

The converter expects three Excel files in the project root:

- `Copy of Product-Level Ad Sales and Metrics (mapped).xlsx`
- `Copy of Product-Level Total Sales and Metrics (mapped).xlsx`
- `Copy of Product-Level Eligibility Table (mapped).xlsx`

These are mapped to the three tables:

- `ad_sales`
- `total_sales`
- `eligibility`

### 4.2. Cleaning & Normalization

For each Excel file:

1. Read with pandas (`pd.read_excel`).
2. Clean column names:
   - Lowercase.
   - Replace spaces, dashes, dots, and `%` with safe characters.
3. Convert `NaN` to Python `None`.

### 4.3. Creating Tables (if needed)

If the tables do not exist yet, the converter creates them using plain SQL with types that match [database/models.py].

### 4.4. Inserting Data

For each row in each cleaned DataFrame, the converter:

- Casts values to appropriate types (floats, ints, dates).
- Inserts into the correct table using asyncpg.
- Skips rows that raise conversion or constraint errors.

At the end, it prints record counts for each table to validate the load.

### 4.5. End-to-End ETL Flow

1. **Excel** → pandas DataFrame (cleaned/normalized).  
2. DataFrame → PostgreSQL tables `ad_sales`, `total_sales`, `eligibility`.  
3. Tables are then ready for:
   - LLaMA-generated SQL queries.
   - Business logic–based SQL generation.
   - Visualizations and BI insights.

To rerun the conversion, run the script as a standalone program (it has a `__main__` entry point using `asyncio.run`).

---

## 5. Database Connectivity & Initialization

Database connectivity is configured in [database/connection.py].

### 5.1. SQLAlchemy Engine & Session

- `DATABASE_URL` points to `postgresql://postgres:ANARIX_AI_2024!@localhost:5432/sales_analytics` (you should change credentials for production).
- `engine` is a `QueuePool` with:
  - `pool_size=20`, `max_overflow=30`, `pool_timeout=300`, `pool_recycle=3600`.
- `SessionLocal` is a `sessionmaker` with `expire_on_commit=False` for BI-style workloads.

`create_tables()` uses `Base.metadata.create_all(bind=engine)` to create all SQLAlchemy models.

### 5.2. Async PostgreSQL Connections

For I/O-bound query execution, asyncpg is used:

- `get_async_connection()` opens an async connection with:
  - Increased `command_timeout`, `work_mem`, and `temp_buffers`.
  - `jit` disabled for predictable performance.

`test_excel_data_connectivity()` and `initialize_anarix_database()`

- Validate that `ad_sales`, `total_sales`, and `eligibility` are populated.
- Run a join test to ensure the schema is consistent and queryable.

---

## 6. LLM Service (Natural Language → SQL)

Defined in [services/llm_service.py]. This is the **core intelligence** of the system.

### 6.1. External Model

- Uses **Ollama** at `http://127.0.0.1:11434/api/generate`.
- Model: `llama3.1:8b`.
- Communication via `aiohttp` (async HTTP client).

### 6.2. Prompt Design

The `LLMService` builds a **very detailed system prompt** containing:

- **Full database schema** (3 tables with column descriptions).
- **Definitions & formulas** for business metrics:
  - ROI, ROAS, CPC, CTR.
- **Question interpretation rules** (e.g., how to treat "top X", "total", "average", etc.).
- **Formatting rules**:
  - SQL only, no extra text.
  - Explicit aliases and ROUND(...::numeric, 2) formatting.
  - Correct clause ordering (SELECT → FROM → WHERE → GROUP BY → ORDER BY → LIMIT).

The model is instructed to output **only a SQL query** that exactly answers the user’s question.

### 6.3. SQL Extraction & Validation

After getting the model response, `LLMService`:

1. Extracts the first `SELECT` statement using regex.
2. Ensures the query:
   - Starts with `SELECT`.
   - Does not reference forbidden table names (e.g., generic names like `SALES_DATA`, `PRODUCT`, `USER`, etc.).
   - Uses only the allowed tables (`ad_sales`, `total_sales`, `eligibility`).
3. Validates that the SQL **matches the question intent**:
   - For ROI questions: uses both `total_sales` and `ad_spend` with the right formula.
   - For ROAS: uses `ad_sales / ad_spend`.
   - For CPC: uses `ad_spend / clicks`.
   - For "top N" questions: ensures `LIMIT N` exists.
   - For "calculate/show all" questions: avoids `LIMIT` unless semantically necessary.

### 6.4. Intelligent Fallbacks

If any of the above checks fail (LLM output is missing, invalid, or misaligned with user intent), the service falls back to **predefined, question-specific SQL templates** that are known to be correct.

Examples:

- CPC questions → `SELECT item_id, ad_spend, clicks, ... as cpc FROM ad_sales ...`.
- ROI questions → join `ad_sales` and `total_sales`.
- ROAS questions → `ad_sales / ad_spend`.
- Total revenue → `SUM(total_sales)` over `total_sales`.

This ensures the system produces **safe and useful SQL** even when the LLM fails or drifts.

---

## 7. FastAPI Applications

There are two main FastAPI entrypoints: [app/main.py] and [app/query_interface.py].

### 7.1. Pure LLaMA API (app/main.py)

This file defines a **pure LLaMA** backend with minimal pattern logic.

Key components:

- `app = FastAPI(title="ANARIX AI Agent – Pure LLaMA System", version="4.0.0")`.
- `llm_service = LLMService()` to convert questions into SQL.
- `viz_service = VisualizationService()` for simple charts.
- `mock_data = MockDataService()` for fallback visualization data.
- `create_tables()` is called on import to ensure the DB schema exists.

#### `/` – Health Check

Returns static metadata about the app, version, and available endpoints.

#### `/query` – Pure LLaMA Query

Request body: `{ "question": "..." }`.

Processing steps:

1. Validate non-empty question.
2. Call `llm_service.convert_to_sql(question)` – **no pattern override**.
3. Execute the **exact generated SQL** using asyncpg:
   - First, try `fetch` (multi-row).
   - If that fails, try `fetchval` (single value).
   - If that fails, try `fetchrow` (single row).
4. Return a JSON structure including:
   - `result` – raw data (list, dict, or scalar).
   - `query_type` – `multi_row`, `single_value`, `single_row`, etc.
   - `record_count`.
   - `generated_sql` – the actual SQL produced by LLaMA.

#### Visualization endpoints

- `/visualize/total-sales`
- `/visualize/roas`
- `/visualize/highest-cpc`

Each endpoint:

1. Tries to query PostgreSQL for relevant metrics.
2. If DB is unavailable, uses `MockDataService` to generate synthetic data.
3. Uses `VisualizationService.generate_chart(...)` (or a thin wrapper) to produce a Plotly chart JSON.

This API is ideal for **programmatic integration** and testing the LLaMA-to-SQL pipeline.

### 7.2. Enhanced BI Interface (app/query_interface.py)

This file builds a **user-facing web experience** with templates and streaming responses.

Key setup:

- `app = FastAPI(title="ANARIX AI Agent - Enhanced BI Platform", ...)`.
- Mounts static files: `app.mount("/static", StaticFiles(directory="static"), name="static")`.
- Jinja2 templates: `templates = Jinja2Templates(directory="templates")`.

#### `/` – HTML Query Interface

- Renders `templates/query_interface.html` – the main user interface.
- Provides a form where the user types a question.

#### `/query` – Form-based Processing

Input: form field `question`.

The endpoint implements a **robust, multi-layered pipeline**:

1. **Attempt LLM SQL generation** using `LLMService`.
2. If LLM fails or the SQL structure doesn’t look valid, call `generate_business_sql(question)`:
   - Pattern-based SQL generator that recognizes business concepts like CPC, ROAS, ROI, and revenue.
3. Run `validate_sql_completely(sql)`:
   - `fix_table_references(sql)` – ensures correct table aliases (`a`, `t`, `e`).
   - `fix_column_references(sql)` – rewrites ambiguous columns to fully qualified forms.
   - `fix_sql_syntax(sql)` – cleans spacing and common syntax issues.
4. Execute the validated SQL via `get_async_connection()`.
5. If the DB execution still fails:
   - Use `generate_safe_fallback_sql(question)` – a conservative, safe query that is guaranteed to work.
6. Once data is retrieved:
   - `generate_enhanced_sql_explanation(...)` – (in the same file) describes what the SQL is doing.
   - `create_guaranteed_visualization(data, question, columns)` – ensures a chart is always produced when possible.
   - `generate_business_insights(data, question)` – produces textual insights.
7. Return a JSON response with:
   - `generated_sql` – the final SQL used.
   - `explanation` – human-readable explanation of the query.
   - `data` and `columns` – result set.
   - `visualization` – chart JSON and metadata.
   - `insights` – bullet-point business insights.
   - `processing_time` – execution time.

This endpoint is more **strict and user-friendly** than the pure LLaMA API.

---

## 8. Visualization & Insights

### 8.1. VisualizationService

Located in [services/visualization_service.py].

Responsibilities:

- Determine if visualization is appropriate (`should_visualize`).
- Convert query results into a pandas DataFrame.
- Detect numeric columns for plotting.
- Create a simple, reliable bar chart as a minimum guaranteed visualization.

Key behavior:

- Chooses `item_id` (if available) as the x-axis; otherwise uses the first column.
- Uses the first numeric column as y-axis.
- Limits to the top 15 rows for readability.
- Returns Plotly figure JSON and metadata (title, recommendations, summary).

Titles and recommendations are tailored based on the question (e.g., CPC, ROAS, ROI, or general revenue/sales).

### 8.2. ResponseFormatter

Located in [services/response_formatter.py].

- Provides generic `format_simple_response(...)` used by the pure LLaMA system.
- Contains legacy helpers for formatting specific metric responses used by visualization endpoints.

This layer standardizes responses so that frontends can rely on a consistent JSON structure.

### 8.3. MockDataService

Located in [data/mock_data.py].

- Generates realistic synthetic business data for testing.
- Provides helpers like `get_total_sales`, `get_roas`, `get_highest_cpc_product`, `get_sales_by_region`.

Used as fallback when the real database is unavailable.

---

## 9. End-to-End Request Flow

This section traces what happens when a user asks a question from the web interface.

### 9.1. User Journey (Web UI)

1. User opens the homepage (query interface HTML) – served by [app/query_interface.py].
2. User types a natural-language question and submits the form.
3. Browser sends a POST to `/query` with `question` in the form body.
4. Backend processes the question (LLM → SQL → DB → visualization).
5. JSON response is used by front-end JavaScript to:
   - Display the structured data table.
   - Render the chart using Plotly.
   - Show explanations and business insights.

### 9.2. User Journey (API Client)

1. Client sends POST to `/query` on the **pure LLaMA** API ([app/main.py]) with JSON body `{ "question": "..." }`.
2. Backend:
   - Calls `LLMService.convert_to_sql` with the detailed system prompt.
   - Executes the exact SQL returned.
   - Returns raw results and metadata as JSON.

This is ideal for experimentation, testing, or integration with other services.

---

## 10. How to Run the System Locally

### 10.1. Prerequisites

- Python 3.8+
- PostgreSQL running locally (`sales_analytics` database).
- Ollama installed with the `llama3.1:8b` model pulled and available.

### 10.2. Python Dependencies

Install required packages from `requirements.txt` (FastAPI, asyncpg, SQLAlchemy, pandas, Plotly, Matplotlib, Seaborn, aiohttp, etc.).

Typical command:

```bash
pip install -r requirements.txt
```

### 10.3. Database Setup

1. Create a PostgreSQL database `sales_analytics` and user with the configured credentials.
2. Run any of the following to create tables:
   - Importing [app/main.py] or [database/connection.py] (they call `create_tables()`).
   - Running `initialize_anarix_database()` in [database/connection.py].
3. Optionally, use [real_data_integration/excel_to_database.py] to import real Excel data.

### 10.4. Running the Pure LLaMA API

From the project root:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Then access:

- `http://localhost:8000/` – health check.
- `POST http://localhost:8000/query` – send questions.

### 10.5. Running the Query Interface App

If [app/query_interface.py] is also run via Uvicorn:

```bash
uvicorn app.query_interface:app --host 0.0.0.0 --port 8001 --reload
```

Then open in the browser:

- `http://localhost:8001/` – interactive query interface.

(Ports are examples; adjust as needed.)

---

## 11. Extending or Reusing This Project

This section explains how to build a **similar project** for a different business domain.

### 11.1. Replace or Extend the Data Model

Steps:

1. Define new SQLAlchemy models in [database/models.py] for your domain.
2. Update ETL scripts (e.g., [real_data_integration/excel_to_database.py]) to convert your Excel/CSV exports into your schema.
3. Add or modify indexes to reflect typical BI queries (e.g., by date, region, product, customer).

### 11.2. Update the LLM Prompt

In [services/llm_service.py]:

1. Update the `self.schema` string with your new table and column definitions.
2. Adjust metric definitions (e.g., margin, churn, LTV) and formulas.
3. Extend **question interpretation rules** to cover your business questions.

This guides LLaMA so it can reason correctly about your domain.

### 11.3. Add New Business-Specific Fallbacks

Extend `_get_question_specific_fallback` and validation logic:

- Add templates for new metrics (e.g., customer lifetime value, retention rate, net revenue, etc.).
- Ensure they use the correct tables and formulas.

This guarantees that, even if the LLM output is imperfect, the system still returns useful answers.

### 11.4. Customize Visualizations

In [services/visualization_service.py]:

- Change chart types based on data (e.g., line charts for time series, stacked bars for category breakdowns).
- Use additional numeric columns or groupings as needed.

You can create new methods that, for example:

- Automatically detect date columns and plot trends.
- Produce KPI summary cards alongside charts.

### 11.5. Enhance the Frontend

In [templates/query_interface.html] and any JavaScript it uses:

- Customize styling and layout to match your brand.
- Add filters, date pickers, or saved query templates.
- Implement streaming results if you want progressive updates.

### 11.6. Deploying to Production

To productionize this system:

- Containerize with Docker (database + FastAPI + Ollama).
- Add authentication/authorization around the APIs.
- Harden database credentials and secrets (environment variables, secret stores).
- Add logging and monitoring for queries, performance, and errors.

---

## 12. Key Design Principles

- **Schema-first**: Explicit, well-indexed tables guide both the LLM and human developers.
- **LLM with guardrails**: The LLM is powerful but constrained:
  - Only 3 allowed tables.
  - Validated formulas.
  - Fallbacks when output is not trustworthy.
- **Separation of concerns**:
  - LLM logic in [services/llm_service.py].
  - Data access in [database/connection.py] and [database/models.py].
  - Visualization in [services/visualization_service.py].
  - Web and API layers in [app/main.py] and [app/query_interface.py].
- **Always-return-something** philosophy:
  - If the LLM fails → business-rule SQL.
  - If that fails → safe fallback SQL.
  - If DB fails → mock data for visualizations.

---

## 13. Summary

By reading this document, you should now understand:

- What ANARIX AI Agent is trying to achieve (NL → SQL BI for ad & sales data).
- How the database, LLM, API, and visualizations fit together.
- How real Excel exports are transformed into a clean, queryable schema.
- How to run the system locally and explore it.
- How to adapt the same architecture to another dataset or business domain.

This project serves as a complete blueprint for building **LLM-powered analytics agents** that sit directly on top of structured business data and provide intuitive, powerful, and explainable insights to non-technical users.