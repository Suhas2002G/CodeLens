# Commit Message Convention & Standards

This repository adheres strictly to the **Conventional Commits 1.0.0** specification. Consistent commit messages ensure clean changelogs, predictable release management, and immediate clarity during code reviews.

---

## 1. Commit Message Structure

```
<type>(<scope>): <short imperative summary>

[optional detailed body]

[optional footer(s)]
```

### 1.1 Header Rules
* **Max Length**: 72 characters.
* **Imperative Mood**: Use imperative, present tense ("add" not "added", "fix" not "fixing", "implement" not "implemented").
* **No Period**: Do not end the subject line with a period.
* **Lowercase**: Type and scope must be lowercase.
* **NO EMOJIS**: Never include emojis, icons, or decorative symbols in commit subjects or bodies.

---

## 2. Commit Types

| Type | When to Use | Example |
|---|---|---|
| `feat` | A new user-facing or platform capability | `feat(diff): add line-to-position mapping in hunk parser` |
| `fix` | A bug fix or defect correction | `fix(auth): fix token expiration calculation for github app` |
| `docs` | Documentation-only updates | `docs(api): document rate limit error response schema` |
| `test` | Adding new tests or refactoring test suites | `test(webhook): add tests for forged signature payloads` |
| `refactor` | Code restructuring without feature or bug change | `refactor(db): extract base repository class with async session` |
| `perf` | A code change that improves execution performance | `perf(ast): cache parsed AST trees for repeated queries` |
| `chore` | Build tasks, package updates, tooling config | `chore(deps): bump pydantic from 2.8.0 to 2.8.2` |
| `ci` | Changes to CI/CD workflows and pipelines | `ci(github): add automated pytest and ruff check workflow` |
| `style` | Formatting, whitespace, semicolon changes | `style(main): reorder imports per isort rules` |

---

## 3. Standard Scopes

* `(webhook)`: GitHub webhook receiver, signature verifier, delivery idempotency.
* `(github)`: GitHub App authentication, Octokit/REST client, installation token manager.
* `(diff)`: Unified diff parser, patch parser, hunk indexer.
* `(ast)`: AST context builder, symbol enclosing scope extractor, import finder.
* `(analyzers)`: Static linters (Ruff, Bandit, Semgrep) runner and parsers.
* `(ai)`: LLM client abstractions, prompt templates, structured output validation.
* `(dedup)`: Finding normalizer, deduplication engine, confidence scoring.
* `(publisher)`: GitHub PR comment publisher, Markdown suggestion formatter, Checks API.
* `(db)`: SQLAlchemy models, Alembic migrations, database connection pooling.
* `(auth)`: JWT authentication, user management, organization RBAC.
* `(frontend)`: React components, dashboard UI, Tailwind styling, state management.
* `(docker)`: Dockerfile, docker-compose, container networking, volumes.

---

## 4. Good vs. Bad Commit Examples

### Good Commit Example
```
feat(diff): implement unified diff hunk parser with line offsets

- Parse git unified diff patches into structured file hunks
- Calculate added/deleted line counts and modified line numbers
- Map line numbers to GitHub diff positions for inline commenting
- Add unit tests covering multiple hunks, additions, and deletions

Refs: #14
```

### Bad Commit Examples
```
git commit -m "fixed bugs"
(Vague, no type, no scope, past tense)

git commit -m "WIP"
(Never commit work in progress without descriptive intent)

git commit -m "feat(webhook): added webhook verification"
(Past tense "added", emojis/capitalization forbidden)

git commit -m "feat(webhook): add signature verification"
(Contains emojis - STRICTLY FORBIDDEN)
```

---

## 5. Breaking Changes
If a commit introduces a breaking change to an internal or external contract, append `!` after the type/scope and document the breaking change in the footer:
```
feat(api)!: migrate auth token endpoint to return bearer structure

BREAKING CHANGE: The `/api/v1/auth/token` endpoint now returns a JSON object with `access_token` and `token_type` fields instead of a raw token string.
```
