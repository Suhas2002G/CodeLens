# Antigravity Operating Rules

- Git Branching: Never write feature code directly to `master` or `development`. Always create a `feature/<name>` branch off `development`.
- Merge Order: `feature/*` -> `development` -> `master`.
- Commits: Conventional Commits (`feat(scope): ...`, `fix(scope): ...`).
- PRs: Must include summary, changes, testing evidence, and security verification.
- Formatting: Never use emojis in commit messages, PR titles, or PR descriptions.
- Code Quality: Python 3.11+, typed, Ruff-linted, Mypy-checked, tested with pytest.
