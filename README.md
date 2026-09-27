# RailSync AI

**Execution-aware maintenance block planning for Indian Railways.**

Smart India Hackathon 2026 — Problem Statement 26027
*AI-Powered Automatic Block Planning to Maximize Asset Availability for Train Operations on Indian Railways*

Team Spirit2.4 · Team ID 166371

---

## What this is, in one paragraph

Indian Railways already has an institutional direction toward coordinated maintenance planning — the 26-week Rolling Block Programme, and more recently a CRIS-built Rolling Block System (RBS) that links block/disconnection demands across Engineering, S&T, and TRD with TMS, TDMS, SMMS, and COA. RailSync AI does not claim to replace any of that. It is an optimization and decision-support layer that sits above these workflows and addresses what they do not: explainable prioritization, deterministic cross-department bundling, constraint-based scheduling that accounts for train impact and weather, resilience testing against disruption scenarios, and — its core differentiator — a closed feedback loop where actual field execution data improves future plans and a real incident triggers automatic re-optimization rather than a manual redo.

## Why this exists

Engineering, S&T, and TRD departments raise maintenance requests independently for shared railway infrastructure while trains are simultaneously scheduled on it. Planned independently, this produces duplicate blocks on the same section, unused or overrun possession time, train-conflict risk, and a growing maintenance backlog. The problem is not the absence of coordination — Indian Railways already pursues that. The gap is in the *quality* of that coordination: whether plans are optimized rather than manually assembled, whether the reasoning behind a decision is visible to the planner, and whether a plan adapts when reality diverges from it instead of remaining static until the next scheduled review.

## What makes this different

Most public prototypes addressing this problem statement converge on the same pattern: a constraint solver, a priority-scoring model, and a dashboard. That pattern is necessary but not sufficient. RailSync AI's contribution sits in four places that pattern typically does not reach:

- **Execution-feedback learning.** Every completed task's actual duration, captured through crew check-in/check-out, updates the estimate used for similar future tasks. The system's planning accuracy is not fixed at build time.
- **Field-triggered replanning.** An incident reported from the crew mobile application — not a simulate-disruption control on a planner's dashboard — identifies affected tasks, freezes already-approved work, and re-solves the remaining schedule.
- **Resilience scoring.** A generated plan is stress-tested against a batch of disruption scenarios (train delay, crew delay, task overrun, block cancellation, weather deterioration, urgent defect) before it is presented as a single robustness figure, rather than shown as one static, untested schedule.
- **Weather as a sourced constraint.** Task-type-to-weather-condition rules are grounded in Indian Railways General & Subsidiary Rules (Rule 15.07, governing work in foggy or tempestuous weather) rather than an invented threshold, and are structured as a configurable policy rather than a hard-coded number.

## What this is not

- Not a replacement for RBS, TMS, COA, SMMS, TDMS, or BDMS.
- Not an autonomous block-authorization system. A human planner approves every plan; the system recommends and validates, it does not authorize.
- Not connected to any live Railway data system. All data in this prototype is synthetic, generated for one representative corridor, and labeled as such throughout the application.
- Not a train-control or signaling safety system.

## System architecture

The system is organized into four zones: an adapter layer that ingests synthetic data shaped like TMS/SMMS/TDMS/COA/BDMS/RBS records with source-provenance metadata; a frontend layer split between a planner web dashboard and a field crew mobile application; a backend layer of FastAPI services covering priority ranking, compatibility checking, impact simulation, optimization, weather policy, validation, resilience testing, and incident handling; and a data and intelligence layer combining a trained priority model, a deterministic cascade simulator, and PostgreSQL storage.

Full architecture diagrams and the complete pipeline are documented in `docs/03_System_Architecture.md`.

Adapters (Zone 1) → Frontend (Zone 2) → Backend Services (Zone 3) → Intelligence & Data (Zone 4)


The planning pipeline itself:

Maintenance Requests → Priority Ranking → Compatibility Check → Impact Simulation
→ CP-SAT Optimization → Resilience Testing → Safety Validation → Planner Approval
→ Crew Execution → Execution Logging → Learning Loop Update


with a rolling-horizon replanning path triggered by field-reported incidents, and a dashed feedback path from execution logging back into priority ranking.

