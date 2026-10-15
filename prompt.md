You are a senior full-stack + optimization + quantum-ML engineer. Build "Q-Charge", a quantum-assisted EV charging optimization platform, end to end in this repository. Work in phases, commit after each phase, run the tests before each commit, and stop to summarize at each phase boundary.

## 1. Product
Q-Charge schedules charging for multiple EVs at a shared station so that energy cost and peak demand are reduced while each driver's energy target and departure deadline are met. It has:
- An operator web dashboard (React) for station config, EV queue, running schedules with a chosen solver, a schedule timeline, solver comparison and metrics.
- A driver view (mobile-responsive page inside the same React app, route /driver) where a driver submits a charging request and sees the assigned window and estimated cost.
- A shared FastAPI backend with PostgreSQL. Business rules live ONLY in the backend. The driver view never computes schedules; it displays the validated schedule returned by the API.
- An optimizer package with a classical baseline and an experimental QUBO/QAOA pathway plus a QML warm-start component.
Initial release uses simulated stations and synthetic data. No real charger control, no OCPP, no payments.

## 2. Hard rules (never violate)
1. Every schedule, from any solver, must pass an independent validator before it is labelled feasible or has metrics stored. Invalid schedules are never marked feasible.
2. The classical solvers are the reference. Never compare against an intentionally weak baseline. Do not claim quantum advantage anywhere (code, docs, UI).
3. Report infeasible instances separately; never count them as successful runs.
4. Never invent results or metrics. All numbers in reports must come from scripts that actually ran. If something could not be run, say so.
5. Everything stochastic takes a seed. Record seeds, solver settings, library versions and runtime with every result.
6. No secrets in the repo. Use environment variables and a .env.example. Never expose secrets in frontend bundles.
7. The optimizer must never send commands to physical chargers. Label simulated data as simulated in the UI.
8. Use synthetic or anonymised data only.
9. Before using any library API (Qiskit, PennyLane, OR-Tools, SciPy), check the installed version's documentation, because these APIs change. Pin versions in requirements.txt. Note in docs/versions.md anything that differs from your expectations.

## 3. Conventions (define once in docs/conventions.md and in code constants, never change)
- Slot length: 15 minutes (0.25 h). Full tier horizon: 24 h. Reduced tier: 4-5 slots.
- Timestamps stored in UTC; displayed in the station's local time zone. Slot t covers [t0 + t*dt, t0 + (t+1)*dt).
- Battery-side energy need: E_need = (target_soc - current_soc) * battery_capacity_kwh.
- Grid-side energy per slot = p * dt. Energy constraint: sum_t p[i,t]*dt >= E_need[i] / eta. Apply efficiency eta exactly once.
- Two problem tiers:
  FULL tier: continuous power p[i,t], solved by LP/MILP, FCFS, EDF. This is the real product scheduler.
  REDUCED tier: binary x[i,t] (EV i charges in slot t at fixed power P_i), shared by brute force, exact solver, simulated annealing, QAOA and QAOA+QML. This is where the quantum comparison happens.

## 4. Stack
- Frontend: React + Vite + TypeScript + Tailwind + Recharts. Operator pages desktop-first; /driver page mobile-first. Handle loading, empty, invalid, error and infeasible states. Metrics shown come from API results, never hard-coded.
- Backend: Python 3.11, FastAPI, Pydantic, SQLAlchemy, Alembic, PostgreSQL (SQLite allowed in dev), JWT auth with roles operator and driver, modern password hashing.
- Jobs: a jobs table plus a separate worker process that polls it (no Celery/Redis). Statuses: queued, running, completed, failed, infeasible. Timeouts and clear error messages.
- Optimization: SciPy milp and/or OR-Tools for LP/MILP; own FCFS and earliest-deadline-first; Qiskit + Aer for QAOA; PennyLane + PyTorch for the QML model; NumPy/Pandas/Matplotlib; Parquet for cached datasets.
- Infra: Docker Compose (db, api, worker, web), GitHub Actions CI running tests and lint, pytest, optional Playwright.

## 5. Repository layout
q-charge/
  services/api/ (routes, auth, schemas, services)
  services/optimizer/ (domain, validator, metrics, solvers/, qubo/, qaoa/, qml/)
  services/worker/
  apps/operator-web/ (includes /driver route)
  experiments/ (configs/, scripts, data/, results/, notebooks/)
  migrations/
  tests/ (unit/, optimization/, integration/)
  docs/ (conventions.md, architecture.md, formulation.md, experiments.md, limitations.md, versions.md)
  docker-compose.yml, requirements.txt, .env.example, README.md

