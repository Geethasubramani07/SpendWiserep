# SpendWise: Expense Tracker with Fraud Alerts

Imports transactions, categorises them (rules + TF-IDF/Logistic Regression), tracks monthly budgets and flags
suspicious transactions with an **Isolation Forest**. Every alert explains *why* it was flagged.
Stack: Python, FastAPI, SQLite, scikit-learn, plain HTML/JS dashboard, Docker, pytest, Selenium.

> Uses **synthetic data only**. Do not upload real bank statements.

## Run in VS Code (Windows / macOS / Linux)

1. Unzip and open the `spendwise` folder in VS Code (File > Open Folder).
2. Open the terminal (Ctrl+`) and create a virtual environment:

   ```bash
   python -m venv venv
   # Windows:  venv\Scripts\activate
   # macOS/Linux:  source venv/bin/activate
   pip install -r requirements.txt
   ```
3. In VS Code press `Ctrl+Shift+P` > **Python: Select Interpreter** > choose the `venv` one.
4. Start the server:

   ```bash
   uvicorn app.main:app --reload
   ```
5. Open <http://127.0.0.1:8000> (dashboard) and <http://127.0.0.1:8000/docs> (API docs).
6. Click **Load demo data**, or **Import CSV** with `sample_data/sample_transactions.csv`.

Python 3.10+ is required.

## Using it

| Action | How |
|---|---|
| Demo data | Dashboard > Load demo data |
| Import your own CSV | Columns: `date,description,amount` (amount > 0) |
| Set a budget | Budgets panel; at 80% = warning, over 100% = exceeded |
| Reset data | Reset button; admin token = `ADMIN_TOKEN` env var (default `admin123` for local dev only) |

## Tests and metrics

```bash
pytest                          # unit + integration tests
pytest --cov=app                # coverage (target 80%)
python scripts/evaluate.py      # categorisation accuracy + fraud recall, writes docs/RESULTS.json
# Selenium (start the server in another terminal first; needs Chrome):
pytest selenium_tests -v
```

Results measured on synthetic data in the build environment (re-run to reproduce):
categorisation accuracy **93.5%** (target 85%), fraud recall **91.1%** on injected anomalies (target 70%),
precision about 57%. Real-world results will differ.

## Docker

```bash
docker compose up --build        # http://localhost:8000
```

## CI/CD

* GitHub Actions: `.github/workflows/ci.yml` (lint, bandit, tests, evaluation, docker build)
* Jenkins: `Jenkinsfile` (webhook-triggered pipeline)

## Project layout

```
app/            main.py (API) service.py (import, budgets, reports) categoriser.py fraud.py synthetic.py db.py static/ (dashboard)
tests/          pytest unit + API tests          selenium_tests/   dashboard test (page-object)
scripts/        evaluate.py                      docs/             SRS, report + PPT templates, AI-usage log
sample_data/    sample CSV                       Dockerfile, docker-compose.yml, Jenkinsfile
```

## How the fraud detector works

Features per transaction: log amount, amount vs your typical spend in that category, night-time (00-05h),
merchant frequency, and number of transactions within 30 minutes. An Isolation Forest (`contamination=0.05`)
scores them; flagged rows get plain-language reasons (e.g. "Amount is 14.2x your typical Shopping spend").

## Known limitations

* Single-user local app: "admin" is a shared token, not full user accounts/login.
* The forest is re-fitted on all stored history on each import (fine for thousands of rows).
* Categoriser and detector are tuned on synthetic data; validate before using on real statements.
* Not production-hardened (no HTTPS, no per-user auth, no rate limiting).


## Deploy on Render (Docker)

1. Push this folder to a GitHub repository.
2. In Render, create a **New Web Service** and connect the repository.
3. Select **Docker** as the runtime. Leave the Dockerfile path as `./Dockerfile`.
4. Add the environment variable `ADMIN_TOKEN` with a long, unique secret value.
5. Deploy. Render supplies the `PORT` variable; the Dockerfile listens on it.

**Storage warning:** This project uses SQLite at `/app/spendwise.db`. Render Free
services have an ephemeral filesystem, so database changes can be lost on restart,
spin-down, or redeploy. Use synthetic/demo data for a free demonstration. Persistent
SQLite storage requires a paid Render service with a persistent disk and
`SPENDWISE_DB=/data/spendwise.db`, with the disk mounted at `/data`.
