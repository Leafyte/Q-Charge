# Q-Charge

**Quantum-assisted EV charging optimization platform.**
Q-Charge schedules charging for multiple electric vehicles at a shared station so that energy cost and peak demand are reduced while each driver's energy target and departure deadline are met. It includes a classical optimization baseline and a small, carefully controlled QUBO / QAOA experiment with a quantum machine learning (QML) warm-start component.

> **Status:** planning / early development. Sections marked *(planned)* describe intended behaviour, not shipped features.
>
> **Honesty note:** the quantum part is an experimental comparison, not a claim of quantum advantage. Every schedule, classical or quantum, must pass an independent validator before it is called feasible. Results in this repo must come from runs that were actually executed.

---

## Table of contents

1. [Goals and non-goals](#1-goals-and-non-goals)
2. [Product overview](#2-product-overview)
3. [Architecture](#3-architecture)
4. [Repository layout](#4-repository-layout)
5. [Tech stack](#5-tech-stack)
6. [Problem formulation](#6-problem-formulation)
7. [Quantum pathway: QUBO, QAOA, QML warm-start](#7-quantum-pathway-qubo-qaoa-qml-warm-start)
8. [Evaluation and benchmarks](#8-evaluation-and-benchmarks)
9. [Data model](#9-data-model)
10. [API (planned)](#10-api-planned)
11. [Getting started](#11-getting-started)
12. [Configuration](#12-configuration)
13. [Testing](#13-testing)
14. [Deployment](#14-deployment)
15. [Security and privacy](#15-security-and-privacy)
16. [Roadmap](#16-roadmap)
17. [Risks and limitations](#17-risks-and-limitations)
18. [Success criteria](#18-success-criteria)
19. [References](#19-references)
20. [Team and license](#20-team-and-license)

---

## 1. Goals and non-goals

**Goals**

- Generate feasible charging schedules for several EVs sharing one station.
- Reduce charging cost by shifting flexible demand to cheaper time slots.
- Limit peak station demand without breaking hard electrical constraints.
- Show operators and drivers understandable schedules, costs and metrics.
- Compare FCFS, exact classical, simulated annealing, QAOA and QAOA+QML on identical instances.
- Keep the architecture extensible (multiple stations, renewable forecasts, later OCPP integration).

**Non-goals (initial release)**

- Controlling real chargers or implementing OCPP.
- Payment processing.
- Utility-grade grid control.
- Any claim of quantum advantage.

---

## 2. Product overview

Two clients share one backend. Business rules live only in the backend.

| Client | Users | Main functions |
|---|---|---|
| **Operator dashboard** (web) | Station operators | Station and charger config, EV queue, run schedules with a chosen solver, timeline view, solver comparison, metrics, CSV export |
| **Driver view** (mobile-responsive page, native app optional later) | EV drivers | Vehicle profile, submit request (arrival, departure, current/target SoC), see assigned window and estimated cost, status |

The driver view never computes schedules. It displays the validated result returned by the backend.

---

## 3. Architecture

```
 Operator Web (React)        Driver View (responsive React)
          \                         /
           \                       /
            FastAPI backend (auth, REST API, job creation)
                         |
        +----------------+-----------------+
        |                                  |
   PostgreSQL                         Worker (polls jobs table)
 (users, requests,                         |
  jobs, schedules,            +------------+-------------+
  experiment results)         |            |             |
                        Classical      QUBO + SA     QAOA / QAOA+QML
                      (FCFS, EDF,                    (Qiskit / PennyLane)
                       LP/MILP)
                              \            |             /
                               Schedule validator + metrics
```

Key rules:

- All solver outputs go through the **same validator** before metrics are stored.
- Long solver runs are **jobs** (`queued → running → completed | failed | infeasible`), never blocking API requests.
- Timestamps are stored in UTC and shown in the station's local time zone.

---

## 4. Repository layout

```
q-charge/
  services/
    api/            # FastAPI routes, auth, domain services
    optimizer/      # domain models, validator, solvers, qubo, qaoa, qml
    worker/         # job runner
  apps/
    operator-web/   # React + Vite + TS (includes the responsive /driver route)
  experiments/      # scripts, configs, saved results, notebooks
  migrations/       # Alembic
  tests/
    unit/
    optimization/
    integration/
  docker-compose.yml
  requirements.txt
  README.md
```

---

## 5. Tech stack

| Layer | Technology |
|---|---|
| Web frontend | React, Vite, TypeScript, Tailwind CSS, Recharts |
| Backend | Python 3.11, FastAPI, Pydantic |
| Database | PostgreSQL (SQLite for early development), SQLAlchemy, Alembic |
| Classical optimization | SciPy (`milp`) and/or OR-Tools |
| Quantum optimization | Qiskit + Aer (simulator) |
| QML | PennyLane + PyTorch (or Qiskit Machine Learning) |
| Analysis | NumPy, Pandas, Matplotlib |
| Testing | pytest, frontend component tests |
| Deployment | Docker; static frontend host, Python API host, managed Postgres |

---

## 6. Problem formulation

### 6.1 Two problem tiers

| Tier | Variables | Solved by | Purpose |
|---|---|---|---|
| **Full** | continuous power `p[i,t]` | LP/MILP, FCFS, EDF | The real product scheduler |
| **Reduced** | binary `x[i,t]` (EV `i` charges in slot `t` at fixed power) | brute force, exact solver, simulated annealing, QAOA, QAOA+QML | Fair quantum vs classical comparison |

Continuous-power scheduling is a linear program that classical solvers handle easily. QUBO/QAOA only applies to the binary reduced tier, so the quantum study is a *comparison*, and classical solvers are expected to win at the sizes a simulator can handle.

### 6.2 Conventions (do not change after they are set)

- Slot length `Δt = 0.25 h` (15 minutes); full tier horizon 24 h; reduced tier 4-5 slots.
- Battery-side energy need: `E_need = (target_soc − current_soc) × battery_capacity_kwh`.
- Grid-side delivery per slot is `p × Δt`; the energy constraint is `Σ_t p[i,t]·Δt ≥ E_need / η`. Apply efficiency `η` once, never twice.
- Slot `t` covers `[t0 + t·Δt, t0 + (t+1)·Δt)` in UTC.

### 6.3 Objective (full tier)

```
min  λc · Σ_t c_t · Δt · Σ_i p[i,t]   +   λp · P_peak   +   λu · Φ_user
```

`P_peak` is the maximum aggregate power, `Φ_user` is an explicit soft penalty for unmet demand. Hard constraints are never replaced by soft penalties. Weights `λc, λp, λu` are documented and tested for sensitivity.

### 6.4 Hard constraints

- **Availability:** `p[i,t] = 0` outside EV `i`'s arrival-departure window.
- **Rate limit:** `0 ≤ p[i,t] ≤ P_max[i]`.
- **Station limit:** `Σ_i p[i,t] ≤ P_grid` for every slot.
- **Chargers:** no more simultaneous sessions than compatible available chargers.
- **Energy and deadline:** each feasible request receives its required energy before departure. Physically impossible requests return `infeasible` with a reason (or a clearly labelled best-effort result).

### 6.5 Classical baselines

1. FCFS (operational reference)
2. Earliest-deadline-first heuristic
3. LP/MILP (exact for the full tier)
4. Brute force (tiny reduced instances)
5. Simulated annealing on the same QUBO (separates "encoding is hard" from "QAOA is weak")

### 6.6 Validator

Independent module that checks windows, per-EV power, station limit, charger count and delivered energy. A schedule is labelled feasible **only** if the validator accepts it.

---

## 7. Quantum pathway: QUBO, QAOA, QML warm-start

### 7.1 QUBO construction (reduced tier)

Binary variables `x[i,t]` for slots inside each EV's window. Energy terms:

```
E(x) = E_cost + λp·E_peak + A·E_energy + B·E_capacity

E_cost     = Σ_i Σ_t  c_t · Δt · P_i · x[i,t]                 # linear (diagonal)
E_peak     = Σ_t ( Σ_i P_i · x[i,t] )²                         # quadratic, flattens load
E_energy   = Σ_i ( Σ_t x[i,t] − k_i )²                         # k_i = slots EV i needs
E_capacity = Σ_t ( Σ_i x[i,t] + s_t − M )²                     # s_t = binary slack, M = max concurrent EVs
```

Notes:

- `max` is not quadratic, so peak demand is modelled as sum of squared slot loads.
- Inequality constraints need slack bits, which add qubits. Budget about 20 qubits or fewer for simulation.
- Charger-assignment variables are dropped in the quantum tier; chargers become a concurrency limit `M`.
- Normalise the QUBO by its largest absolute coefficient before QAOA.
- Verify the QUBO against brute force on every tiny instance: the lowest-energy bitstring must match the exact solver.
- Run a penalty sweep over `A` and `B` and record when infeasible bitstrings begin to score better than feasible ones.

### 7.2 QAOA

- Convert QUBO to an Ising Hamiltonian and build it directly as a `SparsePauliOp`.
- `QAOAAnsatz`, depth `p ∈ {1, 2, 3}`, Aer estimator/sampler, COBYLA, at least 10 seeds.
- Decode sampled bitstrings, validate, and record: variables/qubits, circuit depth, shots, optimizer settings, backend, runtime, best objective, feasibility rate, approximation ratio, optimality gap.
- Optional ablation: slot-count-preserving mixer (XY mixer or Dicke-state start) versus standard mixer with penalties.

### 7.3 QML warm-start (the learning component)

QAOA needs good starting angles `(γ, β)`; finding them takes many slow optimizer iterations. A small **variational quantum circuit** is trained to predict good angles from instance features.

Pipeline:

1. Generate roughly 500-2000 small random instances (3-5 EVs × 4-5 slots, at most about 16-20 qubits) using seeded scenarios, optionally ACN-style arrival patterns and varied tariffs.
2. For each instance, run QAOA from several starts and save the best `(γ, β)` (QUBO normalised first; canonicalise angles to handle symmetries).
3. Features (about 8): normalised price statistics, capacity tightness, window lengths, required slots, QUBO coefficient statistics.
4. Model: angle-encode features onto qubits, add trainable entangling layers, read out `2p` expectation values mapped to `γ ∈ [0, π]`, `β ∈ [0, π/2]`.
5. Training: regression on labels, or (better, costlier) train on the QAOA energy reached with the predicted angles.
6. Integration: optimizer method `qaoa_ml` predicts angles, runs QAOA from them, validates the result, and stores the model version in the experiment record.

Baselines for the warm-start (all required):

- Random initial angles
- One fixed "average good" angle set (often hard to beat)
- Classical model (linear regression or small MLP) with the same inputs and outputs

Metrics on held-out instances: approximation ratio before extra optimization, optimizer iterations to reach 95% of the best value, feasibility rate. If a classical baseline wins, report it as the result.

Fallback if time runs short: a hybrid quantum-classical forecaster for session duration or energy requested, trained on ACN data. It is easier but only loosely tied to the optimizer.

---

## 8. Evaluation and benchmarks

**Metrics:** feasibility rate, energy cost, peak demand, target completion rate, charger utilization, runtime (including setup), objective value, robustness across seeds/tariffs/arrival patterns.

**Experiment design**

1. Fixed benchmark scenarios with documented EV counts, chargers, slots, tariffs and limits.
2. Identical inputs and objective for every method.
3. Multiple seeds for stochastic methods; log every setting and solver version.
4. Validate every candidate schedule before computing comparisons.
5. Report infeasible instances separately.
6. Report limitations, including circuit size and simulation time.

**Benchmark table (to be filled from real runs)**

| Instance (EVs × slots) | Methods | Seeds | Record |
|---|---|---|---|
| 3×4, 3×5, 4×4 | brute force, exact, SA, QAOA p=1/2/3, QAOA+QML | ≥ 10 | feasibility rate, optimality gap, approximation ratio, runtime, qubits, circuit depth |

Also include the penalty-weight sensitivity plot and the warm-start comparison. Never compare a strong classical solver against an intentionally weak quantum baseline.

---

## 9. Data model

| Entity | Key fields |
|---|---|
| User | id, email, password hash, role (`driver`/`operator`), created_at |
| Vehicle | id, owner_id, battery_capacity_kwh, max_charging_kw, connector_type |
| Station | id, name, location, max_grid_power_kw, tariff config, status |
| Charger | id, station_id, max_power_kw, connector_type, status |
| ChargingRequest | id, vehicle_id, station_id, arrival, departure, current_soc, target_soc, status |
| OptimizationJob | id, station_id, method, parameters, seed, status, timestamps, error summary |
| Schedule | id, job_id, request_id, charger_id, time_slot, assigned_power, energy_delivered |
| ExperimentResult | id, job_id, feasibility, total_cost, peak_power, completion_rate, utilization, runtime, solver metadata, model version |

---

## 10. API (planned)

Versioned REST under `/api/v1`. Long jobs return an id and status; clients poll.

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Health check |
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Authenticate |
| GET | `/stations` | List visible stations |
| POST | `/stations` | Create station (operator) |
| GET | `/stations/{id}/chargers` | Charger config and status |
| POST | `/charging-requests` | Submit a request |
| GET | `/charging-requests/{id}` | Request details and status |
| POST | `/optimization-jobs` | Start FCFS / LP / SA / QAOA / QAOA+QML / comparison run |
| GET | `/optimization-jobs/{id}` | Job status and results |
| GET | `/optimization-jobs/{id}/schedule` | Validated schedule |
| GET | `/stations/{id}/metrics` | Cost, peak, utilization, completion |

Job statuses: `queued`, `running`, `completed`, `failed`, `infeasible`.

---

## 11. Getting started

> These commands describe the intended setup *(planned)*. Adjust paths to the actual layout as it is created.

**Prerequisites:** Python 3.11, Node 20+, Docker, Git.

```bash
git clone <your-repo-url> q-charge && cd q-charge

# Python environment
python -m venv .venv && source .venv/bin/activate
pip install fastapi "uvicorn[standard]" sqlalchemy alembic pydantic "psycopg[binary]" \
  pytest numpy pandas scipy ortools matplotlib qiskit qiskit-aer pennylane torch
pip freeze > requirements.txt

# Database
docker compose up -d db
alembic upgrade head

# Backend
uvicorn services.api.main:app --reload

# Worker (separate terminal)
python -m services.worker.main

# Operator web app (separate terminal)
cd apps/operator-web && npm install && npm run dev
```

**Run an experiment** *(planned)*:

```bash
python -m experiments.run_benchmark --config experiments/configs/small.yaml --seeds 10
python -m experiments.generate_qml_dataset --n 1000 --out experiments/data/qml.parquet
python -m experiments.train_qml --data experiments/data/qml.parquet
```

---

## 12. Configuration

Names only. Real values go in your host's secret manager and are never committed.

```
# Backend
DATABASE_URL=postgresql://...
SECRET_KEY=<generated-secret>
CORS_ORIGINS=https://your-dashboard-domain.example

# Web frontend (public value only)
VITE_API_URL=https://your-api-domain.example
```

Anything exposed to a browser bundle is not secret. Keep database credentials and signing keys server-side. Add a `.env.example` and list `.env` in `.gitignore`.

---

## 13. Testing

- **Unit:** SoC-to-energy, efficiency, tariff cost, slot boundaries, charger compatibility, validator.
- **Optimization:** small instances verified by enumeration; impossible deadlines, insufficient station power, overlapping requests, no chargers, zero-energy requests, varying tariffs. Invalid schedules must never be marked feasible. QUBO ground state must equal the exact solution.
- **API / integration:** auth, role permissions, request validation, persistence, job lifecycle, solver failure handling, operator and driver views showing the same stored schedule.
- **Frontend:** loading, empty, invalid, error and infeasible states; metrics come from backend results, never hard-coded.

```bash
pytest tests/unit tests/optimization tests/integration
```

---

## 14. Deployment

Suggested pattern (check each provider's current limits and pricing first):

- Frontend: static host build of `apps/operator-web`.
- API and worker: Python host running FastAPI and the job worker.
- Database: managed PostgreSQL; run migrations on deploy.
- CI: GitHub Actions running tests and builds.

Sequence: create repo and environments → provision Postgres and migrate → deploy API and check `/health` → deploy web app with the API URL → end-to-end test (request → job → validated schedule → both views) → move heavy jobs to the worker if needed → add logs, monitoring and backups.

A local Docker Compose demo is an acceptable fallback if hosting is short on time.

---

## 15. Security and privacy

- HTTPS, role-based authorization, server-side validation of all inputs (time ranges, SoC, power limits, station ids).
- Modern password hashing; no plain-text secrets; rotate any exposed secret.
- ORM / parameterised queries only; no raw exception traces to users.
- Collect only data needed for scheduling; use synthetic or anonymised data in demos.
- Label simulated telemetry and costs clearly.
- The optimizer must never send commands to physical chargers unless a separate, safety-tested hardware layer exists.

---

## 16. Roadmap

About 8 weeks; adjust to your team and deadline.

- [ ] **Phase 0 (days 1-2):** repo, venv, CI, Docker Compose with Postgres
- [ ] **Phase 1 (week 1):** domain models, scenario generator, **validator**, time-slot and energy conventions
- [ ] **Phase 2 (week 2):** FCFS, EDF, LP/MILP, feasibility pre-check, metrics module
- [ ] **Phase 3 (week 3):** reduced problem, QUBO builder, brute-force check, simulated annealing, penalty sweep
- [ ] **Phase 4 (week 4):** QAOA (p = 1-3, multiple seeds), saved reproducible results
- [ ] **Phase 5 (week 5):** QML warm-start: dataset, features, QNN, baselines, evaluation
- [ ] **Phase 6 (week 6):** backend API, auth, job table + worker, persistence, tests
- [ ] **Phase 7 (week 7):** operator dashboard, responsive driver view, error states
- [ ] **Phase 8 (week 8):** end-to-end tests, deployment, benchmark report, demo

**If time gets short, cut in this order:** deployment polish → separate driver page → auth beyond a simple role check → mixer ablation. Never cut the validator, the classical baselines, or the benchmark report.

---

## 17. Risks and limitations

| Risk | Impact | Mitigation |
|---|---|---|
| QUBO size grows fast | QAOA simulation becomes impractical | Small instances, ≤ ~20 qubits, document encoding size |
| Poor penalty weights | Optimizer prefers infeasible schedules | Penalty sweeps, validate every output |
| No quantum advantage observed | Quantum slower or worse | Present as a comparative study |
| QML baseline wins | Warm-start gives no gain | Report honestly; fixed-angle and classical baselines included |
| Infeasible driver requests | Target SoC unreachable | Early detection, clear reason, best-effort option |
| Job timeouts | API blocked | Job records, worker, timeouts |
| Provider limits change | Deployment breaks | Check limits, keep config portable |
| Hardware integration | Safety and protocol complexity | Simulation only for now; OCPP is separate future work |

Known limitations: results come from simulators on small instances; the study does not show or imply quantum advantage; efficiency and tariff models are simplified.

---

## 18. Success criteria

The project is demo-ready when:

- A driver can submit a valid request through the driver view.
- The backend stores it and starts an optimization job.
- The classical solver returns a schedule or a clear infeasibility result.
- QAOA runs on a documented small instance and its output is independently validated.
- The operator dashboard compares solvers on the same scenario and metrics.
- The driver view shows the schedule stored by the backend.
- Failures, invalid input and authorization checks are handled without exposing secrets.
- The report presents reproducible measurements and makes no unsupported quantum-advantage claim.

---

## 19. References

Verify every entry (authors, year, DOI, any corrections) before citing it in a report.

- K. Kea, C. Huot, Y. Han, "Leveraging Knapsack QAOA Approach for Optimal Electric Vehicle Charging," *IEEE Access*, vol. 11, 2023, DOI 10.1109/ACCESS.2023.3320800. Code: https://github.com/QCL-PKNU/QAOA-Knapsack-EV-Charging. A 2025 correction exists; read it.
- C. Si, K. T. Chau, W. Liu, Y. Hou, "Optimal scheduling for electric vehicle charging: A review of methods, technologies, and uncertainty management," *Journal of Energy Storage*, 2025, article 117500.
- Related projects: SAP eMobility Smart Charging (https://github.com/SAP/emobility-smart-charging), ACN-Portal / ACN-Sim (https://github.com/zach401/acnportal), SteVe OCPP server (https://github.com/steve-community/steve), Qiskit Optimization (https://github.com/qiskit-community/qiskit-optimization; check its current status before depending on it).
- Review each repository's license and attribution requirements before reusing any code.

---

## 20. Team and license

- **Team:** [team name and members]
- **License:** [choose a license, e.g. MIT or Apache-2.0, before publishing]
- **Contact:** [email or issue tracker link]
