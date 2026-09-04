# AutoInsight — Automated Sales Reporting Pipeline

An end-to-end automation project: **n8n** orchestrates a scheduled pipeline that
cleans ecommerce sales data, detects anomalies (z-score based spike/dip
detection), asks **Claude** to write a plain-English executive summary, and
either posts it to Slack/Email or serves it to a **React dashboard** styled
like a printed receipt roll.

Built to demonstrate: SQL/Python data analysis, workflow automation (the part
most portfolio projects skip), and applied LLM use — three signals in one
project instead of another static dashboard.

```
autoinsight/
├── data/
│   ├── generate_data.py      # synthetic 120-day ecommerce dataset (4 regions x 4 categories)
│   └── ecommerce_sales.csv   # generated output (7,680 rows, 2 injected anomalies)
├── backend/
│   ├── analysis.py           # aggregation + z-score anomaly detection -> summary.json
│   ├── narrative.py          # Claude API call -> plain-English report (has offline fallback)
│   ├── app.py                # FastAPI: /api/summary /api/narrative /api/refresh
│   ├── requirements.txt
│   └── output/                # summary.json + narrative.json land here
├── n8n/
│   └── autoinsight_workflow.json   # import directly into n8n
└── dashboard/
    ├── src/App.jsx, components/Receipt.jsx, components/TrendChart.jsx
    └── src/data/fallbackData.js   # bundled snapshot so it runs with zero backend
```

## How the pieces connect

```
  n8n (weekly trigger)
        │
        ▼
  POST /api/refresh  ──────►  analysis.py (pandas-free, pure Python: aggregation + anomalies)
        │                            │
        │                            ▼
        │                     narrative.py ──► Claude API ──► plain-English report
        ▼
  Slack / Email message          summary.json + narrative.json
                                        │
                                        ▼
                               React dashboard (/api/summary, /api/narrative)
```

n8n is the trigger + delivery layer. The actual analysis logic lives in
plain, testable Python (`analysis.py`) rather than buried inside n8n's Code
node editor — so you can demo it, unit test it, or reuse it without n8n at
all if an interviewer asks "show me the logic."

## Run it locally

**1. Generate data + run the pipeline once**
```bash
cd autoinsight/data && python3 generate_data.py
cd ../backend && pip install -r requirements.txt
python3 analysis.py
python3 narrative.py          # set ANTHROPIC_API_KEY env var for a real Claude narrative
                               # (falls back to a rule-based summary if unset)
```

**2. Start the API**
```bash
uvicorn app:app --reload --port 8000
```

**3. Run the dashboard**
```bash
cd ../dashboard
npm install
npm run dev
```
Opens at `http://localhost:5173`. Without any config it renders the bundled
sample report (`src/data/fallbackData.js`) — good enough to deploy standalone
to Vercel with zero backend. To go live against your FastAPI backend, create
a `.env` file:
```
VITE_API_BASE=http://localhost:8000
```

**4. Import the n8n workflow**
Open n8n → Workflows → Import from File → `n8n/autoinsight_workflow.json`.
It's wired as: Schedule Trigger → HTTP Request (`POST /api/refresh`) → Code
node (shapes the payload) → Slack node + Gmail node (pick one, or wire both).
Add your Slack/Gmail credentials and swap `localhost:8000` for your deployed
backend URL.

## Deploying (matches your usual stack)

- **Backend** → Render (Python web service, `uvicorn app:app --host 0.0.0.0 --port $PORT`)
- **Dashboard** → Vercel (`VITE_API_BASE` set to your Render URL as an env var)
- **n8n** → n8n Cloud (free tier) or self-hosted; just point its HTTP node at
  the Render backend URL instead of localhost

## Why this is a stronger portfolio piece than a static dashboard

- Shows the full loop: raw data → cleaning/aggregation → statistical anomaly
  detection → LLM narrative → automated delivery — not just a chart.
- The anomaly detection is real, not decorative: `generate_data.py` injects
  a demand spike and a regional dip, and `analysis.py`'s z-score method
  catches both (see `backend/output/summary.json` after running it).
- n8n signals you can automate a recurring analyst task, which is a common
  pain point interviewers ask about directly ("how would you avoid doing
  this report by hand every week?").

## Talking points for interviews

- **Why z-score for anomalies, not just % change?** % change alone flags
  naturally noisy low-volume categories constantly. Z-score normalizes
  against each region/category's own baseline variance, so a spike is only
  flagged when it's genuinely unusual for *that* series.
- **Why keep the narrative logic outside n8n?** Testability — you can run
  `analysis.py` and `narrative.py` standalone, unit test them, or swap n8n
  for Airflow/cron later without rewriting the analysis.
- **What would you change for real production data?** Swap the CSV read for
  a warehouse query (BigQuery/Snowflake), add a small validation layer before
  aggregation, and log anomaly history so recurring flags can be suppressed.
  
