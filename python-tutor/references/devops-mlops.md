# DevOps and MLOps Patterns

Best practices for Docker, CI/CD, deployment, monitoring, and ML operations.

## Docker Patterns

### Multi-Stage Builds

```dockerfile
# Before: Single stage (large image)
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]

# After: Multi-stage (smaller final image)
# Build stage
FROM python:3.11 as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Runtime stage
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

### Layer Optimization

```dockerfile
# Before: Poor layer caching
FROM python:3.11
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# After: Optimized layers (requirements cached separately)
FROM python:3.11
WORKDIR /app

# Install dependencies first (cached layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code (changes frequently)
COPY . .

CMD ["python", "app.py"]
```

### Production Best Practices

```dockerfile
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./app:/app  # Mount for development
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## CI/CD Patterns

### GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

# Run tests
test:
  stage: test
  image: python:3.11
  before_script:
    - pip install -r requirements.txt
    - pip install pytest pytest-cov
  script:
    - pytest tests/unit --cov=app --cov-report=xml
    - pytest tests/integration
  coverage: '/(?i)total.*? (100(?:\.0+)?\%|[1-9]?\d(?:\.\d+)?\%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# Lint and type check
lint:
  stage: test
  image: python:3.11
  before_script:
    - pip install ruff pyright
  script:
    - ruff check .
    - pyright

# Build Docker image
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
  only:
    - main
    - develop

# Deploy to staging
deploy_staging:
  stage: deploy
  image: google/cloud-sdk:alpine
  before_script:
    - echo $GCP_SERVICE_KEY | base64 -d > ${HOME}/gcp-key.json
    - gcloud auth activate-service-account --key-file ${HOME}/gcp-key.json
    - gcloud config set project $GCP_PROJECT_ID
  script:
    - gcloud run deploy my-api-staging
        --image $IMAGE_NAME
        --platform managed
        --region us-central1
        --allow-unauthenticated
  environment:
    name: staging
    url: https://my-api-staging.run.app
  only:
    - develop

# Deploy to production
deploy_production:
  stage: deploy
  image: google/cloud-sdk:alpine
  before_script:
    - echo $GCP_SERVICE_KEY | base64 -d > ${HOME}/gcp-key.json
    - gcloud auth activate-service-account --key-file ${HOME}/gcp-key.json
    - gcloud config set project $GCP_PROJECT_ID
  script:
    - gcloud run deploy my-api-prod
        --image $IMAGE_NAME
        --platform managed
        --region us-central1
        --allow-unauthenticated
  environment:
    name: production
    url: https://my-api-prod.run.app
  when: manual  # Require manual approval
  only:
    - main
```

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run tests
        run: |
          pytest tests/unit --cov=app --cov-report=xml
          pytest tests/integration

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_REGISTRY }}/my-api:${{ github.sha }}
```

## ML Model Deployment

### Model Versioning

```python
# Model registry structure
from pathlib import Path
from datetime import datetime
import json
import joblib

class ModelRegistry:
    def __init__(self, registry_path: Path):
        self.registry_path = registry_path
        self.registry_path.mkdir(parents=True, exist_ok=True)

    def save_model(
        self,
        model,
        version: str,
        metrics: dict,
        metadata: dict = None
    ) -> Path:
        """Save model with version and metadata."""
        version_dir = self.registry_path / version
        version_dir.mkdir(exist_ok=True)

        # Save model
        model_path = version_dir / "model.pkl"
        joblib.dump(model, model_path)

        # Save metadata
        meta = {
            "version": version,
            "timestamp": datetime.now().isoformat(),
            "metrics": metrics,
            "metadata": metadata or {}
        }

        meta_path = version_dir / "metadata.json"
        meta_path.write_text(json.dumps(meta, indent=2))

        # Update latest symlink
        latest_link = self.registry_path / "latest"
        if latest_link.exists():
            latest_link.unlink()
        latest_link.symlink_to(version_dir)

        return model_path

    def load_model(self, version: str = "latest"):
        """Load model by version."""
        if version == "latest":
            model_path = self.registry_path / "latest" / "model.pkl"
        else:
            model_path = self.registry_path / version / "model.pkl"

        return joblib.load(model_path)

    def get_metadata(self, version: str = "latest") -> dict:
        """Get model metadata."""
        if version == "latest":
            meta_path = self.registry_path / "latest" / "metadata.json"
        else:
            meta_path = self.registry_path / version / "metadata.json"

        return json.loads(meta_path.read_text())
```

