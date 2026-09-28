# Pull Request Guidelines & Description Standards

Every change merged into `development` or `master` must be introduced through a disciplined, high-quality Pull Request. This document defines the required PR title, description format, testing evidence, and review criteria.

---

## 1. Pull Request Title Standard

All PR titles must follow Conventional Commits formatting:
```
<type>(<scope>): <concise description of PR purpose>
```
* **Correct**: `feat(webhook): add hmac-sha256 signature verification and idempotency check`
* **Correct**: `fix(ast): handle async function definitions in context extractor`
* **Incorrect**: `Updated webhook code`
* **Incorrect**: `WIP: working on PR parser`

---

## 2. Standard Pull Request Description Template

Whenever opening a Pull Request (either from `feature/*` to `development` or `development` to `master`), use the following markdown template:

```markdown
## Summary & Motivation
<!-- Provide a clear 2-3 sentence overview of what this PR introduces and the problem it solves. -->

## Changes Introduced
<!-- Bullet points detailing exact changes made across modules with file links. -->
- Added `app/modules/diff/parser.py`: Unified git diff hunk parsing.
- Added `app/modules/diff/models.py`: Pydantic models for `DiffHunk` and `ModifiedLine`.
- Updated `backend/tests/unit/test_diff_parser.py`: Added 8 unit tests for hunk boundaries.

## Architectural Decisions & Trade-offs
<!-- Mention any ADR referenced, trade-offs evaluated, or design choices. -->
- Referenced [ADR 0003: Focused Context Extraction Strategy](../docs/adr/0003-focused-context-extraction-strategy.md).
- Chose line-to-position mapping during diff parsing rather than late-stage resolution to keep reviewer publishers decoupled.

## Testing & Verification
<!-- Document how these changes were tested and verified. Include commands and outputs. -->
- [x] Unit tests pass: `pytest backend/tests/unit/test_diff_parser.py` (8 passed in 0.24s)
- [x] Linting passes: `ruff check backend/` (no errors)
- [x] Formatting passes: `ruff format --check backend/`
- [x] Type checking passes: `mypy backend/app/` (success: no issues found)

```bash
# Example test output:
============================== 8 passed in 0.24s ==============================
```

## Security & Performance Considerations
- [x] No untrusted repository code executed directly.
- [x] Inputs validated via Pydantic v2 schemas.
- [x] Diff parsing capped to 1MB to prevent regex ReDoS / worker memory exhaustion.

## Pre-Merge Checklist
- [ ] Code follows project [ENGINEERING_GUIDELINES.md](../ENGINEERING_GUIDELINES.md).
- [ ] Branch is branched off and up to date with `development`.
- [ ] Target branch is `development` (or `master` for release PRs).
- [ ] Commits follow [agent/COMMIT_CONVENTION.md](COMMIT_CONVENTION.md).
- [ ] Documentation updated if public interfaces or architecture changed.
```

---

## 3. Two-Tier PR Review Workflow

### Tier 1: Feature PR (`feature/*` → `development`)
* Focus: Unit correctness, module isolation, edge cases, test coverage (>85%), linting.
* Scope: Single coherent feature increment.
* Merged via **Squash and Merge**.

### Tier 2: Release PR (`development` → `master`)
* Focus: End-to-end integration stability, multi-module coordination, migration safety, performance, and documentation completeness.
* Scope: Cohesive batch of tested features forming a release milestone.
* Merged via **Merge Commit** or Tagged Release.
