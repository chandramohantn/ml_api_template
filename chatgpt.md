# Lead Data Scientist Bootstrap Operating System

## Objective

The goal is not to create random reusable templates.
The goal is to reduce:

* Time-to-first-POC
* Time-to-production
* Cognitive overhead across projects
* Environment inconsistencies
* Repeated infrastructure decisions
* Knowledge silos
* ML experimentation chaos

A good lead data scientist operating model should enable:

1. Rapid project initialization
2. Standardized engineering practices
3. Faster onboarding of future team members
4. Reproducibility
5. Operational maturity
6. AI-assisted development workflows
7. Production-grade experimentation

---

# Phase 1 — Build Your Personal Engineering Foundation

This phase should ideally be completed before joining.

Focus:

* Your reusable tooling
* Local development stack
* Repo standards
* AI augmentation setup
* Infra abstractions

Do not over-engineer initially.

The first target is:

"Can I go from blank repo → working ML service in under 1 hour?"

---

# 1. Repository Templates You Should Build

You should maintain multiple starter repositories.

Not one.

Different problem categories need different scaffolding.

---

# A. ML API Service Template

Use when:

* Serving models
* Online inference
* RAG APIs
* Agent APIs
* Internal ML microservices

## Recommended Stack

* FastAPI
* Pydantic v2
* uv or poetry
* Docker
* pytest
* Ruff
* mypy
* pre-commit
* structured logging
* Prometheus metrics
* OpenTelemetry hooks
* health checks
* model registry abstraction
* feature flags

## Structure

```text
repo/
├── app/
│   ├── api/
│   ├── services/
│   ├── models/
│   ├── schemas/
│   ├── core/
│   ├── middleware/
│   ├── observability/
│   └── config/
├── tests/
├── scripts/
├── infra/
├── docker/
├── notebooks/
├── Makefile
├── pyproject.toml
└── docker-compose.yml
```

## Mandatory Features

### Standardized config system

Support:

* local
* dev
* staging
* prod

using:

```text
.env
.env.dev
.env.prod
```

with pydantic settings.

---

### Observability baked in

Include:

* request tracing
* latency metrics
* GPU metrics hooks
* inference timing
* structured JSON logging
* correlation IDs

Most ML teams add this too late.

---

### Inference abstraction

Create interfaces like:

```python
class BaseInferenceEngine:
    async def predict(self, request):
        pass
```

This allows swapping:

* PyTorch
* TensorRT
* ONNX
* vLLM
* Triton
* OpenVINO

without rewriting API logic.

---

# B. Training Pipeline Template

Use when:

* supervised learning
* fine tuning
* experimentation
* model training
* evaluation pipelines

## Recommended Stack

* Hydra
* PyTorch Lightning or pure PyTorch
* MLflow or Weights & Biases
* DVC optional
* Optuna
* Pandas/Polars
* Pydantic configs

## Key Design Principle

Everything configurable.

No hardcoded:

* datasets
* paths
* hyperparameters
* hardware assumptions

---

## Suggested Structure

```text
training/
├── configs/
├── data/
├── datasets/
├── models/
├── trainers/
├── evaluators/
├── inference/
├── experiments/
├── notebooks/
├── scripts/
└── tests/
```

---

## Critical Automation

### Single-command experiment execution

```bash
make train
```

or:

```bash
python train.py experiment=resnet50_gpu
```

You should avoid projects where every engineer invents their own training entrypoint.

---

### Standard evaluation reports

Automatically generate:

* confusion matrix
* precision/recall
* ROC curves
* calibration plots
* drift reports
* feature importance

Save as:

```text
artifacts/reports/
```

---

# C. RAG / Agentic AI Template

This will become extremely useful.

Most organizations currently have chaotic GenAI repos.

Your advantage as lead is introducing structure.

## Recommended Stack

* LangGraph
* LlamaIndex or Haystack
* FastAPI
* pgvector or Qdrant abstraction
* Redis
* OpenTelemetry
* evaluation harness

## Mandatory Components

### Retriever abstraction

```python
class BaseRetriever:
    async def retrieve(self, query):
        pass
```

---

### LLM abstraction

Support:

* OpenAI
* Anthropic
* local vLLM
* Azure OpenAI
* Ollama

through adapters.

---

### Evaluation Harness

Most teams skip this.

Build reusable:

* groundedness checks
* hallucination checks
* retrieval precision
* answer relevance
* latency benchmarks
* token usage metrics

---

### Prompt registry

Do not scatter prompts across code.

