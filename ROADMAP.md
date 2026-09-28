# Phased Implementation Roadmap

This roadmap breaks down the development of the AI-Powered Code Review Platform into ten disciplined, incremental phases. Each phase delivers a functional, tested increment with clear acceptance criteria.

---

## 🗺️ Phase Overview

```mermaid
gantt
    title Implementation Roadmap
    dateFormat  YYYY-MM-DD
    section Foundation
    Phase 0: Project Setup & Docker Env         :p0, 2026-10-01, 3d
    Phase 1: Database, Models & Multi-Tenancy   :p1, after p0, 4d
    section Ingestion & Processing
    Phase 2: GitHub Webhooks & Queue Gateway    :p2, after p1, 4d
    Phase 3: Diff Parsing & AST Context Engine   :p3, after p2, 5d
    section Analysis Engines
    Phase 4: Static & Security Analyzers        :p4, after p3, 5d
    Phase 5: AI Review Engine & Validation      :p5, after p4, 5d
    section Aggregation & Delivery
    Phase 6: Deduplication & GitHub Publisher   :p6, after p5, 5d
    Phase 7: Observability & Audit Logging      :p7, after p6, 3d
    section Frontend & Hardening
    Phase 8: Web Dashboard (React + TS)         :p8, after p7, 6d
    Phase 9: Security Hardening & CI/CD         :p9, after p8, 4d
```

---

## Phase 0: Architecture, Tooling & Local Environment Setup
* **Goals**: Establish project skeleton, reproducible local Docker Compose environment, linting/formatting configs, and base dependencies.
* **Deliverables**:
  * `docker-compose.yml` with PostgreSQL 16 and Redis 7.
  * Backend structure (`pyproject.toml` or `requirements.txt`, Ruff config, mypy config).
  * Frontend directory setup with Vite, React, TypeScript, and Tailwind CSS.
  * Verified local connectivity to database and Redis.
* **Acceptance Criteria**: `docker-compose up` launches clean database and Redis; local test runner executes cleanly.

---

## Phase 1: Database Models, Migrations & Multi-Tenancy
* **Goals**: Implement SQLAlchemy 2.0 async ORM models, Alembic migrations, and multi-tenant repository abstractions.
* **Deliverables**:
  * ORM models: `Organization`, `User`, `OrganizationMember`, `GitHubInstallation`, `Repository`, `PullRequest`, `PullRequestFile`, `Review`, `AnalysisJob`, `ReviewFinding`, `WebhookEvent`, `AuditLog`.
  * Alembic migration environment configured for asynchronous connection.
  * Repository data access layer with tenant isolation filters.
  * Unit and integration tests for CRUD and tenant scoping.
* **Acceptance Criteria**: Migrations apply cleanly; multi-tenant isolation unit tests pass.

---

## Phase 2: GitHub App Integration & Webhook Ingestion Gateway
* **Goals**: Secure, high-throughput webhook receiver and GitHub App credential management.
* **Deliverables**:
  * `POST /api/v1/webhooks/github` endpoint with HMAC-SHA256 signature verification.
  * Redis-backed idempotency protection using `X-GitHub-Delivery`.
  * Raw event persistence in `webhook_events` table.
  * Asynchronous Celery dispatch for `pull_request` events.
  * GitHub App JWT token generator and installation token caching service.
* **Acceptance Criteria**: Webhook endpoint returns `202 Accepted` within 50ms; invalid signatures return `401`; duplicate webhooks return `200 Ignored`.

---

## Phase 3: Git Diff Parser & AST Context Extraction Engine
* **Goals**: Extract high-precision diff hunks and localized AST context without loading entire repositories.
* **Deliverables**:
  * Unified diff parser: calculates additions, deletions, modified line ranges, and GitHub diff position offsets.
  * AST Context Extractor: identifies enclosing functions, classes, decorators, and imports for modified lines.
  * Unit test suite covering Python functions, classes, async functions, and malformed files.
