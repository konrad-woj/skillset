# Prompt Engineering Patterns

Best practices for crafting effective LLM prompts with structure, templates, and evaluation.

## Structured Prompts

### Basic Prompt Structure

```python
# Before: Unstructured string
prompt = f"Extract the name from: {text}"

# After: Structured with clear sections
prompt = f"""
Task: Extract the person's name from the given text.

Text:
{text}

Instructions:
- Return only the name, no other text
- If no name found, return "NONE"
- Use proper capitalization

Output:
"""
```

### System/User Message Pattern

```python
# Before: Everything in one message
prompt = "You are a helpful assistant. Extract names from this text: " + text

# After: Separate system and user messages
messages = [
    {
        "role": "system",
        "content": "You are a precise name extraction assistant. Extract only person names from text. Return 'NONE' if no name found."
    },
    {
        "role": "user",
        "content": f"Extract the name from this text:\n\n{text}"
    }
]

response = client.chat.completions.create(
    model="gpt-4",
    messages=messages
)
```

## Jinja Templates

### Basic Template Usage

```python
# Before: String formatting
def create_prompt(task, context, examples):
    prompt = f"""
Task: {task}

Context: {context}

Examples:
"""
    for ex in examples:
        prompt += f"Input: {ex['input']}\nOutput: {ex['output']}\n\n"
    return prompt

# After: Jinja template
from jinja2 import Template

PROMPT_TEMPLATE = """
Task: {{ task }}

Context: {{ context }}

Examples:
{% for example in examples %}
Input: {{ example.input }}
Output: {{ example.output }}
{% endfor %}

Now, process this input:
{{ user_input }}
"""

template = Template(PROMPT_TEMPLATE)
prompt = template.render(
    task="Extract entities",
    context="Medical records",
    examples=examples,
    user_input=user_input
)
```

### Template with Conditionals

```python
PROMPT_TEMPLATE = """
Task: {{ task }}

{% if context %}
Context: {{ context }}
{% endif %}

{% if few_shot_examples %}
Examples:
{% for example in few_shot_examples %}
Input: {{ example.input }}
Output: {{ example.output }}
{% endfor %}
{% endif %}

{% if constraints %}
Constraints:
{% for constraint in constraints %}
- {{ constraint }}
{% endfor %}
{% endif %}

Process this input:
{{ user_input }}

{% if output_format %}
Output format: {{ output_format }}
{% endif %}
"""
```

### Template Files

```python
# Store templates in files
from jinja2 import Environment, FileSystemLoader

# Directory structure:
# prompts/
#   ├── extraction.jinja
#   ├── classification.jinja
#   └── generation.jinja

env = Environment(loader=FileSystemLoader('prompts'))

def create_extraction_prompt(text: str, entity_types: list[str]) -> str:
    template = env.get_template('extraction.jinja')
    return template.render(text=text, entity_types=entity_types)

# extraction.jinja:
"""
Extract the following entity types from the text:
{% for entity_type in entity_types %}
- {{ entity_type }}
{% endfor %}

Text:
{{ text }}

Output as JSON:
"""
```

## Few-Shot Prompting

### Dynamic Example Selection

```python
# Before: Fixed examples
def create_prompt(user_input):
    return f"""
Examples:
Input: "Book a flight"
Output: {{"intent": "booking"}}

Input: "{user_input}"
Output:
"""

# After: Dynamic example selection based on similarity
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

class FewShotSelector:
    def __init__(self, example_pool: list[dict], embedding_fn):
        self.examples = example_pool
        self.embedding_fn = embedding_fn
        self.example_embeddings = [
            embedding_fn(ex['input']) for ex in example_pool
        ]

    def select_examples(self, query: str, k: int = 3) -> list[dict]:
        query_embedding = self.embedding_fn(query)

        similarities = cosine_similarity(
            [query_embedding],
            self.example_embeddings
        )[0]

        top_k_indices = np.argsort(similarities)[-k:][::-1]
        return [self.examples[i] for i in top_k_indices]

# Usage
selector = FewShotSelector(example_pool, embedding_function)
relevant_examples = selector.select_examples(user_input, k=3)

template = Template(PROMPT_TEMPLATE)
prompt = template.render(
    examples=relevant_examples,
    user_input=user_input
)
```

### Example Quality Control

```python
# Filter examples by quality criteria
class ExampleFilter:
    @staticmethod
    def is_high_quality(example: dict) -> bool:
        # Check various quality metrics
        checks = [
            len(example['input']) > 10,  # Not too short
            len(example['input']) < 500,  # Not too long
            example.get('verified', False),  # Manually verified
            example.get('confidence', 0) > 0.9  # High confidence
        ]
        return all(checks)

    def filter_examples(self, examples: list[dict]) -> list[dict]:
        return [ex for ex in examples if self.is_high_quality(ex)]

# Usage
filter = ExampleFilter()
high_quality_examples = filter.filter_examples(example_pool)
selected = selector.select_examples(user_input, k=3, pool=high_quality_examples)
```

