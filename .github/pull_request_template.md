## Summary & Motivation
<!-- Provide a clear 2-3 sentence overview of what this PR introduces and the problem it solves. -->

## Changes Introduced
<!-- Bullet points detailing exact changes made across modules with file links. -->
- 

## Architectural Decisions & Trade-offs
<!-- Mention any ADR referenced, trade-offs evaluated, or design choices. -->
- 

## Testing & Verification
<!-- Document how these changes were tested and verified. Include commands and outputs. -->
- [ ] Unit tests pass (pytest backend/tests/unit/)
- [ ] Integration tests pass (pytest backend/tests/integration/)
- [ ] Linting passes (ruff check backend/)
- [ ] Type checking passes (mypy backend/app/)

## Security & Performance Considerations
- [ ] No untrusted repository code executed directly.
- [ ] Input data validated through strict Pydantic schemas.
- [ ] Secrets redacted before sending to external services.

## Pre-Merge Checklist
- [ ] Target branch is development (for feature branches) or master (for release PRs).
- [ ] Commits follow Conventional Commits standard (agent/COMMIT_CONVENTION.md).
- [ ] No emojis in PR title, PR description, or commit messages.
- [ ] Documentation updated if architecture or configuration changed.