### Model Serving with FastAPI

```python
# Production-ready model serving
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
import structlog
from prometheus_client import Counter, Histogram, generate_latest
import time

logger = structlog.get_logger()

# Metrics
prediction_counter = Counter('predictions_total', 'Total predictions')
prediction_latency = Histogram('prediction_latency_seconds', 'Prediction latency')
prediction_errors = Counter('prediction_errors_total', 'Total prediction errors')

app = FastAPI()

# Load model at startup
registry = ModelRegistry(Path("models"))
model = None

@app.on_event("startup")
async def load_model():
    global model
    model = registry.load_model("latest")
    metadata = registry.get_metadata("latest")
    logger.info("model_loaded", version=metadata["version"], metrics=metadata["metrics"])

class PredictionRequest(BaseModel):
    features: list[float]

class PredictionResponse(BaseModel):
    prediction: float
    model_version: str
    latency_ms: float

@app.post("/predict", response_model=PredictionResponse)
async def predict(
    request: PredictionRequest,
    background_tasks: BackgroundTasks
):
    start_time = time.time()

    try:
        # Make prediction
        prediction = model.predict([request.features])[0]

        # Calculate latency
        latency = (time.time() - start_time) * 1000

        # Log prediction (in background)
        background_tasks.add_task(
            log_prediction,
            request.features,
            prediction,
            latency
        )

        # Update metrics
        prediction_counter.inc()
        prediction_latency.observe(latency / 1000)

        # Get model version
        metadata = registry.get_metadata("latest")

        return PredictionResponse(
            prediction=float(prediction),
            model_version=metadata["version"],
            latency_ms=latency
        )

    except Exception as e:
        prediction_errors.inc()
        logger.error("prediction_error", error=str(e))
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics")
def metrics():
    """Prometheus metrics endpoint."""
    return generate_latest()

@app.get("/health")
def health():
    """Health check endpoint."""
    return {
        "status": "healthy",
        "model_loaded": model is not None
    }

def log_prediction(features: list[float], prediction: float, latency: float):
    """Log prediction for monitoring."""
    logger.info(
        "prediction_made",
        features=features,
        prediction=prediction,
        latency_ms=latency
    )
```

## Monitoring and Observability

### Structured Logging

```python
# Configure structured logging
import structlog
from structlog.processors import JSONRenderer

def configure_logging():
    structlog.configure(
        processors=[
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.stdlib.add_log_level,
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            JSONRenderer()
        ],
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

configure_logging()
logger = structlog.get_logger()

# Usage with context
def process_request(request_id: str, data: dict):
    log = logger.bind(request_id=request_id)

    log.info("processing_started", data_size=len(data))

    try:
        result = expensive_operation(data)
        log.info("processing_completed", result_size=len(result))
        return result
    except Exception as e:
        log.error("processing_failed", error=str(e), exc_info=True)
        raise
```

### Application Metrics

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge, Summary
import time

# Counters
requests_total = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
errors_total = Counter('errors_total', 'Total errors', ['error_type'])

# Histograms (for latency)
request_latency = Histogram('http_request_duration_seconds', 'HTTP request latency', ['endpoint'])
model_inference_time = Histogram('model_inference_seconds', 'Model inference time')

# Gauges (for current state)
active_requests = Gauge('active_requests', 'Number of active requests')
model_load_time = Gauge('model_load_time_seconds', 'Time to load model')

# Summary (for quantiles)
prediction_confidence = Summary('prediction_confidence', 'Prediction confidence scores')

