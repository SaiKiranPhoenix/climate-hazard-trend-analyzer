# Climate Hazard Trend Analyzer

A production-grade, full-stack climate analytics web application that ingests historical weather reanalysis data, detects multi-decade climate hazards (specifically heatwave events using standard climatological definitions), computes parametric and non-parametric trend statistics, and provides interactive visualizations and executive reports.

Built for the **Dygnify Ventures Full-Stack Developer Case Study**.

---

## Architecture Overview

```
                      ┌────────────────────────────────────────┐
                      │             User Browser               │
                      └──────────────────┬─────────────────────┘
                                         │
                         HTTPS / JSON    ▼
                      ┌────────────────────────────────────────┐
                      │      Frontend (React 18 + Vite)        │
                      │  - Tailwind CSS Styling                │
                      │  - TanStack Query (Server State)       │
                      │  - Chart.js (Frequency/Duration/Int.) │
                      │  - Leaflet Map (Coordinate Picker)     │
                      │  - jsPDF + Papa Parse (Exports)        │
                      └──────────────────┬─────────────────────┘
                                         │
                   REST API              ▼
   (/api/v1/hazard, /health)  ┌────────────────────────────────────────┐
   Rate Limited (20 req/min)  │       FastAPI Backend (Python 3.11+)   │
                              │  - App Lifespan & CORS Middleware     │
                              │  - Pydantic v2 Bounds Validation       │
                              │  - BaseHazardDetector Architecture     │
                              │  - scipy (OLS) + pymannkendall (MK)    │
                              └──────┬──────────────────────┬──────────┘
                                     │                      │
                      Cache Check    ▼                      ▼ External Fetch
             ┌──────────────────────────────┐     ┌────────────────────────────┐
             │     MongoDB (Motor Async)    │     │      Open-Meteo API        │
             │   - Collection: hazard_cache │     │  - Historical Archive      │
             │   - Unique Index: cache_key  │     │  - Geocoding API           │
             │   - 24-Hour TTL Expiry       │     │  - ERA5 Reanalysis Grid    │
             └──────────────────────────────┘     └────────────────────────────┘
```

---

## Key Features & Evaluation Highlights

### 1. Statistically Rigorous Climate Logic
- **Full-Period Baseline ($p_{95}$)**: Calculates the 95th percentile threshold of daily maximum temperatures ($T_{\max}$) across the **entire multi-decade period** for the region rather than per-year, establishing a true climatological baseline.
- **Consecutive Run Detection**: Detects heatwave events as continuous runs of $\ge 3$ consecutive days exceeding the $p_{95}$ baseline.
- **Year Boundary Handling**: Heatwaves spanning across December 31 and January 1 are correctly assigned to their starting year.
- **Zero-Event Preservation**: Years with zero heatwave events are preserved with explicit zeroed values to prevent sample truncation in time series analysis.

### 2. Multi-Metric Trend Analysis
- **Ordinary Least Squares (OLS) Linear Regression**: Computes trend slope, intercept, $R^2$, and $p$-value.
- **Non-Parametric Mann-Kendall Test**: Employs `pymannkendall` to detect monotonic upward/downward trends without assuming a normal distribution.
- **Sen's Slope Estimator (Theil-Sen)**: Calculates median slope across all pairwise points to guard against extreme climate anomalies.
- **Statistically Honest Insights**: Generates plain-English summaries reporting both percent change (calculated from regression fitted values) and confidence metrics ($R^2$, $p$-value).

### 3. Resilient Data Engineering
- **Missing Data Strategy**: Forward-fills isolated temperature gaps $\le 3$ days; flags and excludes severe gaps $> 3$ days while recording audit counts (`gap_days_filled`, `gap_days_excluded`).
- **Yearly Request Chunking**: Breaks date ranges $> 5$ years into yearly chunks to prevent external API timeouts.
- **Spatial Resolution Tracking**: Captures and exposes snapped Open-Meteo ERA5 grid coordinates and elevation.
- **Offline Demo Resilience**: Bundles pre-cached golden fixtures to ensure flawless presentations during evaluation walk-throughs.

### 4. Interactive Frontend & Executive Reporting
- **Leaflet Map Picker**: Interactive coordinate selection with draggable marker and geocoding search.
- **Benchmark Climate Presets**: 1-click presets covering distinct global climate zones (*Hyderabad [Tropical]*, *New Delhi [Subtropical]*, *Phoenix [Arid]*, *London [Maritime]*).
- **Synchronized Visualizations**:
  - *Frequency Chart*: Heatwave event counts per year with an interactive Linear Regression trend line overlay.
  - *Duration Chart*: Cumulative annual heatwave days.
  - *Intensity Chart*: Mean event intensity alongside peak temperature and the $p_{95}$ baseline reference.
- **Executive Exports**: One-click structured CSV export (Papa Parse) and formatted multi-page PDF advisory report (jsPDF).

---

## Directory Structure