Use:

```text
prompts/
├── system/
├── evaluators/
├── routing/
└── extraction/
```

---

# D. ETL / Batch Pipeline Template

Use when:

* scheduled jobs
* feature pipelines
* analytics
* batch scoring
* ingestion pipelines

## Recommended Stack

* Prefect or Airflow
* Polars
* SQLAlchemy
* dbt optional
* Docker
* Great Expectations optional

## Critical Design

Separate:

* extract
* transform
* validate
* load

strictly.

Do not mix business logic with IO logic.

---

# 2. Standard Development Environment

You should have:

## A. Local AI Development Stack

Containerized local stack:

```text
- PostgreSQL
- Redis
- MinIO
- Qdrant
- MLflow
- Prometheus
- Grafana
- Ollama/vLLM
```

using:

```bash
docker compose up
```

This becomes your local ML platform.

---

## B. Reusable Docker Base Images

This is extremely important.

Build:

### CPU DS Base Image

Contains:

* Python
* uv
* numpy
* pandas
* scikit-learn
* jupyter
* pyarrow
* polars
* common utilities

---

### GPU Deep Learning Base Image

Contains:

* CUDA
* PyTorch
* torchvision
* xformers
* transformers
* deepspeed optional
* tensorboard

---

### Inference Base Image

Optimized for:

* small size
* startup speed
* inference runtime
* production deployment

Potentially multi-stage builds.

---

# 3. CLI Productivity Layer

Create your own internal CLI.

This is massively undervalued.

Example:

```bash
mlops init api-service
mlops init rag-service
mlops init training-pipeline
```

This should:

* clone template
* rename project
* initialize git
* create virtual env
* setup pre-commit
* inject metadata

---

## Suggested Implementation

Use:

* Typer
* Cookiecutter
* Copier

Copier is better for evolving templates.

---

# 4. AI-Augmented Engineering Setup

This is now a force multiplier.

But most engineers use AI inefficiently.

You should systematize it.

---

# A. MCP Servers You Should Consider

## Codebase MCP

For:

* repo indexing
* semantic code retrieval
* architecture navigation

---

## Postgres MCP

For:

* schema exploration
* query generation
* debugging

---

## Kubernetes MCP

For:

* cluster inspection
* deployment debugging
* log retrieval

---

## GitHub MCP

For:

* PR summarization
* issue analysis
* repo insights

---

## Documentation MCP

Useful for:

* internal wiki retrieval
* architecture docs
* API references

---

# B. Reusable AI Skills

Create reusable prompting workflows.

Examples:

## System Design Reviewer

Input:

* architecture
* bottlenecks
* constraints

Output:

* scaling risks
* SPOFs
* deployment risks
* observability gaps

---

## ML Experiment Reviewer

Checks:

* leakage
* train/test contamination
* metric quality
* imbalance handling
* reproducibility

---

## PR Reviewer Skill

Checks:

* typing
* async misuse
* memory leaks
* inference bottlenecks
* missing retries
* missing metrics

---

## RAG Reviewer Skill

Checks:

* chunking quality
* retrieval quality
* prompt injection risks
* evaluation gaps
* hallucination risks

---

# 5. Engineering Workflows To Standardize

This is where lead-level leverage appears.

You are not only writing code.

You are defining how teams operate.

---

# A. New Project Workflow

Target:

New project operational within same day.

## Workflow

### Step 1

Create repo from template.

### Step 2

Bootstrap CI/CD.

### Step 3

Setup observability.

### Step 4

Setup experiment tracking.

### Step 5

Create architecture decision records.

### Step 6

Define:

* SLAs
* latency targets
* dataset ownership
* evaluation metrics

---

# B. Experiment Workflow

Most ML organizations are weak here.

You should standardize:

## Experiment Lifecycle

```text
idea
→ hypothesis
→ dataset version
→ experiment config
→ training
→ evaluation
→ comparison
→ deployment decision
```

---

## Mandatory Metadata

Every experiment should track:

* git commit
* dataset version
* model version
* config
* hardware
* metrics
* seed
* runtime

Without this, reproducibility collapses.

---

# C. Production Deployment Workflow

Standardize:

```text
training
→ validation
→ registry
→ staging
→ shadow traffic
→ canary
→ production
```

---

## Deployment Checklist

Automate checks:

* schema validation
* latency benchmarks
* memory tests
* load tests
* drift baseline
* rollback readiness

---

# D. Incident Workflow

You should prepare this before incidents happen.

## Standard Runbooks

