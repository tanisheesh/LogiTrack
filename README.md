<p align="center">
  <svg xmlns="http://www.w3.org/2000/svg" width="64" height="64" viewBox="0 0 24 24"
       fill="none" stroke="#2563EB" stroke-width="1.5"
       stroke-linecap="round" stroke-linejoin="round">
    <path d="M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/>
    <polyline points="9 22 9 12 15 12 15 22"/>
    <rect x="1" y="3" width="4" height="6" rx="0.5"/>
    <rect x="19" y="3" width="4" height="6" rx="0.5"/>
    <line x1="12" y1="2" x2="12" y2="5"/>
    <circle cx="7" cy="16" r="1.5"/>
    <circle cx="17" cy="16" r="1.5"/>
  </svg>
</p>

<h1 align="center">LogiTrack</h1>

<p align="center">
  <strong>Multi-warehouse inventory management and supply chain optimization dashboard built with Python and Streamlit.</strong>
</p>

<p align="center">
  <a href="https://logitrack-tanishpoddar.streamlit.app">
    <img src="https://img.shields.io/badge/live_demo-2563EB-2563EB?style=flat-square" alt="Live Demo">
  </a>
  <img src="https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat-square&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/Streamlit-1.28%2B-FF4B4B?style=flat-square&logo=streamlit&logoColor=white" alt="Streamlit">
  <img src="https://img.shields.io/badge/Plotly-5.18%2B-3F4F75?style=flat-square&logo=plotly&logoColor=white" alt="Plotly">
  <img src="https://img.shields.io/badge/Prophet-ML_Forecasting-0066CC?style=flat-square" alt="Prophet">
  <img src="https://img.shields.io/badge/SQLAlchemy-2.0%2B-D71F00?style=flat-square" alt="SQLAlchemy">
  <img src="https://img.shields.io/badge/license-GPL--3.0-2563EB?style=flat-square" alt="License">
</p>

---

## What is LogiTrack?

Logistics teams managing inventory across multiple global warehouses spend hours manually deciding where to ship from, which orders to prioritize, and how to keep storage costs in check. LogiTrack eliminates that overhead with a browser-based dashboard that ingests warehouse, order, supplier, and transport data — then runs a cost-distance optimization to generate an allocation plan in seconds. It is built for supply chain analysts and logistics managers who need answers on the fly, not data scientists who need to write SQL.

> **Live demo →** [logitrack-tanishpoddar.streamlit.app](https://logitrack-tanishpoddar.streamlit.app)

---

## What you get

- **Inventory distribution optimizer** — greedy allocation engine using the Haversine formula to compute real-world distances, then minimizes transportation and storage cost across all pending orders
- **Global warehouse map** — interactive Plotly geo-scatter that draws live allocation routes between warehouses and delivery points
- **Demand forecasting** — Prophet-backed ML model with yearly and weekly seasonality; evaluated via MAPE, MAE, and RMSE
- **Supplier scorecards** — aggregated reliability, lead-time, and quality metrics per supplier, surfaced as ranked bar charts
- **Flexible data ingestion** — load bundled sample CSVs, upload your own files, or connect directly to PostgreSQL, MySQL, or SQLite

---

## Stack

| Layer | Tech |
|---|---|
| UI Framework | Streamlit 1.28+ |
| Language | Python 3.9+ |
| Data processing | Pandas 2.1 · NumPy 1.26 |
| Optimization | Custom greedy allocator (Haversine distance + storage cost weighting) |
| Forecasting | Prophet 1.1 · scikit-learn 1.3 |
| Visualization | Plotly 5.18 (geo-scatter, bar, pie) · Folium 0.14 |
| Database | SQLite (default, auto-created) · PostgreSQL · MySQL via SQLAlchemy 2.0 |

---

## Engineering Decisions

**Why a greedy allocator instead of a linear program (PuLP is in requirements.txt)?**
The greedy approach — sort orders by urgency then size, assign each to the nearest warehouse with stock — solves in milliseconds for the typical warehouse counts in this domain. PuLP was evaluated but added solver-setup complexity and ~10s solve time without meaningful accuracy gains at fewer than 50 warehouses. The greedy algorithm is fully transparent: every allocation traces directly to a distance and storage cost.

**Why Streamlit instead of a React + FastAPI split?**
The entire tool is data-exploration-first — operators run optimizations, read charts, and download plans. Streamlit colocates computation and rendering in one Python file, cutting the build surface significantly. A REST API would have added value only if external systems needed to call the optimizer programmatically, which is a v2 concern.

**Why Prophet for demand forecasting instead of ARIMA or a simple moving average?**
Prophet handles missing data and irregular seasonality out of the box, and it produces interpretable yearly and weekly components that an operator can sanity-check. ARIMA requires manual order selection (p,d,q) and is brittle on sparse series; a moving average has no seasonality awareness.

**What would you do differently in v2?**
The optimizer is stateless and recomputes from scratch on every button press. In v2, allocation runs would be persisted to the database with a job ID so results can be audited, compared across runs, and retrieved without re-running the solver. Authentication is also just a username input today — real access control with hashed credentials or OAuth would be required before any production deployment.

---

## Docs

| Document | Description |
|---|---|
| [PRD](docs/PRD.md) | Product requirements — goals, user stories, non-goals |
| [Architecture](docs/ARCHITECTURE.md) | System design, data flow, component breakdown |
| [Decisions](docs/DECISIONS.md) | Every major technical decision and why |
| [Setup](docs/SETUP.md) | Local dev setup, database config, deployment |

---

## Author

**Tanish Poddar** — [tanisheesh.in](https://tanisheesh.in) · [LinkedIn](https://linkedin.com/in/tanisheesh) · [GitHub](https://github.com/tanisheesh)
