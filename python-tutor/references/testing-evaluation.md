# Testing and Evaluation Patterns

Best practices for testing ML systems, creating evaluation datasets, and defining acceptance criteria.

## Unit Testing Patterns

### Basic Test Structure

```python
# Before: Unstructured tests
def test_something():
    result = my_function(input_data)
    assert result == expected

# After: Arrange-Act-Assert pattern
def test_feature_extraction_returns_correct_shape():
    # Arrange
    input_data = create_sample_data(n_samples=100, n_features=10)
    extractor = FeatureExtractor(output_dim=5)

    # Act
    features = extractor.transform(input_data)

    # Assert
    assert features.shape == (100, 5)
    assert features.dtype == np.float32
```

### Parametrized Tests

```python
# Before: Duplicate test code
def test_model_with_input_1():
    assert model.predict(input_1) == expected_1

def test_model_with_input_2():
    assert model.predict(input_2) == expected_2

# After: Parametrized
import pytest

@pytest.mark.parametrize("input_data,expected", [
    (input_1, expected_1),
    (input_2, expected_2),
    (input_3, expected_3),
])
def test_model_predictions(input_data, expected):
    assert model.predict(input_data) == expected
```

### Fixtures for Test Data

```python
# Before: Creating test data in every test
def test_model_accuracy():
    X_train = np.random.rand(100, 10)
    y_train = np.random.randint(0, 2, 100)
    model = MyModel()
    model.fit(X_train, y_train)
    # ...

def test_model_precision():
    X_train = np.random.rand(100, 10)  # Duplicate!
    y_train = np.random.randint(0, 2, 100)
    model = MyModel()
    model.fit(X_train, y_train)
    # ...

# After: Fixtures
@pytest.fixture
def sample_dataset():
    X = np.random.rand(100, 10)
    y = np.random.randint(0, 2, 100)
    return X, y

@pytest.fixture
def trained_model(sample_dataset):
    X, y = sample_dataset
    model = MyModel()
    model.fit(X, y)
    return model

def test_model_accuracy(trained_model, sample_dataset):
    X, y = sample_dataset
    accuracy = trained_model.score(X, y)
    assert accuracy > 0.8

def test_model_precision(trained_model, sample_dataset):
    X, y = sample_dataset
    predictions = trained_model.predict(X)
    precision = precision_score(y, predictions)
    assert precision > 0.75
```

### Mocking External Dependencies

```python
# Before: Testing with real API calls
def test_model_inference():
    result = model_api.predict(test_input)  # Slow, flaky, costs money
    assert result['class'] == 'positive'

# After: Mocking
from unittest.mock import Mock, patch

def test_model_inference():
    mock_response = {'class': 'positive', 'confidence': 0.95}

    with patch('model_api.predict', return_value=mock_response):
        result = model_api.predict(test_input)
        assert result['class'] == 'positive'

# Or with pytest-mock
def test_model_inference_with_pytest_mock(mocker):
    mock_predict = mocker.patch('model_api.predict')
    mock_predict.return_value = {'class': 'positive', 'confidence': 0.95}

    result = model_api.predict(test_input)
    assert result['class'] == 'positive'
    mock_predict.assert_called_once_with(test_input)
```

## Integration Testing

### Testing ML Pipelines

```python
# Test entire pipeline end-to-end
def test_training_pipeline_produces_valid_model():
    # Arrange
    config = TrainingConfig(
        data_path="tests/data/sample.csv",
        model_type="random_forest",
        output_path="tests/output/model.pkl"
    )

    # Act
    train_model(config)

    # Assert
    assert Path(config.output_path).exists()
    model = load_model(config.output_path)
    assert hasattr(model, 'predict')

    # Verify model works
    test_data = load_test_data()
    predictions = model.predict(test_data)
    assert len(predictions) == len(test_data)
```

### Testing API Endpoints

