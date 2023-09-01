# LogiTrack — Product Requirements Document

**Status:** Final
**Owner:** Tanish Poddar
**One-liner:** Multi-warehouse inventory management and supply chain optimization dashboard built with Python and Streamlit.

---

## 1. Problem

Logistics teams managing inventory across multiple global warehouses spend hours manually deciding where to ship from, which orders to prioritize, and how to keep storage costs in check. There is no lightweight, self-contained tool that accepts raw CSV or database data and immediately produces an optimal allocation plan without requiring a data science background. Existing solutions are either heavyweight ERP systems (expensive, months to implement) or raw spreadsheets (no optimization, error-prone at scale). LogiTrack fills the gap: a single-command Python app that goes from raw data to an optimized allocation plan and global map in under 10 seconds.

---

## 2. Goals (v1 / MVP)

1. Accept warehouse, order, product, supplier, and transport data from three sources: bundled sample CSVs, user-uploaded CSVs, or a live PostgreSQL/MySQL/SQLite connection.
2. Run a greedy inventory allocation optimizer that minimizes transportation cost (Haversine distance × rate) plus storage cost, completing in under 1 second for typical datasets.
3. Render an interactive Plotly geo-scatter map showing allocation routes between warehouses and delivery points after each optimization run.
4. Surface supplier performance scorecards (reliability, lead-time, quality) as ranked bar charts.
5. Provide a demand forecasting backend (Prophet) with MAPE/MAE/RMSE evaluation, callable as a module.
6. Validate all ingested data (required columns, coordinate ranges, type coercion) and surface clear errors before any computation begins.
7. Deploy on Streamlit Community Cloud with no required environment variables for sample-data mode.

---

## 3. Non-Goals (explicit scope cuts)

- **PuLP linear programming solver UI** — PuLP is in `requirements.txt` and was evaluated; it is deferred because the greedy allocator is faster and more transparent for typical warehouse counts. An LP-based optimizer is a v2 item for hard-constraint scenarios.
- **Real-time data sync / webhooks** — data is loaded once per session. Live warehouse updates require a persistent backend process that does not fit Streamlit's execution model. Deferred to v2.
- **Forecasting dashboard page** — `DemandForecaster` is fully implemented as a Python module but has no UI page wired to it. Deferred to v2 to keep v1 scope manageable.
- **Multi-user / role-based access** — the login is username-only with no authentication. Multi-tenancy and per-user data isolation are out of scope for a single-analyst portfolio tool.
- **Optimization result persistence** — allocation runs are computed in memory and lost on session end. Persisting runs with a job ID for audit and comparison is a v2 item.
- **Mobile layout** — Streamlit's layout is not optimized for small screens; the primary use case is a desktop analyst workflow.

---

## 4. Users

**Primary:** Supply chain analysts and logistics managers who need to generate an allocation plan from existing warehouse and order data without writing code or queries.

**Secondary:** Recruiters and technical interviewers evaluating this as a portfolio piece — the app must work end-to-end with bundled sample data on first load, with no setup required.

---

## 5. User Stories

1. *As a logistics manager,* I want to load my warehouse and order data from a CSV so that I can see the current inventory status without any database setup.
2. *As a supply chain analyst,* I want to run an inventory optimization so that I get an allocation plan that minimizes transportation and storage cost across all pending orders.
3. *As a logistics manager,* I want to see a global map with allocation routes drawn so that I can visually verify the optimizer's decisions before executing them.
4. *As a procurement lead,* I want to view supplier reliability, lead-time, and quality scores in a ranked chart so that I can identify which suppliers are underperforming.
5. *As an analyst,* I want to connect directly to our PostgreSQL database so that I can run optimizations against live data without exporting CSVs.
6. *As a logistics manager,* I want to see which orders could not be fulfilled and why so that I can take manual action on stockout situations.
7. *As an analyst,* I want to see products that have dropped below their reorder point so that I can trigger replenishment before stock runs out.

---

## 6. Functional Requirements

### 6.1 Data Ingestion

- The system must support three data source modes: sample CSVs (bundled), file upload (five separate CSV files), and database connection (PostgreSQL, MySQL, SQLite).
- After loading, the system must validate required columns for all five datasets (warehouses, sales, products, suppliers, transport) and raise a clear error if validation fails.
- Coordinate columns (latitude, longitude, delivery_latitude, delivery_longitude) must be validated to be within [-90, 90] and [-180, 180] ranges.
- All date columns must be parsed to `datetime` objects and numeric columns type-coerced before any downstream use.

### 6.2 Inventory Optimization