### Model degradation

Checks:

* drift
* upstream schema changes
* null spikes
* distribution shifts
* retrieval failures

---

### GPU OOM

Checks:

* batch size
* fragmentation
* memory leaks
* model loading duplication
* concurrency spikes

---

### Latency spikes

Checks:

* token explosion
* vector DB slowdown
* serialization bottlenecks
* blocking async calls
* cold starts

---

# 6. CI/CD Standards

Every repo should include:

## CI

* lint
* typing
* unit tests
* smoke tests
* docker build validation
* vulnerability scan

---

## CD

* image versioning
* deployment tagging
* rollback support
* canary deployment hooks

---

# 7. Documentation Templates

You should never start projects with blank documentation.

---

# A. Architecture Template

Include:

* business problem
* constraints
* data flow
* scaling assumptions
* latency requirements
* storage decisions
* failure scenarios

---

# B. Model Card Template

Include:

* training data
* intended usage
* limitations
* bias considerations
* evaluation metrics
* deployment assumptions

---

# C. Experiment Report Template

Include:

* hypothesis
* setup
* metrics
* observations
* failure analysis
* next steps

---

# 8. Recommended Tooling Stack

This is a pragmatic modern stack.

Do not adopt all immediately.

Prioritize based on company maturity.

| Area            | Recommended           |
| --------------- | --------------------- |
| API             | FastAPI               |
| Packaging       | uv                    |
| Config          | Hydra + Pydantic      |
| Training        | PyTorch               |
| Tracking        | MLflow/W&B            |
| Orchestration   | Prefect               |
| Vector DB       | Qdrant/pgvector       |
| Data Validation | Great Expectations    |
| Linting         | Ruff                  |
| Typing          | mypy                  |
| Testing         | pytest                |
| Observability   | OpenTelemetry         |
| Metrics         | Prometheus            |
| Dashboards      | Grafana               |
| Containers      | Docker                |
| Deployment      | Kubernetes            |
| CI/CD           | GitHub Actions/GitLab |

---

# 9. What You Should Avoid

These are common lead-level mistakes.

---

## Avoid building giant frameworks early

Start small.

Refactor from real patterns.

Not imagined patterns.

---

## Avoid excessive abstraction initially

Too many base classes early will slow teams.

Abstract only after repetition appears.

---

## Avoid notebook-only culture

Notebooks are exploration tools.

Not production systems.

---

## Avoid hidden tribal knowledge

Everything operational should exist as:

* scripts
* docs
* CI checks
* templates

---

## Avoid environment snowflakes

If reproducibility depends on one engineer's laptop:

system maturity is low.

---

# 10. High-Leverage First 30 Days Strategy

This is probably the most important section.

Do not enter the company trying to immediately redesign architecture.

First:

understand operational pain.

---

# Week 1

Focus:

* infra understanding
* deployment flow
* pain points
* existing tooling
* engineering maturity
* bottlenecks

Deliverables:

* architecture notes
* dependency map
* ML lifecycle map
* deployment map

---

# Week 2

Introduce:

* repo standards
* local dev standards
* linting
* reproducibility improvements
* observability improvements

Avoid massive platform rewrites.

---

# Week 3

Build:

* reusable template
* CI improvements
* experiment tracking
* evaluation framework

Target visible productivity gains.

---

# Week 4

Propose:

* medium-term architecture improvements
* platform standardization
* inference optimization roadmap
* MLOps roadmap
* GenAI governance if applicable

---

# 11. Suggested Folder For Your Personal Knowledge System

```text
lead-ds-os/
├── templates/
│   ├── api-service/
│   ├── training-pipeline/
│   ├── rag-system/
│   └── etl-pipeline/
├── docker-images/
├── cli/
├── workflows/
├── docs/
├── runbooks/
├── prompts/
├── mcp/
└── evaluations/
```

---

# 12. Highest ROI Assets To Build First

If time is limited:

Build in this order.

## Tier 1 (Highest ROI)

1. ML API template
2. Docker base images
3. Local dev stack
4. CI/CD template
5. Experiment tracking setup

---

## Tier 2

6. RAG template
7. Internal CLI scaffolding tool
8. AI reviewer skills
9. Evaluation harness

---

## Tier 3

10. Full internal ML platform
11. Kubernetes abstractions
12. Automated infra provisioning
13. Self-service deployment systems

---

# Final Recommendation

Your leverage as a lead data scientist is not:

"How fast can I write models?"

Your leverage is:

