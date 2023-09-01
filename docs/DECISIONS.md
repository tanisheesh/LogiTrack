# Engineering Decisions — LogiTrack

<!-- PURPOSE OF THIS FILE
     This is not user documentation. This is for technical interviewers
     and senior engineers who want to understand WHY the system is built
     the way it is. Every entry should answer a question an interviewer
     might ask. If you can't explain the tradeoff, the decision doesn't
     belong here yet.
-->

---

## Decision 1 — Greedy allocator over PuLP linear program

**Context:** The core feature is assigning pending orders to warehouses in a way that minimizes transportation and storage cost. Two approaches were evaluated: a greedy heuristic (sort by priority, assign each order to the cheapest viable warehouse) and a proper LP solver using PuLP (which is still in `requirements.txt`).

**Decision:** Greedy heuristic. Orders are sorted by urgency then size; each order is assigned to the warehouse with the lowest score of `distance_km × 10 + storage_cost × quantity × 0.01`.

**Reason:** The greedy approach solves in milliseconds for any realistic warehouse count (the sample data has 6 warehouses and ~20 orders; even 50 warehouses and 200 orders completes in under 1 second). PuLP was evaluated and added ~10s solve time due to solver initialization overhead, without producing meaningfully different allocations at this scale. More importantly, the greedy algorithm is fully auditable — every allocation traces directly to a distance and a cost, with no solver black-box involved.

**Tradeoff:** The greedy algorithm does not guarantee global optimality. For datasets where the early allocation of a large order to a cheap warehouse leaves a later urgent order with only an expensive warehouse, the total cost will be higher than the LP optimum. This is an acceptable tradeoff at portfolio demo scale. An LP mode is deferred to v2.

---

## Decision 2 — Streamlit over React + FastAPI

**Context:** The project needed a user interface for data upload, optimization triggering, and chart rendering. Two architectures were considered: a Streamlit single-process Python app, or a React frontend with a FastAPI backend.

**Decision:** Streamlit.

**Reason:** LogiTrack is a data-exploration tool — the primary workflow is: load data, click a button, read a chart. Streamlit colocates computation and rendering in one Python file, eliminating the need for API contracts, serialization, CORS configuration, and a separate deployment pipeline. The entire codebase is Python, which keeps the tooling surface small (one `requirements.txt`, one `streamlit run` command).

**Tradeoff:** Streamlit's execution model (full script re-run on every widget interaction) means the optimizer runs synchronously on the main thread. For a solver that takes more than a few seconds, this would block the UI. This is not a problem at current scale but would require a background job queue if the LP solver or very large datasets were added in v2. Streamlit also makes it harder to build highly custom UIs — the component library is intentionally limited.

---

## Decision 3 — Prophet over ARIMA or a simple moving average

**Context:** The forecasting module needed to project future demand from historical sales data. Three options were evaluated: Facebook Prophet, ARIMA, and a rolling-window moving average.

**Decision:** Facebook Prophet with yearly and weekly seasonality, multiplicative mode.

**Reason:** Prophet handles missing data and irregular seasonality without manual configuration. It produces interpretable components (trend, yearly, weekly) that an operator can sanity-check visually. ARIMA requires manual selection of (p, d, q) parameters and is brittle on sparse or irregular series. A simple moving average has no seasonality awareness and degrades significantly on business data with weekly/annual cycles.

**Tradeoff:** Prophet is a heavy dependency — it pulls in pystan/cmdstanpy, which requires a C++ compiler and can fail to install on some platforms. Installation is the primary friction point for new users. This is mitigated by marking forecasting as optional and ensuring the main app works without it. Prophet is also slower to train than a moving average — on very long series (years of daily data), fitting can take 10–30 seconds.

---

## Decision 4 — SQLite as the default database with no required setup

**Context:** The app needed some persistence layer for the database connection mode and for the `DatabaseManager` module. Options ranged from no database (CSV-only), to requiring a hosted PostgreSQL instance, to shipping SQLite as a default.

**Decision:** SQLite auto-created at `data/logitrack.db` on first run. PostgreSQL and MySQL are supported as optional connections entered at runtime.

**Reason:** SQLite requires zero infrastructure — no Docker, no cloud account, no connection string in an env var. The app can be cloned and run with a single `pip install` + `streamlit run`. For a portfolio tool where the first-run experience is critical, requiring a separate database setup would add friction and failure modes. SQLite is sufficient for the read-heavy, single-user workload this tool targets.

**Tradeoff:** SQLite does not support concurrent writes, which is irrelevant for a single-analyst tool but rules it out for any production multi-user deployment. The database schema (`warehouses` and `sales` tables) is also a subset of the full data model — products, suppliers, and transport costs are CSV-only and not persisted to SQLite in v1, which means they are re-loaded from file on every session.

---

## What I'd do differently in v2

- **Persist optimization runs** — the optimizer currently recomputes from scratch every time and results are lost when the session ends. I'd add a `runs` table to SQLite with columns for job ID, timestamp, parameters, and the full result JSON, so operators can compare runs and pull historical allocations without re-solving.
- **Wire the forecasting page** — `DemandForecaster` is fully implemented and tested as a backend module but has no Streamlit UI page. Building the page was a deliberate v1 cut; it would add meaningful value and is the next item on the backlog.
- **Replace username-only login** — the current "enter your name" login is suitable only for a portfolio demo. I'd replace it with hashed credentials (bcrypt) stored in SQLite, or GitHub OAuth, before exposing the app to any real users.
- **Add configurable cost weights** — the transport cost formula (`distance_km × 10 + storage_cost × quantity × 0.01`) has hardcoded coefficients. Exposing these as sidebar sliders would let operators tune the optimizer to their actual carrier rates without touching code.

---

## Explicit non-decisions (deferred to v2)

| Feature | Why deferred |
|---|---|
| PuLP LP solver in UI | Greedy is faster and more transparent at this scale; LP adds solver-setup complexity and 10s overhead without meaningful accuracy gains at fewer than 50 warehouses |
| Real-time data sync | Streamlit's execution model requires a full script re-run on interaction; pushing live updates would require a persistent background process outside this model |
| Forecasting dashboard page | Backend fully implemented; UI page was cut to keep v1 scope focused on the optimizer workflow |
| Multi-user / role-based access | Single-analyst use case; multi-tenancy would require per-user data isolation, auth middleware, and billing — none of which are relevant at portfolio scale |
| Optimization result persistence | In-memory compute is sufficient for demo purposes; persistence adds schema complexity without changing the core demo experience |

---

**Tanish Poddar** — [tanisheesh.in](https://tanisheesh.in) · [LinkedIn](https://linkedin.com/in/tanisheesh) · [GitHub](https://github.com/tanisheesh)
