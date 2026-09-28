# ADR 0004: Multi-Provider LLM Abstraction and Delivery Strategy

* **Status**: Accepted
* **Date**: 2026-09-28
* **Deciders**: Software Architect & Platform Lead

---

## Context
1. **LLM Provider Flexibility**: Code review requires high reasoning accuracy. Different deployments may prefer OpenAI (e.g. `gpt-4o-mini` / `gpt-4o`) or Google Gemini (e.g. `gemini-1.5-pro` / `gemini-2.0-flash`), or may require mock execution during automated CI tests.
2. **Review Feedback Delivery**: GitHub supports multiple review mechanisms (individual issue comments, unified PR reviews with inline suggestions, and GitHub Checks API annotations).
3. **Local Development Containerization**: Need a uniform, reproducible environment for developers without conflicting host dependencies.

## Decision
1. **Dynamic LLM Provider Selection via Environment**:
   * Configuration variable: `LLM_PROVIDER=openai|gemini|mock`.
   * Abstract interface `LLMProvider` in `app.modules.ai_review.provider` implemented by:
     * `OpenAIProvider`
     * `GeminiProvider`
     * `MockLLMProvider` (for deterministic unit/integration testing without network or API costs).
   * Factory function `get_llm_provider()` instantiates the configured provider with appropriate API keys (`OPENAI_API_KEY` or `GEMINI_API_KEY`).
2. **Consolidated Pull Request Review Delivery (Option A)**:
   * Group all inline comments into a single batched GitHub Pull Request Review (`POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews`).
   * Include Markdown code suggestions (````suggestion ... ````) on exact modified lines.
   * Provide an overarching markdown summary verdict (`APPROVE`, `REQUEST_CHANGES`, or `COMMENT`).
   * Synchronize with the GitHub Checks API (`CheckRun`) with overall conclusion and line annotations.
3. **Full-Stack Docker Compose Architecture (Option B)**:
   * Docker Compose manages all services: `postgres`, `redis`, `backend` (FastAPI), `celery_worker`, and `frontend` (Vite/React).
   * Host volumes mounted for live code reloading during local development.

## Consequences
* High portability between AI providers without altering orchestration or finding aggregation logic.
* High-signal developer experience on GitHub (single notification per review rather than multiple noisy comments).
* One-command local startup via `docker-compose up --build`.
