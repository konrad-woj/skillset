# Security Review Checklist

## Input Validation & Injection Prevention

### SQL Injection
- **Check**: Raw SQL queries with string formatting or concatenation
- **Bad**: `f"SELECT * FROM users WHERE id = {user_id}"`
- **Good**: Use parameterized queries with SQLAlchemy or psycopg2 parameters

### Command Injection
- **Check**: `os.system()`, `subprocess` with `shell=True` and user input
- **Bad**: `os.system(f"convert {filename} output.pdf")`
- **Good**: Use list arguments: `subprocess.run(["convert", filename, "output.pdf"])`

### Path Traversal
- **Check**: File operations with user-provided paths
- **Bad**: `open(f"/data/{user_file}")`
- **Good**: Validate with `Path().resolve()` and check it's within allowed directory

### XSS (API Responses)
- **Check**: Unescaped user data in HTML responses
- **Good**: FastAPI/Pydantic naturally escapes, but verify custom HTML generation

## Authentication & Authorization

### API Security
- **Check**: Endpoints missing authentication decorators
- **Check**: Authorization logic - does user have permission for this resource?
- **Check**: JWT token validation and expiration

### Secrets Management
- **Check**: Hardcoded secrets, API keys, passwords
- **Bad**: `API_KEY = "sk-proj-..."`
- **Good**: Load from environment variables or AWS Secrets Manager

## Data Protection

### Sensitive Data Logging
- **Check**: Logging PII, passwords, tokens, API keys
- **Bad**: `logger.info(f"User {email} logged in with token {token}")`
- **Good**: Redact sensitive fields: `logger.info(f"User {hash(email)} logged in")`

### Data Encryption
- **Check**: Sensitive data stored in plaintext (passwords, tokens, PII)
- **Good**: Use bcrypt/argon2 for passwords, encrypt PII at rest

## Dependencies & Supply Chain

### Vulnerable Dependencies
- **Check**: Known CVEs in dependencies (usually caught by CI)
- **Action**: Review any new dependencies for necessity and security

### Dependency Confusion
- **Check**: Using `--extra-index-url` without verification
- **Good**: Pin all package versions, use private PyPI with authentication

## LLM-Specific Security

### Prompt Injection
- **Check**: User input directly concatenated into prompts without sanitization
- **Bad**: `f"Extract name from: {user_text}"`
- **Good**: Use structured prompts with clear boundaries and validation

### Model Output Handling
- **Check**: Executing code or commands from LLM responses without validation
- **Check**: Trusting structured output without schema validation

### Data Leakage
- **Check**: Sending proprietary/confidential data to external LLM APIs
- **Good**: Ensure data classification allows external API usage

## API Design Security

### Rate Limiting & DoS
- **Check**: Missing rate limiting on expensive operations
- **Good**: Use FastAPI rate limiting middleware or Redis-based limiting

### Request Size Limits
- **Check**: Unbounded file uploads or request sizes
- **Good**: Set `client_max_body_size` and validate sizes in code

### Error Information Disclosure
- **Check**: Detailed error messages exposing internal structure
- **Bad**: Returning full stack traces or database errors to users
- **Good**: Log detailed errors, return generic messages to users

## Cloud & Infrastructure

### IAM & Permissions (AWS / GCP / Azure)
- **Check**: Overly permissive roles or policies (`s3:*`, `*:*`, `roles/owner`, `Contributor` at subscription scope)
- **Good**: Principle of least privilege — scope to specific resources and actions
- **AWS**: IAM policies, avoid `AdministratorAccess` on service accounts
- **GCP**: IAM bindings, prefer custom roles over `roles/editor`
- **Azure**: RBAC assignments, prefer built-in scoped roles over `Owner`

### Object Storage Security (S3 / GCS / Blob Storage)
- **Check**: Publicly accessible buckets or containers
- **Check**: Pre-signed / SAS URLs without expiration or validation
- **Good**: Private by default, short-lived signed URLs, CDN for delivery
- **Check**: Missing server-side encryption or non-default KMS key

### Secrets Management
- **Check**: Secrets hardcoded or in plain environment variables in container/function definitions
- **Good**:
  - AWS: Secrets Manager or SSM Parameter Store with IAM-based access
  - GCP: Secret Manager with Workload Identity
  - Azure: Key Vault with managed identity
- **Check**: Secrets in Docker `ENV` instructions or Kubernetes `env:` without a Secret reference

### Network & Egress
- **Check**: Overly permissive security groups / VPC firewall rules (0.0.0.0/0 ingress on sensitive ports)
- **Check**: Services with public IPs that should be internal-only
- **Good**: Private endpoints / VPC peering for cloud service access; deny-by-default egress

## Common Python Pitfalls

### Pickle Security
- **Check**: Using `pickle.load()` on untrusted data
- **Risk**: Arbitrary code execution
- **Good**: Use JSON or msgpack for serialization

### YAML Loading
- **Check**: Using `yaml.load()` instead of `yaml.safe_load()`
- **Risk**: Arbitrary code execution
- **Good**: Always use `yaml.safe_load()`

### Regex DoS (ReDoS)
- **Check**: Complex regex on user input (nested quantifiers)
- **Bad**: `re.match(r'(a+)+b', user_input)`
- **Good**: Simplify regex, set timeout, or validate length first

## Review Process

For each code change:
1. Scan for patterns in this checklist
2. Focus on user input handling, authentication, and data flow
3. Flag only medium/high severity issues
4. Provide specific fix with before/after code example
