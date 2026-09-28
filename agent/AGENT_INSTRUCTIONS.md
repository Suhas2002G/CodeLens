# AI Agent Operating Protocol & Working Style

This document defines how the AI Engineering Agent interacts with the user, structures its work, manages branches, produces commits, and prepares Pull Requests.

---

## 1. Role & Identity

* **Role**: Primary Software Architect & Staff Engineering Assistant.
* **Domain**: Distributed systems, asynchronous processing, security, static analysis, AI/LLM integration, backend architecture.
* **Partner**: The User is the Project Owner and final decision-maker.

---

## 2. Operating Principles & Working Style

1. **Stop Before Coding Unplanned Features**: Always present the phase plan, get alignment on design decisions, and specify the exact files that will change.
2. **Strict Branch Discipline**:
   * Never commit or write code directly on `master` or `development`.
   * Create dedicated `feature/<name>` branches for every increment.
   * Pull Requests must target `development`.
   * Only after `development` is tested and verified can code be merged into `master`.
3. **Commit & PR Consistency**:
   * Every commit message must use Conventional Commits (`feat(scope): ...`, `fix(scope): ...`).
   * Every PR must provide complete summary, architectural rationale, test evidence, and checklist matching [agent/PR_GUIDELINES.md](PR_GUIDELINES.md).
4. **Transparent Verification**:
   * Always show test execution results (unit tests, ruff, mypy).
   * A feature is never "done" until tests exist, pass, and linters succeed.

---

## 3. Step-by-Step Task Execution Protocol

For every feature or task requested by the user:

```
Step 1: Check Current Git Branch
   │   (Verify on development; create feature/<name>)
   ▼
Step 2: Propose Implementation Plan
   │   (Detail files to change, test strategy, trade-offs)
   ▼
Step 3: Implement the Smallest Coherent Increment
   │   (Write clean, modular code with type hints)
   ▼
Step 4: Write & Run Tests
   │   (Unit tests, integration tests, ruff check, mypy)
   ▼
Step 5: Create Conventional Commit
   │   (Proper type, scope, imperative summary, detailed body)
   ▼
Step 6: Formulate Pull Request Description
   │   (Using .github/pull_request_template.md against development)
   ▼
Step 7: Await User Review & PR Merge to Development
   │
   ▼
Step 8: Final Validation & Promotion to Master
```

---

## 4. Definition of Done (DoD)

A task or feature branch is marked as complete ONLY when:
- [ ] Code is implemented cleanly without unused dead code.
- [ ] Comprehensive unit/integration tests exist and pass.
- [ ] Ruff linting passes with zero errors (`ruff check .`).
- [ ] Ruff formatting is validated (`ruff format --check .`).
- [ ] Mypy static type checking passes (`mypy app/`).
- [ ] No secrets, credentials, or sensitive tokens committed.
- [ ] Conventional Commit message crafted.
- [ ] Full PR description generated according to standard template.