## Technology stack

| Layer | Technology |
|---|---|
| Planner web dashboard | React, TypeScript, Tailwind CSS |
| Field crew mobile application | React Native (offline-first) |
| Corridor and dependency visualization | React Flow, SVG |
| Backend services | Python, FastAPI |
| Database | PostgreSQL |
| Optimization | Google OR-Tools (CP-SAT) |
| Priority model | Scikit-learn, XGBoost, SHAP |
| Cascade impact modeling | NetworkX (deterministic); a GNN module exists separately as an experimental, non-demo-driving component |
| Data processing | Pandas, NumPy |
| Deployment | Docker |

## Scope

Development follows a strict priority order documented in the Product Requirements Document. Nothing in a later tier is attempted before every item in the current tier is demonstrably working.

**P0 — core differentiators, required before anything else:**
execution-feedback learning loop, field-triggered incident replanning, multi-scenario resilience scoring, deterministic compatibility (shadow-block) engine, sourced weather policy engine.

**P1 — built if P0 is complete and stable:**
explainable "why not scheduled" reason codes, possession utilization as a weighted optimization objective, multi-horizon (weekly and monthly) planning.

**P2 — explicitly out of scope for this prototype:**
GNN-driven live predictions, reinforcement learning, a tamper-evident audit ledger, a full digital twin, computer-vision defect detection, an LLM-based scheduling interface, autonomous block authorization, and any live production integration with Railway systems.

## Data and claims

Every dataset used in this prototype is synthetic and generated for a single representative corridor. Every synthetic value shown in the application, in documentation, or in a demonstration is labeled as a simulation result and is never presented as real Indian Railways operational data. Adapter responses carry `source_system`, `source_record_id`, and `data_quality` fields so that provenance is visible at every layer rather than assumed.

## Research basis

This project is built against a defined set of academic and official sources rather than assumed domain knowledge. The complete, cited list — covering integrated train-and-maintenance scheduling literature, opportunistic maintenance research, possession-utilization studies, delay-propagation and graph-based modeling literature, and official Indian Railways rules and audit findings — is maintained in `docs/06_Research_and_References.md`.

## Repository structure
```
railsync-ai/
├── docs/
│ ├── 01_PRD.md
│ ├── 02_SRS.md
│ ├── 03_System_Architecture.md
│ ├── 04_Database_Schema.md
│ ├── 05_API_Specification.md
│ └── 06_Research_and_References.md
├── backend/
│ ├── app/
│ │ ├── priority_engine/
│ │ ├── compatibility_engine/
│ │ ├── impact_simulator/
│ │ ├── optimizer/
│ │ ├── weather_policy/
│ │ ├── validator/
│ │ ├── resilience/
│ │ └── incidents/
│ ├── adapters/
│ └── tests/
├── frontend/
│ └── dashboard/
├── mobile/
│ └── crew-app/
├── data/
│ └── synthetic/
├── docker-compose.yml
├── .env.example
└── README.md

```
## Getting started

Prerequisites: Python 3.11+, Node.js 18+, PostgreSQL 15+, Docker (optional, for containerized setup).

```bash
git clone https://github.com/MaheshRPawar/RailSync-AI.git
cd RailSync-AI

# Backend
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload

# Frontend
cd ../frontend/dashboard
npm install
npm run dev

# Mobile app
cd ../../mobile/crew-app
npm install
npx react-native start
```

Or, with Docker:

```bash
docker-compose up --build
```

Environment variables are documented in `.env.example`.

## Documentation

- Product Requirements Document — `docs/01_PRD.md`
- Software Requirements Specification — `docs/02_SRS.md`
- System Architecture and High-Level Design — `docs/03_System_Architecture.md`
- Database Schema — `docs/04_Database_Schema.md`
- API Specification — `docs/05_API_Specification.md`
- Research and References — `docs/06_Research_and_References.md`

## Team

Spirit2.4 — Smart India Hackathon 2026, Problem Statement 26027.

## License

MIT License. See `LICENSE`.

## Disclaimer

All performance figures, datasets, and demonstration outputs in this repository are synthetic simulation results produced for prototype and evaluation purposes. They are not Indian Railways operational data and should not be interpreted as such.
