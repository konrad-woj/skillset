# Python Tutor: Worked Examples

## Example 1: FastAPI Endpoint

**Context:** User is adding a new prediction endpoint

```python
@app.post("/predict")
def predict(data: dict):
    features = data["features"]
    result = model.predict(features)
    return {"prediction": result}
```

**Hint: API Design** — Use Pydantic for validation

FastAPI's request validation prevents runtime errors and auto-generates API docs.

```python
# Before
@app.post("/predict")
def predict(data: dict):
    features = data["features"]  # Unsafe — no validation
    result = model.predict(features)
    return {"prediction": result}

# After
from pydantic import BaseModel, Field

class PredictionRequest(BaseModel):
    features: list[float] = Field(..., min_length=1)

class PredictionResponse(BaseModel):
    prediction: float
    confidence: float

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    result = model.predict(request.features)
    return PredictionResponse(prediction=result.value, confidence=result.confidence)
```

**Evaluation:** Add integration tests for this endpoint. See `references/testing-evaluation.md` for FastAPI testing patterns.

---

## Example 2: ML Data Processing

**Context:** User is processing a large dataset

```python
results = []
for item in dataset:
    processed = expensive_function(item)
    results.append(processed)
```

**Hint: Performance** — Use batch processing

Processing one item at a time is slow. Batching dramatically improves throughput.

```python
# Before
results = []
for item in dataset:
    processed = expensive_function(item)  # One at a time
    results.append(processed)

# After
batch_size = 32
results = []
for i in range(0, len(dataset), batch_size):
    batch = dataset[i:i + batch_size]
    batch_results = expensive_function(batch)  # 32 at once
    results.extend(batch_results)
```

**Alternative approaches:**
- **Vectorization with NumPy**: if `expensive_function` can be vectorized (see `references/ai-ml-engineering.md`)
- **Parallel processing**: `multiprocessing` or `joblib.Parallel` for CPU-bound work
- **Async**: `asyncio.gather()` for I/O-bound work

**Evaluation:** Benchmark the improvement. Expected: 10–50x faster depending on batch overhead.

---

## Example 3: LLM Prompt

**Context:** User is building a prompt

```python
prompt = f"Extract the name from: {text}"
response = llm.generate(prompt)
```

**Hint: Prompt Engineering** — Structure prompts clearly

Structured prompts with clear instructions improve consistency and output quality.

```python
# Before
prompt = f"Extract the name from: {text}"

# After
from jinja2 import Template

PROMPT_TEMPLATE = """
Task: Extract the person's name from the given text.

Text:
{{ text }}

Instructions:
- Return only the name, no other text
- If no name found, return "NONE"
- Use proper capitalization

Output:
"""

template = Template(PROMPT_TEMPLATE)
prompt = template.render(text=text)
```

**Evaluation approach:**
1. Create eval dataset with known names: `tests/eval/name_extraction.json`
2. Define success metric: exact match accuracy > 90%
3. Track false positives (extracting non-names)

See `references/prompt-engineering.md` for template organization and evaluation patterns.

---

## Example 4: Missing Evaluation

**Context:** User implemented a new classifier but no tests

```python
class NewClassifier:
    def predict(self, features):
        # ... implementation
        return prediction
```

**Hint: Testing** — Add acceptance criteria and eval dataset

Without evaluation, you can't verify the classifier works or detect regressions.

```python
# Add acceptance criteria
@dataclass
class AcceptanceCriteria:
    min_accuracy: float = 0.85
    min_precision: float = 0.80
    max_inference_time_ms: float = 100

# Create eval dataset
eval_dataset = [
    {"input": [...], "expected": "class_a"},
    {"input": [...], "expected": "class_b"},
]

# Test against criteria
def test_classifier_acceptance():
    classifier = NewClassifier()
    predictions = [classifier.predict(ex["input"]) for ex in eval_dataset]
    actuals = [ex["expected"] for ex in eval_dataset]

    accuracy = accuracy_score(actuals, predictions)
    assert accuracy >= 0.85, f"Accuracy {accuracy:.2%} below threshold"
```

**Next steps:**
1. Define metrics that matter for this use case
2. Collect representative test cases (edge cases, common cases, failures)
3. Set thresholds based on business requirements

See `references/testing-evaluation.md` for comprehensive evaluation patterns.