```python
# Testing FastAPI endpoints
from fastapi.testclient import TestClient

@pytest.fixture
def client():
    return TestClient(app)

def test_predict_endpoint_returns_valid_prediction(client):
    # Arrange
    request_data = {
        "features": [1.0, 2.0, 3.0, 4.0]
    }

    # Act
    response = client.post("/predict", json=request_data)

    # Assert
    assert response.status_code == 200
    result = response.json()
    assert "prediction" in result
    assert "confidence" in result
    assert 0 <= result["confidence"] <= 1

def test_predict_endpoint_handles_invalid_input(client):
    request_data = {"features": []}  # Invalid

    response = client.post("/predict", json=request_data)

    assert response.status_code == 422  # Validation error
```

## Evaluation Dataset Design

### Creating Representative Test Sets

```python
# Before: Random split (can leak info)
from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

# After: Stratified split (maintains class distribution)
from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

# For time-series: time-based split
split_date = "2024-01-01"
train_data = data[data['date'] < split_date]
test_data = data[data['date'] >= split_date]
```

### Evaluation Dataset Structure

```python
# Store evaluation sets with metadata
from dataclasses import dataclass
from pathlib import Path
import json

@dataclass
class EvaluationDataset:
    name: str
    version: str
    samples: list[dict]
    metadata: dict

    def save(self, path: Path):
        output = {
            "name": self.name,
            "version": self.version,
            "samples": self.samples,
            "metadata": self.metadata
        }
        path.write_text(json.dumps(output, indent=2))

    @classmethod
    def load(cls, path: Path):
        data = json.loads(path.read_text())
        return cls(**data)

# Usage
eval_dataset = EvaluationDataset(
    name="user_intent_classification",
    version="v1.2",
    samples=[
        {"input": "Book a flight", "expected_intent": "booking", "expected_confidence": ">0.8"},
        {"input": "What's the weather?", "expected_intent": "query", "expected_confidence": ">0.7"},
    ],
    metadata={
        "created_date": "2024-01-15",
        "source": "production_errors_2023",
        "num_samples": 100
    }
)
eval_dataset.save(Path("eval_datasets/user_intent_v1.2.json"))
```

### Golden Test Sets

```python
# Maintain golden examples for regression testing
class GoldenTestSet:
    """Test set that should never change - for detecting regressions."""

    def __init__(self, path: Path):
        self.path = path
        self.samples = self._load()

    def _load(self) -> list[dict]:
        return json.loads(self.path.read_text())

    def evaluate(self, model) -> dict:
        results = []
        for sample in self.samples:
            prediction = model.predict(sample["input"])
            results.append({
                "input": sample["input"],
                "expected": sample["expected"],
                "predicted": prediction,
                "match": prediction == sample["expected"]
            })

        accuracy = sum(r["match"] for r in results) / len(results)
        return {
            "accuracy": accuracy,
            "results": results,
            "passing": accuracy >= 0.95  # High bar for golden set
        }

# Usage in CI/CD
golden_set = GoldenTestSet(Path("tests/golden/classification_v1.json"))
results = golden_set.evaluate(new_model)
assert results["passing"], f"Golden set accuracy {results['accuracy']:.2%} below threshold"
```

## Acceptance Criteria Patterns

### Metric-Based Criteria

