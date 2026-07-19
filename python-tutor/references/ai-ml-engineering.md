# AI/ML Engineering Patterns

Best practices for building AI/ML pipelines, data processing, and model engineering.

## Data Processing

### Vectorization with NumPy

```python
# Before: Python loops
results = []
for i in range(len(data)):
    results.append(data[i] * 2 + 1)

# After: Vectorized operations (much faster)
import numpy as np
results = data * 2 + 1
```

### Batch Processing

```python
# Before: Processing one at a time
results = []
for item in dataset:
    result = expensive_model.predict(item)
    results.append(result)

# After: Batch processing
batch_size = 32
results = []
for i in range(0, len(dataset), batch_size):
    batch = dataset[i:i + batch_size]
    batch_results = expensive_model.predict(batch)
    results.extend(batch_results)
```

### Efficient Data Loading

```python
# Before: Loading entire dataset into memory
data = load_entire_dataset()
for batch in create_batches(data):
    process(batch)

# After: Streaming/generator approach
def data_generator(file_path, batch_size):
    while True:
        batch = read_next_batch(file_path, batch_size)
        if not batch:
            break
        yield batch

for batch in data_generator("data.csv", batch_size=1000):
    process(batch)
```

### Pandas Optimization

```python
# Before: Iterating rows (slow)
for idx, row in df.iterrows():
    df.at[idx, 'new_col'] = row['col1'] * row['col2']

# After: Vectorized operations
df['new_col'] = df['col1'] * df['col2']

# Or: apply with numpy functions
df['new_col'] = df.apply(lambda row: complex_func(row['col1'], row['col2']), axis=1)

# Better: Use .values for numpy speed
df['new_col'] = complex_func(df['col1'].values, df['col2'].values)
```

## Model Design Patterns

### Model Registry Pattern

```python
# Before: Hard-coded model selection
if model_type == "linear":
    model = LinearRegression()
elif model_type == "rf":
    model = RandomForest()
# ... many more

# After: Registry pattern
from typing import Callable, Type

MODEL_REGISTRY: dict[str, Type] = {}

def register_model(name: str):
    def decorator(cls: Type):
        MODEL_REGISTRY[name] = cls
        return cls
    return decorator

@register_model("linear")
class LinearModel:
    pass

@register_model("rf")
class RandomForestModel:
    pass

# Usage
model = MODEL_REGISTRY[model_type]()
```

### Pipeline Pattern

```python
# Before: Manual chaining
data = load_data()
data = clean_data(data)
data = normalize_data(data)
features = extract_features(data)
predictions = model.predict(features)

# After: Sklearn pipeline
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

pipeline = Pipeline([
    ('cleaner', DataCleaner()),
    ('normalizer', StandardScaler()),
    ('feature_extractor', FeatureExtractor()),
    ('model', MyModel())
])

predictions = pipeline.fit_predict(X, y)
```

### Model Versioning

```python
# Before: No versioning
model.save("model.pkl")

# After: Versioned with metadata
import json
from datetime import datetime
from pathlib import Path

def save_model_versioned(model, metrics: dict, version: str):
    output_dir = Path(f"models/{version}")
    output_dir.mkdir(parents=True, exist_ok=True)

    # Save model
    model.save(output_dir / "model.pkl")

    # Save metadata
    metadata = {
        "version": version,
        "timestamp": datetime.now().isoformat(),
        "metrics": metrics,
        "python_version": sys.version,
    }
    (output_dir / "metadata.json").write_text(json.dumps(metadata, indent=2))
```

## Feature Engineering

### Feature Store Pattern

```python
# Before: Recomputing features everywhere
def train_model():
    features = compute_features(raw_data)
    model.fit(features, labels)

def serve_predictions():
    features = compute_features(raw_data)  # Duplicate computation!
    return model.predict(features)

# After: Feature store
class FeatureStore:
    def __init__(self, cache_dir: Path):
        self.cache_dir = cache_dir

    def get_features(self, key: str, compute_fn: Callable):
        cache_path = self.cache_dir / f"{key}.parquet"
        if cache_path.exists():
            return pd.read_parquet(cache_path)

        features = compute_fn()
        features.to_parquet(cache_path)
        return features

store = FeatureStore(Path("feature_cache"))
features = store.get_features("user_features_v1", lambda: compute_features(raw_data))
```

### Lazy Feature Computation

```python
# Before: Computing all features upfront
features = {
    "basic": compute_basic_features(data),
    "advanced": compute_advanced_features(data),  # Expensive!
    "premium": compute_premium_features(data),     # Very expensive!
}

# After: Lazy computation
from functools import cached_property

class FeatureSet:
    def __init__(self, data):
        self.data = data

    @cached_property
    def basic(self):
        return compute_basic_features(self.data)

    @cached_property
    def advanced(self):
        return compute_advanced_features(self.data)

    @cached_property
    def premium(self):
        return compute_premium_features(self.data)

# Only computes what's accessed
features = FeatureSet(data)
result = model.predict(features.basic)  # Only basic computed
```

## Async ML Operations

Converting sequential per-item model-API calls to concurrent `asyncio.gather`, and offloading CPU-bound preprocessing/inference out of an `async def` path (`asyncio.to_thread` / `run_in_executor`), are exactly what the `python-async-scaling` skill covers in `asyncio-fundamentals.md` — invoke it instead of reproducing the before/after here. It also flags the specific ML libraries (PyTorch/transformers inference, tokenization, docling, EasyOCR) that silently block the event loop if called directly.

## Memory Optimization

### Memory-Mapped Arrays

