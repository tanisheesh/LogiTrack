# LogiTrack — Architecture

<!--
Companion to PRD.md.
PRD says WHAT the system does. This says HOW.
Audience: an engineer who needs to understand the system well
enough to build it, debug it, or extend it.
No marketing language here — be precise.
-->

---

## 1. Stack

| Layer | Tech |
|---|---|
| UI Framework | Streamlit 1.28+ (single-page Python app) |
| Language | Python 3.9+ |
| Data processing | Pandas 2.1 · NumPy 1.26 |
| Optimization | Custom greedy allocator (`src/backend/optimizer.py`) |
| Forecasting | Prophet 1.1 (Facebook/Meta) · scikit-learn 1.3 |
| Visualization | Plotly 5.18 (geo-scatter, bar, pie) · Folium 0.14 (Leaflet maps) |
| Database | SQLite (default, auto-created) · PostgreSQL · MySQL via SQLAlchemy 2.0 |
| Dev tooling | Black · Pylint · Pytest |

---

## 2. Components

```
LogiTrack/
  app.py                      Entry point — Streamlit app class, all page routing
  src/
    config.py                 Central constants (paths, optimization params, colors)
    backend/
      data_loader.py          CSV / file-upload / database ingestion and validation
      optimizer.py            Greedy inventory allocation engine
      forecaster.py           Prophet-based demand forecasting wrapper
    database/
      db_manager.py           SQLite DDL, connection management, CSV import utility
    frontend/
      visualizations.py       Reusable Plotly figure factories (map, cost compare, utilization pie)
    utils/
      helpers.py              Haversine distance, currency formatting, date parsing, export
  data/
    sample_warehouses.csv     6 global warehouse seed records (Mumbai, Singapore, Dubai, NY, London, Tokyo)
    sample_sales.csv          Sample order/sales data
    product_inventory.csv     Product catalogue with reorder points
    supplier_info.csv         Supplier reliability and quality scores
    transportation_costs.csv  Region-to-region cost-per-mile lookup table
```

### app.py — LogiTrackApp

The single Streamlit entry point. Owns session state (login, username), sidebar navigation, and page routing. Calls into `DataLoader` and `InventoryOptimizer` but contains no business logic of its own — it wires UI events to backend methods and renders the results. Also hosts the geo-scatter distribution map inline (duplicating some logic from `visualizations.py`) to keep the optimization result self-contained on one page.

### data_loader.py — DataLoader

Responsible for all data ingestion. Three modes: load from bundled sample CSVs, load from uploaded `st.file_uploader` objects, or connect to a relational database (PostgreSQL, MySQL, or SQLite) via a config dict. After loading, it runs `process_data()` (type coercion, date parsing, sorting) and `validate_data()` (required column checks, coordinate range validation). Exposes domain query methods: `get_pending_orders`, `get_urgent_orders`, `get_warehouse_utilization`, `get_supplier_performance`, `calculate_reorder_needs`.

### optimizer.py — InventoryOptimizer

Implements a greedy allocation algorithm. Orders are sorted by urgency (`status` ascending) then size (quantity descending). For each order, all warehouses with sufficient stock are scored by `transportation cost = distance_km * 10 + storage_cost * quantity * 0.01`. The lowest-cost warehouse is selected, its `current_stock` decremented in a working copy, and the allocation recorded. Unfulfilled orders (no warehouse with enough stock) are collected separately. Distance is Haversine (km). Returns a structured result dict with allocation plan, warehouse utilization stats, unfulfilled orders, total cost, and performance metrics.

### forecaster.py — DemandForecaster

A thin wrapper around Facebook Prophet. `fit()` prepares historical sales data into Prophet's `(ds, y)` format and trains the model with yearly and weekly seasonality enabled. `forecast()` generates a future dataframe for N periods and returns `(yhat, yhat_lower, yhat_upper)`. `evaluate_forecast()` computes MAPE, MAE, and RMSE against a held-out actual dataset. The forecaster is implemented as a backend module but not yet wired into the Streamlit UI — the page is deferred to v2.

### db_manager.py — DatabaseManager

SQLite-backed persistence layer. `_initialize_database()` creates `warehouses` and `sales` tables with `CREATE TABLE IF NOT EXISTS`, and adds an index on `sales(date)`. Provides `import_csv_data()` to bulk-load a CSV into any table via `to_sql`. Supports context manager usage (`with DatabaseManager() as db`). PostgreSQL and MySQL are accessed directly in `DataLoader` without going through this class.

### visualizations.py

Standalone Plotly figure factories: `create_distribution_map` (geo-scatter of warehouses + demand regions with allocation lines), `create_cost_comparison_chart` (bar chart of original vs. optimized cost), `create_utilization_chart` (donut chart of warehouse utilization percentages). All figures consume `VIS_SETTINGS` from `config.py` for consistent colors and map center.

---

## 3. Data Flow

```
[Browser] -- username --> [Login page] -- session_state.logged_in = True -->
    [Sidebar: select data source]
        "Sample Data"   --> DataLoader() reads data/*.csv
        "Upload Data"   --> DataLoader(uploaded_files=...) reads st.file_uploader objects
        "DB Connection" --> DataLoader(db_config=...) reads from PostgreSQL/MySQL/SQLite
    --> DataLoader.validate_data() -- raises ValueError on bad data -->
    [Sidebar: select action]
        "Overview"            --> show_overview_metrics() + show_inventory_status()
        "Inventory Mgmt"      --> show_inventory_status()
        "Order Management"    --> show_order_management() (pending / urgent / history tabs)
        "Supplier Management" --> show_supplier_info()
        "Optimization"        --> InventoryOptimizer.optimize(warehouses, pending_orders)
                                  --> allocation_plan, warehouse_utilization, unfulfilled_orders, total_cost
                                  --> show_distribution_map() renders Plotly geo-scatter with routes
        "User Guide"          --> static inline docs (tabs)
```

