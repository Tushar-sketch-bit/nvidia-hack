# SimPilot: Engineering Simulation Copilot

> Describe a part and a driving scenario in plain language. SimPilot asks for what's missing, runs validated physics in the background, and returns an auditable pass/fail report.

**Status:** In development (hackathon build). Items marked `[ ]` in the [Roadmap](#roadmap) are not finished yet.

---

## The problem

Setting up an engineering simulation (meshing, boundary conditions, load cases, solver input files) takes specialists days. Design engineers who are not simulation experts often wait for another team, or skip early checks and find problems late, when changes are expensive.

## What SimPilot does

1. You upload a part (or choose a template) and describe the scenario in plain language.
2. An **intake agent** asks follow-up questions until every required field is complete and valid.
3. A **planner agent** decides which physics checks apply and which do not.
4. You review and approve a **confirmation summary** with all loads, supports, materials and assumptions.
5. Simulations run as **background jobs**, using deterministic solvers.
6. A **reporter agent** explains the numeric results, compares them to limits, and suggests changes.
7. You edit the design and re-run, with before/after metrics.

## Core principle: the LLM never produces a number

| Job | Done by |
|---|---|
| Understanding the request, asking questions | Nemotron (LLM) |
| Checking whether the request is complete | Plain code (Pydantic) |
| Geometry, meshing, stress, thermal results | CadQuery, Gmsh, CalculiX |
| Safety factor, pass/fail | Plain code |
| Explaining results, suggesting changes | Nemotron (LLM) |

Every number in a report comes from a solver. A post-check flags any number in the written report that is not found in the solver output.

## Model usage (Nemotron on Nebius Token Factory)

| Model | Used for |
|---|---|
| **Nemotron 3 Ultra** | Choosing which checks apply, interpreting results, writing the final report and suggestions |
| **Nemotron 3 Super** | Structured extraction from user text, the clarification conversation, tool-call routing |
| **Nemotron 3 Nano** | Unit hints, status messages, log summaries |

> Confirm exact model IDs in the Nebius Token Factory model list and set them in `.env`.

## Architecture

```
 User (Streamlit UI)
        |
        v
  FastAPI backend ---------> SQLite (runs, requests, reports)
        |
        |--> Intake agent (Super) <--> Pydantic validation (code)
        |--> Planner agent (Ultra) --> failure-mode checklist
        |
        v
  Confirmation (user approves)
        |
        v
  Nebius Serverless Job (Docker)
     CadQuery/STEP --> Gmsh mesh --> CalculiX --> PyVista stress map
        |
        v
  Results (JSON + images) --> Reporter agent (Ultra) --> Report
```

## Tech stack

| Layer | Tools |
|---|---|
| Language | Python 3.11 |
| LLM access | Nebius Token Factory via the `openai` Python SDK (JSON mode / tool calling) |
| Validation | Pydantic, Pint (units) |
| Geometry | CadQuery (OpenCASCADE), STEP files |
| Meshing | Gmsh (Python API) |
| Solver | CalculiX (static and thermal stress) |
| Results | NumPy, meshio, PyVista |
| Backend | FastAPI, SQLite |
| Compute | Docker, Nebius Serverless Jobs |
| Frontend | Streamlit |
| Deploy | Nebius Serverless Endpoints |
| Testing | pytest with analytical benchmarks |

## Project structure

```
simpilot/
├── app/                # FastAPI app and routes
├── agents/             # intake, planner, reporter prompts and logic
├── schemas/            # Pydantic models (SimRequest, Results, Report)
├── sim/                # geometry, meshing, solver input, post-processing
├── jobs/               # entrypoint executed inside the Nebius job
├── ui/                 # Streamlit app
├── data/materials.yaml # material properties with sources
├── benchmarks/         # analytical validation cases
├── tests/
├── docker/Dockerfile
├── .env.example
└── README.md
```

## Setup

### Prerequisites
- Python 3.11
- Docker
- A Nebius account with Token Factory access

### Install
```bash
git clone <your-repo-url> simpilot
cd simpilot
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # then fill in your values
```

### Environment variables (`.env`)
```
NEBIUS_API_KEY=your_key_here
NEBIUS_BASE_URL=<from Nebius Token Factory docs>
MODEL_ULTRA=<exact model id>
MODEL_SUPER=<exact model id>
MODEL_NANO=<exact model id>
MAX_CLARIFY_ROUNDS=4
MAX_LLM_RETRIES=2
```

### Run locally
```bash
# 1. Build the simulation container (Gmsh + CalculiX + PyVista)
docker build -t simpilot-sim -f docker/Dockerfile .

# 2. Start the API
uvicorn app.main:app --reload

# 3. Start the UI
streamlit run ui/app.py
```

### Run the benchmarks
```bash
pytest benchmarks/ -v
```

## Example

**Input**
> "Bracket for a windshield mount, aluminum, city driving, check if it's safe."

**Intake agent asks**
> "What load should the bracket carry, and in which direction? Where is it bolted?"

**Confirmed request (excerpt)**
```json
{
  "part": "l_bracket",
  "dimensions_mm": {"length": 80, "thickness": 4, "hole_radius": 4},
  "material": "aluminum_6061_T6",
  "load": {"magnitude_N": 5000, "direction": "-Z", "face": "top"},
  "supports": ["bolt_holes"],
  "target_safety_factor": {"value": 2.0, "against": "yield"},
  "assumptions": ["Static load", "Room temperature unless thermal check selected"]
}
```

**Report (excerpt)**
```
Static stress:   PASS  (peak Von Mises 112 MPa, safety factor vs yield 2.46)
Thermal stress:  NOT RUN (not requested)
Fatigue:         NOT EVALUATED (cyclic loads likely; recommended before sign-off)
```
*(Illustrative numbers. Real values come from the solver.)*

## Validation

| Benchmark | Reference | Status |
|---|---|---|
| Cantilever beam tip deflection | Euler-Bernoulli formula | `[ ]` |
| Cantilever beam max stress | Bending stress formula | `[ ]` |
| Constrained bar thermal stress | E × α × ΔT | `[ ]` |
| Mesh convergence (3 densities) | Relative change in peak stress | `[ ]` |

Results will be published in `benchmarks/RESULTS.md`.

## Safety and trust design

- The LLM never generates a numeric result.
- Completeness of a request is checked by code, not by the LLM.
- Simulation does not start until the user approves the confirmation summary.
- All defaults are shown as labeled **assumptions**.
- Every report lists **checks run** and **checks NOT run**.
- The LLM never executes code. It only produces JSON validated against a schema and passed to whitelisted functions.
- Retries and rounds are capped to protect credits.

## Limitations (read before relying on results)

- **Pre-screening tool only.** It is not a certification tool and does not replace physical testing, standards compliance, or engineer sign-off.
- Current checks: linear static and thermal stress. Fatigue, buckling, contact, bolt preload, plasticity, and crash dynamics are **not** evaluated unless listed in the report.
- Supports a limited set of part families and one material set in the MVP.
- Stress at sharp corners and point loads can be artificially high (mesh singularities). Mesh settings are documented in the benchmarks.
- Laminated glass and multiphysics (aero, optics, impact) are on the roadmap, not in the MVP.

## Roadmap

**Phase 0: Setup**
- [ ] Repo, venv, `.env`, Nebius key
- [ ] Verified call to Ultra, Super, Nano

**Phase 1: Physics pipeline**
- [ ] Parametric L-bracket (CadQuery) to STEP
- [ ] Gmsh mesh with named face groups
- [ ] CalculiX static run, Von Mises, safety factor
- [ ] Stress map image

**Phase 2: Validation**
- [ ] Cantilever benchmark in pytest
- [ ] Mesh convergence study

**Phase 3: Thermal check**
- [ ] Thermal stress case and benchmark

**Phase 4: Intake agent**
- [ ] Pydantic `SimRequest` and Pint units
- [ ] Missing-field detection in code
- [ ] Clarification loop with round cap
- [ ] Confirmation summary and approval gate

**Phase 5: Planner and reporter**
- [ ] Failure-mode checklist
- [ ] Ultra planner with validated output
- [ ] Reporter with number-verification guard

**Phase 6: Backend and jobs**
- [ ] FastAPI endpoints and run states
- [ ] Dockerized solver
- [ ] Nebius Serverless Job execution

**Phase 7: Frontend**
- [ ] Streamlit chat, upload, approval card, results view
- [ ] Before/after comparison

**Phase 8: Deploy and demo**
- [ ] Nebius Serverless Endpoints deployment
- [ ] Demo scenarios, backup video, pitch

**Stretch / Vision**
- [ ] Fatigue check
- [ ] PDF report export
- [ ] Company-limits file (YAML)
- [ ] More part families
- [ ] Windshield vision: laminated glass, aerodynamics (CFD), HUD optics, impact

## Contributing

Issues and pull requests are welcome. Add a benchmark for any new solver feature before opening a PR.

## License

To be decided (MIT recommended for hackathon submission).

## Acknowledgments

NVIDIA Nemotron models, Nebius Token Factory, CadQuery, Gmsh, CalculiX, PyVista.
