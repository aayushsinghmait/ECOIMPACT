# Implementation Plan: EcoImpact Environmental Screening Platform

Build a production-quality, dual-application environmental screening platform called **EcoImpact** with the tagline **“Assess Before You Build.”**

The platform consists of two connected applications sharing a secure backend, unified database, and real-time synchronization:
1. **EcoImpact User App**: Mobile-first, high-aesthetic environmental screening application with a 7-step assessment wizard, GPS & map location capture, live geospatial and environmental data collection, explainable AI screening agent, what-if scenario sandbox, and PDF report generator.
2. **EcoImpact Admin Dashboard**: High-security management portal with role-based access control (SUPER_ADMIN, ADMIN, REVIEWER), live WebSocket metrics & activity feeds, assessment inspection, user management, report repository, configurable screening thresholds, and audit logging.

---

## User Review Required

> [!IMPORTANT]
> **Statutory Disclaimer Enforcement**:
> In accordance with environmental compliance requirements, the application will display the mandatory disclaimer across both apps, assessment outputs, and generated PDF reports:
> *“EcoImpact is an environmental screening and decision-support tool. It does not replace statutory Environmental Impact Assessment, government environmental clearance, legal compliance, consent/permit requirements, or professional environmental studies.”*
> No AI finding will be claimed as an official government environmental clearance.

> [!IMPORTANT]
> **Authentication & API Credentials**:
> - **Google OAuth 2.0 / GIS**: The app supports official Google Identity Services sign-in with backend ID token verification. A dedicated Demo/Sandbox Google authentication mode is also implemented so the app is immediately testable without requiring the reviewer to pre-configure Google Cloud console credentials.
> - **Zero Password Exposure**: Gmail and Google account passwords will **never** be requested, collected, or stored.
> - **AI Integration**: Backend connects to the Gemini API (`@google/genai`) with an intelligent, deterministic EIA rules engine fallback ensuring explainable results even before an external API key is entered.
> - **Live Geospatial Data**: Incorporates live OpenStreetMap Overpass queries (rivers, water bodies, protected areas, schools, hospitals), Open-Meteo Air Quality & Climate APIs, Elevation APIs, and Bureau of Indian Standards (BIS IS 1893) seismic classification. If any API is unreachable, the system will explicitly state *"Unable to retrieve this dataset at this time"* rather than inventing data.

---

## Architecture & Technology Stack

