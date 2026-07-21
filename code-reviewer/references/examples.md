# Code Reviewer: Example Reviews

## Example 1: Security Issue

**Code:**
```python
def get_user_file(filename: str):
    path = f"/data/uploads/{filename}"
    return open(path).read()
```

**Review:**
```markdown
### 🔴 Critical Issues

**Location**: api.py:42
**Issue**: Path traversal vulnerability
**Why it matters**: User can access any file with `../../etc/passwd`

**Fix**:
from pathlib import Path

UPLOAD_DIR = Path("/data/uploads")

def get_user_file(filename: str):
    file_path = (UPLOAD_DIR / filename).resolve()
    if not file_path.is_relative_to(UPLOAD_DIR):
        raise ValueError("Invalid filename")
    return file_path.read_text()
```

---

## Example 2: Code Duplication

**Code being reviewed:**
```python
# In packages/mdaas-data-translation-ai/src/utils.py
def parse_confidence_score(text: str) -> float:
    match = re.search(r'confidence[:\s]+([0-9.]+)', text, re.IGNORECASE)
    if match:
        return float(match.group(1))
    return 0.0
```

**After intelligent search:**
```bash
# Found: packages/data-utils/src/data_utils/text_parsing.py:extract_confidence
```

**Review:**
```markdown
### 🟡 Important Issues

**Location**: utils.py:15
**Issue**: Duplicate functionality exists in data-utils
**Why it matters**: This logic is identical to `extract_confidence()` in data-utils

**Found match**: `packages/data-utils/src/data_utils/text_parsing.py:34`
Both functions parse confidence scores from text using regex, handle case-insensitive
matching, and return 0.0 as default.

**Fix**:
from data_utils.text_parsing import extract_confidence
score = extract_confidence(text)
```

---

## Example 3: Performance Issue

**Code:**
```python
for doc in documents:
    features = model.extract_features(doc)
    results.append(features)
```

**Review:**
```markdown
### 🟡 Important Issues

**Location**: pipeline.py:78
**Issue**: Processing documents one at a time (no batching)
**Why it matters**: 10-50x slower than batch processing

**Fix**:
BATCH_SIZE = 32
for i in range(0, len(documents), BATCH_SIZE):
    batch = documents[i:i + BATCH_SIZE]
    batch_features = model.extract_features(batch)
    results.extend(batch_features)

**Expected improvement**: ~20x faster for typical document batches
```
