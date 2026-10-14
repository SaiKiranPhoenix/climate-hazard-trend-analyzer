Good. One more thing before I write this — since you're jumping across IDEs, the prompt needs to be **self-contained context**, not IDE-specific syntax. And since it's fully autonomous, the agent needs enough domain knowledge baked in that it doesn't make naive climate-science or architecture decisions on its own.

Let me also check your project context before writing:Good — your PulseOps architecture tells me a lot about how you think and build. The prompt I write will be at that level of structure, not beginner scaffolding.

Here's the real one:

---

## The Project Intelligence Prompt

This is a **living context document**. Paste it at the start of every new chat/session in whichever IDE or agent you're in. It replaces onboarding entirely.

---

```markdown
# PROJECT INTELLIGENCE — Climate Hazard Trend Analyzer
# Treat this as your architectural brain for the entire project.
# You are a senior full-stack engineer. Do not explain basics. Do not add things I didn't ask for.
# Every decision you make must trace back to something stated here.

---

## WHAT WE'RE BUILDING
A production-quality web app that:
- Pulls historical daily weather data from Open-Meteo (free, no auth, reliable)
- Detects heatwave events using a statistically rigorous definition
- Analyzes and visualizes trends over multi-decade time ranges
- Serves everything through a clean FastAPI backend with MongoDB caching
- Has a React frontend that is interactive, not just pretty

This is evaluated on: data integrity, climate logic correctness, API design quality,
frontend clarity, security posture, performance, and critical thinking in insights.
Not on lines of code. Not on how many features exist.

---

## HARD STACK DECISIONS — NO DEVIATIONS

### Backend
- Language: Python 3.11+
- Framework: FastAPI (async throughout — no sync route handlers)
- DB Driver: Motor (async MongoDB) — never use pymongo blocking calls in async context
- DB: MongoDB Atlas M0 (free tier)
- HTTP Client: httpx with AsyncClient — for Open-Meteo calls
- Data Processing: pandas + numpy — all hazard detection and trend math happens here
- Stats: scipy.stats.linregress for linear regression, pymannkendall for Mann-Kendall
- Validation: Pydantic v2 models for all request params and all response shapes
- Rate Limiting: slowapi (Starlette-compatible limiter)
- Config: pydantic-settings — all config from environment, zero hardcoded values
- Env: python-dotenv for local dev

### Frontend
- Framework: React 18 + Vite + TypeScript (strict mode)
- Charts: Chart.js via react-chartjs-2 — register only the components you use
- Maps: Leaflet via react-leaflet — for region selection (this is a bonus but build it early)
- HTTP: Axios with a typed API client module
- Server State: TanStack Query (React Query v5) — no manual loading/error state
- Styling: Tailwind CSS — utility-first, no component libraries (no shadcn, no MUI)
- Export: jsPDF + Papa Parse — for PDF and CSV export respectively
- Types: All API response shapes typed — no `any` unless a TODO comment explains why

### Deployment
- Frontend: Vercel
- Backend: Render (Python web service)
- DB: MongoDB Atlas
- Repo: Public GitHub — monorepo with /frontend and /backend at root

---

## DIRECTORY STRUCTURE — EXACT, NON-NEGOTIABLE

```
climate-hazard-analyzer/
├── .gitignore
├── README.md
│
├── backend/
│   ├── app/
│   │   ├── main.py                   # App factory, CORS, rate limiter, lifespan
│   │   ├── config.py                 # pydantic-settings: all env vars typed
│   │   ├── routers/
│   │   │   └── hazard.py             # All /api/v1/hazard/* routes
│   │   ├── services/
│   │   │   ├── weather_fetcher.py    # Open-Meteo async fetch + parse + gap handling
│   │   │   ├── heatwave_detector.py  # Canonical hazard logic (see below)
│   │   │   └── trend_analyzer.py     # Regression, rolling avg, Mann-Kendall, insight gen
│   │   ├── repositories/
│   │   │   └── hazard_cache.py       # All MongoDB reads/writes — nothing else touches Motor
│   │   ├── models/
│   │   │   ├── request.py            # Pydantic request param models
│   │   │   └── response.py           # Pydantic response models
│   │   └── utils/
│   │       └── date_utils.py         # Date chunking, ISO parsing helpers
│   ├── tests/
│   │   ├── test_heatwave_detector.py # Unit tests: p95 logic, edge cases
│   │   └── test_trend_analyzer.py
│   ├── .env.example
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   │   └── hazardClient.ts       # Axios instance + typed request functions
│   │   ├── components/
│   │   │   ├── RegionSelector/
│   │   │   │   ├── index.tsx         # City search (geocode) + lat/lon fallback
│   │   │   │   └── MapPicker.tsx     # Leaflet map with draggable marker
│   │   │   ├── HazardChart/
│   │   │   │   ├── FrequencyChart.tsx
│   │   │   │   ├── DurationChart.tsx
│   │   │   │   └── IntensityChart.tsx
│   │   │   ├── InsightPanel/
│   │   │   │   └── index.tsx         # Trend summary, percent change, MK result
│   │   │   └── ExportBar/
│   │   │       └── index.tsx         # PDF + CSV export buttons
│   │   ├── hooks/
│   │   │   └── useHazardAnalysis.ts  # TanStack Query hook, typed
│   │   ├── types/
│   │   │   └── hazard.ts             # Mirrors backend Pydantic response models
│   │   ├── utils/
│   │   │   ├── formatters.ts         # Number formatting, date display
│   │   │   └── exportHelpers.ts      # PDF/CSV generation logic
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── .env.example
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
```

---

## OPEN-METEO DATA CONTRACT

Endpoint: https://archive-api.open-meteo.com/v1/archive

Required params:
- latitude, longitude (from user selection)
- start_date, end_date (YYYY-MM-DD format)
- daily=temperature_2m_max
- timezone=auto

Response shape you must handle:
```json
{
  "daily": {
    "time": ["1990-01-01", "1990-01-02", ...],
    "temperature_2m_max": [32.1, null, 34.5, ...]
  }
}
```

Data quality rules — handle ALL of these:
1. Nulls in temperature array: forward-fill gaps ≤ 3 days, flag and exclude gaps > 3 days
2. The time and temperature arrays are always the same length — if not, abort and raise
3. Temperature is in °C — never convert, never assume Fahrenheit
4. Max date range per request: chunk into yearly requests if range > 5 years — Open-Meteo
   can time out on very long ranges
5. On HTTP errors from Open-Meteo: retry once with exponential backoff, then raise a
   structured upstream error to the route handler

---

## HEATWAVE DETECTION — CANONICAL DEFINITION
# This logic lives ONLY in services/heatwave_detector.py

```
Input:  pandas Series of daily tmax values, indexed by date
Output: dict with yearly_stats list and raw event list