"How quickly can I make the entire engineering and ML lifecycle repeatable, reliable, observable, and scalable?"

The strongest leads reduce organizational entropy.

That means:

* fewer custom workflows
* fewer undocumented systems
* fewer deployment surprises
* fewer reproducibility failures
* faster experimentation
* better operational visibility
* cleaner interfaces between data science and engineering

The operating system you build should optimize for those outcomes.

I prepared a detailed operating plan covering:

Reusable repo templates
ML/DL environments
Docker image strategy
AI-agent/MCP setup
Experiment workflows
CI/CD standards
RAG/GenAI scaffolding
Observability standards
Automation ideas
30-day onboarding strategy
High-ROI assets to build first

The focus is on reducing:

time-to-first-POC
deployment friction
reproducibility issues
operational chaos
onboarding overhead

and creating a reusable “Lead DS Operating System” you can carry across organizations.


Start with the **ML API Service Template**.

Not the training pipeline.

Not the RAG framework.

Not the MCP stack.

Not Kubernetes abstractions.

The ML API service template gives the highest leverage because it becomes the convergence point for almost everything else:

* model serving
* RAG systems
* internal tools
* inference APIs
* evaluation services
* batch triggers
* agent backends
* experimentation endpoints
* observability
* deployment standards
* CI/CD standards

If this foundation is weak, every future project becomes inconsistent.

---

# Why This Should Be First

A lead-level mistake is starting with “platform vision” instead of operational bottlenecks.

The first real bottleneck in most organizations is:

> “Nobody can reliably create production-grade ML services quickly.”

Typical symptoms:

* every repo has different structure
* different logging systems
* different config handling
* no observability
* broken Dockerfiles
* inconsistent dependency management
* async misuse
* random notebooks becoming APIs
* no inference abstractions
* no reproducibility

Your first template should solve this.

---

# What We Should Build

We should build a:

# Production-Grade ML Service Starter Kit

This should support:

* classical ML
* deep learning
* RAG
* LLM APIs
* async inference
* batch inference
* GPU inference
* future agent systems

without major restructuring.

---

# What Makes This Difficult

Most “starter templates” online are shallow.

They miss:

* inference lifecycle management
* observability
* GPU readiness
* async correctness
* model loading strategy
* deployment concerns
* config isolation
* dependency isolation
* structured logging
* health/readiness separation
* graceful shutdown
* concurrency control
* inference abstraction
* future extensibility

We should design this correctly from the beginning.

---

# The Actual Goal

The goal is:

```text
new_project/
→ rename project
→ configure model
→ implement predictor
→ deploy
```

within hours.

Not days.

---

# Proposed Architecture

We should build around:

| Area                 | Choice             |
| -------------------- | ------------------ |
| API Framework        | FastAPI            |
| Packaging            | uv                 |
| Validation           | Pydantic v2        |
| Async Server         | uvicorn            |
| Config               | pydantic-settings  |
| Logging              | structlog          |
| Metrics              | prometheus-client  |
| Tracing              | OpenTelemetry      |
| Testing              | pytest             |
| Linting              | Ruff               |
| Typing               | mypy               |
| Containers           | Docker             |
| Task Runner          | Makefile           |
| Dependency Injection | native/lightweight |
| Model Layer          | custom abstraction |

---

# Core Design Principle

The service should separate:

```text
transport layer
≠
business layer
≠
inference layer
≠
model runtime
≠
observability
```

Most teams tightly couple these.

That becomes technical debt very quickly.

---

# What We Should Implement First

We should not attempt the whole framework immediately.

We should implement in stages.

---

# Phase 1 — Skeleton

This is the most important.

We create:

```text
app/
├── api/
├── core/
├── services/
├── schemas/
├── observability/
├── middleware/
├── config/
└── models/
```

with:

* startup lifecycle
* config system
* logging
* health checks
* dependency management
* Docker
* linting
* tests

This alone already creates huge leverage.

---

# Phase 2 — Inference Runtime

Then:

* model loading abstraction
* GPU initialization
* batching support
* inference interfaces
* async inference
* request validation

---

# Phase 3 — Production Hardening

Then:

* metrics
* tracing
* readiness/liveness
* graceful shutdown
* rate limiting
* retries
* circuit breakers
* memory monitoring

---

# Phase 4 — AI/LLM Extensions

Then:

* vector DB integration
* prompt registry
* RAG pipeline abstraction
* LLM provider abstraction
* evaluation hooks

---

# The Most Important Early Decision

This one matters a lot:

## We should design for long-running model processes.

