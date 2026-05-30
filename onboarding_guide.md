# ML API Template — Team Onboarding Guide

> **Purpose**: This document explains how to use the `ml_api_template` repository to build production-grade ML services. Whether you're serving a classical ML model, a deep learning model, a RAG pipeline, or an LLM-backed agent — this template is your starting point.

---

## Table of Contents

1. [Why This Template Exists](#why-this-template-exists)
2. [Architecture Overview](#architecture-overview)
3. [Design Principles](#design-principles)
4. [Folder Structure Deep Dive](#folder-structure-deep-dive)
5. [Getting Started — New Project Workflow](#getting-started--new-project-workflow)
6. [Configuration System](#configuration-system)
7. [Logging & Observability](#logging--observability)
8. [Writing Your First Endpoint](#writing-your-first-endpoint)
9. [Inference Abstraction — Serving Models](#inference-abstraction--serving-models)
10. [Middleware & Cross-Cutting Concerns](#middleware--cross-cutting-concerns)
11. [Health Checks](#health-checks)
12. [Testing Strategy](#testing-strategy)
13. [Docker & Deployment](#docker--deployment)
14. [Makefile Commands](#makefile-commands)
15. [Common Use Cases (With Examples)](#common-use-cases-with-examples)
16. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
17. [FAQ](#faq)
18. [Concepts Deep Dive](#concepts-deep-dive)
19. [ML/DL Production Concerns](#mldl-production-concerns)

---

## Why This Template Exists

Every ML project in our team should:

- Go from blank repo → working service in **under 1 hour**
- Have consistent structure across all projects
- Include observability, health checks, and structured logging from day 1
- Be deployable to staging/production without rewriting infrastructure
- Be testable without spinning up the entire stack

Without a standard template, teams end up with:

- Different logging systems per project
- Broken Dockerfiles
- No observability until production incidents
- Notebooks masquerading as APIs
- Inconsistent dependency management
- Random config handling (`os.getenv` scattered everywhere)

This template eliminates those problems.

---

## Architecture Overview

The template follows a **layered architecture** with strict separation of concerns:

```
┌─────────────────────────────────────────────────────┐
│                   Transport Layer                     │
│              (FastAPI routes, HTTP concerns)          │
├─────────────────────────────────────────────────────┤
│                   Service Layer                       │
│         (Business logic, orchestration, workflows)   │
├─────────────────────────────────────────────────────┤
│                  Inference Layer                      │
│        (Model loading, prediction, batching)         │
├─────────────────────────────────────────────────────┤
│                Infrastructure Layer                   │
│    (Config, logging, metrics, tracing, middleware)   │
└─────────────────────────────────────────────────────┘
```

**Key rule**: Each layer only talks to the layer directly below it. Routes never touch models directly. Services never handle HTTP responses.

### Technology Stack

| Area | Choice | Why |
|------|--------|-----|
| API Framework | FastAPI | Async-native, auto-docs, Pydantic integration |
| Packaging | uv | Deterministic, fast, modern lockfiles |
| Validation | Pydantic v2 | Type-safe request/response contracts |
| Server | uvicorn | ASGI, async, production-ready |
| Config | pydantic-settings | Typed, validated, environment-aware |
| Logging | structlog | JSON structured logs, context binding |
| Metrics | prometheus-client | Industry standard, Grafana-compatible |
| Tracing | OpenTelemetry | Vendor-neutral distributed tracing |
| Testing | pytest | Standard, extensible, async support |
| Linting | Ruff | Fast, replaces flake8+isort+black |
| Typing | mypy | Static type checking |
| Containers | Docker | Reproducible builds |
| Task Runner | Makefile | Universal, no extra dependencies |

---

## Design Principles

These are non-negotiable for any service built from this template:

### 1. Business Logic Must Be Framework-Independent

Your prediction logic, data transformations, and orchestration should work without FastAPI. This means you can:
- Test without HTTP
- Reuse logic in batch jobs, CLI tools, or Kafka consumers
- Swap frameworks if needed

### 2. Configuration Must Be Centralized

No `os.getenv()` scattered across files. All config lives in `app/core/config.py` and is accessed via a typed `Settings` object.

### 3. Logging Must Be Structured From Day 1

Every log line is JSON with correlation IDs. This is required for Grafana, Datadog, ELK, or any observability platform.

### 4. Models Load at Startup, Not Per-Request

Models are loaded once during application startup and stored in `app.state`. Never load a model inside a request handler.

### 5. Async Boundaries Must Be Clean

If you call blocking code (file I/O, CPU-heavy inference) inside an `async` function, you must use `asyncio.to_thread()` or a thread pool. Never block the event loop.

### 6. Health Endpoints Are Mandatory

Every service exposes `/health/live` (process alive) and `/health/ready` (dependencies initialized). This is required for Kubernetes, load balancers, and deployment rollouts.


---

## Folder Structure Deep Dive

```
ml_api_template/
│
├── app/                          # Application source code
│   ├── main.py                   # App factory — creates and configures FastAPI instance
│   │
│   ├── api/                      # Transport layer (HTTP concerns only)
│   │   ├── routes/               # Route handlers grouped by domain
│   │   └── dependencies/         # FastAPI dependency injection functions
│   │
│   ├── core/                     # Application kernel — infrastructure foundations
│   │   ├── config.py             # Centralized settings (pydantic-settings)
│   │   ├── logging.py            # Structured logging setup (structlog)
│   │   ├── lifespan.py           # Startup/shutdown lifecycle management
│   │   └── exceptions.py         # Custom exception classes + global handlers
│   │
│   ├── middleware/               # Cross-cutting HTTP concerns
│   │   ├── request_context.py    # Request ID injection + context propagation
│   │   └── logging_middleware.py # Automatic request/response logging
│   │
│   ├── services/                 # Business logic + orchestration layer
│   │   └── health_service.py     # Health check logic (dependency checks)
│   │
│   ├── schemas/                  # Pydantic models for request/response contracts
│   │   ├── common.py             # Shared schemas (pagination, errors, etc.)
│   │   └── health.py             # Health endpoint schemas
│   │
│   ├── observability/            # Metrics + tracing (decoupled from business logic)
│   │   ├── metrics.py            # Prometheus metrics definitions
│   │   └── tracing.py            # OpenTelemetry tracing setup
│   │
│   └── utils/                    # Pure utility functions (no side effects)
│
├── tests/                        # Test suite
│   ├── conftest.py               # Shared fixtures (test client, mocks, etc.)
│   ├── unit/                     # Unit tests (no I/O, no network)
│   └── integration/              # Integration tests (real HTTP, real DB)
│
├── docker/                       # Container configuration
│   ├── Dockerfile                # Multi-stage production build
│   └── .dockerignore             # Files excluded from Docker context
│
├── scripts/                      # Operational scripts (migrations, seeds, etc.)
│
├── .env                          # Local environment variables (git-ignored)
├── .env.example                  # Template for .env (committed to git)
├── .gitignore                    # Git ignore rules
├── Makefile                      # Developer workflow commands
├── pyproject.toml                # Project metadata + dependencies (uv)
├── docker-compose.yaml           # Local development stack
└── README.md                     # Project overview
```

### What Goes Where — Decision Guide

| I need to... | Put it in... |
|---|---|
| Add a new API endpoint | `app/api/routes/` |
| Add request/response validation | `app/schemas/` |
| Add business logic or orchestration | `app/services/` |
| Add a new model/inference engine | `app/services/` (as an inference service) |
| Change app config or add env vars | `app/core/config.py` |
| Add startup/shutdown logic | `app/core/lifespan.py` |
| Add custom exceptions | `app/core/exceptions.py` |
| Add request-level middleware | `app/middleware/` |
| Add Prometheus metrics | `app/observability/metrics.py` |
| Add tracing spans | `app/observability/tracing.py` |
| Add a reusable dependency (auth, DB session) | `app/api/dependencies/` |
| Add a helper function with no side effects | `app/utils/` |
| Write a test for business logic | `tests/unit/` |
| Write a test that hits the API | `tests/integration/` |
| Add a deployment/migration script | `scripts/` |


---

## Getting Started — New Project Workflow

When starting a new ML service, follow these steps:

### Step 1: Clone and Rename

```bash
# Clone the template
git clone <template-repo-url> my-new-service
cd my-new-service

# Remove template git history
rm -rf .git
git init
```

### Step 2: Configure Your Project

Edit `pyproject.toml`:

```toml
[project]
name = "my-new-service"
version = "0.1.0"
description = "Short description of what this service does"
requires-python = ">=3.11"

dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "structlog>=24.0.0",
    "prometheus-client>=0.20.0",
    "opentelemetry-api>=1.20.0",
    "opentelemetry-sdk>=1.20.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.0",
    "ruff>=0.5.0",
    "mypy>=1.10.0",
    "pre-commit>=3.7.0",
]

# Add ML-specific deps for your use case:
ml = [
    "torch>=2.0.0",
    "transformers>=4.40.0",
    "numpy>=1.26.0",
]

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "ANN", "B", "A", "SIM"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

### Step 3: Setup Environment

```bash
# Install dependencies
uv sync

# Copy environment template
cp .env.example .env

# Edit .env with your local settings
```

### Step 4: Implement Your Service

1. Define your request/response schemas in `app/schemas/`
2. Implement your inference/business logic in `app/services/`
3. Wire up routes in `app/api/routes/`
4. Add model loading to `app/core/lifespan.py`
5. Update config if needed in `app/core/config.py`

### Step 5: Run and Test

```bash
make run          # Start dev server
make test         # Run tests
make lint         # Check code quality
make docker-build # Build container
```

---

## Configuration System

All configuration is centralized in `app/core/config.py` using `pydantic-settings`.

### Example Implementation

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field


class Settings(BaseSettings):
    """Application settings loaded from environment variables."""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    # Application
    app_name: str = "ml-api-template"
    environment: str = Field(default="dev", description="dev | staging | prod")
    debug: bool = False
    version: str = "0.1.0"

    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 1

    # Logging
    log_level: str = "INFO"
    log_format: str = "json"  # "json" for prod, "console" for local dev

    # Model (example — customize per project)
    model_path: str = "./models/model.pt"
    model_device: str = "cpu"  # "cpu" | "cuda" | "mps"
    model_batch_size: int = 32

    # External Services (example)
    redis_url: str = "redis://localhost:6379"
    database_url: str = "postgresql://localhost:5432/app"

    # Observability
    enable_metrics: bool = True
    enable_tracing: bool = False
    otlp_endpoint: str = "http://localhost:4317"


# Singleton — instantiate once, import everywhere
settings = Settings()
```

### Usage Across the Codebase

```python
from app.core.config import settings

# Access any setting with type safety
print(settings.app_name)
print(settings.model_path)
print(settings.log_level)
```

### Environment Files

```bash
# .env.example (committed to git — shows required variables)
APP_NAME=ml-api-template
ENVIRONMENT=dev
DEBUG=true
LOG_LEVEL=DEBUG
LOG_FORMAT=console
HOST=0.0.0.0
PORT=8000
MODEL_PATH=./models/model.pt
MODEL_DEVICE=cpu
ENABLE_METRICS=true
ENABLE_TRACING=false
```

**Rules:**
- `.env` is git-ignored (contains local/secret values)
- `.env.example` is committed (documents required variables)
- Never use `os.getenv()` directly — always go through `settings`
- Add new config fields to `Settings` class with type annotations and defaults


---

## Logging & Observability

### Structured Logging with structlog

All logging uses `structlog` to emit JSON-structured logs with automatic context binding.

```python
# app/core/logging.py
import structlog
from app.core.config import settings


def setup_logging() -> None:
    """Configure structured logging for the application."""

    processors = [
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if settings.log_format == "json":
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )


def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    """Get a named logger instance."""
    return structlog.get_logger(name)
```

### Using the Logger

```python
from app.core.logging import get_logger

logger = get_logger(__name__)

# Basic logging
logger.info("model_loaded", model_name="resnet50", device="cuda", load_time_ms=1200)

# With bound context (all subsequent logs include these fields)
log = logger.bind(user_id="abc123", request_id="req-456")
log.info("prediction_started")
log.info("prediction_complete", latency_ms=45, confidence=0.92)
```

### Log Output (JSON format — production)

```json
{
  "timestamp": "2026-05-23T15:30:00.000Z",
  "level": "info",
  "logger": "app.services.inference",
  "event": "prediction_complete",
  "request_id": "req-456",
  "user_id": "abc123",
  "latency_ms": 45,
  "confidence": 0.92
}
```

### Prometheus Metrics

```python
# app/observability/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["method", "endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

# Inference metrics
INFERENCE_LATENCY = Histogram(
    "model_inference_duration_seconds",
    "Model inference latency",
    ["model_name"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
)

INFERENCE_COUNT = Counter(
    "model_inference_total",
    "Total model inference calls",
    ["model_name", "status"],
)

MODEL_LOADED = Gauge(
    "model_loaded",
    "Whether the model is loaded and ready",
    ["model_name"],
)
```

### OpenTelemetry Tracing

```python
# app/observability/tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from app.core.config import settings


def setup_tracing() -> None:
    """Initialize OpenTelemetry tracing."""
    if not settings.enable_tracing:
        return

    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint=settings.otlp_endpoint)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)


def get_tracer(name: str) -> trace.Tracer:
    """Get a named tracer instance."""
    return trace.get_tracer(name)
```


---

## Writing Your First Endpoint

Here's the complete flow for adding a new endpoint — from schema to route to service.

### Step 1: Define Schemas

```python
# app/schemas/prediction.py
from pydantic import BaseModel, Field


class PredictionRequest(BaseModel):
    """Input schema for prediction endpoint."""
    text: str = Field(..., min_length=1, max_length=10000, description="Input text to classify")
    top_k: int = Field(default=3, ge=1, le=10, description="Number of top predictions to return")


class PredictionResult(BaseModel):
    """Single prediction result."""
    label: str
    confidence: float = Field(..., ge=0.0, le=1.0)


class PredictionResponse(BaseModel):
    """Output schema for prediction endpoint."""
    request_id: str
    predictions: list[PredictionResult]
    model_name: str
    latency_ms: float
```

### Step 2: Implement Service Logic

```python
# app/services/prediction_service.py
import time
from app.core.logging import get_logger
from app.observability.metrics import INFERENCE_LATENCY, INFERENCE_COUNT

logger = get_logger(__name__)


class PredictionService:
    """Handles prediction logic — decoupled from HTTP layer."""

    def __init__(self, model, model_name: str):
        self.model = model
        self.model_name = model_name

    async def predict(self, text: str, top_k: int = 3) -> list[dict]:
        """Run inference and return top-k predictions."""
        start = time.perf_counter()

        try:
            # Run blocking inference in thread pool
            import asyncio
            results = await asyncio.to_thread(self._run_inference, text, top_k)

            latency = (time.perf_counter() - start) * 1000
            INFERENCE_LATENCY.labels(model_name=self.model_name).observe(latency / 1000)
            INFERENCE_COUNT.labels(model_name=self.model_name, status="success").inc()

            logger.info(
                "inference_complete",
                model_name=self.model_name,
                latency_ms=round(latency, 2),
                top_prediction=results[0]["label"] if results else None,
            )
            return results

        except Exception as e:
            INFERENCE_COUNT.labels(model_name=self.model_name, status="error").inc()
            logger.error("inference_failed", model_name=self.model_name, error=str(e))
            raise

    def _run_inference(self, text: str, top_k: int) -> list[dict]:
        """Blocking inference call — runs in thread pool."""
        # Replace with your actual model inference
        outputs = self.model(text)
        # Sort by confidence, return top_k
        sorted_outputs = sorted(outputs, key=lambda x: x["confidence"], reverse=True)
        return sorted_outputs[:top_k]
```

### Step 3: Create the Route

```python
# app/api/routes/prediction.py
import time
import uuid
from fastapi import APIRouter, Depends, Request
from app.schemas.prediction import PredictionRequest, PredictionResponse, PredictionResult
from app.api.dependencies.services import get_prediction_service
from app.services.prediction_service import PredictionService

router = APIRouter(prefix="/v1", tags=["prediction"])


@router.post("/predict", response_model=PredictionResponse)
async def predict(
    request: Request,
    body: PredictionRequest,
    service: PredictionService = Depends(get_prediction_service),
) -> PredictionResponse:
    """Run model inference on input text."""
    start = time.perf_counter()
    request_id = request.state.request_id  # Set by middleware

    results = await service.predict(text=body.text, top_k=body.top_k)

    latency_ms = (time.perf_counter() - start) * 1000

    return PredictionResponse(
        request_id=request_id,
        predictions=[PredictionResult(**r) for r in results],
        model_name=service.model_name,
        latency_ms=round(latency_ms, 2),
    )
```

### Step 4: Wire Up Dependencies

```python
# app/api/dependencies/services.py
from fastapi import Request
from app.services.prediction_service import PredictionService


def get_prediction_service(request: Request) -> PredictionService:
    """Retrieve the prediction service from app state."""
    return request.app.state.prediction_service
```

### Step 5: Register the Route

```python
# app/main.py
from fastapi import FastAPI
from app.core.lifespan import lifespan
from app.api.routes import prediction, health


def create_app() -> FastAPI:
    app = FastAPI(
        title="My ML Service",
        version="0.1.0",
        lifespan=lifespan,
    )

    # Register routes
    app.include_router(health.router)
    app.include_router(prediction.router)

    return app


app = create_app()
```


---

## Inference Abstraction — Serving Models

The template uses an abstraction layer so you can swap model runtimes (PyTorch, ONNX, TensorRT, vLLM) without rewriting API logic.

### Base Interface

```python
# app/services/inference/base.py
from abc import ABC, abstractmethod
from typing import Any


class BaseInferenceEngine(ABC):
    """Abstract base for all inference engines."""

    @abstractmethod
    async def load(self) -> None:
        """Load model into memory. Called during app startup."""
        ...

    @abstractmethod
    async def predict(self, inputs: Any) -> Any:
        """Run inference on inputs."""
        ...

    @abstractmethod
    async def unload(self) -> None:
        """Release model resources. Called during app shutdown."""
        ...

    @property
    @abstractmethod
    def is_ready(self) -> bool:
        """Whether the model is loaded and ready for inference."""
        ...
```

### PyTorch Implementation Example

```python
# app/services/inference/pytorch_engine.py
import asyncio
import torch
from pathlib import Path
from app.services.inference.base import BaseInferenceEngine
from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)


class PyTorchEngine(BaseInferenceEngine):
    """PyTorch model inference engine."""

    def __init__(self, model_path: str, device: str = "cpu"):
        self.model_path = Path(model_path)
        self.device = torch.device(device)
        self.model = None
        self._ready = False

    async def load(self) -> None:
        """Load PyTorch model from disk."""
        logger.info("loading_model", path=str(self.model_path), device=str(self.device))

        def _load():
            model = torch.jit.load(str(self.model_path), map_location=self.device)
            model.eval()
            return model

        self.model = await asyncio.to_thread(_load)
        self._ready = True
        logger.info("model_loaded", path=str(self.model_path))

    async def predict(self, inputs: torch.Tensor) -> torch.Tensor:
        """Run inference."""
        if not self._ready:
            raise RuntimeError("Model not loaded")

        def _infer():
            with torch.no_grad():
                return self.model(inputs.to(self.device))

        return await asyncio.to_thread(_infer)

    async def unload(self) -> None:
        """Release GPU memory."""
        if self.model is not None:
            del self.model
            if self.device.type == "cuda":
                torch.cuda.empty_cache()
            self._ready = False
            logger.info("model_unloaded")

    @property
    def is_ready(self) -> bool:
        return self._ready
```

### HuggingFace Transformers Example

```python
# app/services/inference/hf_engine.py
import asyncio
from transformers import pipeline
from app.services.inference.base import BaseInferenceEngine
from app.core.logging import get_logger

logger = get_logger(__name__)


class HuggingFaceEngine(BaseInferenceEngine):
    """HuggingFace pipeline inference engine."""

    def __init__(self, model_name: str, task: str = "text-classification", device: str = "cpu"):
        self.model_name = model_name
        self.task = task
        self.device = 0 if device == "cuda" else -1
        self.pipe = None
        self._ready = False

    async def load(self) -> None:
        def _load():
            return pipeline(self.task, model=self.model_name, device=self.device)

        logger.info("loading_hf_model", model=self.model_name, task=self.task)
        self.pipe = await asyncio.to_thread(_load)
        self._ready = True
        logger.info("hf_model_loaded", model=self.model_name)

    async def predict(self, inputs: str | list[str]) -> list[dict]:
        if not self._ready:
            raise RuntimeError("Model not loaded")
        return await asyncio.to_thread(self.pipe, inputs)

    async def unload(self) -> None:
        del self.pipe
        self._ready = False

    @property
    def is_ready(self) -> bool:
        return self._ready
```

### Wiring Into Lifespan

```python
# app/core/lifespan.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.core.config import settings
from app.core.logging import setup_logging, get_logger
from app.services.inference.pytorch_engine import PyTorchEngine
from app.services.prediction_service import PredictionService
from app.observability.metrics import MODEL_LOADED

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application startup and shutdown lifecycle."""

    # --- STARTUP ---
    setup_logging()
    logger.info("app_starting", environment=settings.environment)

    # Load model
    engine = PyTorchEngine(
        model_path=settings.model_path,
        device=settings.model_device,
    )
    await engine.load()
    MODEL_LOADED.labels(model_name="primary").set(1)

    # Create services and attach to app state
    app.state.inference_engine = engine
    app.state.prediction_service = PredictionService(
        model=engine,
        model_name="primary",
    )

    logger.info("app_started", environment=settings.environment)

    yield  # App is running

    # --- SHUTDOWN ---
    logger.info("app_shutting_down")
    await engine.unload()
    MODEL_LOADED.labels(model_name="primary").set(0)
    logger.info("app_stopped")
```

**Key points:**
- Models are loaded ONCE at startup, stored in `app.state`
- Services are created with their dependencies and also stored in `app.state`
- Shutdown gracefully releases resources (GPU memory, connections)
- The `yield` separates startup from shutdown logic


---

## Middleware & Cross-Cutting Concerns

### Request Context Middleware

Every request gets a unique ID for tracing through logs:

```python
# app/middleware/request_context.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
import structlog


class RequestContextMiddleware(BaseHTTPMiddleware):
    """Injects request_id into every request and binds it to log context."""

    async def dispatch(self, request: Request, call_next) -> Response:
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id

        # Bind to structlog context (all logs in this request will include it)
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=request_id)

        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

### Logging Middleware

Automatically logs every request with latency:

```python
# app/middleware/logging_middleware.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
from app.core.logging import get_logger
from app.observability.metrics import REQUEST_COUNT, REQUEST_LATENCY

logger = get_logger(__name__)


class LoggingMiddleware(BaseHTTPMiddleware):
    """Logs request/response details and records metrics."""

    async def dispatch(self, request: Request, call_next) -> Response:
        start = time.perf_counter()

        response = await call_next(request)

        latency = (time.perf_counter() - start) * 1000
        status = response.status_code
        method = request.method
        path = request.url.path

        # Log
        logger.info(
            "request_complete",
            method=method,
            path=path,
            status_code=status,
            latency_ms=round(latency, 2),
        )

        # Metrics
        REQUEST_COUNT.labels(method=method, endpoint=path, status_code=status).inc()
        REQUEST_LATENCY.labels(method=method, endpoint=path).observe(latency / 1000)

        return response
```

### Registering Middleware

```python
# In app/main.py create_app()
from app.middleware.request_context import RequestContextMiddleware
from app.middleware.logging_middleware import LoggingMiddleware

def create_app() -> FastAPI:
    app = FastAPI(...)

    # Middleware executes in reverse order (last added = first executed)
    app.add_middleware(LoggingMiddleware)
    app.add_middleware(RequestContextMiddleware)

    # ... routes ...
    return app
```

---

## Health Checks

Two separate endpoints for different purposes:

```python
# app/api/routes/health.py
from fastapi import APIRouter, Request
from app.schemas.health import HealthResponse, ReadinessResponse

router = APIRouter(prefix="/health", tags=["health"])


@router.get("/live", response_model=HealthResponse)
async def liveness() -> HealthResponse:
    """Liveness probe — is the process running?
    
    Used by: Kubernetes liveness probe, load balancer health checks.
    Should NEVER check external dependencies.
    """
    return HealthResponse(status="alive")


@router.get("/ready", response_model=ReadinessResponse)
async def readiness(request: Request) -> ReadinessResponse:
    """Readiness probe — is the app ready to serve traffic?
    
    Used by: Kubernetes readiness probe, deployment rollouts.
    Checks that all critical dependencies are initialized.
    """
    engine = request.app.state.inference_engine
    checks = {
        "model_loaded": engine.is_ready,
    }
    all_ready = all(checks.values())

    return ReadinessResponse(
        status="ready" if all_ready else "not_ready",
        checks=checks,
    )
```

```python
# app/schemas/health.py
from pydantic import BaseModel


class HealthResponse(BaseModel):
    status: str  # "alive"


class ReadinessResponse(BaseModel):
    status: str  # "ready" | "not_ready"
    checks: dict[str, bool]
```

| Endpoint | Purpose | What it checks | When it fails |
|----------|---------|----------------|---------------|
| `GET /health/live` | Process alive | Nothing (always 200 if process runs) | Process crashed |
| `GET /health/ready` | Ready for traffic | Model loaded, DB connected, etc. | During startup, during model reload |

---

## Testing Strategy

### Test Structure

```
tests/
├── conftest.py          # Shared fixtures
├── unit/                # Fast, no I/O, no network
│   ├── test_services.py
│   └── test_schemas.py
└── integration/         # Hits real HTTP endpoints
    └── test_api.py
```

### Shared Fixtures

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from unittest.mock import AsyncMock
from app.main import create_app


@pytest.fixture
def app():
    """Create a test app instance."""
    application = create_app()
    # Mock the model for tests
    mock_engine = AsyncMock()
    mock_engine.is_ready = True
    mock_engine.predict.return_value = [{"label": "positive", "confidence": 0.95}]
    application.state.inference_engine = mock_engine
    return application


@pytest.fixture
async def client(app):
    """Async test client."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
```

### Unit Test Example

```python
# tests/unit/test_services.py
import pytest
from unittest.mock import AsyncMock
from app.services.prediction_service import PredictionService


@pytest.mark.asyncio
async def test_prediction_service_returns_top_k():
    """Service should return only top_k results sorted by confidence."""
    mock_model = AsyncMock()
    mock_model.return_value = [
        {"label": "cat", "confidence": 0.9},
        {"label": "dog", "confidence": 0.7},
        {"label": "bird", "confidence": 0.3},
    ]

    service = PredictionService(model=mock_model, model_name="test")
    results = await service.predict("a photo of a cat", top_k=2)

    assert len(results) == 2
    assert results[0]["label"] == "cat"
    assert results[0]["confidence"] > results[1]["confidence"]
```

### Integration Test Example

```python
# tests/integration/test_api.py
import pytest


@pytest.mark.asyncio
async def test_predict_endpoint(client):
    """POST /v1/predict should return predictions."""
    response = await client.post("/v1/predict", json={
        "text": "This movie was fantastic!",
        "top_k": 3,
    })

    assert response.status_code == 200
    data = response.json()
    assert "predictions" in data
    assert "request_id" in data
    assert "latency_ms" in data


@pytest.mark.asyncio
async def test_health_live(client):
    """GET /health/live should always return 200."""
    response = await client.get("/health/live")
    assert response.status_code == 200
    assert response.json()["status"] == "alive"


@pytest.mark.asyncio
async def test_health_ready(client):
    """GET /health/ready should report model status."""
    response = await client.get("/health/ready")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "ready"
    assert data["checks"]["model_loaded"] is True
```

### Testing Rules

1. **Unit tests** test business logic in isolation — no FastAPI, no HTTP, no I/O
2. **Integration tests** test the full HTTP flow with mocked external dependencies
3. **Never test framework internals** — test YOUR logic
4. **Mock at the boundary** — mock the model, not the service


---

## Docker & Deployment

### Dockerfile (Multi-Stage Build)

```dockerfile
# docker/Dockerfile

# --- Stage 1: Build ---
FROM python:3.11-slim AS builder

WORKDIR /app

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Install dependencies (cached layer)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-editable

# --- Stage 2: Runtime ---
FROM python:3.11-slim AS runtime

WORKDIR /app

# Create non-root user
RUN adduser --disabled-password --no-create-home appuser

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"

# Copy application code
COPY app/ ./app/

# Switch to non-root
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/health/live').raise_for_status()"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### GPU Dockerfile Variant

For services that need GPU inference, use NVIDIA base images:

```dockerfile
# docker/Dockerfile.gpu
FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04 AS runtime

# Install Python
RUN apt-get update && apt-get install -y python3.11 python3-pip && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
COPY app/ ./app/

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yaml (Local Development)

```yaml
# docker-compose.yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - "8000:8000"
    env_file: .env
    volumes:
      - ./app:/app/app  # Hot reload in dev
      - ./models:/app/models
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

### .dockerignore

```
# docker/.dockerignore
.git
.env
__pycache__
*.pyc
.pytest_cache
.mypy_cache
.ruff_cache
tests/
notebooks/
*.md
.venv
```

---

## Makefile Commands

The Makefile is your single interface for all developer workflows:

```makefile
# Makefile
.PHONY: install run lint format typecheck test docker-build docker-run clean

install:
	uv sync

install-dev:
	uv sync --all-extras

run:
	uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

lint:
	uv run ruff check .

format:
	uv run ruff format .

typecheck:
	uv run mypy app

test:
	uv run pytest -v

test-cov:
	uv run pytest --cov=app --cov-report=term-missing

docker-build:
	docker build -f docker/Dockerfile -t ml-api-template .

docker-run:
	docker run -p 8000:8000 --env-file .env ml-api-template

up:
	docker compose up -d

down:
	docker compose down

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type d -name .pytest_cache -exec rm -rf {} +
	find . -type d -name .mypy_cache -exec rm -rf {} +
```

| Command | What it does |
|---------|-------------|
| `make install` | Install production dependencies |
| `make install-dev` | Install all dependencies including dev tools |
| `make run` | Start local dev server with hot reload |
| `make lint` | Check code quality (Ruff) |
| `make format` | Auto-format code (Ruff) |
| `make typecheck` | Run static type checking (mypy) |
| `make test` | Run all tests |
| `make test-cov` | Run tests with coverage report |
| `make docker-build` | Build production Docker image |
| `make docker-run` | Run the Docker image locally |
| `make up` | Start full local stack (app + Redis + monitoring) |
| `make down` | Stop local stack |
| `make clean` | Remove cache directories |

---

## Common Use Cases (With Examples)

### Use Case 1: Classical ML Model (scikit-learn)

Serving a trained scikit-learn model (e.g., fraud detection, churn prediction):

```python
# app/services/inference/sklearn_engine.py
import asyncio
import joblib
from pathlib import Path
from app.services.inference.base import BaseInferenceEngine
from app.core.logging import get_logger

logger = get_logger(__name__)


class SklearnEngine(BaseInferenceEngine):
    def __init__(self, model_path: str):
        self.model_path = Path(model_path)
        self.model = None
        self._ready = False

    async def load(self) -> None:
        self.model = await asyncio.to_thread(joblib.load, str(self.model_path))
        self._ready = True
        logger.info("sklearn_model_loaded", path=str(self.model_path))

    async def predict(self, features: list[list[float]]) -> list[dict]:
        import numpy as np

        def _predict():
            X = np.array(features)
            probas = self.model.predict_proba(X)
            classes = self.model.classes_
            results = []
            for proba in probas:
                top_idx = proba.argmax()
                results.append({
                    "label": str(classes[top_idx]),
                    "confidence": float(proba[top_idx]),
                })
            return results

        return await asyncio.to_thread(_predict)

    async def unload(self) -> None:
        del self.model
        self._ready = False

    @property
    def is_ready(self) -> bool:
        return self._ready
```

### Use Case 2: RAG API (Retrieval-Augmented Generation)

```python
# app/services/rag_service.py
from app.core.logging import get_logger

logger = get_logger(__name__)


class RAGService:
    """Retrieval-Augmented Generation service."""

    def __init__(self, retriever, llm_client):
        self.retriever = retriever
        self.llm_client = llm_client

    async def answer(self, query: str, top_k: int = 5) -> dict:
        """Retrieve context and generate answer."""

        # Step 1: Retrieve relevant documents
        documents = await self.retriever.retrieve(query, top_k=top_k)
        logger.info("documents_retrieved", count=len(documents), query=query[:50])

        # Step 2: Build prompt with context
        context = "\n\n".join([doc["text"] for doc in documents])
        prompt = f"""Answer the question based on the context below.

Context:
{context}

Question: {query}

Answer:"""

        # Step 3: Generate response
        response = await self.llm_client.generate(prompt)
        logger.info("response_generated", tokens=response.get("usage", {}).get("total_tokens"))

        return {
            "answer": response["text"],
            "sources": [doc["metadata"] for doc in documents],
            "model": response.get("model"),
        }
```

```python
# app/schemas/rag.py
from pydantic import BaseModel, Field


class RAGRequest(BaseModel):
    query: str = Field(..., min_length=1, max_length=2000)
    top_k: int = Field(default=5, ge=1, le=20)


class RAGResponse(BaseModel):
    answer: str
    sources: list[dict]
    model: str | None = None
    request_id: str
```

### Use Case 3: LLM Gateway / Agent Backend

```python
# app/services/llm_service.py
from abc import ABC, abstractmethod


class BaseLLMProvider(ABC):
    """Abstract LLM provider — swap between OpenAI, Anthropic, local vLLM."""

    @abstractmethod
    async def generate(self, prompt: str, **kwargs) -> dict:
        ...


class OpenAIProvider(BaseLLMProvider):
    def __init__(self, api_key: str, model: str = "gpt-4o"):
        import openai
        self.client = openai.AsyncOpenAI(api_key=api_key)
        self.model = model

    async def generate(self, prompt: str, **kwargs) -> dict:
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            **kwargs,
        )
        return {
            "text": response.choices[0].message.content,
            "model": response.model,
            "usage": {
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens,
            },
        }


class VLLMProvider(BaseLLMProvider):
    """Local vLLM server — same interface, different backend."""

    def __init__(self, base_url: str = "http://localhost:8001"):
        import httpx
        self.client = httpx.AsyncClient(base_url=base_url)

    async def generate(self, prompt: str, **kwargs) -> dict:
        response = await self.client.post("/v1/completions", json={
            "prompt": prompt,
            "max_tokens": kwargs.get("max_tokens", 512),
        })
        data = response.json()
        return {
            "text": data["choices"][0]["text"],
            "model": data.get("model"),
            "usage": data.get("usage", {}),
        }
```

### Use Case 4: Batch Inference Endpoint

```python
# app/api/routes/batch.py
from fastapi import APIRouter, Request, BackgroundTasks
from pydantic import BaseModel

router = APIRouter(prefix="/v1", tags=["batch"])


class BatchRequest(BaseModel):
    inputs: list[str]
    webhook_url: str | None = None  # Optional callback when done


class BatchResponse(BaseModel):
    job_id: str
    status: str
    total_items: int


@router.post("/batch/predict", response_model=BatchResponse)
async def batch_predict(
    body: BatchRequest,
    request: Request,
    background_tasks: BackgroundTasks,
):
    """Submit a batch prediction job."""
    import uuid

    job_id = str(uuid.uuid4())

    # Run in background
    background_tasks.add_task(
        run_batch_inference,
        job_id=job_id,
        inputs=body.inputs,
        engine=request.app.state.inference_engine,
        webhook_url=body.webhook_url,
    )

    return BatchResponse(
        job_id=job_id,
        status="accepted",
        total_items=len(body.inputs),
    )


async def run_batch_inference(job_id: str, inputs: list, engine, webhook_url: str | None):
    """Background task for batch processing."""
    from app.core.logging import get_logger
    logger = get_logger(__name__)

    logger.info("batch_started", job_id=job_id, total=len(inputs))

    results = []
    for i, inp in enumerate(inputs):
        result = await engine.predict(inp)
        results.append(result)

    logger.info("batch_complete", job_id=job_id, processed=len(results))

    # Optionally notify via webhook
    if webhook_url:
        import httpx
        async with httpx.AsyncClient() as client:
            await client.post(webhook_url, json={"job_id": job_id, "results": results})
```


---

## Exception Handling

Centralized exception handling ensures consistent error responses:

```python
# app/core/exceptions.py
from fastapi import Request
from fastapi.responses import JSONResponse
from app.core.logging import get_logger

logger = get_logger(__name__)


class AppException(Exception):
    """Base application exception."""
    def __init__(self, message: str, status_code: int = 500, error_code: str = "INTERNAL_ERROR"):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code


class ModelNotReadyError(AppException):
    def __init__(self):
        super().__init__(
            message="Model is not loaded or not ready for inference",
            status_code=503,
            error_code="MODEL_NOT_READY",
        )


class InvalidInputError(AppException):
    def __init__(self, message: str):
        super().__init__(message=message, status_code=422, error_code="INVALID_INPUT")


# Global exception handler — register in main.py
async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    logger.warning(
        "app_exception",
        error_code=exc.error_code,
        message=exc.message,
        path=request.url.path,
    )
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code,
            "message": exc.message,
            "request_id": getattr(request.state, "request_id", None),
        },
    )
```

Register in `main.py`:

```python
from app.core.exceptions import AppException, app_exception_handler

def create_app() -> FastAPI:
    app = FastAPI(...)
    app.add_exception_handler(AppException, app_exception_handler)
    # ...
    return app
```

---

## Anti-Patterns to Avoid

### ❌ Loading models per request

```python
# BAD — loads model on every request (slow, wastes memory)
@app.post("/predict")
async def predict(body: PredictionRequest):
    model = load_model("model.pt")  # ← NEVER do this
    return model.predict(body.text)
```

**Fix:** Load in lifespan, access via `app.state`.

---

### ❌ Blocking the event loop

```python
# BAD — requests.get is blocking, freezes all concurrent requests
@app.post("/predict")
async def predict():
    response = requests.get("http://external-api/data")  # ← blocks event loop
    model_output = model.predict(response.json())  # ← also blocking
```

**Fix:** Use `httpx` (async) for HTTP calls, `asyncio.to_thread()` for CPU-bound work:

```python
# GOOD
import httpx
import asyncio

@app.post("/predict")
async def predict():
    async with httpx.AsyncClient() as client:
        response = await client.get("http://external-api/data")
    result = await asyncio.to_thread(model.predict, response.json())
    return result
```

---

### ❌ Scattered configuration

```python
# BAD — os.getenv everywhere, no validation, no defaults
model_path = os.getenv("MODEL_PATH")  # Could be None!
batch_size = int(os.getenv("BATCH_SIZE"))  # Crashes if missing
```

**Fix:** Use the centralized `settings` object:

```python
# GOOD
from app.core.config import settings
model_path = settings.model_path  # Typed, validated, has default
```

---

### ❌ Business logic in route handlers

```python
# BAD — route handler does everything
@app.post("/predict")
async def predict(body: Request):
    # preprocessing
    tokens = tokenizer(body.text)
    # inference
    output = model(tokens)
    # postprocessing
    label = id2label[output.argmax()]
    # logging
    logger.info(f"predicted {label}")
    return {"label": label}
```

**Fix:** Route handlers only handle HTTP concerns. Logic lives in services:

```python
# GOOD
@app.post("/predict")
async def predict(body: PredictionRequest, service = Depends(get_prediction_service)):
    result = await service.predict(body.text)
    return PredictionResponse(**result)
```

---

### ❌ No error handling on inference

```python
# BAD — unhandled exceptions return 500 with no useful info
@app.post("/predict")
async def predict(body: Request):
    return model.predict(body.text)
```

**Fix:** Catch, log, and return structured errors:

```python
# GOOD
from app.core.exceptions import ModelNotReadyError

async def predict(self, text: str):
    if not self.engine.is_ready:
        raise ModelNotReadyError()
    try:
        return await self.engine.predict(text)
    except Exception as e:
        logger.error("inference_failed", error=str(e))
        raise AppException(message="Inference failed", error_code="INFERENCE_ERROR")
```

---

### ❌ Print statements instead of structured logging

```python
# BAD
print(f"Processing request for user {user_id}")
print(f"Model took {time.time() - start}s")
```

**Fix:** Use structlog with context:

```python
# GOOD
logger.info("processing_request", user_id=user_id)
logger.info("inference_complete", latency_ms=round(latency, 2))
```

---

### ❌ No health checks

If your service has no `/health/ready` endpoint, deployments can route traffic to instances that haven't finished loading models.

**Fix:** Always implement both `/health/live` and `/health/ready`.

---

## FAQ

### Q: How do I add a new ML model to an existing service?

1. Create a new engine class implementing `BaseInferenceEngine`
2. Load it in `app/core/lifespan.py` during startup
3. Create a service class that uses the engine
4. Add the service to `app.state`
5. Create a dependency function in `app/api/dependencies/`
6. Add routes in `app/api/routes/`

### Q: How do I handle multiple models in one service?

Store them as a dictionary in `app.state`:

```python
# In lifespan
app.state.models = {
    "sentiment": sentiment_engine,
    "ner": ner_engine,
}

# In dependency
def get_engine(model_name: str, request: Request):
    engine = request.app.state.models.get(model_name)
    if not engine:
        raise InvalidInputError(f"Unknown model: {model_name}")
    return engine
```

### Q: How do I add authentication?

Create a dependency in `app/api/dependencies/auth.py`:

```python
from fastapi import Depends, HTTPException, Header

async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != settings.api_key:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return x_api_key

# Use in routes:
@router.post("/predict", dependencies=[Depends(verify_api_key)])
async def predict(...):
    ...
```

### Q: How do I add a database?

1. Add connection string to `app/core/config.py`
2. Initialize connection pool in `app/core/lifespan.py`
3. Store pool in `app.state`
4. Create a dependency that yields sessions
5. Check DB connectivity in `/health/ready`

### Q: Can I use this for gRPC instead of REST?

Yes. Because business logic lives in `app/services/`, you can add a gRPC transport layer alongside the HTTP one without rewriting any logic.

### Q: How do I version my API?

Use route prefixes:

```python
router_v1 = APIRouter(prefix="/v1")
router_v2 = APIRouter(prefix="/v2")

app.include_router(router_v1)
app.include_router(router_v2)
```

### Q: What about async vs sync model inference?

- If your model is **CPU-bound** (most ML models): wrap in `asyncio.to_thread()`
- If your model has an **async API** (e.g., calling an external LLM API): use `await` directly
- If using **GPU**: wrap in thread pool — PyTorch operations release the GIL during CUDA execution

### Q: How do I add WebSocket support (e.g., streaming LLM responses)?

```python
# app/api/routes/stream.py
from fastapi import WebSocket

@router.websocket("/v1/stream")
async def stream_inference(websocket: WebSocket):
    await websocket.accept()
    async for chunk in llm_service.stream(prompt):
        await websocket.send_text(chunk)
    await websocket.close()
```

---

## Quick Reference — File Responsibilities

| File | Single Responsibility |
|------|----------------------|
| `app/main.py` | Create FastAPI app, register middleware & routes |
| `app/core/config.py` | Define all settings, load from env |
| `app/core/logging.py` | Configure structlog, provide `get_logger()` |
| `app/core/lifespan.py` | Startup (load models, connect DBs) / Shutdown (cleanup) |
| `app/core/exceptions.py` | Custom exceptions + global error handlers |
| `app/middleware/request_context.py` | Inject request ID, bind to log context |
| `app/middleware/logging_middleware.py` | Log every request with latency + record metrics |
| `app/api/routes/*.py` | HTTP handlers — validate input, call service, return response |
| `app/api/dependencies/*.py` | FastAPI `Depends()` — provide services, auth, DB sessions |
| `app/services/*.py` | Business logic — orchestration, inference, workflows |
| `app/schemas/*.py` | Pydantic models — request/response contracts |
| `app/observability/metrics.py` | Define Prometheus counters, histograms, gauges |
| `app/observability/tracing.py` | OpenTelemetry setup and tracer factory |
| `app/utils/*.py` | Pure helper functions (no side effects, no state) |

---

## Checklist Before Deploying

- [ ] All tests pass (`make test`)
- [ ] No lint errors (`make lint`)
- [ ] Type checks pass (`make typecheck`)
- [ ] Docker image builds successfully (`make docker-build`)
- [ ] `/health/live` returns 200
- [ ] `/health/ready` returns 200 (model loaded)
- [ ] Structured logs are emitting correctly
- [ ] Prometheus metrics are exposed (if enabled)
- [ ] `.env.example` is updated with any new variables
- [ ] API docs are accessible at `/docs`
- [ ] Error responses follow the standard format

---

## Contributing to This Template

If you find patterns that should be standardized:

1. Open an issue describing the pattern
2. Propose the change with example code
3. Ensure it doesn't break existing projects using the template
4. Update this onboarding document

The template should evolve from **real patterns observed across projects**, not imagined abstractions.

---

## Concepts Deep Dive

This section explains the foundational concepts behind the template's design decisions.

---

### What is `pyproject.toml`?

It's the **single source of truth** for your Python project's metadata and tooling configuration. Before `pyproject.toml`, you needed separate files: `setup.py`, `setup.cfg`, `requirements.txt`, `MANIFEST.in`, plus individual config files for each tool (`.flake8`, `mypy.ini`, `pytest.ini`, etc.).

`pyproject.toml` replaces all of that. It tells:
- **Package managers** (uv, pip, poetry) what your project is and what it depends on
- **Build tools** how to package your code
- **Dev tools** (ruff, mypy, pytest) how to behave

Think of it as `package.json` for Python.

#### Sections Explained

```toml
[project]                    # WHO you are
name = "ml-api-template"     # Package name
version = "0.1.0"            # Semantic version
description = "..."          # What this project does
requires-python = ">=3.11"   # Minimum Python version

dependencies = [             # WHAT you need to run (production deps)
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]  # EXTRA deps for specific contexts
dev = ["pytest>=8.0", "ruff>=0.5.0", "mypy>=1.10"]  # Only needed during development
ml = ["torch>=2.0", "transformers>=4.40"]            # Only needed for ML projects

[tool.ruff]                  # HOW ruff should lint/format your code
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I"]    # Which rules to enforce

[tool.mypy]                  # HOW mypy should type-check
python_version = "3.11"
strict = true

[tool.pytest.ini_options]    # HOW pytest should run tests
testpaths = ["tests"]
asyncio_mode = "auto"
```

| Section | Purpose |
|---------|---------|
| `[project]` | Identity + production dependencies |
| `[project.optional-dependencies]` | Grouped extras (install with `uv sync --extra dev`) |
| `[tool.ruff]` | Linting/formatting rules (replaces .flake8, black config) |
| `[tool.mypy]` | Type checking strictness |
| `[tool.pytest.ini_options]` | Test discovery and behavior |

The key insight: **one file configures everything**. New team members don't hunt for scattered config files.

---

### What Goes in `lifespan.py`?

`lifespan.py` manages **everything that needs to happen once at startup and cleanup at shutdown**. It's the application's boot sequence.

**What belongs here:**
- Loading ML models into memory/GPU
- Creating database connection pools
- Connecting to Redis
- Initializing vector DB clients
- Setting up OpenTelemetry exporters
- Warming up caches
- Creating service instances and attaching them to `app.state`

**What does NOT belong here:**
- Business logic
- Route definitions
- Per-request operations

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- STARTUP (runs once when server starts) ---
    setup_logging()
    
    engine = PyTorchEngine(model_path=settings.model_path)
    await engine.load()                    # Load model into GPU
    app.state.engine = engine              # Make it accessible to routes
    
    db_pool = await create_pool(settings.database_url)
    app.state.db = db_pool                 # DB connection pool
    
    yield  # ← App is running, serving requests
    
    # --- SHUTDOWN (runs when server stops) ---
    await engine.unload()                  # Free GPU memory
    await db_pool.close()                  # Close DB connections
```

**Why it matters:** Without this, teams either load models per-request (slow, wasteful) or use global variables (untestable, no cleanup). The lifespan pattern gives you controlled initialization with guaranteed cleanup.

---

### The Makefile — Developer Workflow Interface

The Makefile is a **menu of standardized commands** that every team member uses identically.

```makefile
install:          # Set up the project from scratch
    uv sync

run:              # Start local dev server with hot-reload
    uv run uvicorn app.main:app --reload

lint:             # Check code quality (CI will also run this)
    uv run ruff check .

format:           # Auto-fix formatting
    uv run ruff format .

typecheck:        # Static type analysis
    uv run mypy app

test:             # Run test suite
    uv run pytest

docker-build:     # Build production container
    docker build -f docker/Dockerfile -t ml-api-template .
```

**What we're achieving:**
1. **No tribal knowledge** — nobody needs to remember `uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000`. They just type `make run`.
2. **Consistency** — everyone runs the same commands. CI runs the same commands. No "works on my machine."
3. **Discoverability** — new team members type `make` + Tab and see all available workflows.
4. **Composability** — CI pipelines just call `make lint && make test && make docker-build`.

---

### The `dependencies/` Folder

`app/api/dependencies/` stores **FastAPI dependency injection functions** — reusable pieces that provide services, auth, DB sessions, or any shared resource to route handlers.

```python
# app/api/dependencies/services.py
def get_prediction_service(request: Request) -> PredictionService:
    """Provides the prediction service to any route that needs it."""
    return request.app.state.prediction_service

# app/api/dependencies/auth.py
async def verify_api_key(x_api_key: str = Header(...)) -> str:
    """Validates API key — reusable across all protected routes."""
    if x_api_key != settings.api_key:
        raise HTTPException(status_code=401)
    return x_api_key

# app/api/dependencies/database.py
async def get_db_session(request: Request):
    """Yields a DB session, auto-closes after request."""
    async with request.app.state.db.session() as session:
        yield session
```

**Usage in routes:**
```python
@router.post("/predict")
async def predict(
    body: PredictionRequest,
    service: PredictionService = Depends(get_prediction_service),  # injected
    _: str = Depends(verify_api_key),                              # auth check
):
    return await service.predict(body.text)
```

**Why a separate folder?** Dependencies are shared across multiple routes. Putting them in route files creates duplication. Putting them in services creates circular imports. The `dependencies/` folder is the clean middle ground.

---

### Middleware — Cross-Cutting Concerns

Middleware is code that **runs on every request/response** without you explicitly calling it in each route.

```
Request arrives
    → Middleware A (before)
        → Middleware B (before)
            → Your route handler
        → Middleware B (after)
    → Middleware A (after)
Response sent
```

**What middleware solves:** You have 50 endpoints. You want every single one to log request latency. Without middleware, you'd add logging code to all 50 handlers. With middleware, you write it once.

| Middleware | What it does |
|-----------|-------------|
| Request context | Assigns a unique ID to every request for tracing |
| Logging | Logs method, path, status, latency for every request |
| CORS | Handles cross-origin browser requests |
| Auth | Validates tokens before routes execute |
| Rate limiting | Throttles excessive requests |

**Key rule:** Middleware handles things that apply uniformly to all (or most) requests. Business logic never goes in middleware.

#### `request_context.py` — Identity

**Problem:** When debugging production issues, you see thousands of log lines. How do you find all logs belonging to one specific request?

**Solution:** Assign every request a unique `request_id` and attach it to all logs automatically.

What it does:
1. Reads `X-Request-ID` header (if caller sent one) or generates a UUID
2. Stores it in `request.state.request_id` (accessible in route handlers)
3. Binds it to structlog context (all logs in this request automatically include it)
4. Returns it in response header (so caller can reference it)

**Result:** Every log line from a single request shares the same `request_id`. You can filter in Grafana/Datadog by one ID and see the entire request lifecycle.

#### `logging_middleware.py` — Observability

**Problem:** You want to know for every request: what was called, how long it took, what status it returned — without adding logging code to every route.

What it does:
1. Records start time
2. Lets the request execute
3. Logs: method, path, status_code, latency_ms
4. Increments Prometheus counters (request count, latency histogram)

**Why both are needed:**
- `request_context.py` gives you **traceability** (which logs belong together)
- `logging_middleware.py` gives you **visibility** (what's happening, how fast)

They execute in order: request_context runs first (assigns ID), then logging_middleware runs (logs with that ID already bound).

---

### Dependency Injection — Why It's Required

**Dependency Injection** means: instead of a function creating or finding its own dependencies, they're **passed in from outside**.

**Without DI (tightly coupled):**
```python
@app.post("/predict")
async def predict(body: Request):
    model = load_model("model.pt")  # Handler knows HOW to get the model
    db = connect_to_db()            # Handler knows HOW to get DB
    result = model.predict(body.text)
    db.save(result)
    return result
```

Problems:
- Can't test without a real model and real DB
- Can't swap implementations (e.g., mock model for testing)
- Every route repeats the same setup code

**With DI (loosely coupled):**
```python
@app.post("/predict")
async def predict(
    body: Request,
    model: InferenceEngine = Depends(get_engine),   # Injected
    db: Session = Depends(get_db_session),          # Injected
):
    result = await model.predict(body.text)
    await db.save(result)
    return result
```

**Why it helps:**
1. **Testability** — In tests, you inject a mock model. No real GPU needed.
2. **Swappability** — Switch from PyTorch to ONNX by changing one dependency function, not 50 routes.
3. **Separation of concerns** — Routes don't know or care how models are loaded.
4. **Lifecycle management** — DB sessions auto-close, connections auto-return to pool.
5. **Reusability** — Auth check written once, injected into any route that needs it.

---

### REST vs gRPC — When to Use What

**Default: Use REST (FastAPI endpoints) when:**
- External clients consume your API (browsers, mobile apps, third-party integrations)
- You need auto-generated docs (Swagger/OpenAPI)
- Human-readable debugging matters (curl, Postman)
- Request/response payloads are moderate size

**Use gRPC when:**
- **Service-to-service communication** — your ML service is called by another internal service at high frequency
- **Low latency is critical** — gRPC uses HTTP/2 + protobuf (binary), significantly faster than JSON serialization
- **Streaming** — you need bidirectional streaming (e.g., streaming inference tokens, real-time audio)
- **Large payloads** — sending tensors, embeddings, or large feature vectors (protobuf is 3-10x smaller than JSON)
- **Polyglot environment** — callers are in Go, Java, C++ (gRPC has native codegen for all)

**Practical architecture:**

```
[Frontend] → REST → [API Gateway] → gRPC → [ML Inference Service]
                                   → gRPC → [Feature Store]
                                   → gRPC → [Embedding Service]
```

- External-facing: REST (human-friendly, documented)
- Internal model-to-model or service-to-service: gRPC (fast, typed, streaming)

This template uses REST by default because it's simpler, debuggable, and sufficient for most inference workloads. The layered architecture (services separate from transport) means you can add a gRPC transport layer later without rewriting business logic.

---

### The `scripts/` Folder

`scripts/` stores **operational and one-off automation scripts** that support the service but aren't part of the runtime application.

| Script | Purpose |
|--------|---------|
| `download_model.py` | Download model artifacts from S3/registry before deployment |
| `migrate_db.py` | Run database migrations |
| `seed_data.py` | Populate dev/test databases with sample data |
| `export_model.py` | Convert model to ONNX/TorchScript for serving |
| `benchmark.py` | Load test the inference endpoint |
| `generate_client.py` | Generate SDK from OpenAPI spec |
| `warmup.py` | Send warm-up requests after deployment |
| `evaluate.py` | Run offline evaluation on a test dataset |
| `setup_local.sh` | One-command local environment setup |

**What does NOT go here:**
- Application code (that's `app/`)
- Tests (that's `tests/`)
- CI/CD pipeline definitions (that's `.github/` or `.gitlab-ci.yml`)

**Example:**
```python
# scripts/download_model.py
"""Download model from S3 before container starts."""
import boto3
from app.core.config import settings

s3 = boto3.client("s3")
s3.download_file(settings.model_bucket, settings.model_key, settings.model_path)
print(f"Model downloaded to {settings.model_path}")
```

Called in Dockerfile or as init container:
```dockerfile
RUN python scripts/download_model.py
```

The key idea: scripts are **run manually or by CI/CD**, not by the running application. They support the lifecycle around the service (setup, migration, benchmarking, deployment prep).

---

## ML/DL Production Concerns

These are inference-specific concerns that go beyond standard web API practices. Every ML service built from this template should address these.

---

### 1. Model Warmup

The first inference after model load is always slower — CUDA kernel compilation, JIT tracing, memory allocation. Without warmup, your first real user gets 5-10x higher latency.

**Implementation in `lifespan.py`:**

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... load model ...
    await engine.load()

    # Warmup — forces CUDA compilation and buffer allocation
    dummy_input = "warmup request"
    await engine.predict(dummy_input)
    logger.info("model_warmup_complete", warmup_latency_ms=...)

    yield
```

**Rule:** Always warmup after loading. Include warmup latency in your startup health check — `/health/ready` should only return `true` after warmup completes.

---

### 2. Concurrency Control

GPU models cannot handle unlimited concurrent requests. If 100 requests hit a single GPU simultaneously, you get OOM or severe latency degradation.

**Implementation with semaphore:**

```python
# app/services/prediction_service.py
import asyncio

class PredictionService:
    def __init__(self, engine, max_concurrent: int = 4):
        self.engine = engine
        self._semaphore = asyncio.Semaphore(max_concurrent)

    async def predict(self, text: str) -> dict:
        async with self._semaphore:
            return await self.engine.predict(text)
```

| GPU | Recommended `max_concurrent` |
|-----|------------------------------|
| T4 (16GB) | 2-4 |
| A10G (24GB) | 4-8 |
| A100 (80GB) | 8-16 |

Tune based on your model size and batch behavior. Monitor GPU memory to find the sweet spot.

---

### 3. Inference Timeout

A single bad input (extremely long text, adversarial input) can hang inference indefinitely. Always wrap inference with a timeout:

```python
import asyncio
from app.core.exceptions import AppException

class PredictionService:
    async def predict(self, text: str, timeout: float = 30.0) -> dict:
        try:
            return await asyncio.wait_for(
                self.engine.predict(text),
                timeout=timeout,
            )
        except asyncio.TimeoutError:
            logger.error("inference_timeout", input_length=len(text))
            raise AppException(
                message="Inference timed out",
                status_code=504,
                error_code="INFERENCE_TIMEOUT",
            )
```

---

### 4. Input Validation (ML-Specific)

Pydantic validates structure, but ML needs domain-specific validation beyond schema:

```python
class PredictionService:
    MAX_TOKENS = 512
    MAX_BATCH_SIZE = 32

    async def predict(self, text: str) -> dict:
        # Token count validation (not just string length)
        token_count = self.tokenizer.count_tokens(text)
        if token_count > self.MAX_TOKENS:
            raise InvalidInputError(
                f"Input has {token_count} tokens, maximum is {self.MAX_TOKENS}"
            )

        return await self.engine.predict(text)
```

| Input type | What to validate |
|-----------|-----------------|
| Text | Token count, encoding, language |
| Image | Dimensions, file size, format, channels |
| Audio | Duration, sample rate, format |
| Tabular | Feature count, value ranges, null ratio |
| Embeddings | Dimensionality, norm |

---

### 5. Preprocessing & Postprocessing Separation

Never bury preprocessing inside the model. Keep it explicit and measurable:

```python
class PredictionService:
    def __init__(self, preprocessor, engine, postprocessor):
        self.preprocessor = preprocessor
        self.engine = engine
        self.postprocessor = postprocessor

    async def predict(self, text: str) -> dict:
        # Step 1: Preprocess (tokenize, normalize, feature extract)
        start = time.perf_counter()
        features = self.preprocessor.transform(text)
        preprocess_ms = (time.perf_counter() - start) * 1000

        # Step 2: Inference (model forward pass)
        start = time.perf_counter()
        raw_output = await self.engine.predict(features)
        inference_ms = (time.perf_counter() - start) * 1000

        # Step 3: Postprocess (softmax, label map, threshold)
        result = self.postprocessor.format(raw_output)

        logger.info("prediction_complete",
            preprocess_ms=round(preprocess_ms, 2),
            inference_ms=round(inference_ms, 2),
        )
        return result
```

**Why this matters:**
- You can measure where time is spent (preprocessing vs inference)
- You can swap preprocessors without touching the model
- You can cache preprocessed features
- You can test each step independently

---

### 6. Dynamic Batching

Single-request inference wastes GPU throughput. If multiple requests arrive within a short window, batch them for a single forward pass:

```python
import asyncio
from typing import Any

class BatchingEngine:
    """Collects individual requests and runs them as a batch."""

    def __init__(self, engine, max_batch_size: int = 16, max_wait_ms: float = 50):
        self.engine = engine
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        self._queue: asyncio.Queue = asyncio.Queue()
        self._running = False

    async def start(self):
        """Start the batching loop (call in lifespan startup)."""
        self._running = True
        asyncio.create_task(self._batch_loop())

    async def predict(self, input_data: Any) -> Any:
        """Submit a single request, wait for batched result."""
        future = asyncio.get_event_loop().create_future()
        await self._queue.put((input_data, future))
        return await future

    async def _batch_loop(self):
        while self._running:
            batch = []
            futures = []

            # Collect up to max_batch_size items, waiting max_wait_ms
            try:
                item, future = await asyncio.wait_for(
                    self._queue.get(), timeout=self.max_wait_ms / 1000
                )
                batch.append(item)
                futures.append(future)
            except asyncio.TimeoutError:
                continue

            # Drain queue up to batch size
            while len(batch) < self.max_batch_size and not self._queue.empty():
                item, future = self._queue.get_nowait()
                batch.append(item)
                futures.append(future)

            # Run batched inference
            results = await self.engine.predict_batch(batch)

            # Distribute results back to individual callers
            for future, result in zip(futures, results):
                future.set_result(result)
```

**When to use:** High-throughput services (>50 RPS) with GPU models. Not needed for low-traffic or CPU models.

---

### 7. GPU Memory Monitoring

GPU OOM is the #1 production failure for DL services. Expose memory as metrics:

```python
# app/observability/metrics.py
from prometheus_client import Gauge

GPU_MEMORY_USED = Gauge(
    "gpu_memory_used_bytes", "GPU memory currently in use", ["device"]
)
GPU_MEMORY_TOTAL = Gauge(
    "gpu_memory_total_bytes", "Total GPU memory available", ["device"]
)
GPU_UTILIZATION = Gauge(
    "gpu_utilization_percent", "GPU compute utilization", ["device"]
)
```

```python
# app/observability/gpu_monitor.py (periodic background task)
import torch

async def update_gpu_metrics():
    """Called periodically (e.g., every 10s) to update GPU gauges."""
    if not torch.cuda.is_available():
        return
    for i in range(torch.cuda.device_count()):
        mem = torch.cuda.mem_get_info(i)
        free, total = mem
        GPU_MEMORY_USED.labels(device=f"cuda:{i}").set(total - free)
        GPU_MEMORY_TOTAL.labels(device=f"cuda:{i}").set(total)
```

**Alert when:** GPU memory usage > 85% sustained. This means you're close to OOM.

---

### 8. ML-Specific Metrics

Beyond standard HTTP metrics, track these:

```python
# app/observability/metrics.py

# Inference performance
INFERENCE_LATENCY = Histogram(
    "model_inference_seconds", "Pure model forward pass time",
    ["model_name"], buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 5.0]
)
PREPROCESSING_LATENCY = Histogram(
    "preprocessing_seconds", "Input preprocessing time",
    ["model_name"], buckets=[0.001, 0.005, 0.01, 0.05, 0.1]
)

# Input characteristics
INPUT_TOKEN_COUNT = Histogram(
    "input_token_count", "Number of tokens in input",
    ["model_name"], buckets=[10, 50, 100, 200, 500, 1000, 2000]
)

# Output quality signals
OUTPUT_CONFIDENCE = Histogram(
    "prediction_confidence", "Model output confidence score",
    ["model_name"], buckets=[0.1, 0.2, 0.3, 0.5, 0.7, 0.8, 0.9, 0.95, 0.99]
)

# Batching efficiency
BATCH_SIZE = Histogram(
    "inference_batch_size", "Actual batch size per inference call",
    ["model_name"], buckets=[1, 2, 4, 8, 16, 32]
)
```

**Why confidence matters:** If average confidence drops over time, your model may be degrading (data drift, distribution shift). Set alerts on confidence percentiles.

---

### 9. Model Output Caching

Many ML APIs receive repeated inputs (search queries, common documents). Cache predictions to avoid redundant GPU work:

```python
import hashlib
import json

class PredictionService:
    def __init__(self, engine, redis):
        self.engine = engine
        self.redis = redis
        self.cache_ttl = 3600  # 1 hour

    async def predict(self, text: str) -> dict:
        # Check cache
        cache_key = f"pred:{hashlib.sha256(text.encode()).hexdigest()}"
        cached = await self.redis.get(cache_key)
        if cached:
            logger.info("cache_hit", key=cache_key[:16])
            return json.loads(cached)

        # Run inference
        result = await self.engine.predict(text)

        # Cache result
        await self.redis.setex(cache_key, self.cache_ttl, json.dumps(result))
        return result
```

**When to cache:**
- Deterministic models (same input → same output)
- High cache hit ratio expected (search, classification)
- Inference is expensive (>100ms)

**When NOT to cache:**
- Models with randomness (temperature > 0 in LLMs)
- Time-sensitive predictions (real-time features)
- Personalized outputs (user-specific context)

---

### 10. Model Versioning & Hot Reload

Swap models without downtime:

```python
# app/services/model_manager.py
import asyncio

class ModelManager:
    """Manages model lifecycle including hot-reload."""

    def __init__(self):
        self._engine = None
        self._lock = asyncio.Lock()

    @property
    def engine(self):
        return self._engine

    async def load(self, model_path: str, device: str):
        """Initial load."""
        self._engine = await self._create_engine(model_path, device)

    async def reload(self, new_model_path: str, device: str):
        """Hot-reload: load new model, swap atomically, unload old."""
        logger.info("model_reload_starting", new_path=new_model_path)

        # Load new model (old one still serving)
        new_engine = await self._create_engine(new_model_path, device)

        # Atomic swap
        async with self._lock:
            old_engine = self._engine
            self._engine = new_engine

        # Unload old model (no longer receiving requests)
        await old_engine.unload()
        logger.info("model_reload_complete")
```

Expose a reload endpoint (admin-only):

```python
@router.post("/admin/reload-model", dependencies=[Depends(verify_admin_key)])
async def reload_model(request: Request):
    await request.app.state.model_manager.reload(settings.model_path, settings.model_device)
    return {"status": "reloaded"}
```

---

### 11. Graceful Degradation

When the primary model fails or is overloaded, don't return 500. Fall back:

```python
class PredictionService:
    def __init__(self, primary_engine, fallback_engine=None):
        self.primary = primary_engine
        self.fallback = fallback_engine

    async def predict(self, text: str) -> dict:
        try:
            result = await asyncio.wait_for(self.primary.predict(text), timeout=5.0)
            result["model_tier"] = "primary"
            return result
        except (asyncio.TimeoutError, Exception) as e:
            logger.warning("primary_model_failed", error=str(e))

            if self.fallback:
                result = await self.fallback.predict(text)
                result["model_tier"] = "fallback"
                return result

            raise
```

| Strategy | When to use |
|----------|-------------|
| Fallback to lighter model | Primary is GPU, fallback is CPU (slower but available) |
| Return cached result | Input was seen before |
| Return default/safe response | Classification with a "unknown" class |
| Degrade gracefully | Return partial results instead of failing entirely |

---

### 12. A/B Testing & Shadow Mode

When deploying a new model, compare it against the current one without affecting users:

```python
class ABTestingService:
    """Runs both models, returns primary result, logs comparison."""

    def __init__(self, model_a, model_b, traffic_split: float = 0.1):
        self.model_a = model_a  # Current (always returned to user)
        self.model_b = model_b  # Candidate (logged for comparison)
        self.traffic_split = traffic_split

    async def predict(self, text: str) -> dict:
        # Always run primary
        result_a = await self.model_a.predict(text)

        # Run candidate on X% of traffic (shadow mode)
        import random
        if random.random() < self.traffic_split:
            try:
                result_b = await self.model_b.predict(text)
                logger.info("ab_comparison",
                    model_a_label=result_a["label"],
                    model_b_label=result_b["label"],
                    agreement=result_a["label"] == result_b["label"],
                )
            except Exception:
                pass  # Shadow failures don't affect users

        return result_a  # Always return primary
```

---

### Summary Checklist

Before deploying any ML endpoint to production, verify:

- [ ] Model warmup runs during startup
- [ ] Concurrency is limited (semaphore or queue)
- [ ] Inference has a timeout
- [ ] Input is validated beyond schema (token count, dimensions)
- [ ] Preprocessing and postprocessing are separate, measurable steps
- [ ] GPU memory is monitored (if applicable)
- [ ] ML-specific metrics are exposed (inference latency, confidence, batch size)
- [ ] Caching is considered for deterministic, expensive predictions
- [ ] Graceful degradation path exists (fallback model or safe default)
- [ ] Model reload without downtime is possible

---

*Last updated: May 2026*