```python
# Before: Loading large arrays into memory
import numpy as np
data = np.load("huge_data.npy")  # Loads all into RAM

# After: Memory mapping
data = np.load("huge_data.npy", mmap_mode='r')  # Only loads what's accessed
batch = data[1000:2000]  # Efficient
```

### Chunked Processing

```python
# Before: Processing entire dataframe
df = pd.read_csv("huge_file.csv")
df['processed'] = df['column'].apply(expensive_function)

# After: Chunked processing
def process_in_chunks(file_path, chunk_size=10000):
    chunks = pd.read_csv(file_path, chunksize=chunk_size)
    results = []

    for chunk in chunks:
        chunk['processed'] = chunk['column'].apply(expensive_function)
        results.append(chunk)

    return pd.concat(results, ignore_index=True)
```

### Sparse Matrices

```python
# Before: Dense matrix for sparse data
import numpy as np
matrix = np.zeros((10000, 10000))  # Wastes memory
matrix[sparse_indices] = sparse_values

# After: Sparse matrix
from scipy.sparse import csr_matrix
matrix = csr_matrix((sparse_values, sparse_indices), shape=(10000, 10000))
```

## Model Monitoring

### Metric Tracking

```python
# Before: No tracking
accuracy = evaluate(model, test_data)

# After: Comprehensive tracking
from dataclasses import dataclass
from datetime import datetime

@dataclass
class ModelMetrics:
    accuracy: float
    precision: float
    recall: float
    f1: float
    inference_time_ms: float
    timestamp: datetime
    version: str

def evaluate_with_metrics(model, test_data, version: str) -> ModelMetrics:
    start = time.time()
    predictions = model.predict(test_data)
    inference_time = (time.time() - start) * 1000

    return ModelMetrics(
        accuracy=accuracy_score(test_data.labels, predictions),
        precision=precision_score(test_data.labels, predictions),
        recall=recall_score(test_data.labels, predictions),
        f1=f1_score(test_data.labels, predictions),
        inference_time_ms=inference_time,
        timestamp=datetime.now(),
        version=version
    )
```

## LLM-Specific Patterns

### Token Optimization

```python
# Before: Concatenating everything
prompt = f"Context: {huge_context}\nQuestion: {question}\nAnswer:"

# After: Token-aware truncation
from transformers import AutoTokenizer

def create_prompt_with_limit(context: str, question: str, max_tokens: int = 4000):
    tokenizer = AutoTokenizer.from_pretrained(model_name)

    question_tokens = len(tokenizer.encode(question))
    available_for_context = max_tokens - question_tokens - 50  # Buffer

    context_tokens = tokenizer.encode(context)
    if len(context_tokens) > available_for_context:
        context_tokens = context_tokens[:available_for_context]
        context = tokenizer.decode(context_tokens)

    return f"Context: {context}\nQuestion: {question}\nAnswer:"
```

### Response Caching

```python
# Before: Every request hits LLM
def get_answer(question: str) -> str:
    return llm.generate(question)

# After: Semantic caching
from functools import lru_cache
import hashlib

class SemanticCache:
    def __init__(self):
        self.cache = {}

    def get_or_generate(self, question: str, generator: Callable) -> str:
        # Use embedding similarity for semantic matching
        question_hash = self._get_semantic_hash(question)

        if question_hash in self.cache:
            return self.cache[question_hash]

        result = generator(question)
        self.cache[question_hash] = result
        return result

    def _get_semantic_hash(self, text: str) -> str:
        # Simplified: use actual embeddings in production
        return hashlib.md5(text.lower().strip().encode()).hexdigest()

cache = SemanticCache()
answer = cache.get_or_generate(question, lambda q: llm.generate(q))
```

### Streaming Responses

```python
# Before: Wait for entire response
response = llm.generate(prompt)
print(response)

# After: Stream tokens
def stream_response(prompt: str):
    for token in llm.generate_stream(prompt):
        yield token

# FastAPI example
from fastapi.responses import StreamingResponse

@app.post("/generate")
async def generate_endpoint(prompt: str):
    return StreamingResponse(
        stream_response(prompt),
        media_type="text/plain"
    )
```

## Alternatives and Trade-offs

### Model Selection

**Option 1: Simple sklearn model**
- **Pros**: Fast training, interpretable, small memory footprint
- **Cons**: Limited capacity, may underfit complex patterns
- **Use when**: Fast iteration, interpretability critical, small data

**Option 2: Deep learning (PyTorch/TensorFlow)**
- **Pros**: High capacity, handles complex patterns, transfer learning
- **Cons**: Slow training, requires GPU, harder to debug
- **Use when**: Large data, complex patterns, accuracy critical

**Option 3: LLM via API**
- **Pros**: Zero training, handles varied tasks, quick to prototype
- **Cons**: Expensive per request, latency, no fine-tuning control
- **Use when**: Rapid prototyping, diverse tasks, cost acceptable

### Data Processing

**Option 1: Pandas**
- **Pros**: Convenient API, great for exploration
- **Cons**: Single-threaded, memory intensive
- **Use when**: Medium data (<1GB), complex transformations

**Option 2: Polars**
- **Pros**: Parallel execution, lazy evaluation, memory efficient
- **Cons**: Different API, less ecosystem support
- **Use when**: Large data, performance critical

**Option 3: DuckDB**
- **Pros**: SQL interface, very fast aggregations, handles large data
- **Cons**: SQL-only, less flexibility than dataframes
- **Use when**: Analytics queries, data warehouse style operations

**Option 4: Distributed (Spark/Dask)**
- **Pros**: Scales to cluster, handles massive data
- **Cons**: Complex setup, overhead for small data
- **Use when**: Data doesn't fit on one machine (>100GB)