1. User enters a username — no password required; session state tracks login.
2. Data source is selected; `DataLoader` ingests, validates, and type-coerces all five datasets.
3. Navigation action is chosen from the sidebar selectbox.
4. For Optimization: `InventoryOptimizer.optimize()` iterates pending orders, scores each warehouse by cost, allocates greedily, returns structured results in under 1 second for typical dataset sizes.
5. Results are rendered as Streamlit metrics, dataframes, bar charts, and an interactive Plotly geo-scatter map.

---

## 4. Database Schema

SQLite tables (created by `DatabaseManager._initialize_database()`):

- `warehouses` — `warehouse_id TEXT PK`, `location TEXT`, `capacity INT`, `storage_cost REAL`, `latitude REAL`, `longitude REAL`, `created_at TIMESTAMP`, `updated_at TIMESTAMP`
- `sales` — `sale_id INT PK AUTOINCREMENT`, `date DATE`, `region TEXT`, `product_id TEXT`, `quantity INT`, `latitude REAL`, `longitude REAL`, `created_at TIMESTAMP`

CSV-only datasets (not written to SQLite by default):

- `products` — `product_id`, `product_name`, `reorder_point`, `min_order_qty`, `supplier_id`
- `suppliers` — `supplier_id`, `supplier_name`, `reliability_score`, `lead_time_reliability`, `quality_score`
- `transport` — `origin_region`, `destination_region`, `cost_per_mile`

**Indexes:** `idx_sales_date ON sales(date)` — serves the date-range filter in `get_pending_orders` and `get_order_history`.

---

## 5. API Routes

LogiTrack is a Streamlit application — there are no HTTP REST endpoints. All interactions are Streamlit widget callbacks running within the same Python process. The `DatabaseManager` exposes query methods (`get_warehouse_data`, `get_sales_data`) that serve as the internal data access layer.

---

## 6. Security

- **Authentication:** Username-only login stored in `st.session_state`. No passwords, no hashing. Appropriate for a portfolio demo; unsuitable for production without replacing with OAuth or hashed credentials.
- **Secrets:** No API keys are used — the application requires no external service credentials. Database credentials for PostgreSQL/MySQL are entered at runtime in Streamlit sidebar inputs and held in memory only for the session; they are never written to disk or logged.
- **Input validation:** `DataLoader.validate_data()` checks required columns, coordinate ranges (-90/90 lat, -180/180 lon), and type safety before any data is used downstream. Uploaded files are read via Pandas — no arbitrary code execution path.
- **Database:** SQLite file is local to the filesystem and not exposed over the network. PostgreSQL/MySQL connections use SQLAlchemy connection strings constructed from user-supplied config; all queries are `SELECT *` from fixed table names, not parameterized by user input.

---

## 7. Error Handling & Reliability

| Failure | Behaviour |
|---|---|
| Data file missing or malformed | `DataLoader` raises; Streamlit catches and shows `st.error()` with the exception message |
| Required columns absent in upload | `validate_data()` returns `False`; `DataLoader.__init__` raises `ValueError("Data validation failed")` |
| Invalid coordinates in upload | `validate_data()` returns `False`; same raise path |
| Database connection failure | Exception propagates from SQLAlchemy; Streamlit shows `st.error("Database connection failed: ...")` |
| Optimization with no pending orders | `optimizer.optimize()` receives an empty DataFrame; returns `results` with empty allocation plan and zero cost without error |
| Warehouse has insufficient stock | Order is added to `unfulfilled_orders` list; optimization continues; unfulfilled orders are surfaced in the results tab |
| Map coordinates missing | `show_distribution_map()` checks for lat/lon columns before rendering; displays a `st.warning()` explaining what's missing |

---

## 8. Deployment

LogiTrack is designed to run locally or on Streamlit Community Cloud.

1. Install dependencies: `pip install -r requirements.txt`
2. Run: `streamlit run app.py` — accessible at `http://localhost:8501`
3. For Streamlit Community Cloud: push to GitHub, connect repo, set `app.py` as the entry point. No environment variables required for sample-data mode.
4. SQLite database (`data/logitrack.db`) is auto-created on first run by `DatabaseManager._initialize_database()`.

---

## 9. Explicit Scope Cuts

- **PuLP linear programming solver** — included in `requirements.txt` but not used in production code; the greedy allocator was chosen for its speed and transparency. LP integration is deferred to v2 for cases with hard capacity constraints requiring optimality guarantees.
- **Real-time data sync / webhooks** — all data is loaded on session start; there is no push mechanism. Live warehouse stock updates would require a persistent backend process outside Streamlit's execution model.
- **Forecasting UI** — `DemandForecaster` is fully implemented as a backend module but has no Streamlit page wired to it. The page is deferred to v2.
- **Multi-user / role-based access** — single-user session model; no concept of roles, teams, or data isolation between users.
- **Optimization result persistence** — allocation runs are computed in memory and discarded when the session ends. Persisting runs to the database with a job ID is a v2 item.