## Prompt Optimization

### Chain of Thought

```python
# Before: Direct question
prompt = f"Question: {question}\nAnswer:"

# After: Chain of thought
prompt = f"""
Question: {question}

Let's approach this step by step:
1. First, identify the key information
2. Then, reason through the logic
3. Finally, provide the answer

Step 1:
"""
```

### Zero-Shot Chain of Thought

```python
# Adding "Let's think step by step" improves reasoning
prompt = f"""
Question: {question}

Let's think step by step:
"""
```

### Self-Consistency

```python
# Generate multiple reasoning paths and vote
def self_consistent_answer(question: str, n_samples: int = 5) -> str:
    prompt = f"""
Question: {question}

Let's think step by step:
"""

    responses = []
    for _ in range(n_samples):
        response = llm.generate(prompt, temperature=0.7)
        # Extract final answer from reasoning
        answer = extract_answer(response)
        responses.append(answer)

    # Vote on most common answer
    from collections import Counter
    return Counter(responses).most_common(1)[0][0]
```

## Output Formatting

### Structured Output

```python
# Before: Free text output
prompt = f"Extract information from: {text}"

# After: Request JSON format
prompt = f"""
Extract the following information from the text and return as JSON:

Text: {text}

Return format:
{{
    "name": "string",
    "date": "YYYY-MM-DD",
    "amount": float,
    "category": "string"
}}

JSON:
"""
```

### Constrained Generation

```python
# Force specific format
prompt = f"""
Classify the sentiment as exactly one of: positive, negative, neutral

Text: {text}

Sentiment (one word only):
"""

# Or with enum
from enum import Enum

class Sentiment(str, Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

prompt = f"""
Classify sentiment as one of: {', '.join(s.value for s in Sentiment)}

Text: {text}

Sentiment:
"""
```

### Response Parsing

```python
# Robust response parsing
import json
import re

def parse_llm_json_response(response: str) -> dict:
    """Parse JSON from LLM response, handling common issues."""

    # Try direct JSON parsing
    try:
        return json.loads(response)
    except json.JSONDecodeError:
        pass

    # Extract JSON from markdown code block
    json_match = re.search(r'```json\s*(\{.*?\})\s*```', response, re.DOTALL)
    if json_match:
        try:
            return json.loads(json_match.group(1))
        except json.JSONDecodeError:
            pass

    # Extract first JSON object
    json_match = re.search(r'\{.*?\}', response, re.DOTALL)
    if json_match:
        try:
            return json.loads(json_match.group(0))
        except json.JSONDecodeError:
            pass

    raise ValueError(f"Could not parse JSON from response: {response}")
```

## Prompt Evaluation

### Evaluation Dataset

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class PromptExample:
    input: str
    expected_output: str
    metadata: dict = None

@dataclass
class PromptEvaluation:
    prompt_template: str
    examples: list[PromptExample]
    metric_fn: Callable

def evaluate_prompt(
    prompt_template: str,
    examples: list[PromptExample],
    llm_fn: Callable,
    metric_fn: Callable
) -> dict:
    """Evaluate prompt template on test examples."""

    results = []
    for example in examples:
        # Generate prompt from template
        prompt = Template(prompt_template).render(input=example.input)

        # Get LLM response
        response = llm_fn(prompt)

        # Evaluate
        score = metric_fn(example.expected_output, response)

        results.append({
            "input": example.input,
            "expected": example.expected_output,
            "actual": response,
            "score": score
        })

    # Aggregate metrics
    avg_score = sum(r["score"] for r in results) / len(results)

    return {
        "avg_score": avg_score,
        "num_examples": len(results),
        "results": results
    }
```

### A/B Testing Prompts

```python
# Compare two prompt variants
def ab_test_prompts(
    prompt_a: str,
    prompt_b: str,
    test_cases: list[dict],
    llm_fn: Callable,
    metric_fn: Callable
) -> dict:
    """Compare two prompts on the same test cases."""

    eval_a = evaluate_prompt(prompt_a, test_cases, llm_fn, metric_fn)
    eval_b = evaluate_prompt(prompt_b, test_cases, llm_fn, metric_fn)

    return {
        "prompt_a": {
            "avg_score": eval_a["avg_score"],
            "results": eval_a["results"]
        },
        "prompt_b": {
            "avg_score": eval_b["avg_score"],
            "results": eval_b["results"]
        },
        "winner": "a" if eval_a["avg_score"] > eval_b["avg_score"] else "b",
        "improvement": abs(eval_a["avg_score"] - eval_b["avg_score"])
    }
```

### Metrics for Evaluation

```python
# Exact match
def exact_match(expected: str, actual: str) -> float:
    return 1.0 if expected.strip().lower() == actual.strip().lower() else 0.0

# Contains match
def contains_match(expected: str, actual: str) -> float:
    return 1.0 if expected.lower() in actual.lower() else 0.0

