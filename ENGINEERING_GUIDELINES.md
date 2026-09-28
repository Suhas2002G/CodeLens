# Engineering Guidelines & Coding Standards

## 1. Core Engineering Principles

1. **Simplicity Over Cleverness**: Code should be readable, predictable, and maintainable. Avoid obscure metaprogramming, complex inheritance hierarchies, and premature abstractions.
2. **Explicit is Better than Implicit**: Explicit dependency injection, explicit database transactions, explicit error handling, and complete type annotations.
3. **Defense in Depth**: Assume all external input (webhooks, user code, pull request metadata, LLM responses) is potentially malformed or untrusted.
4. **Idempotent by Default**: Webhook handlers, background worker tasks, and database mutations must be safe to execute multiple times.

---

## 2. Python Backend Standards (Python 3.11+)

### 2.1 Typing & Static Analysis
* **Strict Typing**: All function signatures must have complete type annotations:
  ```python
  # Good
  async def get_review_by_id(db: AsyncSession, review_id: UUID) -> Optional[Review]:
      ...

  # Bad
  async def get_review_by_id(db, review_id):
      ...
  ```
* **Linting & Formatting**: Enforced via **Ruff** (line length: 100, rules: E, F, I, B, S, UP).
* **Type Checker**: Checked with **Mypy** under `--strict` mode.

### 2.2 FastAPI Architecture Pattern
Strict layered separation of concerns:
```
Router (HTTP / Serialization / Auth)
  │
  ▼
Service (Business Logic / Orchestration)
  │
  ▼
Repository (Data Access / Queries)
  │
  ▼
Database (PostgreSQL via SQLAlchemy 2.0 AsyncSession)
```

* **Routers**:
  * Never contain direct database queries or raw business logic.
  * Use FastAPI `Depends()` for dependency injection (database sessions, authenticated user, services).
  * Validate inputs and responses using Pydantic v2 schemas.
* **Services**:
  * Implement business rules, coordinate external calls, and manage transaction boundaries.
  * Accept and return domain models or Pydantic schemas.
* **Repositories**:
  * Encapsulate SQLAlchemy queries.
  * Always accept `AsyncSession`.
  * Prevent N+1 queries by using `selectinload` or `joinedload` for relationships.

### 2.3 Celery Worker Standards
* **Pass IDs, Not Objects**: Never pass large objects, ORM instances, or raw file contents through Celery task arguments. Pass only primitive identifiers (`review_id: str`, `webhook_event_id: str`).
* **Handle Concurrency**: Use database row-level locking (`SELECT ... FOR UPDATE`) or atomic status updates to prevent race conditions during parallel processing.
* **Retries & Backoff**:
  ```python
  @celery_app.task(
      bind=True,
      max_retries=3,
      default_retry_delay=5,
      autoretry_for=(TransientNetworkError, HTTP429RateLimitError),
      retry_backoff=True,
      retry_jitter=True,
  )
  def run_analysis_task(self, review_id: str) -> None:
      ...
  ```

---

## 3. Error Handling & Exceptions

1. **Custom Domain Exceptions**: Define domain-specific exceptions in `app/core/exceptions.py`:
   ```python
   class PlatformError(Exception):
       """Base exception for all platform errors."""

   class EntityNotFoundError(PlatformError):
       def __init__(self, entity_name: str, entity_id: str):
           super().__init__(f"{entity_name} with ID '{entity_id}' not found.")
           self.entity_name = entity_name
           self.entity_id = entity_id

   class WebhookSignatureInvalidError(PlatformError):
       """Raised when GitHub HMAC signature validation fails."""
   ```
2. **Global Exception Handlers**: Map domain exceptions to standardized HTTP responses in `app/api/errors.py`.
3. **Never Swallow Exceptions**: Avoid bare `except: pass` or `except Exception: log.error()`. Always log with `exc_info=True` or re-raise.

---

## 4. Logging & Observability Guidelines

* **Structured JSON Logging**: Every log entry must include structured key-value context:
  ```python
  logger.info(
      "pr_review_started",
      review_id=str(review.id),
      pr_number=pr.pr_number,
      repo_name=repo.full_name,
      commit_sha=review.commit_sha,
  )
  ```
* **Log Redaction**:
  * NEVER log API keys, access tokens, webhook signatures, or raw authorization headers.
  * Avoid logging full source code in production logs (log line counts and file paths instead).
* **Correlation IDs**: Every request and Celery task carries a `request_id` or `correlation_id` passed through headers and Celery task headers for cross-service tracing.

---

## 5. Security & Secure Coding Rules

1. **No Code Execution**: Never use `eval()`, `exec()`, or `subprocess.run(..., shell=True)`.
2. **Subprocess Invocations**:
   * Always pass argument lists: `["ruff", "check", "--output-format=json", file_path]`.
   * Enforce a hard timeout (`asyncio.wait_for(proc.communicate(), timeout=60.0)`).
3. **Constant Time Comparison**: Use `hmac.compare_digest` for all cryptographic checks.
4. **Tenant Scoping**: Every database query on multi-tenant entities (`repositories`, `pull_requests`, `reviews`, `findings`) must include `WHERE organization_id = :org_id`.

---

## 6. Testing Standards

* **Naming Convention**: `test_<functionality>_<condition>_<expected_result>()`.
* **Arrange-Act-Assert**: Structure all tests clearly into Arrange, Act, and Assert blocks.
* **Deterministic Tests**: No `time.sleep()` in tests. Use `pytest-asyncio`, mocked clocks, or event synchronizers.
* **Isolated State**: Each test transaction must roll back or use an isolated test database to prevent state leakage.
