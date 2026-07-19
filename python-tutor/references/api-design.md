# API Design Patterns (FastAPI)

Best practices for building robust, scalable APIs with FastAPI.

## Request/Response Models

### Using Pydantic for Validation

```python
# Before: Dict-based request handling
@app.post("/predict")
def predict(data: dict):
    features = data.get("features", [])  # Unsafe
    result = model.predict(features)
    return {"prediction": result}

# After: Pydantic models
from pydantic import BaseModel, Field, validator

class PredictionRequest(BaseModel):
    features: list[float] = Field(..., min_length=1, max_length=100)
    model_version: str = Field(default="v1", pattern="^v[0-9]+$")

    @validator("features")
    def validate_features(cls, v):
        if any(x < 0 for x in v):
            raise ValueError("Features must be non-negative")
        return v

class PredictionResponse(BaseModel):
    prediction: float
    confidence: float = Field(..., ge=0, le=1)
    model_version: str

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    result = model.predict(request.features)
    return PredictionResponse(
        prediction=result.value,
        confidence=result.confidence,
        model_version=request.model_version
    )
```

### Response Model Best Practices

```python
# Before: Exposing internal models
class User:
    id: int
    email: str
    hashed_password: str  # Should not be exposed!

@app.get("/users/{user_id}")
def get_user(user_id: int) -> User:
    return db.get_user(user_id)  # Leaks password!

# After: Separate request/response models
from pydantic import BaseModel

class UserResponse(BaseModel):
    id: int
    email: str
    # No password field

    class Config:
        from_attributes = True  # Previously orm_mode

@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int):
    return db.get_user(user_id)  # Auto-filtered by response_model
```

## Dependency Injection

### Basic Dependencies

```python
# Before: Global state and tight coupling
model = load_model("model.pkl")

@app.post("/predict")
def predict(data: PredictionRequest):
    result = model.predict(data.features)  # Tightly coupled
    return result

# After: Dependency injection
from fastapi import Depends

def get_model():
    return load_model("model.pkl")

@app.post("/predict")
def predict(
    data: PredictionRequest,
    model = Depends(get_model)
):
    result = model.predict(data.features)
    return result
```

### Reusable Dependencies

```python
# Configuration dependency
from functools import lru_cache

class Settings(BaseSettings):
    model_path: str = "models/latest.pkl"
    max_batch_size: int = 32
    redis_url: str = "redis://localhost"

    class Config:
        env_file = ".env"

@lru_cache()
def get_settings() -> Settings:
    return Settings()

# Database session dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Auth dependency
def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials"
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    return get_user(user_id)

# Usage
@app.get("/protected")
def protected_route(
    current_user = Depends(get_current_user),
    settings: Settings = Depends(get_settings)
):
    return {"user": current_user, "config": settings.model_path}
```

### Class-Based Dependencies

```python
# Stateful dependency with initialization
class ModelService:
    def __init__(self, settings: Settings = Depends(get_settings)):
        self.model = load_model(settings.model_path)
        self.cache = {}

    def predict(self, features: list[float]) -> float:
        cache_key = str(features)
        if cache_key in self.cache:
            return self.cache[cache_key]

        result = self.model.predict(features)
        self.cache[cache_key] = result
        return result

@app.post("/predict")
def predict(
    request: PredictionRequest,
    service: ModelService = Depends()
):
    prediction = service.predict(request.features)
    return {"prediction": prediction}
```

## Error Handling

### Custom Exception Handlers

```python
# Before: Generic errors
@app.post("/predict")
def predict(data: PredictionRequest):
    result = model.predict(data.features)
    return result  # What if model.predict fails?

# After: Proper exception handling
from fastapi import HTTPException

class ModelException(Exception):
    """Custom exception for model errors."""
    pass

@app.exception_handler(ModelException)
async def model_exception_handler(request, exc):
    return JSONResponse(
        status_code=500,
        content={
            "error": "model_error",
            "message": str(exc),
            "request_id": request.headers.get("X-Request-ID")
        }
    )

@app.post("/predict")
def predict(data: PredictionRequest):
    try:
        result = model.predict(data.features)
        return {"prediction": result}
    except ValueError as e:
        raise HTTPException(status_code=400, detail=f"Invalid input: {e}")
    except Exception as e:
        raise ModelException(f"Model prediction failed: {e}")
```