```
                                  ┌────────────────────────────────────────────────┐
                                  │             Client Applications                │
                                  ├───────────────────────┬────────────────────────┤
                                  │   EcoImpact User App  │ EcoImpact Admin Portal │
                                  │   (Vite + React + TS) │ (Vite + React + TS)    │
                                  └───────────┬───────────┴───────────┬────────────┘
                                              │ HTTP / REST           │ WebSockets
                                              ▼                       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           EcoImpact Backend (Node.js / Express / TS)             │
├──────────────────────────────────────────────────────────────────────────────────┤
│ ├── Auth Service (Google OAuth token verification, Session JWT, Rate Limiting)   │
│ ├── Human Verification Service (reCAPTCHA / anti-bot token validation)           │
│ ├── Environmental Pipeline (OSM Overpass, Open-Meteo AQI, Open-Elevation, BIS)  │
│ ├── AI Screening Agent (Gemini API + Rule-based fallback + India EIA 2006 logic) │
│ ├── Risk Engine (Explainable 9-factor matrix: Air, Water, Land, Noise, Ecology..)│
│ ├── What-If Simulation Sandbox (Dynamic factor recalibration)                    │
│ ├── PDF Report Engine (Multi-page styled PDF generator with charts & tables)    │
│ ├── WebSocket Gateway (Real-time live updates from User actions to Admin)        │
│ └── Admin Audit Logger (Immutable audit trail of all administrative actions)    │
└────────────────────────────────────────┬─────────────────────────────────────────┘
                                         ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   Unified SQLite Database (Relational Schema)                    │
│ users | admins | roles | assessments | projects | locations | environmental_data  │
│ risk_assessments | mitigation_recommendations | scenarios | reports | audit_logs │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Proposed Changes

### 1. Backend Service (`/server`)

Create a robust, typed Express server with real-time WebSockets and relational SQLite database.

#### [NEW] `server/package.json`
Dependencies:
- `express`, `cors`, `dotenv`, `better-sqlite3` (or `sqlite3`), `jsonwebtoken`, `bcryptjs`, `ws` (WebSockets)
- `google-auth-library` (Google token verification)
- `axios` (external environmental API queries)
- `pdfkit` / `jspdf` / `@google/genai`
- TypeScript dev dependencies (`tsx`, `typescript`, `@types/*`)

#### [NEW] `server/src/db/schema.sql` & `server/src/db/database.ts`
Full relational schema supporting all required entities:
- `users`: ID, google_sub, name, email, avatar_url, role, account_status, created_at, last_login
- `admins`: ID, username, email, password_hash, role (`SUPER_ADMIN`, `ADMIN`, `REVIEWER`), last_login, created_at
- `assessments`: ID, user_id, project_name, project_category, project_stage, status, created_at, updated_at
- `projects`: ID, assessment_id, description, purpose, developer_org, contact, start_date, operational_date, scale_data (JSON: area, workers, water, power, fuel, traffic, etc.)
- `locations`: ID, assessment_id, latitude, longitude, address, accuracy, confirmed_at, method (gps/map/manual)
- `environmental_data`: ID, assessment_id, factor_category, feature_name, distance_meters, data_source, data_date, confidence_level, raw_metadata (JSON)
- `risk_assessments`: ID, assessment_id, overall_score, status (`LOW`, `MODERATE`, `HIGH`, `CRITICAL`), factor_scores (JSON), eia_category (`Category A`, `Category B1`, `Category B2`), methodology_notes
- `mitigation_recommendations`: ID, assessment_id, factor, recommendation_text, type (`AI_RECOMMENDATION` vs `STATUTORY_REQUIREMENT`), priority
- `scenarios`: ID, assessment_id, name, parameters_delta (JSON), recalculated_score, created_at
- `reports`: ID, assessment_id, user_id, report_number, pdf_path, generated_at, status
- `screening_rules`: ID, factor, weight, buffer_meters, sensitivity_level, threshold_high, updated_at
- `audit_logs`: ID, admin_id, admin_name, action, target_type, target_id, details (JSON), ip_address, timestamp

#### [NEW] `server/src/services/authService.ts`
- Google OAuth token validation using Google API / tokeninfo endpoint
- Fallback sandbox Google token verifier for zero-configuration local review
- JWT session issuance & middleware verification (`authenticateUser`, `authenticateAdmin`, `requireRole`)
- Anti-bot human verification handler (reCAPTCHA token evaluation & rate-limiter)

#### [NEW] `server/src/services/environmentalPipeline.ts`
- **OpenStreetMap Overpass API**: live radius queries around `(lat, lng)`:
  - Water bodies (rivers, lakes, wetlands, reservoirs)
  - Ecological zones (forests, nature reserves, national parks)
  - Sensitive human receptors (schools, universities, hospitals, residential clusters)
  - Infrastructure (railways, highways, industrial zones)
- **Air Quality & Meteorology**: Open-Meteo Air Quality API (PM2.5, PM10, NO2, SO2, O3, US/European AQI)
- **Elevation**: Open-Elevation API (elevation in meters, terrain calculation)
- **Indian Seismic Zoning**: Calculation of Seismic Zone (II, III, IV, V) based on coordinates according to IS 1893:2016
- **Data Integrity & Provenance**: Every retrieved factor has `source`, `dataset`, `date_retrieved`, `distance_km`, `confidence` (`High`, `Medium`, `Low`).
- If an external API is down or times out, flags factor as `"Unable to retrieve this dataset at this time."`

#### [NEW] `server/src/services/aiScreeningAgent.ts`
- **EcoImpact Environmental Screening Agent**:
  - Connects to Gemini API via `@google/genai` or direct API call using `GEMINI_API_KEY`
  - Integrated with India EIA Notification 2006 classification logic (Category A requiring MoEFCC Central Clearance, Category B1 requiring State EIA / SEAC, Category B2 screened out / EMP)
  - Multi-category reasoning: Air, Water, Soil/Land, Noise, Ecology, Waste, Traffic, Climate, Social Receptors
  - Explainable risk engine: combines likelihood (1-5), severity (1-5), exposure (1-5), and data confidence to compute transparent factor scores and overall 0-100 score
  - High-risk explainability: explicitly details *Why it is high risk*, *Evidence*, *Data source*, *Mitigation*, *Confidence*
  - Distinguishes AI advice from legal compliance requirements

#### [NEW] `server/src/services/scenarioService.ts`
- Recalculates risk factors dynamically when user alters variables: project area, water consumption, workforce, traffic, green area %, renewables %, emission controls.
- Returns delta diff comparing Baseline vs Scenario.

#### [NEW] `server/src/services/reportService.ts`
- Generates a comprehensive, professional PDF Environmental Screening Report with:
  1. EcoImpact Cover Page with official badge & disclaimer
  2. Assessment Metadata & Project Identification
  3. Spatial Location & Geocoordinates
  4. Environmental Baseline & Proximity Table
  5. Potential Impacts by Category
  6. Explainable Risk Scorecard & Methodology
  7. AI Screening Agent Reasoning & Findings
  8. Prioritized Mitigation Recommendations (AI vs Statutory)
  9. What-If Scenario Comparison Matrix
  10. Data Provenance & Limitations Disclosure
  11. Final Screening Conclusion & Prominent Statutory Legal Disclaimer

#### [NEW] `server/src/services/realtimeGateway.ts`
- WebSocket server managing live connections from Admin Dashboards
- Broadcasts real-time events: `user_registered`, `assessment_created`, `location_confirmed`, `analysis_completed`, `report_generated`

#### [NEW] `server/src/routes/*.ts`
- `authRoutes.ts`: `/api/auth/google`, `/api/auth/verify-human`, `/api/auth/admin-login`, `/api/auth/me`
- `assessmentRoutes.ts`: CRUD for assessments, project scale, location
- `environmentalRoutes.ts`: `/api/environmental/analyze` (triggers pipeline)
- `aiRoutes.ts`: `/api/ai/screen`
- `scenarioRoutes.ts`: `/api/scenarios/simulate`
- `reportRoutes.ts`: `/api/reports/generate`, `/api/reports/download/:id`
- `adminRoutes.ts`: `/api/admin/metrics`, `/api/admin/users`, `/api/admin/assessments`, `/api/admin/rules`, `/api/admin/audit-logs`
- `configRoutes.ts`: `/api/config/status` (checks connected API keys and provides setup guides)

---

### 2. Frontend Client (`/client`)

A single high-performance Vite + React + TypeScript web application hosting both the **User App** and the **Admin Dashboard** with seamless client-side routing, protected routes, and shared design tokens.

#### [NEW] `client/package.json` & Configuration
- React 18 / Vite / TypeScript
- Lucide React (environmental & dashboard icons)
- Leaflet / React-Leaflet / Google Maps loader (interactive spatial map with pin dropping, GPS capture, search)
- Canvas-confetti / Recharts (radar charts, gauge charts, risk bars)
- Custom CSS design system with nature-tech palette:
  - Primary: Deep Forest (`#064e3b`), Emerald (`#059669`), Mint (`#34d399`)
  - Dark Mode & Glassmorphism: Slate (`#0f172a`), Deep Indigo (`#1e293b`), translucent glass cards
  - Risk Badges: Low (Emerald), Moderate (Amber), High (Orange), Critical/Very High (Rose)
  - Modern typography: Inter / Outfit via Google Fonts

#### [NEW] `client/src/context/AuthContext.tsx`
- Manages user session, Google Identity Services initialization, Google Login modal, Sandbox test accounts, account metadata, and sign out.
- Anti-bot human verification modal integration before creating assessments.

#### [NEW] `client/src/context/AdminAuthContext.tsx`
- Dedicated session management for EcoImpact Admin with role verification (`SUPER_ADMIN`, `ADMIN`, `REVIEWER`).
- Automatic session timeout and logout.

#### [NEW] `client/src/components/common/*`
- `Header.tsx`: Brand logo, tagline "Assess Before You Build", navigation links, live backend health pill, profile avatar
- `DisclaimerBanner.tsx`: High-visibility statutory EIA disclaimer required on all screening pages
- `HumanVerificationModal.tsx`: Anti-bot challenge (Google reCAPTCHA / interactive proof-of-work)
- `GoogleAuthButton.tsx`: Official Google Identity Services integration with fallback sandbox button
- `ApiConfigModal.tsx`: Transparent guide showing active vs unconfigured APIs (Google Maps, Gemini AI, OSM, Open-Meteo)

#### [NEW] `client/src/pages/user/*`
- `HomePage.tsx`: Hero section, EIA decision-support value proposition, interactive sample preview, 4-step workflow, statutory compliance notice
- `NewAssessmentWizard.tsx`: 7-Step guided wizard:
  1. **PROJECT**: Name, category (Residential, Industrial, Mining, Highway, Data Center, Solar/Wind, Hospital, etc.), developer, dates
  2. **DETAILS**: Dynamic scale inputs based on category (Area, built-up, water demand, power, fuel, workers, traffic, raw materials)
  3. **LOCATION**:
     - Option A: GPS / Current location with permission request
     - Option B: Interactive Map with place search, pan, zoom, draggable pin
     - Option C: Manual Latitude/Longitude validation
     - Permanent location confirmation & address display
  4. **ENVIRONMENT**: Radar scanning animation querying real live environmental data; transparent cards showing distances to rivers, forests, schools, elevation, air quality, seismic hazard
  5. **ANALYSIS**: AI Environmental Screening Agent reasoning; 9-factor radar chart; explainable risk engine score (0-100); India EIA Category classification
  6. **SCENARIOS**: What-if interactive sandbox (sliders to alter scale, water, traffic, green belt; real-time risk delta computation)
  7. **RESULTS & REPORT**: Comprehensive results dashboard; prioritized mitigations (AI vs Statutory); 1-click professional PDF report generator & immediate download
- `MyAssessmentsPage.tsx`: Grid/table of user assessments with status badges, risk levels, and direct links to view or download reports
- `ReportsPage.tsx`: Repository of generated PDF reports with previewer and download
- `ProfilePage.tsx`: Profile photo, Google email, account creation date, assessment counts, privacy notice, data deletion request option
- `HelpAboutPage.tsx`: Explains EIA screening methodology, data sources, Indian EIA 2006 framework, API configuration instructions

#### [NEW] `client/src/pages/admin/*`
- `AdminLoginPage.tsx`: High-security portal with role selection / credentials
- `AdminDashboard.tsx`:
  - KPI counters: Total Users, Active Users, Total Assessments, Generated Reports, High Risk Alerts
  - Live Real-time Activity Feed (connected via WebSocket)
  - Risk Distribution chart & Project Category breakdown
- `AdminAssessments.tsx`: Searchable, filterable list of all user assessments, interactive spatial inspector, AI risk breakdown viewer
- `AdminUsers.tsx`: View registered users, Google emails, assessment counts, account status (active/suspended) - **NEVER displays passwords**
- `AdminReports.tsx`: All generated PDF reports with download, verification, and archive tools
- `AdminRulesConfig.tsx`: Interactive editor for screening parameters (e.g. eco-sensitive buffer distances, risk weights, category sensitivity)
- `AdminAuditLogs.tsx`: Complete audit trail with timestamps, admin ID, actions, targets, and IP addresses

---

## Verification Plan

### Automated Build & Execution Verification
1. Verify backend installs cleanly:
   ```bash
   npm.cmd --prefix server install
   ```
2. Verify backend TypeScript compilation:
   ```bash
   npm.cmd --prefix server run build (or typecheck)
   ```
3. Verify frontend installs cleanly:
   ```bash
   npm.cmd --prefix client install
   ```
4. Verify frontend TypeScript & Vite build:
   ```bash
   npm.cmd --prefix client run build
   ```

### Functional End-to-End Verification
1. **Authentication Flow**:
   - Verify Google OAuth / Sandbox login flow succeeds.
   - Verify human verification / anti-bot challenge is enforced.
   - Verify user metadata is recorded and no password field exists in user schema.
2. **Assessment Creation & Location Flow**:
   - Create a project (e.g. "Greenfield Industrial Park, Bengaluru").
   - Test Location Option A (GPS), Option B (Map Pin dropping), Option C (Manual coordinates 12.9716° N, 77.5946° E).
   - Verify coordinates are validated and location confirmation is locked.
3. **Environmental Context Pipeline**:
   - Verify real external queries execute for water bodies, protected areas, schools, elevation, air quality, seismic zone.
   - Verify data source provenance, distances, and confidence levels are rendered.
   - Verify graceful fallback message if an external service is unavailable.
4. **AI Screening Agent & Risk Engine**:
   - Verify 9-factor scores and overall risk score (0-100) are computed with explainable reasoning.
   - Verify high-risk triggers display evidence and mitigation.
   - Verify Indian EIA 2006 category classification (Category A / B1 / B2).
5. **What-If Scenario Sandbox**:
   - Tweak sliders (e.g. increase green belt to 33%, switch to solar, reduce water demand).
   - Verify real-time risk score reduction and delta indicators.
6. **PDF Report Generation**:
   - Click "Generate PDF Report".
   - Verify generated PDF includes all 17+ sections, cover page, maps, risk scores, mitigations, and mandatory statutory disclaimer.
   - Verify PDF is saved to user account and downloaded to browser.
7. **Admin Dashboard & Real-Time Sync**:
   - Open Admin Dashboard in parallel.
   - Verify live WebSocket notification fires when User creates assessment / generates report.
   - Verify Admin can view assessment details, download report, manage users, edit screening rules, and inspect audit logs.
8. **Browser Subagent Visual & Functional Testing**:
   - Use browser subagent to interactively click through the User wizard, generate a report, and verify the Admin dashboard.
