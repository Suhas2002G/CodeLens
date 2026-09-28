# GEMINI Agent Operating Guidelines & Workspace Standards

This document defines the strict operating procedures, git branching strategy, commit conventions, Pull Request standards, and working style for the AI agent and all contributors working on this repository.

---

## 1. Core Operating Directive

* **No Direct Commits to Main/Master or Development**: Never commit feature or bugfix code directly to `master` or `development`.
* **Branch-Based Workflow**: All code changes must occur on a dedicated `feature/*`, `fix/*`, `chore/*`, or `docs/*` branch.
* **Pull Request Discipline**:
  * Feature branches are merged into `development` via Pull Requests after passing local tests and linting.
  * Only after `development` is verified, tested, and validated as stable is a PR created from `development` into `master`.
* **Zero Assumption on Commands**: Always explain changes clearly and follow the agreed phase roadmap.
* **Strictly No Emojis**: Never use emojis, decorative icons, or pictographs in commit messages, PR titles, or PR descriptions.

---

## 2. Git Branching Strategy

```
master (Production / Release)
  ▲
  │ (Release PR after complete testing on dev)
development (Integration & Verification)
  ▲
  │ (Feature PR after unit/integration tests pass)
feature/<feature-name> (Active Development)
```

| Branch | Purpose | Protection Rules | Merge Target |
|---|---|---|---|
| `master` | Production-ready, stable codebase | Locked. No direct pushes. Merges only from `development`. | Production / Releases |
| `development` | Integration branch for testing cohesive increments | Locked. No direct pushes. Merges only from `feature/*` or `fix/*` via PRs. | `master` |
| `feature/<name>` | Individual feature development | Created off `development`. | `development` |
| `fix/<name>` | Bug fix or issue resolution | Created off `development` (or `master` for hotfixes). | `development` |
| `chore/<name>` | Tooling, dependencies, or repo maintenance | Created off `development`. | `development` |
| `docs/<name>` | Documentation updates | Created off `development`. | `development` |

---

## 3. Commit Style & Message Pattern (Conventional Commits)

All commits must strictly follow the **Conventional Commits 1.0.0** specification.

### 3.1 Structure
```
<type>(<scope>): <short imperative summary>

[optional body explaining WHY the change was made and trade-offs]

[optional footer(s) for Breaking Changes or Issue References]
```

### 3.2 Allowed Types
* `feat`: A new feature or capability.
* `fix`: A bug fix or defect resolution.
* `docs`: Documentation only changes.
* `style`: Formatting, missing semicolons, whitespace (no code change).
* `refactor`: Code change that neither fixes a bug nor adds a feature.
* `perf`: Code change that improves performance.
* `test`: Adding missing tests or correcting existing tests.
* `build`: Changes that affect the build system or external dependencies.
* `ci`: Changes to CI/CD configuration files and scripts.
* `chore`: Maintenance tasks, configuration updates, or tooling.

### 3.3 Scopes
Use relevant module scopes:
* `(webhook)`
* `(diff-parser)`
* `(ast-context)`
* `(static-analyzer)`
* `(ai-engine)`
* `(dedup)`
* `(publisher)`
* `(db)`
* `(auth)`
* `(frontend)`
* `(docker)`

### 3.4 Commit Message Example
```
feat(webhook): implement hmac-sha256 signature verification

- Add constant-time hmac comparison in GitHub signature verification service
- Add Redis idempotency guard using X-GitHub-Delivery header
- Add unit tests for valid, tampered, and missing signature headers

Closes #12
```

---

## 4. Pull Request Standards & Creation Pattern

Every Pull Request must follow the standardized structure defined in `agent/PR_GUIDELINES.md` and `.github/pull_request_template.md`.

### 4.1 PR Title Format
```
<type>(<scope>): <concise description of PR purpose>
```
*Example*: `feat(ast-context): implement enclosing function and class extractor`

### 4.2 PR Description Template
Every PR must contain:
1. **Summary & Motivation**: 2-3 sentences explaining what this PR accomplishes and why.
2. **Changes Introduced**: Bulleted list of additions, modifications, or removals with file links.
3. **Architectural Decisions & Trade-offs**: Any ADR references or non-obvious choices.
4. **Testing & Verification**: Exact commands run, test pass/fail metrics, coverage impact.
5. **Security Considerations**: Authentication, input sanitization, or permission checks verified.
6. **Checklist**: Completed checkboxes confirming code standards, linting, tests, and documentation.

---

## 5. Working Style & Definition of Done

For each development task:
1. **Step 1 - Branch Creation**: Ensure on latest `development` branch and branch to `feature/<name>`.
2. **Step 2 - Planning & Alignment**: Explain what will be implemented, files modified, and test plan.
3. **Step 3 - Implementation**: Write clean, typed, modular code adhering to `ENGINEERING_GUIDELINES.md`.
4. **Step 4 - Testing**: Run unit and integration tests; verify linting with Ruff and type checks with Mypy.
5. **Step 5 - Commit**: Craft disciplined Conventional Commit messages.
6. **Step 6 - PR Preparation**: Prepare complete PR documentation against `development`.
7. **Step 7 - Integration Validation**: After merging to `development`, validate end-to-end functionality before planning release PR to `master`.
