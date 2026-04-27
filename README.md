# SAM.gov Opportunities Tracker

> Part of the **[DevOps Lab](https://github.com/github-kbaker/devops_openshift-lab/tree/main)** collection —
> a hands-on DevOps / OpenShift reference project.

A full-stack web app that pulls federal contract opportunities from
[SAM.gov](https://sam.gov)'s official Opportunities API, scores them against
your keywords and NAICS codes, and gives your business-development team a
clean dashboard to triage, favorite, and export the matches.

Built from the [original CLI script](#original-script) into a production-ready
dashboard with persistence, favorites, history, and CSV export.

---

## What it does (in one paragraph)

You paste your SAM.gov API key into **Settings**, click **Fetch Now**, and the
app queries `https://api.sam.gov/opportunities/v2/search` across every NAICS ×
notice-type combination you configured, paginates through all results, scores
each opportunity (+1 per keyword match in title/description, +3 if NAICS
matches, +1 if active, capped at 10), deduplicates by `noticeId`, and saves
anything at or above your score threshold to MongoDB. The dashboard shows the
latest run; you can filter, star favorites, browse past runs, and export CSV.

---

## Tech stack

| Layer | Stack |
|---|---|
| Frontend | React 19 · React Router · Tailwind · shadcn/ui · lucide-react · sonner |
| Backend | FastAPI · Motor (async MongoDB) · `requests` |
| Data | MongoDB (collections: `settings`, `opportunities`, `fetch_runs`) |
| Typography | Chivo (headings) · IBM Plex Sans (body) · IBM Plex Mono (data) |
| Theme | Swiss / high-contrast light |

---

## Quick start (live instance)

The app is already running at:

> **https://232a7e6c-d012-4ca7-be76-414da73d2be5.preview.emergentagent.com**

1. Open the URL above.
2. Click **Settings** (top-right).
3. Paste your SAM.gov API key → **Save settings**.
4. Click **Fetch Now**. Wait 15–30 seconds.
5. Your top opportunities appear, sorted by score.

Don't have a SAM.gov key yet? See [Getting an API key](#getting-a-samgov-api-key).

---

## Use cases

### 1. Daily BD triage (primary use case)
You're a government-contractor BD rep. Every morning:

1. Open the dashboard.
2. Click **Fetch Now**. A toast tells you how many new matches were saved.
3. Scan the table, top-down (highest score first). Each row shows:
   title · agency · NAICS · posted date · response deadline · score · keyword chips.
4. Click a row to open the detail drawer with full description and the direct
   SAM.gov link.
5. **Star** opportunities your team should pursue. They're saved across future
   fetches (deduplicated by notice ID).
6. Click **Export CSV** to share the day's pipeline with colleagues.

### 2. Hunting a specific capability
Your company just launched a Kubernetes-migration practice. You want to see
every federal opportunity mentioning it.

1. Open **Settings**.
2. Clear existing keywords and add only your target terms (e.g., `Kubernetes`,
   `container`, `migration`, `modernization`).
3. Raise **Score Threshold** to 5 (ensures NAICS bonus + keyword are both present).
4. **Fetch Now**. Only high-signal results are saved.

### 3. Auditing what you may have missed
Not sure if today's fetch was complete? Click the **History** tab.

1. Every run is listed with status (success/failed), fetched count, matched count,
   duration, and any error message.
2. Click any historical run to jump back to Opportunities filtered to that run's
   snapshot — useful for re-checking what you triaged last Tuesday.

### 4. Spending-window research
Contracting budgets spike at end-of-fiscal-year (September). Widen the window:

1. **Settings → Days to look back** → set to `30` (or `60`, `90`).
2. Add seasonal keywords (e.g., `EOFY`, `obligation`).
3. **Fetch Now**. You now have a month's backlog to mine.

---

## Scoring explained

The score is deterministic, predictable, and capped at 10:

| Condition | Points |
|---|---|
| Each keyword that appears in title or description | +1 per match |
| NAICS code matches one of your configured codes | +3 |
| `active == "yes"` on the opportunity | +1 |
| **Cap** | 10 |

> ⚠️ **SAM.gov gotcha**: the `description` field on v2 results is usually a
> **URL**, not text. That means keyword matching effectively runs on titles
> only. To compensate, keep your keyword list focused on terms that actually
> appear in contract titles (e.g., `Kubernetes`, `cybersecurity`, `AI`) rather
> than verbose phrases.
>
> **Practical effect**: a typical match lands at 4–7/10. If you see 0 matches
> on a large fetch, lower your threshold to `3` or `4`.

---

## Simplifications & clarifications from the original script

The starting point was a one-shot CLI script that printed the top 10 results to
stdout. Here's what changed and why:

| Original script | This build | Why |
|---|---|---|
| Key read from `SAM_API_KEY` env var | Stored in MongoDB via Settings UI | Team can update it without redeploys; no secrets in env files |
| Hardcoded `KEYWORDS`, `NAICS_CODES`, `NOTICE_TYPES` | All editable in Settings drawer | Different practices need different filters |
| Hardcoded threshold (`>= 5`) | User-adjustable slider | Lower when results are sparse; raise when noisy |
| Fixed 3-day look-back | 1–90 day range | Supports both daily triage and quarterly research |
| Writes CSV file to disk | Streams CSV via `/api/export.csv` | Works in the browser; no server filesystem |
| Prints to terminal | Full dashboard with filtering, favorites, history | BD workflow, not a dev tool |
| No persistence | MongoDB: runs + opportunities + favorites | Track what you've already triaged |
| No deduplication | Dedupe by `noticeId` | SAM.gov returns the same notice under multiple NAICS; avoid duplicates |
| 30s request timeout | 90s per page | Broad NAICS queries are slow on SAM.gov's side |
| No error handling | Failed fetches stored in History with error message | Debuggable without server logs |

**Kept identical to the script**: the scoring formula, default keywords,
default NAICS codes (541511/541512/541513/541519), default notice types
(Solicitation + Combined Synopsis/Solicitation).

---

## Getting a SAM.gov API key

1. Go to **https://sam.gov** → Sign in via Login.gov (free account).
2. Click your username → **Account Details**.
3. Scroll to **API Key** → click **Request API Key** (or **Regenerate**).
4. Re-enter your account password when prompted.
5. Copy the key **immediately** — it's only shown once.
6. Wait **~15 minutes** for the key to activate in SAM.gov's API gateway.
7. Paste into **Settings → SAM.gov API Key** in the dashboard and save.

Rate limit: ~1,000 requests/day by default (one fetch = ~8 requests minimum,
more with pagination).

---

## Architecture

```
┌─────────────────────────┐        ┌──────────────────────────┐
│  React Dashboard        │  /api  │  FastAPI Backend         │
│  (pages/Dashboard.jsx)  │◄──────►│  (backend/server.py)     │
└─────────────────────────┘        └──────┬───────────────────┘
                                          │
                            ┌─────────────┼─────────────┐
                            ▼             ▼             ▼
                      ┌──────────┐  ┌─────────┐  ┌──────────────┐
                      │ settings │  │  opps   │  │ fetch_runs   │
                      │  (Mongo) │  │ (Mongo) │  │   (Mongo)    │
                      └──────────┘  └─────────┘  └──────────────┘
                                          ▲
                                          │ requests.get
                                          │
                                   ┌──────┴───────┐
                                   │  SAM.gov v2  │
                                   │     API      │
                                   └──────────────┘
```

### Backend routes

All routes are prefixed with `/api`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Health/greeting |
| GET | `/settings` | Read current settings |
| PUT | `/settings` | Update any subset of settings |
| GET | `/notice-types` | List SAM.gov notice-type codes + labels |
| POST | `/fetch` | Trigger a full SAM.gov fetch → score → persist |
| GET | `/opportunities` | List opportunities (filters: `run_id`, `min_score`, `naics`, `notice_type`, `favorite_only`, `search`, `latest_run_only`, `limit`) |
| GET | `/opportunities/{id}` | One opportunity |
| PATCH | `/opportunities/{id}/favorite` | Toggle favorite (propagates to all docs sharing `notice_id`) |
| GET | `/history` | List fetch runs (newest first) |
| GET | `/stats` | Dashboard stats strip |
| GET | `/export.csv` | Download CSV of current filter |

### Frontend structure

```
frontend/src/
├── App.js                          # Router + Toaster
├── lib/api.js                      # Axios client + typed helpers
├── pages/Dashboard.jsx              # Main page (state, filters, tabs)
└── components/
    ├── TopBar.jsx                  # Logo, Fetch Now, Settings, Export
    ├── FiltersSidebar.jsx          # Search, score slider, NAICS/type selects, favorites switch
    ├── OpportunityTable.jsx        # Score badges, keyword chips, row hover
    ├── OpportunityDetailSheet.jsx  # Right drawer with full metadata
    ├── SettingsSheet.jsx           # API key, chip inputs, threshold slider
    └── HistoryList.jsx             # Past fetch runs with status
```

---

## Local development

### Prerequisites
- Python 3.11+
- Node 18+ / Yarn 1.22
- MongoDB running (or use the provided `MONGO_URL`)

### Backend
```bash
cd backend
pip install -r requirements.txt
# server runs via supervisor on 0.0.0.0:8001
```

### Frontend
```bash
cd frontend
yarn install
yarn start      # http://localhost:3000
```

### Environment
- `backend/.env` → `MONGO_URL`, `DB_NAME`, `CORS_ORIGINS`
- `frontend/.env` → `REACT_APP_BACKEND_URL` (points to the `/api` backend)

---

## Testing

- **Backend**: 18 pytest integration tests, 100% passing (see
  `backend/tests/test_sam_opportunities.py` and
  `test_reports/iteration_1.json`).
- Covers: settings CRUD + clamping, fetch 400/502 paths, history ordering,
  opportunity filters, favorite 404, CSV headers, stats, and guarantees no
  MongoDB `_id` leaks into any response.

Run backend tests:
```bash
cd backend
pytest tests/ -v
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "SAM.gov API key is not configured" | Open Settings → paste key → Save |
| 401/403 from SAM.gov | Key invalid or not yet activated (wait 15 min after regeneration) |
| Fetched 5,000 · saved 0 | Threshold too high, or keywords don't appear in titles. Lower threshold to 3–4, or add title-friendly keywords |
| Fetch times out | Broad NAICS × notice-type combos = many paginated calls. Narrow NAICS list or lower `days_back` |
| Key looks right but still fails | Regenerate at sam.gov → Account Details → API Key → Regenerate. Wait 15 min |
| Favorites disappear after re-fetch | They shouldn't — favorites propagate by `noticeId`. If they do, check the opportunity actually has a `noticeId` in the response |

---

## Roadmap

**P1** — Scheduled daily auto-fetch + email digest (revenue hook for Pro tier)
**P1** — Multi-user auth with shared team watchlists
**P2** — Custom scoring-rule editor (per-keyword weights, per-NAICS weights)
**P2** — Charts: opportunities per agency / per day / score distribution
**P3** — Bulk actions, saved filter presets, attachment listing

---

## Original script

The CLI script that seeded this build is preserved conceptually in
`backend/server.py` (`fetch_sam_sync` + `score_opportunity`). Scoring rules,
defaults, and pagination behavior are identical — the dashboard simply layers
persistence, UI, and team workflows on top.

---

## Project reference

This tracker is part of the **DevOps Lab** portfolio by
[@github-kbaker](https://github.com/github-kbaker):

> 🔗 **https://github.com/github-kbaker/devops_openshift-lab/tree/main**

A companion hands-on lab covering DevOps tooling, OpenShift, and cloud-native
deployment patterns.

---

## License# sam_opportunities_trakker-README.MD
SAM.GOV Opportunities
