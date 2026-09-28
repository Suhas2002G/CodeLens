# ADR 0001: Modular Monolith Architecture Pattern

* **Status**: Accepted
* **Date**: 2026-09-28
* **Deciders**: Software Architect & Platform Lead

---

## Context
When architecting an enterprise-grade automated code review platform, an immediate architectural decision is whether to build microservices (e.g. separate Webhook Service, Diff Parser Service, Static Analysis Service, AI Review Service, Web Dashboard Backend) or a unified backend architecture.

Microservices introduce significant operational complexity:
* Distributed transactions and eventual consistency across review steps.
* Network latency overhead between parsing, linting, and LLM orchestration.
* Complex local development environments requiring multiple containerized microservices.
* Observability and distributed tracing overhead early in development.

## Decision
We will build the backend as a **Modular Monolith** using Python (FastAPI) and an asynchronous distributed worker cluster (Celery + Redis).

Key characteristics of this architecture:
1. **Single Deployable Unit**: The core API server and Celery workers run from the same codebase, ensuring synchronized schema versions and shared domain models.
2. **Strict Internal Boundaries**: Modules (`auth`, `github`, `pull_requests`, `analyzers`, `ai_engine`, `reviews`) maintain clear domain boundaries, interacting via explicit public interfaces.
3. **Dedicated Workqueues**: Different tasks (webhook processing, static analysis, AI calls, GitHub comment publishing) are routed to dedicated Celery queues with tailored concurrency and timeouts.
4. **Extraction-Ready**: Should a particular module require distinct scaling characteristics (e.g., static analysis runners needing sandboxed Kubernetes pods), its clean interface will allow extraction into a standalone microservice with minimal refactoring.

## Consequences

### Positive
* Fast developer iteration: Single test suite, unified database migrations, simple local setup with Docker Compose.
* Zero inter-service network latency for orchestrating internal domain logic.
* Atomic database transactions when updating review status and persisting normalized findings.

### Negative / Trade-offs
* If one module crashes the Python interpreter (e.g., a native C-binding issue in an analyzer), the worker process terminates (mitigated by isolated subprocess execution for external linters).
* Requires strict engineering discipline to prevent circular dependencies between internal domain modules.