- The optimizer must accept a warehouse DataFrame and a pending-orders DataFrame and return a structured result dict containing: allocation plan, warehouse utilization stats, unfulfilled orders list, total cost, solve time, and fulfillment rate.
- Orders must be sorted by urgency (status ascending) then size (quantity descending) before allocation.
- Transportation cost per allocation must be calculated as `distance_km × 10 + storage_cost × quantity × 0.01`, where distance is Haversine (km).
- Warehouses with insufficient stock for an order must be skipped; orders with no viable warehouse must be recorded in `unfulfilled_orders`.
- Optimization must complete in under 1 second for datasets with fewer than 50 warehouses and 200 orders.

### 6.3 Visualization

- After optimization, the system must render a Plotly geo-scatter map displaying warehouse locations (square markers), delivery locations (circle markers), and allocation routes (line traces) on an equirectangular world projection.
- The map must be skipped gracefully (with an explanatory `st.warning`) if latitude/longitude columns are absent.
- Warehouse utilization must be rendered as a bar chart accessible from both the Overview and Optimization result tabs.
- Supplier performance metrics (reliability, lead-time, quality) must be rendered as a bar chart indexed by supplier name.

### 6.4 Dashboard Navigation

- The sidebar must expose six navigation options: Overview, Inventory Management, Order Management, Supplier Management, Optimization, User Guide.
- The Overview page must display four KPI metrics: total inventory, pending orders, urgent orders, products to reorder.
- The Order Management page must expose three tabs: Pending Orders, Urgent Orders (deadline within 2 days), Order History (last 7 days).

---

## 7. Non-Functional Requirements

- **Latency:** Optimization must complete in under 1 second for typical datasets; the full page must render in under 3 seconds on a standard laptop.
- **Security:** No API keys or secrets required; database credentials entered at runtime are held in memory only and never logged or written to disk.
- **Cost:** Zero ongoing infrastructure cost — runs entirely on local Python or free Streamlit Community Cloud tier.
- **Reliability:** Data validation failures must produce a visible error via `st.error()` and halt further processing cleanly; the app must never crash silently.
- **Portability:** Must run on Python 3.9+ on macOS, Linux, and Windows with a single `pip install -r requirements.txt`.

---

## 8. Success Metrics

| Metric | Target |
|---|---|
| Time from `streamlit run app.py` to populated dashboard | Under 10 seconds with sample data |
| Optimization solve time | Under 1 second for standard sample dataset (6 warehouses, ~20 orders) |
| Data validation coverage | All five required datasets validated before any computation |
| Deployment | Live and accessible via Streamlit Community Cloud with no env vars required |

---

## 9. Risks & Open Questions

- **Prophet installation complexity** — Prophet has heavy transitive dependencies (pystan, cmdstanpy). Installation can fail on some platforms without a C++ compiler. Mitigated by marking it as optional in comments and ensuring the app works without the forecasting page.
- **Database credential security** — PostgreSQL/MySQL credentials are entered into Streamlit sidebar text inputs. These are not encrypted in transit (unless the app is served over HTTPS). Mitigated by noting this is a portfolio demo; production deployment would require HTTPS and a secrets manager.
- **Large dataset performance** — the greedy allocator is O(orders × warehouses). For very large datasets (1,000+ orders, 100+ warehouses), solve time could exceed 1 second. Mitigated by the scope cut: this tool targets analyst-scale datasets, not ERP-scale.
- **Open question:** Should the optimizer expose a configurable cost weight slider (transport vs. storage) rather than the hardcoded `distance * 10` formula? Deferred to v2 — the current formula is sufficient for demonstrating the concept.

---

## 10. v2 Candidates

- **Optimization result persistence** — save each run to SQLite with a job ID, timestamp, and parameters so results can be compared across runs and retrieved without re-solving.
- **Forecasting dashboard page** — wire `DemandForecaster` into a Streamlit page with period selector, component plots, and accuracy metrics displayed inline.
- **PuLP LP solver mode** — offer an optional LP-based optimizer for users who need optimality guarantees under hard capacity constraints.
- **Real authentication** — replace the username-only login with hashed credentials or OAuth so the app can be safely deployed in a shared environment.
- **Configurable cost weights** — expose transport cost rate and storage cost multiplier as sidebar sliders so operators can tune the optimizer to their actual pricing.

---

**Tanish Poddar** — [tanisheesh.in](https://tanisheesh.in) · [LinkedIn](https://linkedin.com/in/tanisheesh) · [GitHub](https://github.com/tanisheesh)