# Usage in endpoint
@app.post("/predict")
@active_requests.track_inprogress()
async def predict(request: PredictionRequest):
    start = time.time()

    try:
        # Make prediction
        with model_inference_time.time():
            result = model.predict(request.features)

        # Track metrics
        requests_total.labels(method='POST', endpoint='/predict', status='200').inc()
        request_latency.labels(endpoint='/predict').observe(time.time() - start)
        prediction_confidence.observe(result.confidence)

        return result

    except Exception as e:
        errors_total.labels(error_type=type(e).__name__).inc()
        requests_total.labels(method='POST', endpoint='/predict', status='500').inc()
        raise
```

### Model Monitoring

```python
# Monitor model performance in production
from dataclasses import dataclass
from datetime import datetime, timedelta
from collections import deque

@dataclass
class PredictionLog:
    timestamp: datetime
    features: list[float]
    prediction: float
    confidence: float
    actual: float = None  # Filled in later when ground truth available

class ModelMonitor:
    def __init__(self, window_size: int = 1000):
        self.predictions = deque(maxlen=window_size)
        self.drift_threshold = 0.1

    def log_prediction(self, features, prediction, confidence):
        """Log a prediction."""
        self.predictions.append(PredictionLog(
            timestamp=datetime.now(),
            features=features,
            prediction=prediction,
            confidence=confidence
        ))

    def update_actual(self, prediction_id: int, actual: float):
        """Update with ground truth when available."""
        if prediction_id < len(self.predictions):
            self.predictions[prediction_id].actual = actual

    def check_data_drift(self) -> bool:
        """Check for data drift using feature distributions."""
        if len(self.predictions) < 100:
            return False

        # Get recent and historical feature distributions
        recent = list(self.predictions)[-100:]
        historical = list(self.predictions)[:-100]

        # Compare distributions (simplified)
        from scipy.stats import ks_2samp

        for feature_idx in range(len(recent[0].features)):
            recent_vals = [p.features[feature_idx] for p in recent]
            hist_vals = [p.features[feature_idx] for p in historical]

            statistic, pvalue = ks_2samp(recent_vals, hist_vals)

            if pvalue < 0.05:  # Significant difference
                logger.warning(
                    "data_drift_detected",
                    feature_idx=feature_idx,
                    statistic=statistic,
                    pvalue=pvalue
                )
                return True

        return False

    def calculate_accuracy(self) -> float:
        """Calculate accuracy on predictions with ground truth."""
        with_actuals = [p for p in self.predictions if p.actual is not None]

        if len(with_actuals) < 10:
            return None

        correct = sum(
            abs(p.prediction - p.actual) < 0.1
            for p in with_actuals
        )

        return correct / len(with_actuals)

    def get_metrics(self) -> dict:
        """Get monitoring metrics."""
        return {
            "num_predictions": len(self.predictions),
            "avg_confidence": sum(p.confidence for p in self.predictions) / len(self.predictions),
            "accuracy": self.calculate_accuracy(),
            "data_drift": self.check_data_drift()
        }

# Use in application
monitor = ModelMonitor()

@app.post("/predict")
def predict(request: PredictionRequest):
    result = model.predict(request.features)

    # Log for monitoring
    monitor.log_prediction(
        request.features,
        result.prediction,
        result.confidence
    )

    return result

@app.get("/monitor/metrics")
def monitoring_metrics():
    return monitor.get_metrics()
```

## Deployment Strategies

### Blue-Green Deployment

```python
# Route traffic between two versions
from fastapi import Request

class DeploymentRouter:
    def __init__(self):
        self.blue_model = load_model("v1")
        self.green_model = load_model("v2")
        self.traffic_split = 0.0  # 0 = all blue, 1 = all green

    def set_traffic_split(self, split: float):
        """Set traffic split (0.0 to 1.0)."""
        self.traffic_split = max(0.0, min(1.0, split))

    def get_model(self, request_id: str):
        """Route to model based on traffic split."""
        import hashlib
        hash_val = int(hashlib.md5(request_id.encode()).hexdigest(), 16)
        use_green = (hash_val % 100) / 100 < self.traffic_split

        return self.green_model if use_green else self.blue_model

router = DeploymentRouter()