```python
# Define clear acceptance criteria
@dataclass
class AcceptanceCriteria:
    min_accuracy: float = 0.85
    min_precision: float = 0.80
    min_recall: float = 0.80
    max_inference_time_ms: float = 100
    max_memory_mb: float = 512

def evaluate_acceptance(model, test_data, criteria: AcceptanceCriteria) -> dict:
    # Performance metrics
    predictions = model.predict(test_data.X)
    accuracy = accuracy_score(test_data.y, predictions)
    precision = precision_score(test_data.y, predictions, average='weighted')
    recall = recall_score(test_data.y, predictions, average='weighted')

    # Speed metrics
    import time
    start = time.time()
    _ = model.predict(test_data.X[:100])
    inference_time = (time.time() - start) / 100 * 1000

    # Memory metrics
    import sys
    memory_mb = sys.getsizeof(model) / (1024 * 1024)

    results = {
        "accuracy": accuracy,
        "precision": precision,
        "recall": recall,
        "inference_time_ms": inference_time,
        "memory_mb": memory_mb,
    }

    # Check criteria
    passes = (
        accuracy >= criteria.min_accuracy and
        precision >= criteria.min_precision and
        recall >= criteria.min_recall and
        inference_time <= criteria.max_inference_time_ms and
        memory_mb <= criteria.max_memory_mb
    )

    return {
        "metrics": results,
        "passes": passes,
        "failures": _identify_failures(results, criteria)
    }

def _identify_failures(results: dict, criteria: AcceptanceCriteria) -> list[str]:
    failures = []
    if results["accuracy"] < criteria.min_accuracy:
        failures.append(f"Accuracy {results['accuracy']:.2%} below {criteria.min_accuracy:.2%}")
    if results["precision"] < criteria.min_precision:
        failures.append(f"Precision {results['precision']:.2%} below {criteria.min_precision:.2%}")
    # ... more checks
    return failures
```

### Business Logic Tests

```python
# Test business rules, not just metrics
def test_model_never_predicts_invalid_category():
    """Business rule: Model should only predict valid product categories."""
    valid_categories = {"electronics", "clothing", "food", "books"}

    test_inputs = load_diverse_test_inputs()
    predictions = [model.predict(input) for input in test_inputs]

    invalid_predictions = [p for p in predictions if p not in valid_categories]
    assert len(invalid_predictions) == 0, f"Found invalid predictions: {invalid_predictions}"

def test_model_high_confidence_predictions_are_accurate():
    """Business rule: When model is >90% confident, it should be >95% accurate."""
    test_data = load_test_data()

    high_conf_samples = []
    for sample in test_data:
        prediction, confidence = model.predict_with_confidence(sample.input)
        if confidence > 0.90:
            high_conf_samples.append({
                "prediction": prediction,
                "actual": sample.label,
                "confidence": confidence
            })

    if len(high_conf_samples) > 0:
        accuracy = sum(s["prediction"] == s["actual"] for s in high_conf_samples) / len(high_conf_samples)
        assert accuracy >= 0.95, f"High-confidence accuracy {accuracy:.2%} below 95%"
```

## Property-Based Testing

```python
# Use hypothesis for property-based testing
from hypothesis import given, strategies as st

@given(st.lists(st.floats(min_value=0, max_value=100), min_size=1, max_size=100))
def test_feature_extractor_output_range(input_values):
    """Property: Feature extractor output should always be normalized [0, 1]."""
    features = feature_extractor.transform(input_values)
    assert all(0 <= f <= 1 for f in features)

@given(st.lists(st.text(min_size=1, max_size=1000), min_size=1, max_size=50))
def test_tokenizer_preserves_length(texts):
    """Property: Number of tokenized documents equals input documents."""
    tokens = tokenizer.tokenize(texts)
    assert len(tokens) == len(texts)
```

## Model Comparison Testing

```python
# Compare new model against baseline
def test_new_model_outperforms_baseline():
    baseline_model = load_model("models/baseline_v1.pkl")
    new_model = load_model("models/candidate_v2.pkl")
    test_data = load_test_data()

    baseline_accuracy = baseline_model.score(test_data.X, test_data.y)
    new_accuracy = new_model.score(test_data.X, test_data.y)

    # New model must be at least 2% better
    improvement = new_accuracy - baseline_accuracy
    assert improvement >= 0.02, f"New model only {improvement:.2%} better than baseline"

# A/B test simulation
def test_ab_test_simulation():
    """Simulate A/B test to ensure new model is better."""
    from scipy import stats

    baseline_results = run_model_on_dataset(baseline_model, test_data)
    new_results = run_model_on_dataset(new_model, test_data)

    # Statistical significance test
    t_stat, p_value = stats.ttest_ind(baseline_results, new_results)

    assert p_value < 0.05, "Improvement not statistically significant"
    assert new_results.mean() > baseline_results.mean(), "New model not better"
```