### Validation Error Handling

```python
# Custom validation error response
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": " -> ".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })

    return JSONResponse(
        status_code=422,
        content={
            "error": "validation_error",
            "details": errors
        }
    )
```

## Caching

### In-Memory Caching

```python
# Before: No caching
@app.get("/expensive-computation/{param}")
def compute(param: int):
    result = expensive_function(param)  # Always recomputes
    return {"result": result}

# After: LRU cache
from functools import lru_cache

@lru_cache(maxsize=128)
def cached_expensive_function(param: int) -> int:
    return expensive_function(param)

@app.get("/expensive-computation/{param}")
def compute(param: int):
    result = cached_expensive_function(param)
    return {"result": result}
```

### Redis Caching

```python
# Dependency for Redis
import redis
import json

def get_redis():
    return redis.Redis.from_url("redis://localhost")

# Cache decorator
from functools import wraps

def cache_in_redis(ttl: int = 3600):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, redis_client = Depends(get_redis), **kwargs):
            # Create cache key from function name and args
            cache_key = f"{func.__name__}:{args}:{kwargs}"

            # Check cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Compute and cache
            result = await func(*args, **kwargs) if asyncio.iscoroutinefunction(func) else func(*args, **kwargs)
            redis_client.setex(cache_key, ttl, json.dumps(result))
            return result

        return wrapper
    return decorator

# Usage
@app.get("/predictions/{item_id}")
@cache_in_redis(ttl=300)  # 5 minutes
def get_prediction(item_id: int, model = Depends(get_model)):
    return model.predict(item_id)
```

### Response Caching with Headers

```python
from fastapi import Response

@app.get("/static-data")
def get_static_data(response: Response):
    # Set cache headers
    response.headers["Cache-Control"] = "public, max-age=3600"
    response.headers["ETag"] = "version-1.0"

    return {"data": "static content"}
```

## Background Tasks

### Simple Background Tasks

```python
# Before: Blocking operation
@app.post("/process")
def process_data(data: DataRequest):
    result = long_running_task(data)  # Blocks request
    send_email(result)  # Also blocks
    return {"status": "done"}

# After: Background tasks
from fastapi import BackgroundTasks

def send_email_task(email: str, message: str):
    # Send email asynchronously
    email_service.send(email, message)

@app.post("/process")
def process_data(
    data: DataRequest,
    background_tasks: BackgroundTasks
):
    result = long_running_task(data)  # Still blocking, but email isn't

    background_tasks.add_task(send_email_task, data.email, "Processing complete")

    return {"status": "processing", "result": result}
```

### Async Background Processing

```python
# For truly long-running tasks, use task queue
from celery import Celery

celery_app = Celery('tasks', broker='redis://localhost')

@celery_app.task
def expensive_ml_task(data: dict):
    # Long-running ML task
    result = train_model(data)
    return result

@app.post("/train")
def start_training(data: TrainingRequest):
    task = expensive_ml_task.delay(data.dict())

    return {
        "task_id": task.id,
        "status": "queued"
    }

@app.get("/task/{task_id}")
def get_task_status(task_id: str):
    task = celery_app.AsyncResult(task_id)
    return {
        "task_id": task_id,
        "status": task.state,
        "result": task.result if task.ready() else None
    }
```

## API Versioning

### URL Path Versioning

```python
# Version 1
@app.get("/v1/predictions")
def get_predictions_v1():
    return {"version": "1.0", "data": []}

# Version 2 with breaking changes
@app.get("/v2/predictions")
def get_predictions_v2():
    return {
        "version": "2.0",
        "predictions": [],  # Changed field name
        "metadata": {}      # New field
    }
```

### APIRouter for Versioning

```python
from fastapi import APIRouter

# v1 router
v1_router = APIRouter(prefix="/v1")

@v1_router.get("/predictions")
def get_predictions():
    return {"version": "1.0"}

# v2 router
v2_router = APIRouter(prefix="/v2")

@v2_router.get("/predictions")
def get_predictions():
    return {"version": "2.0"}

# Register routers
app.include_router(v1_router)
app.include_router(v2_router)
```

## Request Validation

### Path and Query Parameters