## 6. Domain, validator, simulator (build first)
- Pydantic/dataclass models: Station, Charger, Vehicle, ChargingRequest, Tariff, Scenario, Schedule.
- Scenario generator: seeded, configurable EV count, chargers, slots, tariff shape, station limit, arrival/departure patterns. Support optional loading of ACN-Data-style session files if I provide them; otherwise synthetic only.
- Validator checks: availability windows, per-EV max power, station power limit per slot, number of simultaneous sessions vs compatible chargers, energy delivered vs need (with eta), deadline. Returns structured violations. Unit-test it with hand-built good and bad schedules.
- Feasibility pre-check: if max_power * window_length * eta < need, return infeasible with a reason before invoking any solver.

## 7. Classical solvers (FULL tier)
- FCFS and earliest-deadline-first baselines.
- LP/MILP minimising lambda_c * energy cost + lambda_p * peak power (+ explicit soft penalty for unmet energy only where the request is allowed to be partial). Hard constraints stay hard. Weights are configurable and documented; add a sensitivity test.
- Metrics module: energy cost, peak demand, energy delivered, target completion rate, charger utilization, runtime, objective value.
- Done when: all solvers produce validated schedules on 20+ seeded scenarios, infeasible cases are detected, and the LP is no worse than FCFS on cost.

## 8. QUBO and baselines (REDUCED tier)
Variables x[i,t] for slots inside each EV's window. Build the QUBO matrix in code:
- Cost (linear, diagonal): sum c_t * dt * P_i * x[i,t].
- Peak: lambda_p * sum_t ( sum_i P_i x[i,t] )^2. Do NOT use a max(); the sum of squared slot loads is quadratic and flattens the load.
- Energy: A * sum_i ( sum_t x[i,t] - k_i )^2, where k_i is the number of slots EV i needs.
- Capacity: B * sum_t ( sum_i x[i,t] + s_t - M )^2 with s_t a binary-encoded slack and M the max concurrent EVs (chargers become a concurrency limit in this tier; drop charger-assignment variables).
- Normalise the QUBO by its max absolute coefficient before QAOA.
Required checks:
- Brute-force every tiny instance; the lowest-energy bitstring of the QUBO must match the exact solver's optimal feasible solution (when penalties are adequate). Unit-test this.
- Implement simulated annealing on the same QUBO as a baseline.
- Run a penalty sweep over A and B and record the point where infeasible bitstrings begin to score better than feasible ones. Save plots.
- Keep qubit counts about 20 or fewer including slack bits; document the encoding size for each instance.

## 9. QAOA
- Convert QUBO to an Ising Hamiltonian and build it directly (for example as a SparsePauliOp). Do not depend on qiskit-optimization unless it is current and maintained in the installed version; document the choice.
- QAOA ansatz with depth p in {1, 2, 3}, Aer simulator, COBYLA (configurable), at least 10 seeds per setting.
- Sample bitstrings, decode to schedules, run the validator, and record: number of variables/qubits, circuit depth, depth p, shots, optimizer and settings, backend, runtime, best objective, feasibility rate, approximation ratio, optimality gap vs the exact solution.
- Optional ablation (only after everything else works): a slot-count-preserving mixer (XY mixer or Dicke-state initial state) vs the standard mixer with penalties.

## 10. QML warm-start (the learning component)
Goal: a small variational quantum circuit predicts good QAOA angles (gamma, beta) from instance features, so QAOA starts near the optimum and needs fewer optimizer iterations.
- Dataset: generate 500-2000 small seeded instances (3-5 EVs x 4-5 slots, about 16-20 qubits max). For each, run QAOA from several starting points and store the best (gamma, beta) after canonicalising for symmetry/periodicity. Cache to Parquet. Make generation resumable and parallel-safe because it is the main compute cost.
- Features (about 8): normalised price statistics, capacity tightness (total demand vs station limit), window lengths, required slots, QUBO coefficient statistics.
- Model: angle-encode features onto qubits, trainable entangling layers (for example StronglyEntanglingLayers), read out 2p expectation values mapped to gamma in [0, pi] and beta in [0, pi/2]. PyTorch training with a train/validation/test split by instance.
- Training objective: start with regression on labels; if label ambiguity hurts, switch to training on the QAOA energy reached with the predicted angles.
- Integration: optimizer method "qaoa_ml" predicts angles, runs QAOA from them, validates the result, and stores the model version/hash in the experiment record.
- REQUIRED baselines on held-out instances: (a) random initial angles, (b) one fixed "average good" angle set, (c) a classical model (linear regression and a small MLP) with the same inputs/outputs. Metrics: approximation ratio before extra optimisation, optimizer iterations to reach 95% of the best value, feasibility rate. If a classical baseline wins, report that plainly.

