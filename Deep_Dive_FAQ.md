# ML API Template — Deep Dive FAQ

Detailed answers to common questions about the template's architecture, tooling, and design decisions.

---

## Table of Contents

1. [What does pyproject.toml do?](#1-what-does-pyprojecttoml-do)
2. [How to use uv with pyproject.toml?](#2-how-to-use-uv-with-pyprojecttoml)
3. [Can I include uv + pyproject.toml setup in the Makefile?](#3-can-i-include-uv--pyprojecttoml-setup-in-the-makefile)
4. [What does app/core/logging.py do?](#4-what-does-appcoreloggingpy-do)
5. [What does setup_logging() set up and where to use it?](#5-what-does-setup_logging-set-up-and-where-to-use-it)
6. [Which files should import the logging module?](#6-which-files-should-import-the-logging-module)
7. [What goes in app/schemas/?](#7-what-goes-in-appschemas)
8. [Exception handling flow — where to catch, where to raise?](#8-exception-handling-flow--where-to-catch-where-to-raise)
9. [Request tracing — full lifecycle through logs](#9-request-tracing--full-lifecycle-through-logs)
10. [What is the dependencies/ folder for?](#10-what-is-the-dependencies-folder-for)
11. [Where should API versioning be implemented?](#11-where-should-api-versioning-be-implemented)
12. [What goes in lifespan.py?](#12-what-goes-in-lifespanpy)
13. [What goes in middleware?](#13-what-goes-in-middleware)
14. [What goes in create_app()?](#14-what-goes-in-create_app)
15. [How to add custom exceptions?](#15-how-to-add-custom-exceptions)
16. [REST vs gRPC — where to use each in ML?](#16-rest-vs-grpc--where-to-use-each-in-ml)
17. [Dependency injection — where and why?](#17-dependency-injection--where-and-why)

---

## 1. What does `pyproject.toml` do?

`pyproject.toml` is the **single configuration file** for your entire Python project. It replaces what used to require 5-6 separate files (`setup.py`, `requirements.txt`, `.flake8`, `mypy.ini`, `pytest.ini`, `setup.cfg`).

### What it controls:

| Concern | Old way | pyproject.toml way |
|---------|---------|-------------------|
| Project metadata | `setup.py` | `[project]` section |
| Dependencies | `requirements.txt` | `dependencies = [...]` |
| Dev dependencies | `requirements-dev.txt` | `[project.optional-dependencies]` |
| Linting config | `.flake8` | `[tool.ruff]` |
| Type checking | `mypy.ini` | `[tool.mypy]` |
| Test config | `pytest.ini` | `[tool.pytest.ini_options]` |

### Sections explained:

```toml
# ─── PROJECT IDENTITY ───────────────────────────────────────
[project]
name = "ml-api-template"
version = "0.1.0"
description = "Production-grade ML API service template"
requires-python = ">=3.11"

# ─── PRODUCTION DEPENDENCIES ────────────────────────────────
# These get installed in the Docker image / production environment
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "structlog>=24.0.0",
    "prometheus-client>=0.20.0",
    "httpx>=0.27.0",
]

# ─── OPTIONAL/GROUPED DEPENDENCIES ──────────────────────────
# Install with: uv sync --extra dev --extra ml
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=5.0.0",
    "ruff>=0.5.0",
    "mypy>=1.10.0",
    "pre-commit>=3.7.0",
]
ml = [
    "torch>=2.0.0",
    "transformers>=4.40.0",
    "numpy>=1.26.0",
    "scikit-learn>=1.5.0",
]

# ─── TOOL CONFIGURATIONS ────────────────────────────────────

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM"]
# E=pycodestyle, F=pyflakes, I=isort, N=naming, W=warnings
# UP=pyupgrade, B=bugbear, SIM=simplify

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --tb=short"
```

### How to read it:

- `[project]` → "What is this project and what does it need to run?"
- `[project.optional-dependencies]` → "What extra groups of packages exist for specific contexts?"
- `[tool.X]` → "How should tool X behave when run in this project?"

---

## 2. How to use `uv` with `pyproject.toml`?

`uv` reads `pyproject.toml` directly. You don't need a separate `requirements.txt`. Here's the workflow:

### Initial setup (new project):

```bash
# Initialize a new project (creates pyproject.toml + uv.lock)
uv init my-project

# Or if pyproject.toml already exists, just sync:
uv sync
```

### Daily workflow:

```bash
# Install all dependencies (reads pyproject.toml, creates/updates uv.lock)
uv sync

# Install with optional groups
uv sync --extra dev --extra ml

# Add a new dependency (updates pyproject.toml AND uv.lock)
uv add fastapi
uv add pytest --dev    # adds to [project.optional-dependencies] dev group

# Remove a dependency
uv remove pandas

# Run a command inside the virtual environment
uv run uvicorn app.main:app --reload
uv run pytest
uv run ruff check .

# Update all dependencies to latest compatible versions
uv lock --upgrade
uv sync
```

### Things to take care of:

1. **Always commit `uv.lock`** — This file pins exact versions. Without it, different machines may install different versions.

2. **Use `uv run` instead of activating venv** — `uv run` automatically uses the project's virtual environment. No need for `source .venv/bin/activate`.

3. **Version pinning strategy:**
   ```toml
   # In pyproject.toml — use minimum version constraints
   dependencies = [
       "fastapi>=0.115.0",    # Allow compatible updates
   ]
   # uv.lock will pin the EXACT version installed
   ```

4. **The `.venv` is local** — uv creates `.venv/` in your project root. It's git-ignored. Each developer gets their own.

5. **Python version** — uv can manage Python versions too:
   ```bash
   uv python install 3.11
   uv python pin 3.11
   ```

---

## 3. Can I include uv + pyproject.toml setup in the Makefile?

Yes, and you **should**. The Makefile becomes the single entry point for all workflows:

```makefile
.PHONY: install install-dev run lint format typecheck test test-cov docker-build clean

# ─── SETUP ──────────────────────────────────────────────────
install:                          ## Install production dependencies
	uv sync

install-dev:                      ## Install all dependencies (dev + ml)
	uv sync --extra dev --extra ml

# ─── DEVELOPMENT ────────────────────────────────────────────
run:                              ## Start dev server with hot reload
	uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# ─── CODE QUALITY ───────────────────────────────────────────
lint:                             ## Check code quality
	uv run ruff check .

format:                           ## Auto-format code
	uv run ruff format .

typecheck:                        ## Run static type checking
	uv run mypy app

# ─── TESTING ────────────────────────────────────────────────
test:                             ## Run tests
	uv run pytest

test-cov:                         ## Run tests with coverage
	uv run pytest --cov=app --cov-report=term-missing --cov-report=html

# ─── DOCKER ─────────────────────────────────────────────────
docker-build:                     ## Build production Docker image
	docker build -f docker/Dockerfile -t ml-api-template .

docker-run:                       ## Run Docker container
	docker run -p 8000:8000 --env-file .env ml-api-template

# ─── UTILITIES ──────────────────────────────────────────────
clean:                            ## Remove cache files
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type d -name .pytest_cache -exec rm -rf {} +
	find . -type d -name .mypy_cache -exec rm -rf {} +
	find . -type d -name .ruff_cache -exec rm -rf {} +

add:                              ## Add a dependency: make add pkg=fastapi
	uv add $(pkg)

add-dev:                          ## Add a dev dependency: make add-dev pkg=pytest
	uv add $(pkg) --dev

help:                             ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'
```

### New developer onboarding becomes:

```bash
git clone <repo>
cd ml-api-template
make install-dev   # Everything is ready
make run           # Server starts
```

### CI pipeline becomes:

```yaml
steps:
  - run: make install-dev
  - run: make lint
  - run: make typecheck
  - run: make test
  - run: make docker-build
```

**Key point:** The Makefile wraps `uv` commands so nobody needs to remember flags or arguments. It's documentation-as-code for your workflows.


---

## 4. What does `app/core/logging.py` do?

This file is the **centralized logging configuration** for the entire application. It sets up `structlog` to emit structured, machine-parseable logs instead of plain text.

### Full implementation with documentation:

```python
# app/core/logging.py
"""
Centralized logging configuration.

This module configures structlog for structured JSON logging with:
- Automatic timestamp injection
- Log level as a field
- Request correlation ID binding (via contextvars)
- Exception formatting
- Environment-aware rendering (JSON for prod, colored console for dev)

Usage:
    from app.core.logging import setup_logging, get_logger

    # Call once at startup:
    setup_logging()

    # Get a logger anywhere:
    logger = get_logger(__name__)
    logger.info("something_happened", key="value", count=42)
"""

import logging
import structlog
from app.core.config import settings


def setup_logging() -> None:
    """
    Configure structlog and stdlib logging for the application.

    This function should be called ONCE during application startup
    (inside lifespan.py). It configures:

    1. structlog processors — the pipeline that transforms every log event
    2. stdlib logging — so third-party libraries (uvicorn, httpx) also
       emit structured logs
    3. Output format — JSON for production, colored console for local dev

    Processors execute in order. Each one adds/transforms fields in the log event:
        merge_contextvars → add_log_level → add_logger_name → TimeStamper
        → StackInfoRenderer → format_exc_info → Renderer
    """

    # Shared processors for both structlog and stdlib logging
    shared_processors: list[structlog.types.Processor] = [
        # Merge any context bound via structlog.contextvars.bind_contextvars()
        # This is how request_id gets into every log line automatically
        structlog.contextvars.merge_contextvars,

        # Add "level" field: info, warning, error, etc.
        structlog.stdlib.add_log_level,

        # Add "logger" field: the name passed to get_logger()
        structlog.stdlib.add_logger_name,

        # Add "timestamp" field in ISO 8601 format
        structlog.processors.TimeStamper(fmt="iso"),

        # If stack_info=True was passed, render the stack
        structlog.processors.StackInfoRenderer(),

        # Format exception info into a readable string
        structlog.processors.format_exc_info,
    ]

    # Choose renderer based on environment
    if settings.log_format == "json":
        # Production: machine-parseable JSON (for Grafana, Datadog, ELK)
        renderer = structlog.processors.JSONRenderer()
    else:
        # Local dev: colored, human-readable console output
        renderer = structlog.dev.ConsoleRenderer()

    # Configure structlog
    structlog.configure(
        processors=shared_processors + [renderer],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    # Also configure stdlib logging (for uvicorn, httpx, sqlalchemy, etc.)
    # This ensures third-party library logs also go through structlog
    logging.basicConfig(
        format="%(message)s",
        level=getattr(logging, settings.log_level.upper()),
        handlers=[logging.StreamHandler()],
    )


def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    """
    Get a named, structured logger instance.

    Args:
        name: Logger name, typically __name__ of the calling module.
              This appears as the "logger" field in log output.

    Returns:
        A bound logger that supports .info(), .warning(), .error(), etc.
        All methods accept arbitrary keyword arguments that become log fields.

    Example:
        logger = get_logger(__name__)
        logger.info("model_loaded", model_name="resnet50", load_time_ms=1200)

        # Output (JSON):
        # {"timestamp": "...", "level": "info", "logger": "app.services.inference",
        #  "event": "model_loaded", "model_name": "resnet50", "load_time_ms": 1200}
    """
    return structlog.get_logger(name)
```

### What each processor does:

| Processor | What it adds to every log line |
|-----------|-------------------------------|
| `merge_contextvars` | Any fields bound via `bind_contextvars()` (e.g., `request_id`) |
| `add_log_level` | `"level": "info"` |
| `add_logger_name` | `"logger": "app.services.prediction"` |
| `TimeStamper` | `"timestamp": "2026-05-27T01:00:00.000Z"` |
| `StackInfoRenderer` | Stack trace if `stack_info=True` |
| `format_exc_info` | Formatted exception if one was raised |
| `JSONRenderer` | Serializes everything to JSON string |

---

## 5. What does `setup_logging()` set up and where to use it?

### What it sets up:

`setup_logging()` configures the **global logging pipeline**. After calling it:
- Every `get_logger()` call returns a properly configured logger
- Every log line automatically includes timestamp, level, logger name
- Context variables (like `request_id`) are automatically merged into logs
- Third-party library logs (uvicorn, httpx) are also structured

### Where to call it:

**Call it exactly ONCE, at application startup — inside `lifespan.py`:**

```python
# app/core/lifespan.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.core.logging import setup_logging, get_logger

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ← First thing that happens at startup
    setup_logging()
    logger.info("app_starting", environment=settings.environment)

    # ... load models, connect DBs ...
    yield
    # ... cleanup ...
```

### Why only once?

`structlog.configure()` sets global state. Calling it multiple times would reconfigure the pipeline mid-flight, potentially losing context bindings or changing format unexpectedly.

### What happens if you forget to call it?

Logs will still work (structlog has defaults), but:
- No timestamps
- No JSON formatting
- No context variable merging
- No consistent format across the app

---

## 6. Which files should import the logging module?

**Short answer:** Any file that needs to log something.

### Import pattern:

```python
from app.core.logging import get_logger

logger = get_logger(__name__)
```

### Where you SHOULD use it:

| File/Layer | What to log |
|-----------|-------------|
| `app/services/*.py` | Business events: inference started/completed, external API calls, errors |
| `app/core/lifespan.py` | Startup/shutdown events: model loaded, DB connected |
| `app/middleware/*.py` | Request lifecycle: request received, latency, status |
| `app/api/routes/*.py` | Sparingly — only for route-specific events not covered by middleware |
| `app/core/exceptions.py` | Exception handler logging |

### Where you should NOT use it:

| File | Why not |
|------|---------|
| `app/schemas/*.py` | Schemas are pure data definitions — no side effects |
| `app/core/config.py` | Config loads before logging is set up |
| `app/utils/*.py` | Utils should be pure functions (no side effects) |

### Example across layers:

```python
# app/services/prediction_service.py
from app.core.logging import get_logger

logger = get_logger(__name__)

class PredictionService:
    async def predict(self, text: str) -> dict:
        logger.info("prediction_started", input_length=len(text))

        result = await self.engine.predict(text)

        logger.info("prediction_complete",
            label=result["label"],
            confidence=result["confidence"],
            latency_ms=result["latency_ms"],
        )
        return result
```

```python
# app/core/lifespan.py
from app.core.logging import get_logger

logger = get_logger(__name__)

# During startup:
logger.info("model_loading", path=settings.model_path)
logger.info("model_loaded", load_time_ms=1200, device="cuda")

# During shutdown:
logger.info("model_unloaded")
```


---

## 7. What goes in `app/schemas/`?

Yes — this folder contains **all Pydantic models that define the shape of data** flowing in and out of your API. Think of schemas as **contracts** between your API and its consumers.

### What belongs here:

| File | Contains |
|------|----------|
| `common.py` | Shared schemas used across multiple endpoints (error responses, pagination) |
| `health.py` | Health endpoint request/response models |
| `prediction.py` | Prediction endpoint input/output models |
| `rag.py` | RAG endpoint schemas |
| One file per domain | Keep schemas grouped by feature/endpoint |

### Example structure:

```python
# app/schemas/common.py
from pydantic import BaseModel, Field
from datetime import datetime


class ErrorResponse(BaseModel):
    """Standard error response returned by all endpoints on failure."""
    error: str                    # Machine-readable error code
    message: str                  # Human-readable description
    request_id: str | None = None
    timestamp: datetime = Field(default_factory=datetime.utcnow)


class PaginationParams(BaseModel):
    """Reusable pagination input."""
    page: int = Field(default=1, ge=1)
    page_size: int = Field(default=20, ge=1, le=100)
```

```python
# app/schemas/prediction.py
from pydantic import BaseModel, Field


class PredictionRequest(BaseModel):
    """Input for the prediction endpoint."""
    text: str = Field(..., min_length=1, max_length=10000)
    top_k: int = Field(default=3, ge=1, le=10)
    model_name: str = Field(default="default")


class PredictionResult(BaseModel):
    """Single prediction output."""
    label: str
    confidence: float = Field(..., ge=0.0, le=1.0)


class PredictionResponse(BaseModel):
    """Full response from prediction endpoint."""
    request_id: str
    predictions: list[PredictionResult]
    model_name: str
    latency_ms: float
```

### Rules:

1. **Schemas are pure data** — no methods with side effects, no imports from services
2. **Use Field() for validation** — min/max lengths, ranges, descriptions
3. **One file per domain** — don't put everything in one giant file
4. **Schemas are NOT database models** — they define API contracts, not storage

---

## 8. Exception handling flow — where to catch, where to raise?

This is a critical question. Let's trace through a real example:

### Scenario:

```
POST /v1/predict
    → prediction route (transport layer)
        → PredictionService (business layer)
            → ThirdPartyAPIClient (external call)
                → HTTP request to external service
```

The third-party API might fail (timeout, 500, invalid response). Where do we handle this?

### Principle: **Raise at the source, catch at the boundary, translate for the consumer.**

### Full example:

```python
# app/core/exceptions.py — Define custom exceptions
class AppException(Exception):
    """Base for all application exceptions."""
    def __init__(self, message: str, status_code: int = 500, error_code: str = "INTERNAL_ERROR"):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code


class ExternalServiceError(AppException):
    """Raised when an external API call fails."""
    def __init__(self, service_name: str, detail: str):
        super().__init__(
            message=f"External service '{service_name}' failed: {detail}",
            status_code=502,
            error_code="EXTERNAL_SERVICE_ERROR",
        )


class ModelInferenceError(AppException):
    """Raised when model inference fails."""
    def __init__(self, detail: str):
        super().__init__(
            message=f"Inference failed: {detail}",
            status_code=500,
            error_code="INFERENCE_ERROR",
        )
```

```python
# app/services/external_client.py — The third-party API client
import httpx
from app.core.logging import get_logger
from app.core.exceptions import ExternalServiceError

logger = get_logger(__name__)


class FeatureStoreClient:
    """Client for external feature store API."""

    def __init__(self, base_url: str, timeout: float = 5.0):
        self.client = httpx.AsyncClient(base_url=base_url, timeout=timeout)

    async def get_features(self, user_id: str) -> dict:
        """Fetch features from external service.

        RAISES ExternalServiceError if the call fails.
        The service layer decides what to do with it.
        """
        try:
            response = await self.client.get(f"/features/{user_id}")
            response.raise_for_status()
            return response.json()

        except httpx.TimeoutException:
            logger.error("feature_store_timeout", user_id=user_id)
            raise ExternalServiceError("feature-store", "Request timed out")

        except httpx.HTTPStatusError as e:
            logger.error("feature_store_http_error",
                user_id=user_id, status=e.response.status_code)
            raise ExternalServiceError("feature-store", f"HTTP {e.response.status_code}")

        except httpx.RequestError as e:
            logger.error("feature_store_connection_error", error=str(e))
            raise ExternalServiceError("feature-store", "Connection failed")
```

```python
# app/services/prediction_service.py — Business logic layer
from app.core.logging import get_logger
from app.core.exceptions import ModelInferenceError, ExternalServiceError

logger = get_logger(__name__)


class PredictionService:
    def __init__(self, engine, feature_client):
        self.engine = engine
        self.feature_client = feature_client

    async def predict(self, user_id: str, text: str) -> dict:
        """
        Orchestrates the prediction flow.

        This layer:
        - Calls external services
        - Handles/translates exceptions from lower layers
        - Adds business context to errors
        - Does NOT catch AppException (let it bubble to the handler)
        """
        # Step 1: Get features (may raise ExternalServiceError)
        # We LET it propagate — the global handler will catch it
        features = await self.feature_client.get_features(user_id)

        # Step 2: Run inference
        try:
            result = await self.engine.predict(text, features)
        except Exception as e:
            # Translate unknown engine errors into our domain exception
            logger.error("inference_failed", error=str(e), user_id=user_id)
            raise ModelInferenceError(detail=str(e))

        return result
```

```python
# app/api/routes/prediction.py — Transport layer (route handler)
from fastapi import APIRouter, Depends, Request
from app.schemas.prediction import PredictionRequest, PredictionResponse

router = APIRouter(prefix="/v1", tags=["prediction"])


@router.post("/predict", response_model=PredictionResponse)
async def predict(
    request: Request,
    body: PredictionRequest,
    service: PredictionService = Depends(get_prediction_service),
):
    """
    Route handler — THIN layer.

    Does NOT catch exceptions here.
    AppException subclasses bubble up to the global exception handler.
    This keeps routes clean and consistent.
    """
    result = await service.predict(user_id=body.user_id, text=body.text)

    return PredictionResponse(
        request_id=request.state.request_id,
        predictions=result["predictions"],
        model_name=result["model_name"],
        latency_ms=result["latency_ms"],
    )
```

```python
# app/main.py — Global exception handler (catches everything)
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from app.core.exceptions import AppException


async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    """Converts any AppException into a structured JSON error response."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code,
            "message": exc.message,
            "request_id": getattr(request.state, "request_id", None),
        },
    )


def create_app() -> FastAPI:
    app = FastAPI(...)
    app.add_exception_handler(AppException, app_exception_handler)
    return app
```

### The flow when the third-party API times out:

```
1. FeatureStoreClient.get_features() catches httpx.TimeoutException
2. FeatureStoreClient RAISES ExternalServiceError("feature-store", "Request timed out")
3. PredictionService.predict() does NOT catch it (lets it bubble)
4. Route handler does NOT catch it (lets it bubble)
5. FastAPI's exception handler catches AppException
6. Returns JSON: {"error": "EXTERNAL_SERVICE_ERROR", "message": "...", "request_id": "..."}
7. Client receives HTTP 502
```

### Summary of responsibilities:

| Layer | Exception responsibility |
|-------|------------------------|
| External clients (`services/`) | Catch transport errors (timeout, connection), raise domain exceptions |
| Service layer (`services/`) | Catch unexpected errors from engines, translate to domain exceptions |
| Route handlers (`api/routes/`) | Do NOT catch exceptions — let them bubble |
| Global handler (`main.py`) | Catch all `AppException`, return structured JSON |


---

## 9. Request tracing — full lifecycle through logs

Here's how a single request flows through the system and how every log line connects back to it.

### High-level flow:

```
Client sends POST /v1/predict
    │
    ▼
┌─ RequestContextMiddleware ─────────────────────────────┐
│  • Extracts/generates request_id                       │
│  • Binds request_id to structlog contextvars           │
│  • From this point, ALL logs include request_id        │
└────────────────────────────────────────────────────────┘
    │
    ▼
┌─ LoggingMiddleware ────────────────────────────────────┐
│  • Records start time                                  │
│  • After response: logs method, path, status, latency  │
│  • Records Prometheus metrics                          │
└────────────────────────────────────────────────────────┘
    │
    ▼
┌─ Route Handler ────────────────────────────────────────┐
│  • Validates request body (Pydantic)                   │
│  • Calls service                                       │
└────────────────────────────────────────────────────────┘
    │
    ▼
┌─ PredictionService ────────────────────────────────────┐
│  • Logs "prediction_started"                           │
│  • Calls engine.predict()                              │
│  • Logs "prediction_complete" with latency             │
└────────────────────────────────────────────────────────┘
    │
    ▼
Response returned with X-Request-ID header
```

### What the logs look like for ONE request:

```json
{"timestamp":"2026-05-27T01:00:00.001Z","level":"info","logger":"app.middleware.request_context","event":"request_received","request_id":"abc-123","method":"POST","path":"/v1/predict"}

{"timestamp":"2026-05-27T01:00:00.002Z","level":"info","logger":"app.services.prediction","event":"prediction_started","request_id":"abc-123","input_length":45}

{"timestamp":"2026-05-27T01:00:00.050Z","level":"info","logger":"app.services.prediction","event":"prediction_complete","request_id":"abc-123","label":"positive","confidence":0.94,"latency_ms":48.2}

{"timestamp":"2026-05-27T01:00:00.051Z","level":"info","logger":"app.middleware.logging","event":"request_complete","request_id":"abc-123","method":"POST","path":"/v1/predict","status_code":200,"latency_ms":50.1}
```

**Key insight:** Every log line has `request_id: "abc-123"`. In Grafana/Datadog, you filter by this ID and see the complete story of one request.

### Implementation:

```python
# app/middleware/request_context.py
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class RequestContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        # Use caller's ID or generate one
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))

        # Store on request (accessible in route handlers)
        request.state.request_id = request_id

        # Bind to structlog — ALL subsequent logs in this request include it
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=request_id)

        response = await call_next(request)

        # Return in response header (caller can use it for support tickets)
        response.headers["X-Request-ID"] = request_id
        return response
```

### Adding custom trace context in services:

```python
# Inside any service — you can bind additional context
import structlog

async def predict(self, text: str):
    # Add model_name to all subsequent logs in this request
    structlog.contextvars.bind_contextvars(model_name=self.model_name)

    logger.info("prediction_started", input_length=len(text))
    # This log will include BOTH request_id AND model_name
```

### For distributed tracing (multiple services):

If your prediction service calls another microservice, pass the `request_id` forward:

```python
async def call_feature_store(self, user_id: str, request_id: str):
    response = await self.client.get(
        f"/features/{user_id}",
        headers={"X-Request-ID": request_id},  # Propagate!
    )
```

Now both services' logs share the same `request_id` — you can trace across service boundaries.

---

## 10. What is the `dependencies/` folder for?

`app/api/dependencies/` contains **FastAPI dependency functions** — reusable factories that provide resources to route handlers via `Depends()`.

### Why it exists:

Routes need access to services, database sessions, auth validation, etc. Instead of creating these inside each route (duplication) or importing globals (untestable), you use dependency injection.

### What goes here:

```
app/api/dependencies/
├── services.py      # Provides service instances from app.state
├── auth.py          # Authentication/authorization checks
├── database.py      # DB session lifecycle (yield + cleanup)
└── common.py        # Shared utilities (pagination params, rate limits)
```

### Example implementations:

```python
# app/api/dependencies/services.py
"""
Dependency functions that provide service instances to route handlers.

These pull pre-initialized services from app.state (set up in lifespan.py).
"""
from fastapi import Request
from app.services.prediction_service import PredictionService
from app.services.rag_service import RAGService


def get_prediction_service(request: Request) -> PredictionService:
    """Provides the prediction service to routes."""
    return request.app.state.prediction_service


def get_rag_service(request: Request) -> RAGService:
    """Provides the RAG service to routes."""
    return request.app.state.rag_service
```

```python
# app/api/dependencies/auth.py
"""
Authentication dependencies — reusable across any protected route.
"""
from fastapi import Header, HTTPException, Depends
from app.core.config import settings


async def verify_api_key(x_api_key: str = Header(...)) -> str:
    """Validates API key from request header."""
    if x_api_key != settings.api_key:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return x_api_key


async def verify_admin_key(x_api_key: str = Header(...)) -> str:
    """Validates admin-level API key."""
    if x_api_key != settings.admin_api_key:
        raise HTTPException(status_code=403, detail="Admin access required")
    return x_api_key
```

```python
# app/api/dependencies/database.py
"""
Database session dependency — auto-closes after request.
"""
from fastapi import Request
from typing import AsyncGenerator


async def get_db_session(request: Request) -> AsyncGenerator:
    """Yields a DB session, ensures cleanup after request."""
    async with request.app.state.db_pool.session() as session:
        yield session
        # Session is automatically closed here, even if route raised an exception
```

### How routes use them:

```python
# app/api/routes/prediction.py
from fastapi import APIRouter, Depends
from app.api.dependencies.services import get_prediction_service
from app.api.dependencies.auth import verify_api_key

router = APIRouter(prefix="/v1")


@router.post("/predict")
async def predict(
    body: PredictionRequest,
    service: PredictionService = Depends(get_prediction_service),
    _: str = Depends(verify_api_key),  # Auth check runs automatically
):
    return await service.predict(body.text)
```

### Why not just import services directly?

```python
# BAD — tight coupling, untestable
from app.services.prediction_service import prediction_service  # global instance

@router.post("/predict")
async def predict(body: PredictionRequest):
    return await prediction_service.predict(body.text)
```

Problems:
- Can't swap the service in tests (need to monkeypatch globals)
- Service must exist at import time (circular import issues)
- No lifecycle management

With `Depends()`, you can override any dependency in tests:

```python
# In tests — swap real service for mock
app.dependency_overrides[get_prediction_service] = lambda: mock_service
```


---

## 11. Where should API versioning be implemented?

API versioning is implemented via **route prefixes**, configured in `app/main.py` when including routers.

### Approach: URL-based versioning with prefixed routers

```python
# app/api/routes/v1/prediction.py
from fastapi import APIRouter

router = APIRouter(tags=["prediction-v1"])

@router.post("/predict")
async def predict_v1(body: PredictionRequestV1, ...):
    """V1 prediction — returns top-k labels."""
    ...
```

```python
# app/api/routes/v2/prediction.py
from fastapi import APIRouter

router = APIRouter(tags=["prediction-v2"])

@router.post("/predict")
async def predict_v2(body: PredictionRequestV2, ...):
    """V2 prediction — returns labels + explanations."""
    ...
```

```python
# app/main.py — Wire versions here
from app.api.routes.v1 import prediction as prediction_v1
from app.api.routes.v2 import prediction as prediction_v2
from app.api.routes import health

def create_app() -> FastAPI:
    app = FastAPI(title="ML Service", version="2.0.0", lifespan=lifespan)

    # Unversioned (health checks don't need versioning)
    app.include_router(health.router)

    # Versioned endpoints
    app.include_router(prediction_v1.router, prefix="/v1")
    app.include_router(prediction_v2.router, prefix="/v2")

    return app
```

### Resulting endpoints:

```
POST /v1/predict  → old clients still work
POST /v2/predict  → new clients use enhanced response
GET  /health/live → no version prefix needed
```

### Folder structure with versioning:

```
app/api/routes/
├── health.py           # Unversioned
├── v1/
│   ├── __init__.py
│   └── prediction.py   # V1 contract
└── v2/
    ├── __init__.py
    └── prediction.py   # V2 contract (new fields, different schema)
```

### When to version:

- **Breaking schema changes** (removing fields, changing types) → new version
- **Adding optional fields** → no new version needed (backward compatible)
- **Changing behavior** (different model, different logic) → new version

---

## 12. What goes in `lifespan.py`?

`lifespan.py` is the **boot sequence and shutdown sequence** of your application. Everything that needs to be initialized once and cleaned up on exit lives here.

### Complete checklist of what to include:

```python
# app/core/lifespan.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.core.config import settings
from app.core.logging import setup_logging, get_logger

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Application lifecycle manager.

    STARTUP (before yield):
    - Everything that must be ready before the first request is served.
    - If anything here fails, the app does NOT start.

    SHUTDOWN (after yield):
    - Graceful cleanup of all resources.
    - Runs even if the app is killed with SIGTERM.
    """

    # ═══════════════════════════════════════════════════════════
    # STARTUP
    # ═══════════════════════════════════════════════════════════

    # 1. Logging (must be first — everything else logs)
    setup_logging()
    logger.info("app_starting", environment=settings.environment)

    # 2. Database connection pool
    # db_pool = await create_async_pool(settings.database_url)
    # app.state.db_pool = db_pool
    # logger.info("db_pool_created", pool_size=db_pool.size)

    # 3. Redis connection
    # redis = await aioredis.from_url(settings.redis_url)
    # app.state.redis = redis
    # logger.info("redis_connected")

    # 4. ML Model loading (heaviest operation — do last)
    # engine = PyTorchEngine(settings.model_path, settings.model_device)
    # await engine.load()
    # app.state.engine = engine
    # logger.info("model_loaded", path=settings.model_path, device=settings.model_device)

    # 5. Create service instances (depend on above resources)
    # app.state.prediction_service = PredictionService(engine=engine)
    # app.state.rag_service = RAGService(retriever=retriever, llm=llm_client)

    # 6. Observability setup
    # setup_tracing()  # OpenTelemetry
    # logger.info("tracing_initialized")

    # 7. Warmup (optional — send a dummy request to pre-compile)
    # await engine.predict("warmup")
    # logger.info("warmup_complete")

    logger.info("app_started", environment=settings.environment)

    yield  # ← Application is running and serving requests

    # ═══════════════════════════════════════════════════════════
    # SHUTDOWN
    # ═══════════════════════════════════════════════════════════

    logger.info("app_shutting_down")

    # Release resources in reverse order
    # await engine.unload()          # Free GPU memory
    # await redis.close()            # Close Redis
    # await db_pool.close()          # Close DB connections
    # shutdown_tracing()             # Flush remaining spans

    logger.info("app_stopped")
```

### Checklist of items to handle:

| Item | Startup | Shutdown |
|------|---------|----------|
| Logging | `setup_logging()` | — |
| Database pool | Create pool | Close pool |
| Redis | Connect | Disconnect |
| ML Model | Load into memory/GPU | Unload, free GPU memory |
| Vector DB client | Connect | Disconnect |
| HTTP clients (httpx) | Create `AsyncClient` | `await client.aclose()` |
| OpenTelemetry | Initialize tracer provider | Flush + shutdown |
| Service instances | Create and attach to `app.state` | — |
| Feature flags | Fetch initial config | — |
| Cache warmup | Pre-populate hot cache | — |

### Rules:

1. **Order matters** — logging first, models last (so model loading is logged)
2. **Shutdown in reverse order** — last initialized = first cleaned up
3. **If startup fails, app doesn't start** — this is intentional (fail fast)
4. **Store everything on `app.state`** — this is how routes access shared resources

---

## 13. What goes in middleware?

Middleware handles **cross-cutting concerns** — things that apply to every (or most) requests uniformly, without being part of business logic.

### Complete list of what to implement:

| Middleware | File | Purpose |
|-----------|------|---------|
| Request Context | `request_context.py` | Assign unique request_id, bind to logs |
| Logging | `logging_middleware.py` | Log every request with latency + record metrics |
| CORS | (built-in FastAPI) | Allow cross-origin requests from frontends |
| Error handling | `error_middleware.py` | Catch unhandled exceptions, return structured JSON |
| Rate limiting | `rate_limit.py` | Throttle excessive requests per client |
| Auth (optional) | `auth_middleware.py` | Validate tokens globally (vs per-route with Depends) |

### What middleware should do:

```python
# app/middleware/request_context.py — MUST HAVE
# Assigns request_id, binds to structlog context
# Every log line in the request automatically includes request_id

# app/middleware/logging_middleware.py — MUST HAVE
# Logs: method, path, status_code, latency_ms
# Records: Prometheus request_count, request_latency histograms
```

### What middleware should NOT do:

- Business logic (use services)
- Database queries (use dependencies)
- Model inference (use services)
- Complex authorization logic (use dependencies — more testable)

### Execution order matters:

```python
# app/main.py — middleware executes in REVERSE registration order
def create_app() -> FastAPI:
    app = FastAPI(...)

    # Registered last → executes FIRST (outermost)
    app.add_middleware(LoggingMiddleware)

    # Registered first → executes LAST (innermost, closest to route)
    app.add_middleware(RequestContextMiddleware)

    # Actual execution order:
    # Request → RequestContext → Logging → [route] → Logging → RequestContext → Response
```

Wait — that's confusing. Let me clarify:

**Starlette middleware executes in registration order for the request path:**

```python
# First added = outermost = runs first on request, last on response
app.add_middleware(RequestContextMiddleware)  # 1st: assigns request_id
app.add_middleware(LoggingMiddleware)         # 2nd: logs (has request_id available)
```

Actually in Starlette/FastAPI, **last added = first executed**. So:

```python
app.add_middleware(LoggingMiddleware)          # Added 2nd, runs 1st? No...
app.add_middleware(RequestContextMiddleware)   # Added 1st, runs 2nd? No...
```

**Correct rule:** In FastAPI, middleware added **last** wraps the **outermost** layer. So add in this order:

```python
# Correct order — RequestContext must run BEFORE Logging
app.add_middleware(LoggingMiddleware)         # Inner (runs after context is set)
app.add_middleware(RequestContextMiddleware)  # Outer (runs first, sets context)
```

This ensures `request_id` is available when `LoggingMiddleware` logs.


---

## 14. What goes in `create_app()`?

`create_app()` is the **app factory** — it assembles all the pieces (middleware, routes, exception handlers, lifespan) into a configured FastAPI instance.

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.lifespan import lifespan
from app.core.exceptions import AppException, app_exception_handler
from app.middleware.request_context import RequestContextMiddleware
from app.middleware.logging_middleware import LoggingMiddleware
from app.api.routes import health
from app.api.routes.v1 import prediction as prediction_v1


def create_app() -> FastAPI:
    """
    Application factory.

    Assembles:
    1. FastAPI instance with metadata
    2. Lifespan (startup/shutdown)
    3. Exception handlers
    4. Middleware stack
    5. Route registration

    This is the ONLY place where wiring happens.
    """

    # ─── 1. Create FastAPI instance ──────────────────────────
    app = FastAPI(
        title="ML API Service",
        description="Production ML inference API",
        version="1.0.0",
        lifespan=lifespan,
        docs_url="/docs",        # Swagger UI
        redoc_url="/redoc",      # ReDoc
        openapi_url="/openapi.json",
    )

    # ─── 2. Exception handlers ───────────────────────────────
    app.add_exception_handler(AppException, app_exception_handler)

    # ─── 3. Middleware (last added = outermost = runs first) ─
    app.add_middleware(LoggingMiddleware)
    app.add_middleware(RequestContextMiddleware)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],     # Restrict in production
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # ─── 4. Routes ───────────────────────────────────────────
    app.include_router(health.router)                    # /health/live, /health/ready
    app.include_router(prediction_v1.router, prefix="/v1")  # /v1/predict

    return app


# Entry point for uvicorn
app = create_app()
```

### What `create_app()` is responsible for:

| Responsibility | Example |
|---------------|---------|
| App metadata | title, version, description |
| Lifespan binding | `lifespan=lifespan` |
| Exception handlers | `app.add_exception_handler(...)` |
| Middleware registration | `app.add_middleware(...)` |
| Route registration | `app.include_router(...)` |
| CORS configuration | `CORSMiddleware` settings |
| OpenAPI customization | docs_url, redoc_url |

### What it should NOT do:

- Load models (that's lifespan)
- Define routes (that's `api/routes/`)
- Implement business logic (that's `services/`)
- Configure logging (that's lifespan calling `setup_logging()`)

### Why a factory function instead of just `app = FastAPI()`?

1. **Testability** — tests can call `create_app()` to get a fresh instance
2. **Configuration** — you can pass different settings for test vs prod
3. **Clarity** — all wiring is visible in one place

---

## 15. How to add custom exceptions?

### Step 1: Define exceptions in `app/core/exceptions.py`

```python
# app/core/exceptions.py
from fastapi import Request
from fastapi.responses import JSONResponse


class AppException(Exception):
    """Base exception — all custom exceptions inherit from this."""
    def __init__(
        self,
        message: str,
        status_code: int = 500,
        error_code: str = "INTERNAL_ERROR",
    ):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code


# ─── Specific exceptions ────────────────────────────────────

class ModelNotReadyError(AppException):
    """Model hasn't finished loading or failed to load."""
    def __init__(self):
        super().__init__(
            message="Model is not loaded or not ready for inference",
            status_code=503,
            error_code="MODEL_NOT_READY",
        )


class ModelNotFoundError(AppException):
    """Requested model doesn't exist."""
    def __init__(self, model_name: str):
        super().__init__(
            message=f"Model '{model_name}' not found",
            status_code=404,
            error_code="MODEL_NOT_FOUND",
        )


class InvalidInputError(AppException):
    """Input failed business validation (beyond Pydantic schema validation)."""
    def __init__(self, detail: str):
        super().__init__(
            message=detail,
            status_code=422,
            error_code="INVALID_INPUT",
        )


class ExternalServiceError(AppException):
    """An external dependency (API, DB, cache) failed."""
    def __init__(self, service: str, detail: str):
        super().__init__(
            message=f"Service '{service}' failed: {detail}",
            status_code=502,
            error_code="EXTERNAL_SERVICE_ERROR",
        )


class RateLimitExceededError(AppException):
    """Client exceeded rate limit."""
    def __init__(self):
        super().__init__(
            message="Rate limit exceeded. Try again later.",
            status_code=429,
            error_code="RATE_LIMIT_EXCEEDED",
        )


# ─── Global handler ─────────────────────────────────────────

async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    """
    Converts any AppException into a consistent JSON error response.
    Registered in create_app() via app.add_exception_handler().
    """
    from app.core.logging import get_logger
    logger = get_logger("app.exceptions")

    logger.warning(
        "app_exception",
        error_code=exc.error_code,
        message=exc.message,
        status_code=exc.status_code,
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

### Step 2: Raise them in services

```python
# app/services/prediction_service.py
from app.core.exceptions import ModelNotReadyError, InvalidInputError

class PredictionService:
    async def predict(self, text: str) -> dict:
        if not self.engine.is_ready:
            raise ModelNotReadyError()

        if len(text.split()) > 512:
            raise InvalidInputError("Input exceeds maximum 512 tokens")

        return await self.engine.predict(text)
```

### Step 3: Client receives structured error

```json
// HTTP 503
{
    "error": "MODEL_NOT_READY",
    "message": "Model is not loaded or not ready for inference",
    "request_id": "abc-123"
}
```

### Key principle:

- **Routes never catch `AppException`** — they bubble to the global handler
- **Services raise domain-specific exceptions** — not generic `HTTPException`
- **One handler catches all** — consistent error format across the entire API

---

## 16. REST vs gRPC — where to use each in ML?

### Decision matrix:

| Scenario | Use | Why |
|----------|-----|-----|
| External clients (web, mobile, partners) | REST | Human-readable, Swagger docs, easy debugging |
| Internal service-to-service (high frequency) | gRPC | Binary protocol, 5-10x faster serialization |
| Streaming responses (LLM token generation) | gRPC or WebSocket | Bidirectional streaming support |
| Large tensor/embedding transfer | gRPC | Protobuf is 3-10x smaller than JSON for numeric arrays |
| Browser-based clients | REST | gRPC-web exists but adds complexity |
| Quick prototyping / POC | REST | Faster to build, easier to test with curl |
| Batch scoring between services | gRPC | Efficient for high-throughput numeric data |
| Public API | REST | Universal compatibility |

### Typical ML architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                     External Clients                          │
│              (browsers, mobile, third-party)                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ REST (JSON)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│              (FastAPI — this template)                        │
└───────┬──────────────────┬──────────────────┬───────────────┘
        │ gRPC             │ gRPC             │ gRPC
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
│ ML Inference │  │  Embedding   │  │  Feature Store   │
│   Service    │  │   Service    │  │                  │
└──────────────┘  └──────────────┘  └──────────────────┘
```

- **External boundary** → REST (this template)
- **Internal communication** → gRPC (when latency/throughput matters)

### When to NOT bother with gRPC:

- You have < 5 internal services
- Latency requirements are > 100ms (REST overhead is negligible)
- Your team doesn't have gRPC experience
- You're in early stages (POC, MVP)

### How this template supports both:

Because business logic lives in `app/services/` (framework-independent), you can add a gRPC transport layer alongside REST without rewriting anything:

```
app/
├── api/           # REST transport (FastAPI)
│   └── routes/
├── grpc/          # gRPC transport (grpcio) — added later
│   └── servicers/
└── services/      # Shared business logic (used by both)
```

---

## 17. Dependency injection — where and why?

### What is it in this project?

FastAPI's `Depends()` system. When a route declares a parameter with `Depends(some_function)`, FastAPI calls that function and passes the result to your route handler.

### Where to use it in this template:

| Use Case | Dependency | Why DI instead of import? |
|----------|-----------|--------------------------|
| Get prediction service | `Depends(get_prediction_service)` | Testable, swappable |
| Get DB session | `Depends(get_db_session)` | Auto-cleanup after request |
| Validate API key | `Depends(verify_api_key)` | Reusable across routes |
| Get current user | `Depends(get_current_user)` | Decouples auth from business logic |
| Pagination params | `Depends(get_pagination)` | Reusable validation |

### Concrete example in this project:

```python
# WITHOUT DI — tightly coupled, untestable
@router.post("/predict")
async def predict(request: Request, body: PredictionRequest):
    # Route knows implementation details
    engine = request.app.state.engine
    service = PredictionService(engine)  # Creates new instance every request!
    return await service.predict(body.text)


# WITH DI — clean, testable, swappable
@router.post("/predict")
async def predict(
    body: PredictionRequest,
    service: PredictionService = Depends(get_prediction_service),
):
    # Route only knows the interface
    return await service.predict(body.text)
```

### Pros:

| Benefit | How |
|---------|-----|
| **Testability** | Override dependencies in tests: `app.dependency_overrides[get_service] = mock` |
| **Separation of concerns** | Routes don't know how services are created |
| **Reusability** | Auth check written once, used in 50 routes |
| **Lifecycle management** | DB sessions auto-close via `yield` dependencies |
| **Composability** | Dependencies can depend on other dependencies |
| **Swappability** | Change from PyTorch to ONNX by changing one function |

### Cons:

| Drawback | Mitigation |
|----------|-----------|
| **Indirection** | Keep dependency functions simple (1-3 lines) |
| **Learning curve** | FastAPI docs are excellent; team learns once |
| **Over-engineering risk** | Only use DI for shared resources, not simple values |
| **Debugging** | Stack traces show the dependency chain — readable enough |

### When NOT to use DI:

```python
# DON'T use Depends() for simple values
@router.post("/predict")
async def predict(
    body: PredictionRequest,
    top_k: int = Depends(get_default_top_k),  # Overkill — just use a default
):
    ...

# DO just use a default value
@router.post("/predict")
async def predict(body: PredictionRequest):
    top_k = body.top_k or 3  # Simple, no DI needed
    ...
```

### Rule of thumb:

Use `Depends()` when the dependency:
- Is **shared** across multiple routes (services, auth, DB)
- Has **lifecycle** concerns (needs cleanup — DB sessions, HTTP clients)
- Needs to be **swappable** in tests (mock models, mock external APIs)
- Involves **side effects** (auth checks, rate limiting)

Don't use `Depends()` for:
- Simple default values
- Pure computations
- Things only used in one place

---

*Last updated: May 2026*