```python
from fastapi import Path, Query

@app.get("/items/{item_id}")
def get_item(
    item_id: int = Path(..., gt=0, description="The item ID"),
    skip: int = Query(0, ge=0, description="Number of items to skip"),
    limit: int = Query(10, ge=1, le=100, description="Number of items to return")
):
    return {"item_id": item_id, "skip": skip, "limit": limit}
```

### Custom Validators

```python
from pydantic import validator, root_validator

class TrainingRequest(BaseModel):
    model_type: str
    epochs: int
    batch_size: int
    learning_rate: float

    @validator("model_type")
    def validate_model_type(cls, v):
        allowed = {"linear", "rf", "nn"}
        if v not in allowed:
            raise ValueError(f"model_type must be one of {allowed}")
        return v

    @validator("epochs")
    def validate_epochs(cls, v):
        if v < 1 or v > 1000:
            raise ValueError("epochs must be between 1 and 1000")
        return v

    @root_validator
    def validate_batch_size_with_model(cls, values):
        model_type = values.get("model_type")
        batch_size = values.get("batch_size")

        if model_type == "nn" and batch_size < 8:
            raise ValueError("Neural network requires batch_size >= 8")

        return values
```

## Middleware

### Logging Middleware

```python
import time
import structlog
from fastapi import Request

logger = structlog.get_logger()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()

    # Log request
    logger.info(
        "request_started",
        method=request.method,
        path=request.url.path,
        client=request.client.host
    )

    # Process request
    response = await call_next(request)

    # Log response
    duration = time.time() - start_time
    logger.info(
        "request_completed",
        method=request.method,
        path=request.url.path,
        status_code=response.status_code,
        duration_ms=duration * 1000
    )

    return response
```

### CORS Middleware

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],  # Or ["*"] for all
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Testing FastAPI

### Test Client

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_predict_endpoint():
    response = client.post(
        "/predict",
        json={"features": [1.0, 2.0, 3.0]}
    )
    assert response.status_code == 200
    assert "prediction" in response.json()

def test_predict_validation_error():
    response = client.post(
        "/predict",
        json={"features": []}  # Invalid - too few features
    )
    assert response.status_code == 422
```

### Testing with Dependencies

```python
# Override dependencies in tests
def get_mock_model():
    class MockModel:
        def predict(self, features):
            return 42.0
    return MockModel()

app.dependency_overrides[get_model] = get_mock_model

def test_with_mock_model():
    response = client.post(
        "/predict",
        json={"features": [1.0, 2.0]}
    )
    assert response.json()["prediction"] == 42.0

# Clean up
app.dependency_overrides = {}
```

## Documentation

### Enhanced OpenAPI Docs

```python
@app.post(
    "/predict",
    response_model=PredictionResponse,
    summary="Make a prediction",
    description="Predicts the target value based on input features using the trained model.",
    responses={
        200: {
            "description": "Successful prediction",
            "content": {
                "application/json": {
                    "example": {
                        "prediction": 42.5,
                        "confidence": 0.95,
                        "model_version": "v1"
                    }
                }
            }
        },
        400: {"description": "Invalid input features"},
        500: {"description": "Model prediction error"}
    },
    tags=["predictions"]
)
def predict(request: PredictionRequest):
    """
    Make a prediction using the trained model.

    - **features**: List of numeric feature values
    - **model_version**: Version of the model to use (default: v1)
    """
    pass
```

## Performance Optimization

### Async Endpoints

```python
# Before: Blocking I/O
@app.get("/fetch-data")
def fetch_data():
    response = requests.get("https://api.example.com/data")  # Blocks
    return response.json()

# After: Async
import httpx

@app.get("/fetch-data")
async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

### Connection Pooling

```python
# Reuse HTTP client
from httpx import AsyncClient

class HTTPClientManager:
    def __init__(self):
        self.client = None

    async def __aenter__(self):
        self.client = AsyncClient()
        return self.client

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.client.aclose()

@app.on_event("startup")
async def startup():
    app.state.http_client = AsyncClient()

@app.on_event("shutdown")
async def shutdown():
    await app.state.http_client.aclose()

@app.get("/data")
async def get_data(request: Request):
    client = request.app.state.http_client
    response = await client.get("https://api.example.com/data")
    return response.json()
```
