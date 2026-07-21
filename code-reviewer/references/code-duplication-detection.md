# Intelligent Code Duplication Detection

## Philosophy

**Don't just search for similar text - understand what the code does and find functionally similar implementations.**

## Detection Process

### Step 1: Understand the Code's Purpose

Before searching, analyze what the code actually does:

**Example:**
```python
def extract_material_from_text(text: str) -> str:
    pattern = r'\b[A-Z][a-z]+\s+\d+\b'
    match = re.search(pattern, text)
    return match.group(0) if match else None
```

**Purpose**: Extracts material identifiers from text using regex pattern matching.

**Key characteristics**:
- Input: text string
- Output: extracted identifier or None
- Method: regex pattern matching
- Domain: material identification

### Step 2: Identify Search Keywords

Based on purpose, derive semantic search terms:

**For the example above:**
- Function purpose: "extract", "material", "identifier", "pattern"
- Technical approach: "regex", "search", "pattern matching"
- Domain: "material", "classification"

**Not just**: "extract_material" (literal string matching)

### Step 3: Search with Context

Use the Grep tool to search for semantically related code:

**Search for similar purpose:**
- Pattern: `"def extract.*material"`
- Type: `py`
- Path: `packages/`

**Search for similar technical approach:**
- Pattern: `"re\.(search|findall|match).*material"`
- Type: `py`
- Path: `packages/`

**Search in likely locations:**
- Pattern: `"material.*identifier"`
- Type: `py`
- Path: `packages/data-utils/`

These searches use Claude Code's Grep tool, not bash commands.

### Step 4: Analyze Matches for Functional Similarity

For each match, ask:
1. **Does it solve the same problem?** Not just similar names
2. **Same input/output contract?** Similar function signature
3. **Same approach?** Similar algorithm or method
4. **Same domain?** Similar business logic

**True duplicate example:**
```python
# In packages/data-utils/text_processing.py
def extract_material_code(document: str) -> Optional[str]:
    """Extract material identifier from document text."""
    material_pattern = r'\b[A-Z][a-z]+\s+\d+\b'
    result = re.search(material_pattern, document)
    return result.group(0) if result else None
```
✅ **This IS a duplicate** - same purpose, same approach, same logic

**False positive example:**
```python
# In packages/classification-ai/predictor.py
def extract_features(text: str) -> np.ndarray:
    """Extract feature vector from text for classification."""
    return vectorizer.transform([text])
```
❌ **This is NOT a duplicate** - different purpose (feature extraction vs identifier extraction)

## Common Duplication Patterns to Check

### 1. Data Processing Utilities

**Check locations:**
- `packages/data-utils/` - Generic data processing
- Other packages - Package-specific processing

**What to compare:**
- Data transformation logic
- Validation functions
- Parsing utilities

**Example:**
If seeing JSON parsing with validation, check if `data-utils` already has a generic version.

### 2. Pydantic Models

**Check locations:**
- `packages/data-models/` - Shared models
- Other packages - Package-specific models

**What to compare:**
- Field names and types
- Validation logic
- Business entities (User, Document, etc.)

**Example:**
If defining a `Document` model, check if `data-models` already has one that can be extended.

### 3. LLM Integration Patterns

**Check locations:**
- `packages/llm-clients/` - LLM client wrappers
- Other packages - Direct LLM usage

**What to compare:**
- Prompt construction patterns
- Response parsing logic
- Error handling for LLM calls

**Example:**
If implementing retry logic for OpenAI calls, check if `llm-clients` already handles this.

### 4. API Utilities

**Check locations:**
- `packages/api-services-utils/` - Shared API utilities
- Other packages - API endpoints

**What to compare:**
- Error handling patterns
- Request/response schemas
- Authentication logic

### 5. Logging and Monitoring

**Check locations:**
- `packages/logger/` - Logging utilities
- Other packages - Custom logging

**What to compare:**
- Logger configuration
- Structured logging patterns
- Performance timing decorators

## Search Strategy by Code Type

### For Utility Functions

1. **Understand**: What transformation/operation does it perform?
2. **Search keywords**: Operation verbs + domain nouns
3. **Check**: `data-utils` first, then other packages
4. **Compare**: Input/output types and transformation logic

### For Data Models

1. **Understand**: What entity does it represent?
2. **Search keywords**: Entity name + key fields
3. **Check**: `data-models` first
4. **Compare**: Field names, types, and validation rules

### For API Endpoints

1. **Understand**: What business operation does it perform?
2. **Search keywords**: Operation + resource
3. **Check**: Other API packages
4. **Compare**: Request/response contracts and business logic

### For ML/AI Logic

1. **Understand**: What ML task does it perform?
2. **Search keywords**: Task type + domain
3. **Check**: Similar AI packages
4. **Compare**: Model architecture, data processing, inference logic

## Multi-Step Search Process

### Level 1: Exact Functional Match
Search for functions that do exactly the same thing.

**Example: Looking for material extraction logic**
- Use Grep tool with pattern: `"extract.*material"`
- Type: `py`
- Path: `packages/data-utils/`

### Level 2: Similar Approach
Search for similar technical approaches in the domain.

**Example: Looking for regex-based extraction**
- Use Grep tool with pattern: `"re\.search.*pattern"`
- Type: `py`
- Path: `packages/`
- Glob: `**/src/**/*extract*.py`

### Level 3: Shared Utilities
Check if smaller components (helpers) already exist.

**Example: Looking for validation utilities**
- Use Grep tool with pattern: `"def validate_"`
- Type: `py`
- Path: `packages/data-utils/`

## Red Flags for Duplication

### High Confidence - Likely Duplicate
- Same function name in `data-utils` or `data-models`
- Same business logic in another package
- Same regex pattern for same domain
- Identical validation logic

### Medium Confidence - Investigate Further
- Similar function name with different signature
- Similar approach but different domain
- Partial overlap in logic

### Low Confidence - Probably Not Duplicate
- Same operation verb but different domain
- Same domain but different operation
- Similar name but completely different logic

## Review Guidelines

When reviewing code for duplication:

1. **Understand first**: What does this code do? Why does it exist?
2. **Search intelligently**: Use semantic keywords, not just text matching
3. **Read the matches**: Don't just count grep results - read and understand them
4. **Compare functionality**: Does it solve the same problem?
5. **Suggest consolidation**: If duplicate found, explain how to use existing code

## Example Review Comment

**Instead of:**
```
This function looks similar to something in data-utils.
```

**Write:**
```
This material extraction logic appears to duplicate functionality in
`packages/data-utils/src/data_utils/text_extraction.py:42`.

Both functions:
- Extract material identifiers using regex
- Handle None cases identically
- Use the same pattern format

Suggestion: Import and use `extract_material_identifier()` from data-utils:

from data_utils.text_extraction import extract_material_identifier

result = extract_material_identifier(text)
```

## When NOT to Flag Duplication

- **Similar but different domain**: Same operation, different context
- **Intentional specialization**: Domain-specific version of generic utility
- **Performance optimization**: Optimized version for specific use case
- **Different abstraction level**: High-level vs low-level implementation

Always prioritize understanding over pattern matching.
