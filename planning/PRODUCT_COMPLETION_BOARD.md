# PRODUCT_COMPLETION_BOARD.md
# Climate Hazard Trend Analyzer — Master Task Board
# Legend: [ ] = pending | [x] = done | [~] = blocked
# Rule: Update this file after EVERY completed task before moving to the next one.

---

## PHASE 0 — PROJECT SCAFFOLDING

### Repository & Root Setup
[x] Initialize git repository
[x] Create .gitignore (Python, Node, OS files, .env)
[x] Create root README.md (empty placeholder)
[x] Create PRODUCT_COMPLETION_BOARD.md (this file)
[ ] Create root docker-compose.yml (backend + mongo + frontend 1-command startup)

### Backend Scaffold
[ ] Create backend/ directory structure (all folders and empty files)
[ ] Create backend/.env.example with all required variable names
[ ] Create backend/requirements.txt with all dependencies pinned
[ ] Create backend/Dockerfile

### Frontend Scaffold
[ ] Create frontend/ directory structure (all folders and empty files)
[ ] Create frontend/.env.example with all required variable names
[ ] Initialize package.json with all dependencies
[ ] Create tsconfig.json with strict mode enabled
[ ] Create vite.config.ts

---

## PHASE 1 — BACKEND FOUNDATION

### Configuration & App Factory
[ ] Implement config.py — pydantic-settings with all env vars typed
[ ] Implement main.py — FastAPI app factory with lifespan context manager
[ ] Add CORS middleware — reads ALLOWED_ORIGIN from config
[ ] Add slowapi rate limiter — 20 req/min per IP on all /api/* routes
[ ] Add observability middleware — X-Process-Time-Ms and X-Cache headers
[ ] Implement GET /health endpoint — returns status, active db ping latency (db.command('ping')), version

### Database Layer
[ ] Implement db/mongo.py — Motor async client
[ ] Add MongoDB connect on app startup (lifespan)
[ ] Add MongoDB disconnect on app shutdown (lifespan)
[ ] Create unique index on hazard_cache.cache_key on startup (idempotent)
[ ] Verify index creation doesn't error on re-run

### Extensibility Architecture & Pydantic Models
[ ] Define Abstract Base Class BaseHazardDetector for future multi-hazard extension (heatwave, drought, rainfall)
[ ] Implement models/request.py — HazardRequest with all validators
[ ] Validate lat range [-90, 90]
[ ] Validate lon range [-180, 180]
[ ] Validate start_year min 1950
[ ] Validate end_year max current_year - 1
[ ] Validate end_year > start_year
[ ] Validate minimum 2-year range
[ ] Implement models/response.py — YearlyStats, TrendResult, DataQuality, RecordEvent, GridCoordinates, HazardResponse
[ ] Implement structured error response model

---

## PHASE 2 — BACKEND CORE SERVICES

### Weather Fetcher (services/weather_fetcher.py)
[ ] Implement async Open-Meteo fetch with httpx AsyncClient
[ ] Set 30s connect timeout, 60s read timeout
[ ] Implement yearly chunking for date ranges > 5 years
[ ] Parse response — zip time array with temperature_2m_max array
[ ] Capture Open-Meteo elevation and snapped grid coordinates in response
[ ] Validate time and temperature arrays are same length — raise if not
[ ] Handle null values — forward-fill gaps ≤ 3 days
[ ] Flag and exclude gaps > 3 days from analysis
[ ] Track gap_days_filled and gap_days_excluded counts
[ ] Implement retry logic — one retry with exponential backoff on HTTP error
[ ] Provide offline fallback fixture (e.g. Hyderabad demo dataset) to guarantee resilience during live reviews
[ ] Return structured upstream error after failed retry

### Heatwave Detector (services/heatwave_detector.py)
[ ] Implement BaseHazardDetector inheritance for heatwave detector
[ ] Load daily tmax as pandas Series indexed by date
[ ] Compute p95 across ENTIRE date range (not per-year)
[ ] Expose p95_threshold_celsius baseline in detector output
[ ] Create boolean hot_day mask: tmax > p95
[ ] Detect runs of consecutive True values
[ ] Filter runs to heatwave events: length >= 3 consecutive days
[ ] Record per-event: start_date, end_date, duration, mean_intensity, max_intensity
[ ] Track all-time historical record event (longest duration, peak tmax, date)
[ ] Assign each event to the year it STARTS in
[ ] Aggregate per year: event_count, total_heatwave_days, avg_intensity, max_intensity
[ ] Include years with zero events in output (zeroed values, not omitted)
[ ] Handle heatwave spanning Dec 31 → Jan 1 (assign to start year)
[ ] Handle year with no data — exclude from stats, note in metadata
[ ] Handle all days below p95 — return cleanly with zero events
[ ] Handle single-year range — return stats, skip trend (needs >= 5 years)

### Trend Analyzer (services/trend_analyzer.py)
[ ] Extract event_count per year as time series
[ ] Run scipy.stats.linregress(years, counts)
[ ] Compute percent_change from regression predicted values (not raw first/last)
[ ] Generate fitted regression line points for frontend chart overlay
[ ] Compute Sen's Slope (Theil-Sen estimator) for robust non-parametric trend magnitude
[ ] Run pymannkendall.original_test on event_count series
[ ] Determine significance: p_value < 0.05
[ ] Generate insight string — increasing case
[ ] Generate insight string — decreasing case
[ ] Generate insight string — no significant trend case
[ ] Append R² and p-value to all insight strings
[ ] Return null trend object if fewer than 5 years of data, with explanation
[ ] Include MK trend, MK p-value, MK tau, and Sen's slope in trend result

### Cache Repository (repositories/hazard_cache.py)
[ ] Implement cache lookup by cache_key
[ ] Check expires_at > now — treat expired as miss
[ ] Delete expired document before recompute
[ ] Implement cache write — insert full result document
[ ] Set expires_at = computed_at + 24 hours on write
[ ] Return cached: true flag on cache hit
[ ] Return cached: false flag on cache miss

### Hazard Router (routers/hazard.py)
[ ] Implement GET /api/v1/hazard route
[ ] Inject query params via HazardRequest model
[ ] Cache lookup first — return early on hit
[ ] On miss: fetch weather → detect hazard → analyze trend → cache → return
[ ] Return typed HazardResponse on success
[ ] Return structured error JSON on validation failure (422)
[ ] Return structured error JSON on upstream failure (502)
[ ] Return structured error JSON on insufficient data (422)

---

## PHASE 3 — BACKEND TESTS

### Unit Tests
[ ] Test p95 computed across full range, not per year
[ ] Test p95 threshold value correctly reported in response
[ ] Test run detection — exactly 3 consecutive days = 1 event
[ ] Test run detection — 2 consecutive days = 0 events
[ ] Test event assigned to start year when spanning Dec 31
[ ] Test zero-event year appears in output with zeroed values
[ ] Test all-below-p95 data returns cleanly
[ ] Test historical record event correctly identified
[ ] Test gap fill — gap ≤ 3 days is filled
[ ] Test gap exclusion — gap > 3 days is excluded and counted
[ ] Test percent_change uses regression predicted values
[ ] Test Sen's slope computation aligns with pymannkendall
[ ] Test insight string content for each trend direction
[ ] Test trend returns null for < 5 years input
[ ] Test cache key format is consistent
[ ] Test offline fallback activates when upstream is unreachable

---

## PHASE 4 — FRONTEND FOUNDATION

### API Client
[ ] Implement api/hazardClient.ts — Axios instance with VITE_API_BASE_URL
[ ] Add typed request function for GET /api/v1/hazard
[ ] Add typed response interfaces matching backend HazardResponse exactly
[ ] Handle structured error responses from backend — parse error field

### TanStack Query Hook
[ ] Implement hooks/useHazardAnalysis.ts
[ ] Configure query key with all params (lat, lon, start_year, end_year)
[ ] Enable query only when lat, lon are set
[ ] Expose: data, isLoading, isError, error, refetch
[ ] Type the query return — no any

---

## PHASE 5 — FRONTEND CORE COMPONENTS

### Region Selector (components/RegionSelector/)
[ ] Implement city search input
[ ] On input change (debounced 300ms) — call Open-Meteo geocoding API
[ ] Display top 5 city suggestions as dropdown
[ ] Add 1-click climate benchmark city presets (Hyderabad, New Delhi, Phoenix, Seville, London)
[ ] On city select — update lat/lon state and close dropdown
[ ] Implement direct lat/lon number inputs as fallback
[ ] Validate lat/lon ranges client-side before allowing Analyze
[ ] Display selected region name, elevation, and coordinates below selector

### Year Range Inputs
[ ] Implement start_year number input (min 1950)
[ ] Implement end_year number input (max current_year - 1)
[ ] Validate end_year > start_year — show inline error, disable Analyze if invalid
[ ] Show warning if range < 5 years (trend analysis unavailable)

### Charts (components/HazardChart/)
[ ] Register only required Chart.js components (no full bundle import)
[ ] Implement FrequencyChart.tsx — line chart, x=year, y=event_count
[ ] Add regression trend line overlay toggle on FrequencyChart
[ ] Add labeled x-axis (Year), labeled y-axis (Heatwave Events)
[ ] Add point markers on line
[ ] Add tooltip showing exact year and count on hover
[ ] Implement DurationChart.tsx — bar chart, x=year, y=total_heatwave_days
[ ] Add labeled axes, tooltip with exact values
[ ] Implement IntensityChart.tsx — line chart, y=avg_intensity_celsius
[ ] Add max_intensity as secondary dashed line on same chart
[ ] Add p95 baseline threshold indicator reference line/badge
[ ] Add legend distinguishing avg vs max intensity
[ ] Forward chart ref on all three charts (needed for PDF export)

### Insight Panel (components/InsightPanel/)
[ ] Display insight_summary string prominently (large text)
[ ] Show trend direction badge — red upward arrow / green downward / neutral
[ ] Show stats row: R², p-value, Sen's slope, MK trend, MK p-value, MK tau
[ ] Show Historical Record Event card (longest heatwave & peak temperature date)
[ ] Show data_quality row: days filled, days excluded, days analyzed, p95 baseline temp
[ ] Hide InsightPanel entirely until data is loaded

### App Layout
[ ] Implement App.tsx — layout: sidebar (selector) + main (charts + insight)
[ ] Show loading spinner while query is in-flight
[ ] Show error panel with backend error type + detail message on failure
[ ] Show empty state before first Analyze (prompt user to select region)
[ ] ExportBar appears only after successful data load

---

## PHASE 6 — BONUS FEATURES

### Map Region Selector (components/RegionSelector/MapPicker.tsx)
[ ] Initialize Leaflet map centered on default region (Hyderabad)
[ ] Place draggable marker at current lat/lon
[ ] On marker drag end — update lat/lon state in parent
[ ] On city geocode selection — move marker to new position
[ ] Show zoom controls, tile layer attribution

### CSV Export (components/ExportBar/)
[ ] Implement CSV export via Papa Parse unparse
[ ] Include columns: year, event_count, total_heatwave_days, avg_intensity, max_intensity
[ ] Add header row with units noted
[ ] Trigger browser download on button click
[ ] Filename: hazard-analysis-{lat}-{lon}-{start_year}-{end_year}.csv

### PDF Export (components/ExportBar/)
[ ] Capture all three chart canvases via chart ref + toDataURL
[ ] Generate PDF via jsPDF with executive advisory layout
[ ] Page 1: Header, region info, elevation, grid coords, year range, insight_summary, KPI summary cards
[ ] Page 1: Historical record event and data quality stats
[ ] Page 2: Frequency chart (with regression line) & duration chart images
[ ] Page 3: Intensity chart image & methodology notes
[ ] Filename: hazard-report-{lat}-{lon}-{start_year}-{end_year}.pdf

### Mann-Kendall Surface in UI
[ ] InsightPanel shows MK trend label (e.g. "Mann-Kendall: increasing")
[ ] InsightPanel shows MK tau value and Sen's slope magnitude
[ ] InsightPanel shows MK p-value with significance indicator

---

## PHASE 7 — INTEGRATION & QA

### End-to-End Testing
[ ] Test full flow: Hyderabad, 1990–2020 — data loads, charts render, insight shows
[ ] Test benchmark presets — click Delhi/Phoenix, verify immediate update and marker movement
[ ] Test city search — type "Mumbai", select, verify lat/lon updates, map marker moves
[ ] Test direct lat/lon input — enter custom coordinates, analyze
[ ] Test invalid input — end_year < start_year — verify inline error, Analyze disabled
[ ] Test < 5 year range — verify warning shows, trend section shows null gracefully
[ ] Test cache hit — run same query twice, verify second returns cached: true and X-Cache: HIT
[ ] Test rate limit — verify 21st request in a minute returns 429
[ ] Test offline fallback — simulate network cutoff, verify golden dataset renders with fallback notice
[ ] Test error state — disconnect backend, verify frontend shows structured error

### Cross-browser & Responsive Check
[ ] Charts render correctly on Chrome
[ ] Charts render correctly on Firefox
[ ] Layout is usable on 1280px width
[ ] Layout is usable on 375px mobile width (basic, not perfect)

---

## PHASE 8 — DEPLOYMENT

### Backend (Render)
[ ] Create Render Python web service
[ ] Set all environment variables in Render dashboard
[ ] Confirm /health returns 200 on deployed URL
[ ] Confirm /api/v1/hazard returns data on deployed URL

### Frontend (Vercel)
[ ] Create Vercel project connected to GitHub repo
[ ] Set VITE_API_BASE_URL to deployed Render backend URL
[ ] Confirm deployed frontend loads with default Hyderabad state
[ ] Confirm full analysis flow works on deployed URLs end-to-end

---

## PHASE 9 — DOCUMENTATION

### README.md (Final)
[ ] Project overview section
[ ] Architecture section (ASCII diagram or clear description)
[ ] Multi-hazard extensibility design (BaseHazardDetector architecture)
[ ] 1-command Docker Compose setup instructions (evaluator quickstart)
[ ] Local setup instructions — backend
[ ] Local setup instructions — frontend
[ ] All environment variables listed with descriptions
[ ] Data source explanation & caveats (reanalysis ERA5 grid resolution, p95 regional baseline vs WMO normals)
[ ] Heatwave detection logic explained in plain English
[ ] Statistical methodology explained (OLS regression, Sen's slope, Mann-Kendall test)
[ ] Assumptions documented
[ ] What I would improve with more time
[ ] AI tools used and how (required by case study)
[ ] Deployed app URL
[ ] Demo video link (if recorded)

---

## BONUS TRACKING

[ ] Leaflet map region selection — implemented
[ ] CSV export — implemented
[ ] PDF export (Executive Report format) — implemented
[ ] Mann-Kendall test — implemented in backend
[ ] Sen's slope (Theil-Sen estimator) — implemented in backend
[ ] Mann-Kendall & Sen's slope result — surfaced in frontend UI
[ ] Linear regression trend line overlay — implemented
[ ] 1-Click benchmark climate presets — implemented
[ ] 1-Command Docker Compose stack — implemented
[ ] Offline demo resilience fixture — implemented
[ ] Demo video recorded and linked

---
_Last updated by agent after: Initial root setup (.gitignore, README.md, planning board)_