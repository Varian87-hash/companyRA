# CompanyRA — Company Recruitment Analytics

> Track **where** top tech companies are hiring, how that changes over time, and spot emerging hiring hubs before they become obvious.

## What This Is

CompanyRA scrapes job listings from major tech companies on a weekly schedule, normalizes location data, and stores historical snapshots in a PostgreSQL database (Neon). A FastAPI backend exposes the data, and a built-in dashboard lets you explore it visually.

**Core questions it answers:**
- Which countries/cities is Amazon hiring in right now?
- Is Google expanding or shrinking its EU headcount over the past 6 months?
- Which companies post the most remote-friendly roles?

## Supported Companies

| Company | Source |
|---------|--------|
| Amazon | careers.amazon.com |
| Apple | jobs.apple.com |
| Google | careers.google.com |
| Intel | jobs.intel.com |
| Meta | metacareers.com |
| Microsoft | careers.microsoft.com |
| Nokia | careers.nokia.com |
| Nvidia | nvidia.wd5.myworkdayjobs.com |

## Stack

- **Backend:** Python, FastAPI, psycopg2
- **Database:** Neon (serverless PostgreSQL)
- **Ingestion:** custom scrapers per company, weekly cron schedule
- **Dashboard:** static HTML served at `/app`
- **Tests:** unittest

## Architecture

```
collectors/          # one scraper per company
pipeline/
  ingest_weekly.py   # orchestrator, runs all or selected companies
  common.py          # shared ingestion logic
storage/neon.py      # DB connection + writes
api/main.py          # FastAPI: /v1/companies, /v1/.../locations/current, /v1/trends/countries
api/static/          # dashboard.html
migrations/          # SQL schema + materialized views
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/companies` | List all tracked companies |
| GET | `/v1/companies/{id}/locations/current` | Latest hiring locations by country + city |
| GET | `/v1/trends/countries` | Monthly avg job counts per country (filterable by company/date range) |
| GET | `/app` | Web dashboard |
| GET | `/healthz` | Health check |

## Quick Start

**Requirements:** Python 3.11+, a Neon database URL.

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Set database URL
echo NEON_DATABASE_URL=postgresql://... > .env

# 3. Run migrations
psql $NEON_DATABASE_URL -f migrations/0001_init.sql
psql $NEON_DATABASE_URL -f migrations/0002_mv.sql
psql $NEON_DATABASE_URL -f migrations/0003_weekly_snapshots_monthly_avg.sql

# 4. Run first ingestion
python -m backend.py.pipeline.ingest_weekly --companies amazon,google

# 5. Start API
uvicorn api.main:app --host 127.0.0.1 --port 8000
# Open: http://127.0.0.1:8000/app
```

## Ingestion

Run all companies:
```bash
python -m backend.py.pipeline.ingest_weekly
```

Run selected companies:
```bash
python -m backend.py.pipeline.ingest_weekly --companies amazon,apple,nvidia
```

Schedule weekly (Linux cron):
```bash
bash scripts/register_weekly_cron.sh /path/to/companyRA "amazon,apple" "0 3 * * 0"
```

## Tests

```bash
python -m unittest tests.test_api_validation -v
```

## Deployment Notes

- Set `NEON_DATABASE_URL` as a server environment variable, not in `.env`, in production.
- `scripts/*.ps1` are Windows task scheduler helpers; use `scripts/*.sh` on Linux.
- Materialized views (`mv_*`) are refreshed automatically after each ingestion run.