## Continuous Evaluation

```python
# Monitor model performance over time
class ModelMonitor:
    def __init__(self, model, eval_dataset: EvaluationDataset):
        self.model = model
        self.eval_dataset = eval_dataset
        self.history = []

    def evaluate_and_log(self) -> dict:
        """Evaluate model and log results."""
        metrics = {}

        for sample in self.eval_dataset.samples:
            prediction = self.model.predict(sample["input"])
            expected = sample["expected"]

            # Track per-sample metrics
            metrics[sample["id"]] = {
                "prediction": prediction,
                "expected": expected,
                "correct": prediction == expected
            }

        # Aggregate metrics
        accuracy = sum(m["correct"] for m in metrics.values()) / len(metrics)

        result = {
            "timestamp": datetime.now().isoformat(),
            "accuracy": accuracy,
            "num_samples": len(metrics),
            "dataset_version": self.eval_dataset.version
        }

        self.history.append(result)
        return result

    def check_for_degradation(self, threshold: float = 0.05) -> bool:
        """Check if performance has degraded significantly."""
        if len(self.history) < 2:
            return False

        recent_accuracy = self.history[-1]["accuracy"]
        baseline_accuracy = self.history[0]["accuracy"]

        degradation = baseline_accuracy - recent_accuracy
        return degradation > threshold
```

## Evaluation Metrics Reference

### Classification Metrics

```python
from sklearn.metrics import (
    accuracy_score,
    precision_recall_fscore_support,
    confusion_matrix,
    classification_report,
    roc_auc_score
)

def comprehensive_classification_eval(y_true, y_pred, y_prob=None):
    """Generate comprehensive classification metrics."""

    metrics = {
        "accuracy": accuracy_score(y_true, y_pred),
        "confusion_matrix": confusion_matrix(y_true, y_pred).tolist(),
    }

    # Per-class metrics
    precision, recall, f1, support = precision_recall_fscore_support(y_true, y_pred)
    metrics["per_class"] = {
        "precision": precision.tolist(),
        "recall": recall.tolist(),
        "f1": f1.tolist(),
        "support": support.tolist()
    }

    # AUC if probabilities available
    if y_prob is not None:
        metrics["roc_auc"] = roc_auc_score(y_true, y_prob, multi_class='ovr')

    return metrics
```

### Regression Metrics

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

def comprehensive_regression_eval(y_true, y_pred):
    """Generate comprehensive regression metrics."""
    return {
        "mae": mean_absolute_error(y_true, y_pred),
        "rmse": mean_squared_error(y_true, y_pred, squared=False),
        "r2": r2_score(y_true, y_pred),
        "mape": mean_absolute_percentage_error(y_true, y_pred)
    }

def mean_absolute_percentage_error(y_true, y_pred):
    y_true, y_pred = np.array(y_true), np.array(y_pred)
    return np.mean(np.abs((y_true - y_pred) / y_true)) * 100
```

## Testing Checklist

Before deploying a model, ensure:

1. **Unit tests pass**: All components tested independently
2. **Integration tests pass**: Pipeline works end-to-end
3. **Acceptance criteria met**: Metrics exceed thresholds
4. **Business rules validated**: Domain constraints satisfied
5. **Comparison tests pass**: New model beats baseline
6. **Golden set maintained**: No regressions on known-good examples
7. **Edge cases covered**: Handles invalid/unusual inputs gracefully
8. **Performance acceptable**: Inference time and memory within limits
9. **Evaluation dataset documented**: Clear provenance and versioning
10. **Monitoring in place**: Can detect degradation post-deployment