* **Acceptance Criteria**: Given a multi-hunk diff, extracts the exact enclosing AST signatures and maps line numbers to diff positions with 100% test accuracy.

---

## Phase 4: Deterministic Static & Security Analysis Engine
* **Goals**: Run fast, deterministic linters and security scanners locally in isolated worker processes.
* **Deliverables**:
  * Static analyzer runners: Ruff, Bandit, Semgrep.
  * Safe subprocess execution with process timeouts, unprivileged user limits, and error handling.
  * Output parsers converting tool JSON outputs into normalized `RawFinding` records.
  * Unit tests with mocked subprocess responses and realistic tool fixtures.
* **Acceptance Criteria**: Analyzers execute on sample diff files within 10s and output standardized finding structures.

---

## Phase 5: Context-Aware AI Review Engine
* **Goals**: Intelligent, high-signal review engine using provider-agnostic LLM interface with structured Pydantic outputs.
* **Deliverables**:
  * LLM client abstraction interface (`LLMProvider`) supporting OpenAI, Anthropic, and Gemini (plus `MockLLMProvider`).
  * Prompt assembly engine with strict XML code fencing and prompt injection hardening.
  * Pydantic v2 structured output validation (`AIReviewOutput`, `AIFinding`).
  * Out-of-bounds line validation and secret redaction pre-flight scanner.
* **Acceptance Criteria**: AI engine returns validated structured findings against sample bugs (SQLi, race condition, resource leak) without hallucinated line offsets.

---

## Phase 6: Finding Aggregator, Deduplicator & GitHub Review Publisher
* **Goals**: Merge findings across tools, eliminate duplicates, and publish rich reviews to GitHub.
* **Deliverables**:
  * Deduplication engine: merges static and AI findings on identical line spans, boosts confidence.
  * GitHub Review Publisher: posts consolidated review with line-level diff comments, Markdown suggestion blocks, and summary verdict.
  * GitHub Checks API integration (`CheckRun` status and annotations).
  * Review state machine updating PostgreSQL records.
* **Acceptance Criteria**: End-to-end simulation posts a multi-tool review with inline suggestions and passes all checks.

---

## Phase 7: Observability, Metrics & Audit Logging
* **Goals**: Production monitoring, Prometheus metrics, structured logs, and audit trails.
* **Deliverables**:
  * Prometheus metrics (`code_review_webhooks_total`, `code_review_duration_seconds`, `code_review_llm_tokens_total`, `code_review_findings_by_severity`).
  * JSON structured logging with request ID correlation.
  * Audit logging middleware tracking critical configuration changes.
  * `/health/live` and `/health/ready` endpoints.
* **Acceptance Criteria**: Prometheus endpoint scrapes valid metrics; health probes correctly reflect DB and Redis status.

---

## Phase 8: Web Dashboard & Finding Explorer
* **Goals**: Clean, modern web interface for engineers and team leads to view repositories, reviews, and findings.
* **Deliverables**:
  * React + TypeScript + Tailwind CSS application.
  * Dashboard views: Repository list, Pull Request review history, Detailed Review View with diff viewer and finding highlights.
  * Repository configuration modal: adjust enabled linters, severity floor, and AI review toggle.
  * API client integrating with `/api/v1` routes.
* **Acceptance Criteria**: Fully functional UI displaying real reviews and findings with responsive design.

---

## Phase 9: Hardening, CI/CD Pipeline & End-to-End Testing
* **Goals**: Final security audit, worker sandboxing, automated CI/CD pipeline, and complete documentation review.
* **Deliverables**:
  * Subprocess security hardening and resource limits.
  * GitHub Actions CI workflow (linting, type checking, pytest with coverage, docker build).
  * End-to-end integration test suite validating the entire flow from webhook to database.
* **Acceptance Criteria**: CI passes with >85% test coverage; security scans report zero high-severity issues.
