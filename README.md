# RailSync AI

Maintenance block planning for Indian Railways that keeps working after the plan is made.

Smart India Hackathon 2026 · Problem Statement SIH26027 · Ministry of Railways
Team Spirit2.4 · Team ID 166371

---

## The short version

Three departments (Engineering, S&T, TRD) ask for track time on the same section. Trains want the same section. Someone has to decide who gets which window, and today that decision is largely manual.

Most solutions to this problem stop once a schedule has been produced. RailSync AI starts there. It generates and validates a block plan, hands it to a human planner for approval, sends the work to field crew on a mobile app, and then uses what actually happens on the ground (real durations, real incidents) to correct the plan and to make the next one better.

It is a decision-support layer. It does not replace RBS, BDMS, TMS, SMMS, TDMS or COA, and it never authorizes a block on its own.

---

## Contents

1. [A worked example](#a-worked-example)
2. [How the pieces fit](#how-the-pieces-fit)
3. [The maths behind it](#the-maths-behind-it)
4. [Weather policy](#weather-policy)
5. [Design decisions and why](#design-decisions-and-why)
6. [The two applications](#the-two-applications)
7. [Data](#data)
8. [Project status](#project-status)
9. [Known limitations](#known-limitations)
10. [Running it](#running-it)
11. [Repository layout](#repository-layout)
12. [Glossary](#glossary)
13. [References](#references)

---

## A worked example

All numbers below are synthetic and chosen to make the logic easy to follow. They are not Railway data.

**The situation.** Section A–B has one candidate window on Tuesday, 10:00 to 13:00 (180 minutes), with a lighter-traffic window later that evening. Three requests are waiting:

| Task | Dept | Needs | Priority score |
|---|---|---|---|
| ENG-042 sleeper replacement | Engineering | 120 min | 8.6 |
| TRD-031 OHE inspection | TRD | 60 min | 7.9 |
| SNT-017 signal cable check | S&T | 45 min | 6.2 |
| ENG-055 vegetation clearing | Engineering | 40 min | 4.1 |

**What a department-by-department process tends to do.** Each department books its own block. That is three or four separate possessions on one section in the same week.

**What RailSync does, step by step.**

1. Priority ranking orders the four tasks and records why each scored what it did.
2. The compatibility engine checks every pair. ENG-042 and TRD-031 can share a block. ENG-042 and SNT-017 cannot, because of an isolation conflict, so the engine says so and names the reason.
3. The impact simulator overlays the Tuesday window on the timetable. A 10:00 to 13:00 block clips one passenger service. The evening window clips none.
4. The optimizer weighs those trade-offs. It bundles ENG-042 and TRD-031 into one block (120 minutes, since they run in parallel), which leaves 60 minutes of granted time unused.
5. Opportunistic fill looks at that gap. SNT-017 is incompatible, but ENG-055 (40 minutes) is compatible and fits with a buffer, so it is offered as an addition. Utilization for that block goes from 120/180 to 160/180.
6. SNT-017 is not dropped. It goes into the evening window with a stated reason: incompatible with the morning bundle, no conflicting trains in the evening.
7. The validator re-checks the whole plan independently. It passes.
8. The planner sees the plan, the reasons, and the resilience score, and approves it.

**Then reality intervenes.** During execution the TRD crew reports an equipment fault and the inspection will overrun by 25 minutes. The system finds the affected tasks, freezes what is already complete, re-solves the remaining horizon, and shows the planner exactly what changed before asking for re-approval. When the work is finally closed out, the actual duration is fed back into the estimate for TRD OHE inspections.

That last part is what the rest of the design is built around.

---

## How the pieces fit

```
   Adapters (synthetic, source-labelled)
   BDMS/RBS · TMS · SMMS · TDMS · COA · weather
                     │
                     ▼
   ┌──────────────────────────────────────────────┐
   │                Backend (FastAPI)              │
   │                                               │
   │  Priority ─► Compatibility ─► Impact sim      │
   │                                  │            │
   │                                  ▼            │
   │              Weather policy ─► CP-SAT         │
   │                                  │            │
   │                        Resilience testing     │
   │                                  │            │
   │                     Independent validator     │
   └──────────────────────────────────┬───────────┘
                                      ▼
                            Planner approval (human)
                                      │
                    ┌─────────────────┴────────────────┐
                    ▼                                  ▲
            Crew mobile app ──► execution log ──► duration learning
                    │                                  │
                    └──► incident ──► frozen-horizon replan
```

The validator is a separate code path from the optimizer on purpose. If the solver has a bug, the validator should still catch a bad plan. The compatibility engine and validator use no machine learning at all.

---

## The maths behind it

### Priority score

The cold-start score is a weighted sum. It works with zero training data, which matters because there is no real labelled history to train on.

```
priority = 0.35 · criticality
         + 0.30 · urgency
         + 0.20 · failure_risk
         + 0.15 · pending_duration        (each input scaled 0–10)
```

The weights are configuration, not truth. Once enough (synthetic) history exists, an XGBoost model can take over the ranking, and SHAP values explain each prediction. The API always reports which method produced a score (`rule_based_fallback` or `xgboost`), so it is never ambiguous.

XGBoost here is trained on synthetic data and is described that way everywhere. It is a ranking aid, not a validated failure predictor.

### Optimization

Sets: tasks *i*, block windows *j*, train events *q*.
Main variables: `X[i,j]` (task in block), `C[i]` (task completed), `D[q]` (delay on train event), `O[i]` (overrun exposure).

```
maximize
    Σ priority_i · C_i
  − λ1 · Σ D_q                          train delay
  + λ2 · Σ (productive_j / granted_j)   possession utilization
  − λ3 · Σ scenario_penalty_s           resilience
  − λ4 · Σ O_i                          overrun risk
  − λ5 · Σ plan_change_i                stability (replans only)
```

Hard constraints include: a task must fit inside its window, must sit on the right section, cannot share a block with an incompatible task, cannot exceed crew or equipment capacity, must respect predecessors, and cannot be assigned to a weather-prohibited slot. Mandatory emergency work is either scheduled or explicitly escalated. It is never silently dropped.

The solver runs under a time limit and always reports its status (`OPTIMAL`, `FEASIBLE`, `INFEASIBLE`, or time limit reached). A `FEASIBLE` answer is not called optimal.

### Utilization has two meanings

| Metric | Definition |
|---|---|
| Occupied utilization | time the block was in use ÷ time granted |
| Productive utilization | time spent on actual maintenance work ÷ time granted |

A crew waiting on power isolation occupies a block without being productive. Reporting only the first number flatters the plan, so the dashboard shows both.

### Duration learning

```
new_estimate = 0.3 · actual + 0.7 · old_estimate
```

Applied per task type after each completed execution. The 0.3 is a tunable smoothing factor. The dashboard reports how far estimates have moved and the error before and after.

### Resilience score

Each generated plan is re-solved against 20 to 50 sampled disruptions (train delay, crew delay, overrun, block cancellation, weather change, urgent defect), with approved work frozen. The score is:

```
resilience = 100 − (unmet_critical_penalty + train_impact_penalty + recovery_penalty)
```

This is a product metric for comparing plans. It is not a probability of safe operation and is never presented as one.

### Replanning

On an incident: find affected tasks by section and time overlap, freeze completed and active work plus anything inside a lockout horizon, re-solve the rest with the previous plan supplied as a hint, and penalize unnecessary changes to assignments crews have already been told about. The planner gets a diff, not just a new plan.

---

## Weather policy

Weather is handled as a configurable policy table with four outcomes per task type and condition:

| Outcome | Effect |
|---|---|
| Prohibited | hard exclusion in the solver |
| Discouraged | soft penalty |
| Acceptable | no effect |
| Unknown | flagged for planner review |

Any row marked Prohibited must carry a source citation, and the UI shows that citation next to the exclusion. The fog and tempestuous-weather restriction is seeded from Rule 15.07 of the Indian Railways General & Subsidiary Rules.

Rows beyond what a cited rule directly supports are team defaults for demonstration. They are labelled that way in the config and would need review by Railway domain staff before any real use. The repo does not claim a single universal rainfall threshold, because a blanket number would not be defensible.

---

## Design decisions and why

**CP-SAT rather than a MILP solver.** Much of the academic literature uses MILP, and that is a fine choice. CP-SAT handles optional intervals, no-overlap constraints, and logical conditions more naturally, is free, and is easier to demonstrate interactively. It is not a machine-learning method and is not described as one.

**A deterministic impact simulator rather than a GNN.** Graph neural networks are well studied for delay propagation, but they need large volumes of real historical train-event data. Trained on synthetic data, a GNN would mostly learn the assumptions of the generator. The event-based simulator can show its reasoning: this train is affected because its section entry overlaps the block, and the delay reaches the next station because its departure depends on that arrival. A GNN module may exist in the repo as an experiment, but it does not drive any figure the application shows.

**Rules, not ML, for compatibility and validation.** Whether two tasks can safely share a block is a safety question. It should be reproducible, inspectable and auditable, so it is written as explicit rules.

**A hybrid priority engine.** Rules give a working system on day one. A learned model can improve on them when data justifies it. Neither is presented as more than it is.

**Frozen horizon during replanning.** Rebuilding everything after each incident would reshuffle crews unnecessarily. Freezing committed work and penalizing change keeps replans usable.

**Adapters instead of pretend integrations.** Real CRIS interfaces are not publicly documented and are not accessible to us. Mock adapters return synthetic records with `source_system`, `source_record_id` and `data_quality` fields. Swapping in a real source later means implementing the same contract.

---

## The two applications

### Planner dashboard (web)

Used by controllers, department engineers and planning officers.

- Priority queue with per-task explanations
- Corridor view with block windows, conflicts and weather overlay
- Weekly and monthly block planner
- Optimization panel: objective value, solver status, bundles, utilization
- Resilience panel: scenario results and score breakdown
- Rejected tasks with a specific reason for each
- Live feed of check-ins and incident reports from the field
- Approval screen recording who approved which plan version, and under which data-quality labels

### Crew app (mobile, offline-first)

Used by gangmen, trackmen, linesmen, patrolmen and inspectors.

- Today's assigned blocks
- Check-in and check-out with time and location
- Incident report with category, photo and location
- Weather and safety alerts for assigned work
- Actions queue locally and sync when the network returns

The app is deliberately narrow. Railways and enterprise vendors already have field apps, so the point of this one is the connection: what a crew member does on the track changes what the planner sees and what the next plan looks like.

---

## Data

Everything in the prototype is synthetic. It models one representative corridor, in the region of 10 to 20 stations, 20 to 40 sections, 50 to 150 maintenance tasks and a few hundred train movements across Engineering, S&T and TRD.

Every record carries provenance:

```json
{
  "source_system": "TMS",
  "source_record_id": "TMS-00421",
  "ingested_at": "2026-01-15T08:00:00Z",
  "data_quality": "synthetic"
}
```

Any number shown in the application, in documents or in a demo that comes from this data is labelled a simulation result. None of it is Indian Railways operational data.

One figure in the project is not synthetic. The CAG Report No. 45 of 2018 audited a sample of 25 track machines across selected sections of five zonal railways in 2016 to 2017 and found that, on average, 33% of the block time available to those machines could not be utilized. That is a finding about that sample and is cited as such. It motivates the utilization objective and is not extrapolated to the whole network.

---

## Project status

Update this table as work lands. It is here so that a reader can tell what exists from what is planned.

| Area | Status |
|---|---|
| Problem analysis, PRD, SRS, architecture, schema, API spec | Written |
| Synthetic corridor dataset generator | Planned |
| Mock adapters with provenance | Planned |
| Priority engine (rules, then XGBoost + SHAP) | Planned |
| Compatibility engine | Planned |
| Impact simulator | Planned |
| CP-SAT optimizer | Planned |
| Weather policy engine | Planned |
| Independent validator | Planned |
| Planner dashboard | Planned |
| Crew app (check-in/out, incident report) | Planned |
| Incident-triggered replanning | Planned |
| Duration learning loop | Planned |
| Resilience scoring | Planned |
| Weekly and monthly horizons | Planned |

Build order follows the priority tiers in the PRD. Nothing from a later tier starts before the current tier works end to end.

Out of scope for this prototype: live Railway integration, autonomous block authorization, reinforcement learning, a full digital twin, computer-vision defect detection, an LLM-based scheduler, and a tamper-evident audit ledger.

---

## Known limitations

- The data is synthetic, so results show that the method works, not how much it would help on a real corridor.
- The XGBoost model has no real failure history behind it.
- Weather rows not backed by a cited rule are defaults awaiting domain review.
- CP-SAT solve time grows with problem size. The prototype targets a bounded corridor, and larger networks would need decomposition.
- The resilience score depends on the scenario distributions chosen, and those are configured, not measured.
- Public documentation of RBS, TMS, SMMS, TDMS and COA is limited. The adapters reflect our understanding of what those systems hold, not their actual schemas.
- Real deployment would need Railway validation, approved data access and safety review. None of that has happened.

---

## Running it

Requirements: Python 3.11+, Node 18+, PostgreSQL 15+, Docker optional.

```bash
git clone https://github.com/MaheshRPawar/RailSync-AI.git
cd RailSync-AI

# backend
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload

# dashboard
cd ../frontend/dashboard
npm install && npm run dev

# crew app
cd ../../mobile/crew-app
npm install && npx react-native start
```

Or:

```bash
docker-compose up --build
```

Copy `.env.example` to `.env` first. Synthetic weather is the default. Live weather is opt-in and off unless you set it.

---

## Repository layout

```
RailSync-AI/
├── docs/                 PRD, SRS, architecture, schema, API spec, references
├── backend/
│   ├── app/
│   │   ├── priority_engine/
│   │   ├── compatibility_engine/
│   │   ├── impact_simulator/
│   │   ├── optimizer/
│   │   ├── weather_policy/
│   │   ├── validator/
│   │   ├── resilience/
│   │   └── incidents/
│   ├── adapters/
│   └── tests/
├── frontend/dashboard/
├── mobile/crew-app/
├── data/synthetic/
├── docker-compose.yml
└── .env.example
```

---

## Glossary

| Term | Meaning |
|---|---|
| Block / possession | A window in which a section is closed to trains so work can be done |
| BDMS | Departmental block and disconnection demand workflow |
| RBS | Rolling Block System, the CRIS platform for cross-department block planning |
| TMS / SMMS / TDMS | Track, signalling and traction distribution management systems |
| COA | Control Office Application, holds control charts and corridor information |
| TRD | Traction Distribution (overhead equipment and power) |
| S&T | Signal and Telecommunication |
| PWI | Permanent Way Inspector |
| Shadow block / bundle | Several compatible tasks sharing one block |
| CP-SAT | Constraint-programming solver from Google OR-Tools |
| EMA | Exponential moving average |

---

## References

Academic
- Zhang, D'Ariano, He, Peng (2019). Microscopic optimization model and algorithm for integrating train timetabling and track maintenance task scheduling. *Transportation Research Part B*. doi:10.1016/j.trb.2019.07.010
- Zhang, Lusby, Shang, Zhu (2020). Simultaneously re-optimizing timetables and platform schedules under planned track maintenance for a high-speed railway network. *Transportation Research Part C*, 121, 102823.
- Dao, Basten, Hartmann (2018). Maintenance scheduling for railway tracks under limited possession time. *Journal of Transportation Engineering, Part A*. doi:10.1061/JTEPBS.0000163
- Famurewa, Kumar. Scheduling of railway infrastructure maintenance tasks using train free windows.
- Yan et al. (2026). Coordinative optimization strategy for group track maintenance planning and train scheduling. *Reliability Engineering and System Safety*.
- Huang et al. (2024). Data science in transportation networks with graph neural networks: a review. *Data Science for Transportation*.

Official
- Indian Railways General & Subsidiary Rules, Rule 15.07
- Railway Board: Rolling Block Programme and Rolling Block System implementation letter (2025)
- CAG Report No. 45 of 2018, utilisation of resources and infrastructure for track maintenance
- RDSO documentation on SMMS

Full details, with links, are in `docs/`.

---

## Team

Spirit2.4 · Smart India Hackathon 2026

## License

MIT. See `LICENSE`.

## Disclaimer

Every dataset, metric and output in this repository is a synthetic simulation produced for prototyping and evaluation. None of it represents Indian Railways operational data or performance.