NOT request-time model loading.

Meaning:

BAD:

```python
@app.post("/predict")
def predict():
    model = load_model()
```

GOOD:

```python
@app.on_event("startup")
async def startup():
    app.state.model = load_model()
```

This affects:

* latency
* GPU memory
* concurrency
* scaling
* cold starts

---

# Another Critical Decision

We should use:

# `uv`

instead of pip + requirements.txt.

Because:

* deterministic
* much faster
* cleaner lockfiles
* modern Python packaging

This matters in teams.

---

# What We Should Build Together First

The first milestone should be:

# A Minimal Production-Grade ML API Skeleton

with:

* FastAPI
* uv
* Docker
* structured logging
* config management
* health endpoints
* proper repo structure
* Makefile
* test scaffolding
* linting/type checking

No model yet.

No RAG yet.

No GPU yet.

Just:

* architecture correctness
* operational correctness
* extensibility

That foundation matters more than adding ML immediately.

---

# Why I’m Not Starting With Training Pipelines

Because training code changes heavily across projects.

Serving infrastructure changes less.

Inference systems become organizational standards.

Training pipelines are more domain-dependent.

---

# Why I’m Not Starting With AI Agents

Because most organizations still lack:

* stable APIs
* reproducibility
* observability
* deployment maturity

Agent frameworks without engineering maturity become chaos very quickly.

---

# What I Recommend We Do Next

Next step should be:

# Design the exact repository structure and lifecycle

including:

* how startup works
* how dependency injection works
* how configs work
* how logging works
* how services communicate
* where models live
* how inference abstraction works
* how observability hooks are injected

before writing code.

This architecture step is extremely important.

Most template repos fail because they start coding before defining boundaries.


The value is not FastAPI itself.

The value is:

* lifecycle correctness
* clean boundaries
* operational maturity
* extensibility
* standardization

---

# Step 1 — Define The Design Principles

These principles should drive the repo.

---

# 1. Business Logic Must Be Framework Independent

This is one of the most important decisions.

BAD:

```python
@app.post("/predict")
async def predict():
    result = some_logic()
```

This couples:

* transport layer
* orchestration
* business logic

Instead:

```text
API Layer
    ↓
Service Layer
    ↓
Domain Logic
```

This enables:

* testing without FastAPI
* batch execution reuse
* CLI reuse
* future gRPC support
* future Kafka consumers
* future async workers

without rewriting logic.

---

# 2. Config Must Be Centralized

No scattered:

```python
os.getenv(...)
```

Use:

* `pydantic-settings`
* typed configuration
* environment-aware config

Example:

```python
settings.database.host
settings.server.port
```

This becomes critical later for:

* secrets management
* Kubernetes
* cloud deployment
* runtime validation

---

# 3. Logging Must Be Structured From Day 1

Most teams regret this later.

We should emit:

```json
{
  "timestamp": "...",
  "level": "INFO",
  "request_id": "...",
  "route": "...",
  "latency_ms": 12
}
```

NOT:

```python
print("request failed")
```

Because later:

* Grafana
* Loki
* Datadog
* ELK
* OpenTelemetry

all depend on structured logs.

---

# 4. Health Endpoints Must Be Separated

We should support:

| Endpoint        | Purpose            |
| --------------- | ------------------ |
| `/health/live`  | Process alive      |
| `/health/ready` | App ready to serve |

This matters for:

* Kubernetes
* load balancers
* deployment rollouts

Even if you don’t use K8s immediately.

---

# 5. Startup Lifecycle Must Be Explicit

Avoid:

* global state
* import-time initialization

We should use:

* FastAPI lifespan
* app state
* controlled initialization

This becomes essential later for:

* models
* DB pools
* Redis
* vector DBs
* telemetry

---

# 6. Async Boundaries Must Be Clean

One major FastAPI anti-pattern:

```python
async def endpoint():
    requests.get(...)
```

This blocks the event loop.

We should clearly separate:

* async-safe code
* blocking code

from the beginning.

---

# Step 2 — Define Repo Structure

This is where most repos become messy.

I recommend this:

```text
ml-api-template/
│
├── app/
│   ├── api/
│   │   ├── routes/
│   │   └── dependencies/
│   │
│   ├── core/
│   │   ├── config.py
│   │   ├── logging.py
│   │   ├── lifespan.py
│   │   └── exceptions.py
│   │
│   ├── middleware/
│   │   ├── request_context.py
│   │   └── logging_middleware.py
│   │
│   ├── services/
│   │   └── health_service.py
│   │
│   ├── schemas/
│   │   ├── health.py
│   │   └── common.py
│   │
│   ├── observability/
│   │   ├── metrics.py
│   │   └── tracing.py
│   │
│   ├── utils/
│   │
│   └── main.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
│
├── docker/
│   ├── Dockerfile
│   └── .dockerignore
│
├── scripts/
│
├── .env
├── .env.example
├── Makefile
├── pyproject.toml
├── README.md
└── docker-compose.yml
```

---

# Why This Structure Matters

---

# `api/`

Only transport concerns.

Contains:

* routes
* request validation
* response mapping
* dependencies

Should NOT contain:

* business logic
* DB logic
* model logic

---

# `services/`

Application orchestration layer.

This is where:

* workflows
* coordination
* orchestration

happen.

Future:

* inference services
* retrieval services
* feature services

will live here.

---

# `core/`

Global application infrastructure.

Examples:

* config
* logging
* lifecycle
* exceptions

This becomes the “kernel” of your service.

---

# `middleware/`

Cross-cutting concerns:

* request IDs
* logging
* tracing
* auth hooks

Avoid mixing middleware logic into routes.

---

# `schemas/`

Only request/response contracts.

Not business logic.

Use Pydantic v2 models here.

---

# `observability/`

Very important separation.

This allows:

* Prometheus
* OpenTelemetry
* tracing exporters

to evolve independently.

---

# Step 3 — Define Configuration System

This part is important.

---

# Recommended Approach

Use:

* `.env`
* `.env.dev`
* `.env.prod`

with:

```python
from pydantic_settings import BaseSettings
```

Example:

```python
class Settings(BaseSettings):
    app_name: str = "ml-api-template"
    environment: str = "dev"
    log_level: str = "INFO"
    host: str = "0.0.0.0"
    port: int = 8000

    class Config:
        env_file = ".env"
```

---

# Important Design Decision

Create:

```python
settings = Settings()
```

ONCE.

Do not instantiate repeatedly.

---

# Step 4 — Logging Architecture

This is where many repos fail.

---

# Use `structlog`

Not plain logging.

Because:

* JSON structured logs
* context binding
* request correlation
* async friendliness

---

# We Need Request Correlation IDs

Every request should carry:

```text
X-Request-ID
```

and logs should automatically include it.

This becomes essential later for:

* debugging
* distributed tracing
* incident analysis

---

# Step 5 — Health Endpoint Design

We should implement:

```text
GET /health/live
GET /health/ready
```

Difference:

---

## Live

Means:

```text
process running
```

Minimal checks.

---

## Ready

Means:

```text
dependencies initialized
```

Later this may include:

* DB connectivity
* model loaded
* Redis alive
* vector DB reachable

Even though we won’t implement those yet.

---

# Step 6 — Makefile Standards

The Makefile becomes your operational interface.

This matters a lot.

Example:

```make
install:
\tuv sync

run:
\tuv run uvicorn app.main:app --reload

lint:
\tuv run ruff check .

format:
\tuv run ruff format .

typecheck:
\tuv run mypy app

test:
\tuv run pytest

docker-build:
\tdocker build -f docker/Dockerfile -t ml-api-template .
```

This standardizes developer workflows.

---

# Step 7 — Testing Philosophy

We should separate:

| Type              | Purpose        |
| ----------------- | -------------- |
| Unit tests        | business logic |
| Integration tests | API behavior   |
| Smoke tests       | startup sanity |

---

# Important Early Rule

Never make FastAPI TestClient your primary testing strategy.

Most business logic should be testable without HTTP.

---

# Step 8 — Linting & Type Safety

Mandatory:

* Ruff
* mypy

Optional later:

* pyright

---

# Critical Rule

Treat typing as architecture.

Not autocomplete.

Typing:

* improves maintainability
* reduces runtime bugs
* improves AI-assisted coding quality

---

# Step 9 — Docker Philosophy

We should optimize for:

* reproducibility
* predictable runtime
* small image size
* layer caching

We should use:

* multi-stage builds
* non-root users
* pinned dependencies

from the beginning.

---

# What We Should Build Next

The next step should NOT be:
“generate all files.”

That usually produces shallow architecture.

Instead:

# We should first design:

1. `pyproject.toml`
2. dependency strategy
3. FastAPI lifespan pattern
4. logging bootstrap
5. config bootstrap
6. middleware chain
7. app factory pattern

These are the actual architectural foundations.

After that:
we generate the repo skeleton.
