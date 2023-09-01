# Local Setup — LogiTrack

> This guide is for running LogiTrack locally or self-hosting it.
> The app works out of the box with bundled sample data — no environment variables, no external accounts, and no database setup required.

---

## Prerequisites

- Python 3.9+
- pip (comes with Python)
- A C++ compiler (required by Prophet/cmdstanpy — see note below)

> **Prophet note:** Facebook Prophet requires a C++ toolchain. On macOS this is Xcode Command Line Tools (`xcode-select --install`). On Windows, install [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/). On Linux, `gcc` and `g++` are usually pre-installed. If Prophet installation fails, the rest of the app (optimizer, visualizations, data management) still works — only the forecasting module is affected.

---

## 1. Clone and install

```bash
git clone https://github.com/tanisheesh/LogiTrack
cd LogiTrack
pip install -r requirements.txt
```

---

## 2. Environment variables

None required for sample-data mode. The app runs entirely offline with bundled CSVs.

For the **Database Connection** mode, credentials are entered directly in the Streamlit sidebar at runtime — no `.env` file needed. If you want to connect to a hosted database:

| Field | Where to get it |
|---|---|
| Host | Your PostgreSQL/MySQL server hostname or IP |
| Port | `5432` for PostgreSQL, `3306` for MySQL |
| Database name | Name of your logistics database |
| Username / Password | Your database user credentials |

> Credentials entered in the sidebar are held in `st.session_state` memory only and are never written to disk or logged.

---

## 3. Data setup

No migration step is needed. On first run, `DatabaseManager._initialize_database()` auto-creates `data/logitrack.db` (SQLite) with `warehouses` and `sales` tables.

To use your own data instead of the bundled samples, prepare five CSV files matching these schemas and upload them via the "Upload Data" sidebar option, or download the templates from within the app:

| File | Key columns |
|---|---|
| `sample_warehouses.csv` | `warehouse_id, name, capacity, current_stock, location, storage_cost, latitude, longitude` |
| `sample_sales.csv` | `order_id, date, product_id, quantity, delivery_deadline, status, delivery_latitude, delivery_longitude, region` |
| `product_inventory.csv` | `product_id, product_name, reorder_point, min_order_qty, supplier_id` |
| `supplier_info.csv` | `supplier_id, supplier_name, reliability_score, lead_time_reliability, quality_score` |
| `transportation_costs.csv` | `origin_region, destination_region, cost_per_mile` |

---

## 4. Run locally

```bash
streamlit run app.py
```

LogiTrack will be running at `http://localhost:8501`.

On first load, select **Sample Data** from the sidebar to populate the dashboard immediately with the six bundled global warehouse records (Mumbai, Singapore, Dubai, New York, London, Tokyo).

---

## 5. Deploy to production

### Streamlit Community Cloud (recommended — free)

1. Push the repo to GitHub.
2. Go to [share.streamlit.io](https://share.streamlit.io) and click **New app**.
3. Select your repo, set branch to `main`, and set the entry point to `app.py`.
4. No environment variables are required for sample-data mode.
5. Click **Deploy** — the app will be live at a `*.streamlit.app` URL within ~2 minutes.

### Local / self-hosted

Run `streamlit run app.py` behind a reverse proxy (nginx/Caddy) if you need HTTPS. Streamlit listens on port `8501` by default; configure `--server.port` to change it.

---

## Known local-only limitations

- **Prophet installation** — may fail without a C++ toolchain. The rest of the app works without it; only the `DemandForecaster` backend module is affected.
- **Database Connection mode requires a reachable host** — if connecting to a cloud PostgreSQL or MySQL instance, ensure your local machine's IP is whitelisted in the database firewall rules.
- **SQLite concurrency** — SQLite does not support concurrent writes. Running multiple Streamlit instances pointing at the same `data/logitrack.db` will work for reads but may cause lock errors on write operations. Use a dedicated instance per environment.

---

**Tanish Poddar** — [tanisheesh.in](https://tanisheesh.in) · [LinkedIn](https://linkedin.com/in/tanisheesh) · [GitHub](https://github.com/tanisheesh)
