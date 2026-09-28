# REST API Specification (v1)

## 1. Global Conventions

* **Base URL**: `/api/v1`
* **Transport**: HTTPS Only
* **Authentication**: Bearer Token (JWT) in `Authorization: Bearer <token>` header (except public webhooks and health checks).
* **Content Type**: `application/json` (except webhooks which verify raw binary HMAC).
* **Timestamps**: ISO 8601 UTC (e.g. `2026-09-28T14:30:00Z`).
* **Pagination**: Standard limit/offset: `?limit=20&offset=0` with `total`, `limit`, `offset` in response.

### Standard Error Response Format
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Pull request with id '3fa85f64-5717-4562-b3fc-2c963f66afa6' does not exist.",
    "details": {},
    "request_id": "req_01H1234567890ABCDEF"
  }
}
```

---

## 2. API Endpoints

### 2.1 Health & Observability

#### `GET /health/live`
Liveness probe for container orchestrators.
* **Response `200 OK`**:
  ```json
  { "status": "ok", "timestamp": "2026-09-28T14:30:00Z" }
  ```

#### `GET /health/ready`
Readiness probe verifying DB and Redis connectivity.
* **Response `200 OK`**:
  ```json
  {
    "status": "ready",
    "dependencies": {
      "database": "connected",
      "redis": "connected",
      "celery_workers": 4
    }
  }
  ```

#### `GET /metrics`
Prometheus metrics export endpoint (secured or private network only).

---

### 2.2 Webhooks

#### `POST /api/v1/webhooks/github`
Public webhook ingestion receiver for GitHub App events.
* **Headers Required**:
  * `X-GitHub-Event`: `pull_request` (or `installation`, `ping`)
  * `X-GitHub-Delivery`: Unique UUID delivery ID
  * `X-Hub-Signature-256`: `sha256=...` (HMAC signature)
* **Response `202 Accepted`**:
  ```json
  {
    "status": "accepted",
    "delivery_id": "720c0610-e555-11eb-8c65-8b3874c76b92",
    "job_id": "celery-job-uuid-1234"
  }
  ```
* **Response `200 OK`**: (If duplicate delivery ID)
  ```json
  { "status": "ignored", "reason": "duplicate_delivery" }
  ```
* **Errors**: `401 Unauthorized` (Invalid signature).

---

### 2.3 Authentication & User Management

#### `POST /api/v1/auth/github/callback`
Exchanges GitHub OAuth temporary code for platform JWT token.
* **Request**:
  ```json
  { "code": "github_temp_oauth_code" }
  ```
* **Response `200 OK`**:
  ```json
  {
    "access_token": "jwt_token_string",
    "token_type": "bearer",
    "expires_in": 86400,
    "user": {
      "id": "c1f7b76a-5c12-4c22-bcf5-998811223344",
      "email": "developer@company.com",
      "full_name": "Jane Developer",
      "github_user_id": "1234567"
    }
  }
  ```

#### `GET /api/v1/auth/me`
Fetches current authenticated user profile and organization memberships.

---

### 2.4 Organizations & Repositories

#### `GET /api/v1/organizations`
List organizations the current user belongs to.

#### `GET /api/v1/organizations/{id}/repositories`
List repositories tracked under the organization.
* **Response `200 OK`**:
  ```json
  {
    "items": [
      {
        "id": "b1b2b3b4-1111-2222-3333-444455556666",
        "full_name": "acme-corp/payment-service",
        "default_branch": "main",
        "is_active": true,
        "config_overrides": {
          "enabled_linters": ["ruff", "bandit", "semgrep"],
          "severity_threshold": "MEDIUM",
          "ai_review_enabled": true
        },
        "created_at": "2026-09-01T10:00:00Z"
      }
    ],
    "total": 1
  }
  ```

#### `PATCH /api/v1/repositories/{id}`
Update repository configuration (e.g. toggle analyzers, severity threshold).

---

### 2.5 Pull Requests & Reviews

#### `GET /api/v1/repositories/{id}/pull-requests`
Query pull requests for a repository with pagination and status filters.
* **Query Parameters**: `?state=open&limit=20&offset=0`

#### `GET /api/v1/pull-requests/{id}`
Retrieve PR details, files, and review history.

#### `POST /api/v1/pull-requests/{id}/trigger-review`
Manually trigger or rerun an analysis review for the latest commit SHA.
* **Response `202 Accepted`**:
  ```json
  {
    "review_id": "e9b21111-aaaa-bbbb-cccc-112233445566",
    "status": "pending",
    "commit_sha": "a1b2c3d4e5f60718293a4b5c6d7e8f9012345678"
  }
  ```

#### `GET /api/v1/reviews/{id}`
Retrieve a review run, status, verdict, duration, and summary.
* **Response `200 OK`**:
  ```json
  {
    "id": "e9b21111-aaaa-bbbb-cccc-112233445566",
    "pull_request_id": "pr_uuid_123",
    "commit_sha": "a1b2c3d4e5f60718293a4b5c6d7e8f9012345678",
    "status": "completed",
    "verdict": "changes_requested",
    "summary_markdown": "### Code Review Summary\nFound 1 High Severity Security Issue and 1 Logic Flaw.",
    "total_findings": 2,
    "total_tokens_used": 1420,
    "total_duration_seconds": 18.4,
    "started_at": "2026-09-28T14:31:00Z",
    "completed_at": "2026-09-28T14:31:18Z"
  }
  ```

#### `GET /api/v1/reviews/{id}/findings`
Retrieve all normalized findings for a review.
* **Response `200 OK`**:
  ```json
  {
    "items": [
      {
        "id": "f1f2f3f4-5555-6666-7777-888899990000",
        "file_path": "services/payment.py",
        "line_start": 42,
        "line_end": 44,
        "is_in_diff": true,
        "diff_position": 14,
        "category": "SECURITY",
        "severity": "HIGH",
        "confidence": 0.98,
        "source": "HYBRID",
        "rule_id": "B608-SQL-INJECTION",
        "title": "Possible SQL Injection via String Concatenation",
        "description": "Raw SQL query constructed using unsanitized user parameter `order_id`.",
        "code_evidence": "cursor.execute(f'SELECT * FROM orders WHERE id = {order_id}')",
        "suggested_fix": "cursor.execute('SELECT * FROM orders WHERE id = :id', {'id': order_id})",
        "status": "open"
      }
    ],
    "total": 1
  }
  ```

#### `PATCH /api/v1/findings/{id}`
Update finding status from the dashboard (e.g. mark as `false_positive`, `dismissed`, `resolved`).
* **Request**:
  ```json
  {
    "status": "false_positive",
    "feedback_note": "This is an internal admin script with sanitized input."
  }
  ```
