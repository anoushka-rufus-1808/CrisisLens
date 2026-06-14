# CG State Risk Engine

> A real-time disaster risk monitoring and forecasting platform for Chhattisgarh, India — combining live weather data, vulnerability assessments, and ML-powered predictive risk modeling for critical infrastructure.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Folder Structure](#folder-structure)
- [Installation & Setup](#installation--setup)
- [Environment Variables](#environment-variables)
- [Run Instructions](#run-instructions)
- [API Overview](#api-overview)
- [Troubleshooting](#troubleshooting)

---

## Overview

CG State Risk Engine is a full-stack web application built to help government officials and facility managers in Chhattisgarh monitor, assess, and forecast disaster risk across schools and hospitals. It ingests real-time weather data from Open-Meteo, runs risk scoring algorithms, and leverages two machine-learning models (Facebook Prophet and Random Forest) to forecast risk trends over 30, 60, or 90-day horizons.

The platform is role-gated so that state-level administrators get a full system view while individual facility users see only their relevant data.

---

## Features

- **Live Weather Monitoring** — Real-time weather data fetched from the Open-Meteo API with a 15-minute localStorage cache.
- **Interactive Risk Map** — Leaflet-powered district-level map showing risk overlays across Chhattisgarh.
- **ML Risk Forecasting** — Dual-model forecasting (Prophet & Random Forest) with configurable horizons (30/60/90 days) and MAPE accuracy reporting.
- **Model Comparison** — Side-by-side comparison of Prophet vs. Random Forest predictions on the same dataset.
- **Bulk Facility Forecasting** — Background bulk-run tasks streamed live via Server-Sent Events (SSE).
- **Facility Management** — Full CRUD operations for schools and hospitals stored in SQLite.
- **Vulnerability Assessments** — Risk scoring engine based on structural, demographic, and environmental factors.
- **Automated Email Alerts** — EmailJS integration for dispatch of automated risk alerts.
- **Role-Based Access Control** — Admin (Government Official) and User (Facility-specific) roles.
- **Historical Data Analysis** — District-level CSV datasets used for model training and backtesting.

---

## Tech Stack

### Frontend
| Technology | Purpose |
|---|---|
| React 18 + TypeScript | UI framework |
| Vite | Build tool & dev server |
| Tailwind CSS | Utility-first styling |
| Radix UI / shadcn/ui | Accessible component primitives |
| Framer Motion | Animations & transitions |
| React-Leaflet / Leaflet | Interactive maps |
| Recharts | Data visualization & charts |
| TanStack Query | Server-state management & caching |
| Wouter | Lightweight client-side routing |
| EmailJS | Client-side email dispatch |
| date-fns | Date formatting & manipulation |

### Backend (`forecast-service`)
| Technology | Purpose |
|---|---|
| FastAPI (Python) | REST API framework |
| SQLite (`facilities.db`) | Persistent facility storage |
| Pandas / NumPy | Data manipulation |
| Scikit-learn | Random Forest model |
| Prophet (Facebook) | Time-series forecasting |
| Httpx | Async HTTP client |
| Uvicorn | ASGI server |

### External APIs
| Service | Usage |
|---|---|
| Open-Meteo | Real-time & historical weather data |

---

## Folder Structure

```
cg-state-risk-engine/
├── forecast-service/           # Python FastAPI backend
│   ├── routes/                 # Modular API route handlers
│   │   ├── forecast.py         # Forecasting endpoints
│   │   └── facilities.py       # Facility CRUD endpoints
│   ├── db.py                   # SQLite database setup & queries
│   ├── main.py                 # FastAPI app entry point
│   └── requirements.txt        # Python dependencies
│
├── public/
│   └── data/historical/        # District-level CSV files for model training
│
├── src/                        # React + TypeScript frontend
│   ├── components/             # Reusable UI components (shadcn/ui + custom)
│   ├── config/
│   │   └── emailjs.ts          # EmailJS service configuration
│   ├── context/
│   │   ├── AuthContext.tsx     # Authentication & session management
│   │   └── DataContext.tsx     # Shared application data state
│   ├── data/                   # Static data (district lists, LGD codes)
│   ├── engine/                 # Risk scoring & prediction logic
│   ├── hooks/                  # Custom hooks (useWeather, useFacilities, …)
│   ├── pages/                  # Top-level page components
│   └── utils/                  # ML helpers & risk factor utilities
│
├── package.json                # Node dependencies & scripts
├── vite.config.ts              # Vite configuration
├── tsconfig.json               # TypeScript configuration
├── components.json             # shadcn/ui configuration
```

---

## Installation & Setup

### Prerequisites

- **Node.js** v18+ and **npm** / **pnpm**
- **Python** 3.11+
- **pip** (Python package manager)

### 1. Clone the repository

```bash
git clone https://github.com/your-org/cg-state-risk-engine.git
cd cg-state-risk-engine
```

### 2. Install frontend dependencies

```bash
npm install
```

### 3. Install backend dependencies

```bash
cd forecast-service
pip install -r requirements.txt
cd ..
```

> **Note:** Prophet requires `pystan` and may need additional system packages. On Debian/Ubuntu: `sudo apt-get install gcc g++ libpython3-dev`.

---

## Environment Variables

### EmailJS (Frontend)

EmailJS credentials are configured in `src/config/emailjs.ts`. Fill in your account values:

```ts
// src/config/emailjs.ts
export const EMAILJS_CONFIG = {
  serviceId: "YOUR_SERVICE_ID",
  templateId: "YOUR_TEMPLATE_ID",
  publicKey: "YOUR_PUBLIC_KEY",
};
```

Obtain these values from your [EmailJS dashboard](https://www.emailjs.com/).

### Vite

No `.env` file is required for the base setup. Optional Vite environment variables:

| Variable | Default | Description |
|---|---|---|
| `VITE_API_BASE_URL` | `http://localhost:8001` | Backend API base URL |

Create a `.env` file in the project root if you need to override defaults:

```env
VITE_API_BASE_URL=http://localhost:8001
```

### Backend

The FastAPI service uses no external environment variables by default. The SQLite database (`facilities.db`) is created automatically at first run inside `forecast-service/`.

---

## Run Instructions

### Development (both services together)

The `package.json` uses `concurrently` to start both the Vite dev server and the Python backend simultaneously:

```bash
npm run dev
```

This starts:
- **Frontend** → `http://localhost:5173`
- **Backend API** → `http://localhost:8001`

### Run services individually

**Frontend only:**
```bash
npm run dev:client
```

**Backend only:**
```bash
cd forecast-service
uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```

### Production build

```bash
npm run build
npm run preview
```

### Default Login Credentials

| Role | Email | Password |
|---|---|---|
| Admin (Government Official) | `admin@cgdisaster.gov.in` | `Admin@123` |

> Facility-level user accounts are managed through the admin panel.

---

## API Overview

The FastAPI backend runs on **port 8001** and exposes the following endpoint groups:

| Group | Base Path | Description |
|---|---|---|
| Health | `/healthz` | Service health & cache status |
| Forecasting | `/forecast` | ML risk forecasts (Prophet & Random Forest) |
| Bulk Forecasting | `/forecast/run`, `/forecast/stream/{id}` | Background bulk runs with SSE streaming |
| Model Comparison | `/forecast/compare` | Side-by-side model comparison |
| Facilities | `/facilities` | CRUD for schools & hospitals |

For full endpoint documentation including request/response schemas, status codes, and cURL examples, see **[API_DOCS.md](./API_DOCS.md)**.

---


## Troubleshooting

### Prophet installation fails
Prophet requires a C++ compiler and `pystan`. Install system dependencies first:
```bash
# Ubuntu / Debian
sudo apt-get install gcc g++ python3-dev

pip install pystan==2.19.1.1
pip install prophet
```

### Backend returns `502` or is unreachable
Ensure the FastAPI service is running on port `8001`:
```bash
curl http://localhost:8001/healthz
```
If the port is in use, change it in `forecast-service/main.py` and update `VITE_API_BASE_URL`.

### Weather data not loading
Open-Meteo is a public API with no key required. If it fails:
- Check your internet connection.
- Open browser DevTools → Network tab and inspect the failing Open-Meteo request.
- The app caches responses in `localStorage` for 15 minutes. Clear cache with:
  ```bash
  curl -X DELETE http://localhost:8001/cache
  ```

### Map tiles not rendering
Ensure Leaflet CSS is imported. If tiles are blank, check your network connection — tiles are served by OpenStreetMap CDN.

### SQLite database not found
The `facilities.db` file is created automatically on first run. If you see a database error, ensure the FastAPI process has write permissions in the `forecast-service/` directory.

### Frontend shows blank page after build
Run `npm run build` and check for TypeScript errors. Ensure `VITE_API_BASE_URL` points to the correct backend URL in the production environment.

### EmailJS alerts not sending
- Verify `serviceId`, `templateId`, and `publicKey` in `src/config/emailjs.ts`.
- Check the EmailJS dashboard for quota limits (free tier: 200 emails/month).