# Semantic similarity (using embeddings)
from sklearn.metrics.pairwise import cosine_similarity

def semantic_similarity(expected: str, actual: str, embedding_fn: Callable) -> float:
    exp_emb = embedding_fn(expected)
    act_emb = embedding_fn(actual)
    return cosine_similarity([exp_emb], [act_emb])[0][0]

# Custom LLM-based evaluation
def llm_judge(expected: str, actual: str, llm_fn: Callable) -> float:
    """Use LLM to judge response quality."""
    judge_prompt = f"""
Compare the expected and actual outputs. Rate the actual output on a scale of 0-1.

Expected: {expected}
Actual: {actual}

Consider:
- Factual accuracy
- Completeness
- Format compliance

Rating (0.0 to 1.0):
"""
    rating_str = llm_fn(judge_prompt)
    try:
        return float(rating_str.strip())
    except ValueError:
        return 0.0
```

### Tracking Prompt Performance

```python
# Log prompt performance over time
from datetime import datetime
import json
from pathlib import Path

class PromptMetricsTracker:
    def __init__(self, log_path: Path):
        self.log_path = log_path

    def log_evaluation(
        self,
        prompt_version: str,
        metrics: dict,
        test_set_version: str
    ):
        entry = {
            "timestamp": datetime.now().isoformat(),
            "prompt_version": prompt_version,
            "test_set_version": test_set_version,
            "metrics": metrics
        }

        with self.log_path.open("a") as f:
            f.write(json.dumps(entry) + "\n")

    def get_history(self, prompt_version: str = None) -> list[dict]:
        history = []
        with self.log_path.open("r") as f:
            for line in f:
                entry = json.loads(line)
                if prompt_version is None or entry["prompt_version"] == prompt_version:
                    history.append(entry)
        return history

# Usage
tracker = PromptMetricsTracker(Path("prompt_metrics.jsonl"))

# After evaluation
tracker.log_evaluation(
    prompt_version="v2.1",
    metrics={"accuracy": 0.87, "avg_tokens": 150},
    test_set_version="golden_v1"
)

# View history
history = tracker.get_history(prompt_version="v2.1")
```

## Cost Optimization

### Token Counting

```python
# Count tokens before calling API
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("gpt2")  # Approximate

def estimate_cost(prompt: str, max_tokens: int, model: str = "gpt-4") -> float:
    """Estimate API call cost."""

    # Token counts
    prompt_tokens = len(tokenizer.encode(prompt))
    total_tokens = prompt_tokens + max_tokens

    # Pricing (example, check current pricing)
    pricing = {
        "gpt-4": {"input": 0.03, "output": 0.06},  # per 1K tokens
        "gpt-3.5-turbo": {"input": 0.001, "output": 0.002}
    }

    model_pricing = pricing.get(model, pricing["gpt-4"])

    cost = (
        (prompt_tokens / 1000) * model_pricing["input"] +
        (max_tokens / 1000) * model_pricing["output"]
    )

    return cost
```

### Prompt Compression

```python
# Remove unnecessary whitespace and formatting
def compress_prompt(prompt: str) -> str:
    """Compress prompt to reduce token count."""

    # Remove extra whitespace
    lines = [line.strip() for line in prompt.split('\n')]
    lines = [line for line in lines if line]  # Remove empty lines

    # Join with single newline
    compressed = '\n'.join(lines)

    return compressed

# Before
verbose_prompt = """
Task: Extract names

Instructions:
    - Find person names
    - Return as list
    - Use proper capitalization

Text:
    {text}

Output:
"""

# After
compressed_prompt = compress_prompt(verbose_prompt)
```

### Caching Strategies

```python
# Cache prompt responses
from functools import lru_cache
import hashlib

class PromptCache:
    def __init__(self):
        self.cache = {}

    def _hash_prompt(self, prompt: str) -> str:
        return hashlib.md5(prompt.encode()).hexdigest()

    def get_or_generate(
        self,
        prompt: str,
        llm_fn: Callable,
        use_cache: bool = True
    ) -> str:
        if not use_cache:
            return llm_fn(prompt)

        cache_key = self._hash_prompt(prompt)

        if cache_key in self.cache:
            return self.cache[cache_key]

        response = llm_fn(prompt)
        self.cache[cache_key] = response

        return response

cache = PromptCache()
response = cache.get_or_generate(prompt, llm.generate)
```

## Best Practices Summary

1. **Structure prompts clearly** - Use sections, headers, and formatting
2. **Use templates** - Jinja2 for reusability and maintainability
3. **Few-shot when needed** - Select relevant examples dynamically
4. **Request structured output** - JSON, enums, specific formats
5. **Evaluate systematically** - Create test sets, track metrics over time
6. **Optimize costs** - Count tokens, cache responses, compress prompts
7. **Version prompts** - Track changes and performance over versions
8. **Test edge cases** - Empty input, very long input, malformed data
9. **Handle parsing errors** - Robust extraction of structured data
10. **Monitor in production** - Track performance, costs, and failures
