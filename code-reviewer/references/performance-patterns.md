# Performance Review Patterns

## Data Processing Performance

### Vectorization vs Loops
- **Check**: Processing large arrays/DataFrames with Python loops
- **Bad**: `[expensive_op(x) for x in large_array]`
- **Good**: Vectorize with NumPy/Pandas operations
- **Example**: `np.apply_along_axis()` or native vectorized operations

### Batch Processing
- **Check**: Processing items one-by-one when batching is possible
- **Bad**: Multiple single-item API calls or DB queries
- **Good**: Batch operations (batch size 32-128 for ML, 100-1000 for DB)
- **Impact**: 10-50x speedup

### Memory-Mapped Files
- **Check**: Loading entire large files into memory
- **Bad**: `data = json.load(open("big_file.json"))`
- **Good**: Use `np.memmap()` for numeric data or streaming parsers

## Database Performance

### N+1 Query Problem
- **Check**: Loop making one query per iteration
- **Bad**:
  ```python
  for user in users:
      orders = db.query(Order).filter(Order.user_id == user.id).all()
  ```
- **Good**: Use joins or `IN` queries to fetch all at once

### Missing Indexes
- **Check**: Queries on columns without indexes
- **Action**: Suggest adding indexes on frequently queried columns

### Unbounded Queries
- **Check**: Queries without `LIMIT` or pagination
- **Bad**: `SELECT * FROM large_table`
- **Good**: Add pagination and limits

## API Performance

### Async Opportunities
- **Check**: Multiple independent `await` calls executed sequentially instead of concurrently
- **Good**: `results = await asyncio.gather(fetch_data_1(), fetch_data_2())`
- For the fuller pattern set (TaskGroup, cancellation, fire-and-forget tasks, event loop blocking) invoke `python-async-scaling` — this file only covers the review checklist item, not the underlying async guidance.

### Caching Missing
- **Check**: Expensive computations or API calls repeated with same inputs
- **Good**: Add `@lru_cache`, Redis caching, or response caching
- **Example**: Model predictions, external API calls, computed aggregations

### Response Size
- **Check**: Returning entire objects when only fields needed
- **Good**: Use response models to filter fields
- **Example**: FastAPI `response_model` with specific fields

## ML/AI Performance

### Model Loading
- **Check**: Loading model on every request
- **Bad**: `model = load_model()` inside request handler
- **Good**: Load once at startup, use singleton pattern

### Inference Batching
- **Check**: Running inference on single items when batch available
- **Good**: Accumulate requests and batch inference (with timeout)

### GPU Utilization
- **Check**: Small batches on GPU (underutilization)
- **Good**: Increase batch size to saturate GPU

### Tokenization
- **Check**: Tokenizing same text multiple times
- **Good**: Cache tokenized results or tokenize once

## LLM-Specific Performance

### Token Optimization
- **Check**: Verbose prompts or unnecessary examples
- **Good**: Compress prompts, use fewer examples, remove filler words

### Streaming Responses
- **Check**: Waiting for full LLM response before returning
- **Good**: Stream responses for better UX and lower TTFB

### Caching Strategies
- **Check**: Repeated identical or similar prompts
- **Good**: Implement semantic caching (embed + similarity search)

### Parallel API Calls
- **Check**: Sequential LLM calls when they could be parallel
- **Good**: Use `asyncio.gather()` for independent calls (see `python-async-scaling` for cancellation/error-handling nuances)

## Memory Performance

### Large Data Structures
- **Check**: Loading large datasets entirely in memory
- **Good**: Use generators, chunking, or streaming
- **Example**: `pd.read_csv(chunksize=10000)`

### Memory Leaks
- **Check**: Circular references, unclosed files/connections
- **Good**: Use context managers, explicitly close resources

### Copy vs Reference
- **Check**: Deep copying large objects unnecessarily
- **Bad**: `new_df = df.copy()` when reference would work
- **Good**: Use views or references when possible

## Asyncio Event Loop Blocking

Any `async def` handler with a sync call that takes more than ~100ms (ML inference, `open()`/`Path.read_bytes()`, `subprocess.run()`, a sync DB driver) freezes the entire event loop for every other request — flag as 🔴 Critical, especially on Azure Container Apps/Kubernetes where a stalled liveness probe restarts the container mid-request. This is `python-async-scaling`'s core principle (see its `asyncio-fundamentals.md` and `checklists.md`) — invoke that skill for the full pattern set and code examples rather than re-deriving them here.

## Python-Specific Performance

### String Concatenation
- **Check**: Repeated string concatenation in loop
- **Bad**: `s = ""; for x in items: s += str(x)`
- **Good**: `"".join(str(x) for x in items)`

### List Comprehensions
- **Check**: Building list with `append()` in loop
- **Good**: Use list comprehension (faster and more readable)

### Premature Optimization
- **Check**: Complex optimizations for non-bottleneck code
- **Principle**: Profile first, optimize hot paths only

## Review Guidelines

Flag performance issues when:
1. **Obvious impact**: N+1 queries, no batching on large datasets
2. **Known bottleneck**: Based on application characteristics (ML inference, API calls)
3. **Easy fix**: Simple change with significant impact

Skip micro-optimizations unless in a tight loop or proven bottleneck.

Always suggest:
- The specific pattern to change
- Expected performance improvement magnitude
- Code example of the fix