## 11. Backend API (/api/v1)
GET /health; POST /auth/register; POST /auth/login; GET /stations; POST /stations (operator); GET /stations/{id}/chargers; POST /charging-requests; GET /charging-requests/{id}; POST /optimization-jobs (method: fcfs | edf | lp | sa | qaoa | qaoa_ml | compare); GET /optimization-jobs/{id}; GET /optimization-jobs/{id}/schedule; GET /stations/{id}/metrics; plus CSV export endpoints for experiment results.
- Server-side validation of time ranges, SoC percentages, power limits, ids. Role-based authorization (drivers see only their requests; operators manage stations and run jobs).
- Data model: User, Vehicle, Station, Charger, ChargingRequest, OptimizationJob, Schedule, ExperimentResult (feasibility, total cost, peak power, completion rate, utilization, runtime, solver metadata, seeds, versions, model version). Alembic migrations for all tables.
- QAOA/QML jobs run in the worker, never in a request handler. Larger instances than the quantum limit fall back to classical methods with a clear message.
- Structured logs without unnecessary personal data; no raw exception traces returned to clients.

## 12. Frontend
Operator: station overview, charger status, EV queue, config (chargers, power limits, tariff), "Run scenario" with method picker, schedule timeline (per EV and per charger), solver comparison table + bar charts (cost, peak, completion, utilization, runtime, feasibility), experiment history, CSV export. Show a clear "simulated data" label.
Driver (/driver, mobile-first): vehicle profile, request form (arrival, departure, current/target SoC), status, assigned charging window, charger, estimated cost, and a clear message plus best-effort alternative when infeasible.
Style: clean, accessible (contrast, labels, large touch targets), consistent loading/error/empty states.

## 13. Testing
- Unit: SoC to energy, efficiency, tariff cost, slot boundaries, charger compatibility, validator.
- Optimization: tiny instances verified by enumeration; impossible deadlines, insufficient station power, overlapping requests, no chargers, zero-energy requests, varying tariffs; QUBO ground state equals exact solution; invalid schedules never marked feasible.
- API/integration: auth and roles, validation errors, persistence, job lifecycle, solver failure and timeout handling, operator view and driver view showing the same stored schedule.
- Frontend: component tests for all states; metrics come from API data.
- CI runs everything on each push.

## 14. Experiments and reporting
Create experiments/ with config-driven scripts and a README:
- run_benchmark: methods (brute force, exact, SA, QAOA p=1/2/3, QAOA+ML) over instance sizes (for example 3x4, 3x5, 4x4 EVs x slots), >= 10 seeds, identical inputs and objective for all methods.
- Outputs: results tables (CSV/Parquet), plots (feasibility rate, optimality gap, approximation ratio, runtime vs size, qubits/circuit depth, penalty sensitivity, warm-start comparison), and a generated markdown report.
- The report states limitations (simulator only, small instances, simplified tariff/efficiency models) and does not claim advantage.

## 15. Phases (stop and summarize after each; definition of done = tests pass, docker compose up works, docs accurate)
0. Repo scaffold, CI, Docker Compose, conventions doc.
1. Domain models, scenario generator, validator, feasibility pre-check.
2. Classical solvers (FCFS, EDF, LP/MILP), metrics.
3. Reduced tier: QUBO builder, brute force, simulated annealing, penalty sweep.
4. QAOA with saved reproducible results.
5. QML warm-start: dataset, model, baselines, evaluation.
6. Backend API, auth, jobs table + worker, persistence.
7. Operator dashboard and /driver page.
8. End-to-end tests, benchmark report, deployment config (static frontend, API + worker, managed Postgres), README and demo script.
If time or compute is short, cut in this order: deployment polish, separate driver page polish, auth beyond role checks, mixer ablation. Never cut the validator, classical baselines, or the benchmark report.

## 16. How to work
- Start by writing a short plan and a list of assumptions, then proceed without waiting unless there is a real blocker.
- Small, tested commits. Pinned dependencies. Type hints and docstrings on public functions.
- Keep solver code, API code and UI code strictly separated. No business logic in the frontend.
- Write docs: README (setup, run, test), docs/architecture.md (with a diagram), docs/formulation.md (full math for both tiers), docs/experiments.md, docs/limitations.md.
- At the end, output a checklist of things I must do or confirm myself: deployment accounts and limits, real data sources, any hackathon/track requirements, reference verification, license choice.

Open questions with my default assumptions (change if I say otherwise):
- Timeline: [8 WEEKS / CHANGE]
- Hosting: [LOCAL DOCKER ONLY / CLOUD]
- Real session data (ACN-Data) available: [YES/NO]
- GPU available: [YES/NO] (everything must also run on CPU)
- Native mobile app (Expo) wanted: [NO, responsive page only / YES]