Algorithm:
1. Compute p95 = np.percentile(series.dropna(), 95) across the ENTIRE date range
   (not per-year — this is the baseline threshold for the region)
2. Create boolean mask: hot_day = tmax > p95
3. Use itertools.groupby or pandas rle to find runs of consecutive True values
4. A heatwave event = any run of length >= 3
5. For each event: record start_date, end_date, duration (days), mean_intensity (mean tmax)
6. Aggregate per calendar year:
   - event_count: number of heatwave events that START in that year
   - total_heatwave_days: sum of durations for events starting that year
   - avg_intensity_celsius: mean of mean_intensity across events in that year
   - max_intensity_celsius: max single-day tmax across all heatwave days in that year
7. Years with zero events still appear in yearly_stats with zeroed values
```

Edge cases you MUST handle:
- Year with no data at all: exclude from yearly_stats, note in response metadata
- Single-year ranges: valid, return that year's stats, no trend (trend requires ≥ 5 years)
- All days below p95: valid result — zero heatwave events, return cleanly
- Heatwave spanning Dec 31 → Jan 1: assign to the year the event STARTS in

---

## TREND ANALYSIS — services/trend_analyzer.py

Inputs: yearly_stats list (from heatwave_detector)
Outputs: trend object + insight_summary string

Steps:
1. Extract event_count per year as a time series [year, count]
2. Linear regression via scipy.stats.linregress(years, counts)
   - slope, intercept, r_value, p_value, std_err
3. Percent change: ((predicted_last - predicted_first) / predicted_first) * 100
   - Use predicted values (regression line), not raw first/last — more honest
4. Mann-Kendall test via pymannkendall.original_test(counts)
   - Returns: trend ("increasing"/"decreasing"/"no trend"), p, Tau, s, var_s, slope
5. Statistical confidence: only call a trend "significant" if p_value < 0.05
6. Insight string generation rules:
   - If significant increasing: "Heatwaves have increased by {X}% in this region since {start_year}."
   - If significant decreasing: "Heatwaves have decreased by {X}% in this region since {start_year}."
   - If not significant: "No statistically significant trend in heatwave frequency was detected for this region between {start_year} and {end_year}."
   - Always append: " (R²={r2:.2f}, p={p:.3f})"
   - Never fabricate confidence. Honest insight > impressive-sounding insight.

---

## MONGODB CACHE CONTRACT

Collection: hazard_cache
Unique index on: cache_key (create on app startup via Motor, idempotent)

Document shape:
```json
{
  "cache_key": "lat:17.38_lon:78.47_1990_2020_heatwave",
  "latitude": 17.38,
  "longitude": 78.47,
  "start_year": 1990,
  "end_year": 2020,
  "hazard_type": "heatwave",
  "computed_at": "<ISODate>",
  "expires_at": "<ISODate: computed_at + 24h>",
  "data_quality": {
    "gap_days_filled": 12,
    "gap_days_excluded": 0,
    "total_days_analyzed": 10958
  },
  "yearly_stats": [...],
  "trend": {
    "slope": 0.12,
    "intercept": 1.4,
    "r_squared": 0.61,
    "p_value": 0.003,
    "direction": "increasing",
    "percent_change": 40.2,
    "mk_trend": "increasing",
    "mk_p_value": 0.018,
    "mk_tau": 0.34
  },
  "insight_summary": "Heatwaves have increased by 40.2% in this region since 1990. (R²=0.61, p=0.003)"
}
```

Cache lookup logic in hazard_cache.py:
- Match on cache_key
- Check expires_at > now — if expired, delete document and recompute
- On cache hit: return document + flag cached: true in response
- On cache miss: compute → insert → return

---

## API CONTRACT

### GET /api/v1/hazard
Query params (all validated via Pydantic):
- lat: float, required, range [-90, 90]
- lon: float, required, range [-180, 180]
- start_year: int, default 1990, min 1950
- end_year: int, default 2020, max current_year - 1
- hazard_type: Literal["heatwave"], default "heatwave"

Validations to enforce:
- end_year > start_year (not >=, they must differ)
- end_year - start_year >= 1 (minimum 2-year range for any result)
- Trend analysis requires >= 5 years — return trend: null with explanation if < 5

Success response (200):
```json
{
  "cached": false,
  "computed_at": "2026-10-14T10:00:00Z",
  "region": { "latitude": 17.38, "longitude": 78.47 },
  "hazard_type": "heatwave",
  "yearly_stats": [
    {
      "year": 1990,
      "event_count": 3,
      "total_heatwave_days": 14,
      "avg_intensity_celsius": 38.4,
      "max_intensity_celsius": 42.1
    }
  ],
  "trend": {
    "slope": 0.12,
    "r_squared": 0.61,
    "p_value": 0.003,
    "direction": "increasing",
    "percent_change": 40.2,
    "significant": true,
    "mk_trend": "increasing",
    "mk_p_value": 0.018
  },
  "insight_summary": "...",
  "data_quality": { "gap_days_filled": 12, "gap_days_excluded": 0, "total_days_analyzed": 10958 }
}
```

Error responses — always structured, never raw exceptions:
```json
{ "error": "VALIDATION_ERROR", "detail": "end_year must be greater than start_year" }
{ "error": "UPSTREAM_ERROR", "detail": "Open-Meteo API unavailable after retry" }
{ "error": "INSUFFICIENT_DATA", "detail": "Less than 5 years of data available for trend analysis" }
```

### GET /health
```json
{ "status": "ok", "db": "connected", "version": "1.0.0" }
```

---

## FRONTEND BEHAVIOUR — EXACT

Default state on load:
- Region: Hyderabad, India (lat: 17.385, lon: 78.4867) — pre-filled, user can change
- Hazard type: Heatwave (locked as default, shown as label not dropdown for now)
- Year range: 1990–2020
- Map: shows Hyderabad marker on Leaflet map, draggable

User flow:
1. User types a city name → geocode via Open-Meteo Geocoding API
   (https://geocoding-api.open-meteo.com/v1/search?name=Hyderabad&count=5)
   → pick first result → update lat/lon + move map marker
2. OR user drags map marker → update lat/lon fields
3. OR user types lat/lon directly
4. User sets year range via two number inputs (not a slider)
5. User clicks "Analyze" → TanStack Query fires → loading state shows
6. On success: all three charts render + InsightPanel shows + ExportBar appears
7. On error: show error type + message from backend JSON, not a generic "Something went wrong"

Charts (all Chart.js):
- Frequency chart: Line chart, x=year, y=event_count, with point markers
- Duration chart: Bar chart, x=year, y=total_heatwave_days
- Intensity chart: Line chart, x=year, y=avg_intensity_celsius, with max_intensity as a
  secondary dashed line on the same chart
- All charts: labeled axes, legend, tooltip showing exact values on hover

InsightPanel shows:
- The insight_summary string — large, prominent
- Trend direction badge (green down arrow / red up arrow / neutral)
- R², p-value, Mann-Kendall result as a small stats row
- data_quality row: "X days filled, Y days excluded, Z days analyzed"

ExportBar:
- "Export CSV": downloads yearly_stats as CSV via Papa Parse unparse
- "Export PDF": generates a single-page PDF via jsPDF with chart images (use chart ref
  + canvas toDataURL) + insight text + data quality note

---

## SECURITY RULES — NON-NEGOTIABLE

1. Rate limiting: 20 req/min per IP on all /api/* routes via slowapi
2. CORS: ALLOWED_ORIGIN env var — only that origin. localhost:5173 only in dev.
3. Input sanitization: Pydantic validators reject anything outside defined bounds silently
   (return 422, not 500)
4. No secrets in source: .env files in .gitignore, only .env.example committed
5. Open-Meteo calls: always server-side only, never expose from frontend
6. MongoDB: connection string in env var MONGODB_URI — never hardcoded
7. httpx timeout: 30s connect, 60s read — never infinite waits
8. Error messages returned to client: never include stack traces, file paths, or internal
   variable names

---

## WHAT THE EVALUATORS ARE ACTUALLY LOOKING FOR
(Read this before making any architectural decision)

- Data handling (25%): They will look for how you handle nulls, gaps, and inconsistency.
  The gap-filling logic and data_quality metadata in the response demonstrate this directly.
- Climate logic (20%): The p95 definition must be implemented correctly. Using per-year p95
  instead of full-period p95 is a common wrong answer. Don't do it.
- Backend skills (20%): Clean layered architecture (router → service → repository),
  async throughout, structured errors, caching design.
- Frontend skills (20%): The charts must be informative and labeled. A chart with no axis
  labels or tooltips will score poorly.
- Critical thinking (15%): The insight string must be statistically honest. R² and p-value
  in the summary text directly signals this.
- Security + performance (10%): Rate limiting, input validation, caching TTL, timeout
  handling — all of this must be present and visible.
- Bonus (up to +10 points): Leaflet map, CSV/PDF export, Mann-Kendall — all three
  are implemented in this project.

---

## DEVELOPMENT SEQUENCE — FOLLOW THIS ORDER

Phase 1 — Foundation (do not skip ahead)
  1. Scaffold full directory structure (both sides), empty files with correct names
  2. Backend: config.py + main.py (app factory, CORS, health endpoint, lifespan)
  3. Backend: db/mongo.py → Motor client, connect on startup, unique index on cache_key
  4. Backend: Pydantic request + response models

Phase 2 — Core Backend
  5. weather_fetcher.py: fetch from Open-Meteo, parse, handle nulls, chunk by year
  6. heatwave_detector.py: full p95 algorithm with edge cases
  7. trend_analyzer.py: regression + Mann-Kendall + insight string
  8. hazard_cache.py: cache read → miss → compute → write pipeline
  9. hazard.py router: wire everything, return typed response

Phase 3 — Core Frontend
  10. hazardClient.ts: typed Axios client
  11. useHazardAnalysis.ts: TanStack Query hook
  12. RegionSelector: city geocode + lat/lon inputs
  13. FrequencyChart + DurationChart + IntensityChart
  14. InsightPanel
  15. Wire App.tsx: full end-to-end flow working with Hyderabad default

Phase 4 — Bonus
  16. MapPicker: Leaflet map with draggable marker
  17. ExportBar: CSV first (simpler), then PDF
  18. Verify Mann-Kendall is surfaced in InsightPanel (it's already in the backend)

Phase 5 — Ship
  19. Write .env.example for both sides
  20. Deploy backend to Render, frontend to Vercel
  21. Smoke test on deployed URLs
  22. Write README (sections listed in evaluation rubric order)

---

## CODING STANDARDS

Python:
- Type annotations on every function signature
- Docstring on every service-layer function (one-liner is fine)
- No bare except: — always catch specific exceptions
- Async all the way: no asyncio.run inside routes, no sync DB calls
- pandas operations: use .copy() after slices to avoid SettingWithCopyWarning

TypeScript:
- strict: true in tsconfig — no exceptions
- Every API response typed — types/hazard.ts mirrors backend response models exactly
- No inline styles — Tailwind classes only
- Chart refs typed as RefObject<ChartJS<...>> for canvas export in PDF

Git:
- Commit at end of each phase, not file-by-file
- Commit message format: "phase-2: implement heatwave detector with p95 logic"
- .env, __pycache__, node_modules, .DS_Store in .gitignore from the start

---

## CONSTRAINTS ON YOUR BEHAVIOUR AS THE AGENT

- When I say "do Phase N step X" — do exactly that step, show output, stop.
- If a step touches more than one file, list all files changed at the end.
- If you're about to make a design decision not covered here, state the two options
  and your recommended one — do not decide silently.
- Never refactor something from a previous step unless I ask for it.
- Never add a dependency not listed in this document without asking first.
- The word "simple" in your code comments means: add a note explaining why this
  approach was chosen over the more complex one, not that you should simplify it.
- If a test fails, show me the exact failure output and your root cause analysis
  before touching any code.


## TASK BOARD PROTOCOL — READ THIS FIRST

A file called `PRODUCT_COMPLETION_BOARD.md` exists at the project root.
It is the single source of truth for project progress.

Rules you must follow without exception:
- After completing ANY task from the board: mark it [x] in the file immediately
- Update the "_Last updated by agent after:_" line at the bottom of the board
- Never mark a task [x] unless you have actually produced the output for it
- Never skip ahead — complete tasks in board order within each phase
- If I ask "where are we?", read the board and report: last [x] task, next [ ] task
- If a task fails and cannot be completed, mark it [~] and add a one-line note inline:
  `[~] Task name — BLOCKED: reason`
- Never delete tasks. Never reorder tasks. Only mark them.