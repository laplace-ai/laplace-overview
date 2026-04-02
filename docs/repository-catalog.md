# Repository Catalog — laplace-ai GitHub Organization

Last updated: 2026-04-02

## Active Repositories (in product)

These repos are cloned locally in `laplace-digital-twin/` and are part of the current product.

| Repo | Description |
|------|-------------|
| **laplace-overview** | Central repo for cross-cutting issues, architecture decisions, and platform-wide documentation |
| **laplace-platform** | Frontend web application (Next.js, deployed on Vercel at app.laplacelog.com) |
| **laplace-platform-api** | REST API serving data from Cloud SQL to the frontend (FastAPI, Cloud Run) |
| **laplace-data-warehouse** | ETL service — consumes GCS CSV paths from Data Lake, transforms, loads into Cloud SQL (Cloud Run) |
| **laplace-data-lake** | Multi-tenant ingestion — syncs CSV files from Google Drive to GCS, publishes Pub/Sub events (Cloud Run) |
| **laplace-service-loss-prediction** | Loss prediction service — LightGBM classifier for CTRC cargo loss probability (Cloud Run) |
| **digital-model** | Domain/simulation library — entities, events, model aggregate for the digital twin |
| **laplace-docs** | Product documentation platform (deployed at docs.laplacelog.com) |
| **laplace-handbook** | Internal handbook (deployed at handbook.laplacelog.com) |
| **laplace-landing-page** | Product landing page (laplacelog.com) |
| **laplace-health-monitor** | Health monitoring wrapper for Laplace infrastructure |
| **laplace-toolkit** | Shared utilities and tooling |
| **kill-gcp** | Cloud Function triggered by Pub/Sub when billing exceeds budget — safety kill switch |
| **dummy-platform** | Test/staging frontend for development |

---

## Legacy Repositories

### Transfer & Consolidation Analysis (relevant to base 200)

| Repo | Description | Relevance to Base 200 |
|------|-------------|----------------------|
| **transfer-optimization-eda** | Exploratory data analysis for transfer optimization. Python pipeline that ingests raw base 200 CSVs, cleans them, filters by vehicle type, creates route columns, generates `ID_VIAGEM` (trip ID), calculates vehicle occupancy. Has notebooks for savings analysis, fleet analysis, route maps, and per-trip analysis across 21 branches. | **HIGH** — directly processes base 200 manifests. Contains ETL logic (`ingest_transform_save_200.py`, `filter_200.py`, `group_by_trip.py`) and a YAML config defining 200 columns. Key source for reusable trip grouping logic. |
| **calculate-savings-second-lever** | Transfer simulation using bases 455 + 930. Reconstructs CTRC transfer chains from occurrence codes (95=departure, 94=arrival, 93=destination arrival, 01/1/11=delivery). Full simulation engine with `CTRCState`, `SimulationState`, `Truck` classes. Day-by-day simulation packing trucks prioritized by slack days ("gordurinha"). | **HIGH** — does not use base 200 directly, but simulates the consolidation problem. Contains transfer chain reconstruction logic reusable for the digital model. |

### Early Prototypes (UI mockups, no backend)

| Repo | Description |
|------|-------------|
| **digital_twin_mvp** | Early network simulation prototype (~1500-line Python). Simulates CTRCs moving through branches with vehicles. Predecessor to the current `digital-model` repo. Uses Supabase backend for a Lovable UI. |
| **consolidador-laplace** | Lovable-generated React UI for a consolidation tool. Pages for trip sequencing, prioritized lists, processing, and AI agent. No backend logic. |
| **roteirizador-laplace** | Lovable-generated React UI for a routing tool. Pages for planning, routing, branches. No data processing. |
| **atlas-route-optimizer** | Lovable-generated React UI for route optimization. Pages for dashboard, map, simulation, recommendations. No backend logic. |
| **logiview-twin** | Lovable-generated React UI for a digital twin viewer. Pages for map, database, recommendations. No data processing. |
| **agent-aware** | Lovable-generated React UI for an AI analyst agent interface. Components for analysts, chat, reports. Early product concept. |
| **laplaceai** | Early/abandoned Lovable-generated platform UI prototype, before `laplace-platform` became the main frontend. |
| **laplaceai-landing-page** | Company marketing landing page (Lovable). Logos for CIETEC, USP, Centrale Lille. Separate from the product landing page. |

### Legacy ML Pipeline (superseded by current architecture)

These repos were the original loss prediction pipeline before it was rebuilt as `laplace-service-loss-prediction` + `laplace-platform-api`.

| Repo | Description |
|------|-------------|
| **ativa-model-pipeline-extravio** | ML training repo. Full pipeline: CSV ingestion, DuckDB transformations, LightGBM training, Optuna hyperparameter optimization, AutoML experiments. Multiple YAML configs for temporal studies. |
| **ativa-inference** | Local inference pipeline. Downloads base 455/930, builds features in DuckDB, runs LightGBM model. Development version. |
| **ativa-inference-automation-gcp** | Production automation version of ativa-inference. Runs on a GCP VM, orchestrated by bash scripts, uses rclone for data download. |
| **ativa-api** | Standalone FastAPI prediction API. OAuth2 auth, LightGBM model serving. Predecessor to prediction endpoints in `laplace-platform-api`. |

### Empty / Placeholder Repos (never used)

| Repo | Description |
|------|-------------|
| **ativa-plotting** | Intended for visualization utilities. Empty — no commits. |
| **state-logger** | Intended for state change logging. Only contains .gitignore and placeholder README. |
| **laplace-backlog** | Intended for cross-cutting backlog. Empty — superseded by GitHub Issues on individual repos. |
| **EDA-CargOn** | Intended for EDA on CargOn database (different client/prospect). Empty — never pursued. |
| **disc-ocr-reader** | Intended for OCR/PDF reader experiments. Only placeholder README. |
| **demo-repository** | GitHub's default demo repo template. Auto-generated when org was created. |
