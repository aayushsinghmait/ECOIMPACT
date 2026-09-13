# 🌿 EcoImpact — “Assess Before You Build.”

**Production-Quality AI-Powered Environmental Impact Assessment (EIA) Screening & Decision-Support Platform**

---

## ⚠️ Statutory Legal Compliance Notice

> «“EcoImpact is an environmental screening and decision-support tool. It does not replace statutory Environmental Impact Assessment, government environmental clearance, legal compliance, consent/permit requirements, or professional environmental studies.”»
> 
> *Never falsely claim that an AI-generated screening is an official government EIA or environmental clearance.*

---

## 🏛️ Platform Architecture

EcoImpact consists of **TWO connected applications** powered by a single secure backend and unified relational database:

1. **EcoImpact User App** (Proponents, Planners, Environmental Consultants)
   - Guided 7-step assessment wizard: `PROJECT → DETAILS → LOCATION → ENVIRONMENT → ANALYSIS → RESULTS → REPORT`
   - GPS location & Interactive Map pin selection with coordinates validation
   - Live geospatial environmental context scanner (OpenStreetMap Overpass, Open-Meteo Air Quality & CAMS, Copernicus DEM 90m, BIS IS 1893:2016 Seismic Hazard)
   - EcoImpact Environmental Screening Agent & Explainable Risk Matrix (9 factors: Air, Water, Land, Noise, Ecology, Waste, Traffic, Climate, Social)
   - Interactive What-If Scenario Sandbox for testing mitigation trade-offs
   - Multi-page professional PDF Environmental Screening Report generator

2. **EcoImpact Admin Dashboard** (Regulators, Supervisors, Reviewers)
   - Role-Based Access Control (`SUPER_ADMIN`, `ADMIN`, `REVIEWER`)
   - Real-time WebSocket event gateway broadcasting live user activities (signups, location locks, screenings, report generation)
   - Proponent user directory with status toggling (**zero passwords stored**)
   - Full spatial assessment inspector
   - PDF reports repository with 1-click downloads
   - Regulatory screening rule & threshold configuration editor
   - Immutable administrative audit trail

---

## 🚀 Quick Start & Local Execution

### 1. Start the Backend Server (Express + WebSockets + SQLite)
```bash
cd server
npm.cmd install
npm.cmd run dev
```
*Backend runs at `http://localhost:5000` with WebSocket endpoint at `ws://localhost:5000/ws`.*

### 2. Start the Frontend Client (Vite + React + TypeScript + Tailwind)
```bash
cd client
npm.cmd install
npm.cmd run dev
```
*Frontend opens at `http://localhost:5173`.*

---

## 🔐 Authentication & Zero Password Security Rule

EcoImpact strictly adheres to the highest identity security protocols:
- **Google OAuth / Google Identity Services (GIS)**: Genuine Google login flow with server-side ID token verification.
- **Zero Secrets Rule**: The application **NEVER** asks for, collects, transmits, or stores Gmail passwords, Google passwords, Google recovery passwords, or OTP codes.
- **Sandbox Test Accounts**: For reviewers testing without Google Cloud Console credentials, authentic test profiles (Alex Sharma / Priya Sundaram) are pre-configured.
- **Human Verification / Anti-Bot Protection**: Server-side proof-of-work challenge and Google reCAPTCHA v3 support.

### Admin Credentials (Pre-Configured)
- **Super Administrator**: `admin@ecoimpact.org` / `Admin@EcoImpact2026!`
- **Reviewer**: `priya.reviewer@ecoimpact.org` / `Reviewer@2026!`

---

## 📡 Data Provenance & Real Data Disclosures

EcoImpact guarantees **zero invented environmental data**:
- **Water Bodies & Forests**: Live spatial queries via OpenStreetMap Overpass API around exact coordinates.
- **Ambient Air Quality**: Real-time Copernicus Atmosphere Monitoring Service (PM2.5, PM10, AQI).
- **Surface Elevation**: Copernicus 90m Global Elevation Model.
- **Seismic Hazard**: Bureau of Indian Standards IS 1893:2016 zoning (Zone II to Zone V).
- **Statutory Framework**: Ministry of Environment, Forest and Climate Change (MoEFCC) EIA Notification 2006 (Category A / Category B1 / Category B2).

If any third-party environmental API is unreachable, the system explicitly reports:
*“Unable to retrieve this dataset at this time.”*