```
climate-hazard-trend-analyzer/
├── .gitignore                         # Python, Node, OS, and secret exclusions
├── README.md                          # Project documentation
├── docker-compose.yml                 # 1-command stack orchestration
│
├── planning/
│   ├── PRODUCT_COMPLETION_BOARD.md    # Master task board (single source of truth)
│   └── SYSTEM_PROMPT.md               # Architecture and domain constraints
│
├── backend/
│   ├── app/
│   │   ├── main.py                    # App factory, CORS, rate limiting, lifespan
│   │   ├── config.py                  # Pydantic-settings environment configuration
│   │   ├── routers/
│   │   │   └── hazard.py              # GET /api/v1/hazard & /health routes
│   │   ├── services/
│   │   │   ├── weather_fetcher.py     # Open-Meteo async client & gap handling
│   │   │   ├── base_detector.py       # Pluggable BaseHazardDetector interface
│   │   │   ├── heatwave_detector.py   # Canonical p95 heatwave algorithm
│   │   │   └── trend_analyzer.py      # Regression, Sen's slope, Mann-Kendall
│   │   ├── repositories/
│   │   │   └── hazard_cache.py        # Motor async MongoDB repository with TTL
│   │   ├── models/
│   │   │   ├── request.py             # HazardRequest parameter validators
│   │   │   └── response.py            # Pydantic response models
│   │   └── utils/
│   │       └── date_utils.py          # Date chunking and ISO helpers
│   ├── tests/
│   │   ├── test_heatwave_detector.py  # Unit tests for p95 logic & edge cases
│   │   └── test_trend_analyzer.py     # Unit tests for statistical models
│   ├── .env.example
│   ├── requirements.txt
│   └── Dockerfile
│
└── frontend/
    ├── src/
    │   ├── api/
    │   │   └── hazardClient.ts        # Typed Axios instance
    │   ├── components/
    │   │   ├── RegionSelector/        # Geocoding search, presets & Leaflet MapPicker
    │   │   ├── HazardChart/           # Frequency, Duration & Intensity charts
    │   │   ├── InsightPanel/          # Trend badges, stats row & record events
    │   │   └── ExportBar/             # CSV and executive PDF generators
    │   ├── hooks/
    │   │   └── useHazardAnalysis.ts   # TanStack Query hook
    │   ├── types/
    │   │   └── hazard.ts              # TypeScript interfaces mirroring API
    │   ├── utils/
    │   │   ├── formatters.ts          # Number and date formatters
    │   │   └── exportHelpers.ts       # PDF and CSV export logic
    │   ├── App.tsx
    │   └── main.tsx
    ├── .env.example
    ├── index.html
    ├── package.json
    ├── tsconfig.json
    └── vite.config.ts
```

---

## Quickstart & Local Setup

### Option 1: Docker Compose (Recommended)
Launch the entire system (FastAPI backend, MongoDB, and React frontend) with a single command:
```bash
docker compose up --build
```
- Frontend: `http://localhost:5173`
- Backend API Docs: `http://localhost:8000/docs`
- Health Probe: `http://localhost:8000/health`

### Option 2: Manual Setup

#### 1. Backend (Python 3.11+)
```bash
cd backend
python -m venv venv
# On Windows:
.\venv\Scripts\activate
# On Linux/macOS:
source venv/bin/activate

pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload --port 8000
```

#### 2. Frontend (Node.js 18+)
```bash
cd frontend
npm install
cp .env.example .env
npm run dev
```

---

## Environment Configuration

### Backend (`backend/.env`)
| Variable | Description | Default |
| :--- | :--- | :--- |
| `ENVIRONMENT` | Runtime environment (`development`, `production`) | `development` |
| `ALLOWED_ORIGIN` | Allowed CORS origin URL | `http://localhost:5173` |
| `MONGODB_URI` | MongoDB connection string | `mongodb://localhost:27017` |
| `MONGODB_DB_NAME` | Database name | `climate_hazard_db` |
| `RATE_LIMIT_PER_MINUTE` | Per-IP rate limit | `20/minute` |

### Frontend (`frontend/.env`)
| Variable | Description | Default |
| :--- | :--- | :--- |
| `VITE_API_BASE_URL` | Backend API base URL | `http://localhost:8000` |

---

## Scientific Assumptions & Methodology Notes

1. **Regional Baseline ($p_{95}$)**: Following World Meteorological Organization (WMO) and IPCC climatological guidelines, extreme temperature thresholds are determined relative to local historical distributions rather than fixed global temperatures. 40°C represents an extreme heatwave in London, whereas it is normal summer weather in Hyderabad.
2. **Minimum Data Requirements**: Trend calculations require a minimum 5-year observation span ($N \ge 5$). When shorter ranges are submitted, yearly statistics are computed but statistical trend metrics return `null` with an explanatory message.
3. **Sen's Slope vs OLS**: Both linear regression slope and Sen's slope are provided. Sen's slope is recommended when evaluating regions prone to high year-to-year volatility.

---

## AI Tools Disclosure

As required by the case study guidelines:
- **Tools Utilized**: Advanced Agentic Coding via Gemini 3.8 / Antigravity IDE.
- **Application**: Architectural design verification, test-driven validation, algorithmic heatwave edge case analysis, and task board tracking.
- **Code Ownership**: All business logic, climate algorithms, and architectural implementations were verified, tested, and reviewed line-by-line.

---

## Project Status & Tracking
Project execution progress is tracked in real-time in the master task board:
**[planning/PRODUCT_COMPLETION_BOARD.md](file:///d:/projects/climate-hazard-trend-analyzer/planning/PRODUCT_COMPLETION_BOARD.md)**