@app.post("/predict")
def predict(request: PredictionRequest, req: Request):
    request_id = req.headers.get("X-Request-ID", str(time.time()))
    model = router.get_model(request_id)

    return model.predict(request.features)

@app.post("/admin/traffic-split")
def set_traffic_split(split: float):
    """Gradually shift traffic to new model."""
    router.set_traffic_split(split)
    return {"traffic_split": router.traffic_split}
```

### Canary Deployment

```python
# Gradually roll out new version
class CanaryDeployment:
    def __init__(self, stable_model, canary_model):
        self.stable = stable_model
        self.canary = canary_model
        self.canary_percentage = 0
        self.canary_metrics = {"success": 0, "errors": 0}
        self.stable_metrics = {"success": 0, "errors": 0}

    def should_use_canary(self) -> bool:
        """Decide if this request should use canary."""
        import random
        return random.random() < (self.canary_percentage / 100)

    def predict(self, features):
        """Route to appropriate model."""
        use_canary = self.should_use_canary()
        model = self.canary if use_canary else self.stable
        metrics = self.canary_metrics if use_canary else self.stable_metrics

        try:
            result = model.predict(features)
            metrics["success"] += 1
            return result
        except Exception as e:
            metrics["errors"] += 1
            # Fallback to stable on canary error
            if use_canary:
                return self.stable.predict(features)
            raise

    def increase_canary(self, increment: int = 10):
        """Increase canary traffic if metrics look good."""
        canary_error_rate = (
            self.canary_metrics["errors"] /
            max(1, self.canary_metrics["success"] + self.canary_metrics["errors"])
        )

        stable_error_rate = (
            self.stable_metrics["errors"] /
            max(1, self.stable_metrics["success"] + self.stable_metrics["errors"])
        )

        # Only increase if canary error rate is acceptable
        if canary_error_rate <= stable_error_rate * 1.1:  # Allow 10% worse
            self.canary_percentage = min(100, self.canary_percentage + increment)
            return True

        return False

    def get_status(self) -> dict:
        return {
            "canary_percentage": self.canary_percentage,
            "canary_metrics": self.canary_metrics,
            "stable_metrics": self.stable_metrics
        }
```

## Infrastructure as Code

### Terraform for Cloud Resources

```hcl
# terraform/main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Cloud Run service
resource "google_cloud_run_service" "api" {
  name     = "ml-api"
  location = var.region

  template {
    spec {
      containers {
        image = var.image_url

        resources {
          limits = {
            cpu    = "2"
            memory = "2Gi"
          }
        }

        env {
          name  = "MODEL_PATH"
          value = "gs://${google_storage_bucket.models.name}/latest/model.pkl"
        }
      }

      service_account_name = google_service_account.api.email
    }

    metadata {
      annotations = {
        "autoscaling.knative.dev/minScale" = "1"
        "autoscaling.knative.dev/maxScale" = "10"
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}

# Storage bucket for models
resource "google_storage_bucket" "models" {
  name     = "${var.project_id}-ml-models"
  location = var.region

  versioning {
    enabled = true
  }
}

# Service account
resource "google_service_account" "api" {
  account_id   = "ml-api-sa"
  display_name = "ML API Service Account"
}

# IAM binding for storage access
resource "google_storage_bucket_iam_member" "api_models_access" {
  bucket = google_storage_bucket.models.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.api.email}"
}

# Output
output "api_url" {
  value = google_cloud_run_service.api.status[0].url
}
```

## Best Practices Summary

1. **Docker**: Use multi-stage builds, optimize layers, run as non-root
2. **CI/CD**: Automated testing, linting, and deployment pipelines
3. **Model versioning**: Track models with metadata and metrics
4. **Model serving**: Production-ready API with error handling and monitoring
5. **Monitoring**: Structured logging, metrics, and model performance tracking
6. **Deployment strategies**: Blue-green or canary deployments for safe rollouts
7. **Infrastructure as Code**: Version control infrastructure definitions
8. **Security**: Non-root containers, secrets management, least privilege IAM
9. **Scalability**: Horizontal scaling, load balancing, caching
10. **Observability**: Logs, metrics, traces for debugging and optimization
