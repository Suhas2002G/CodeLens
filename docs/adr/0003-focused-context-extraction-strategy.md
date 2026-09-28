# ADR 0003: Focused Context Extraction vs Full-Repo Ingestion

* **Status**: Accepted
* **Date**: 2026-09-28
* **Deciders**: Software Architect & Platform Lead

---

## Context
Code review requires understanding context beyond the isolated lines changed in a git diff. However, ingesting an entire Git repository or entire files:
* Rapidly exceeds LLM token limits and budgets.
* Increases review latency to minutes instead of seconds.
* Dilutes LLM attention, resulting in lower recall of actual bugs in modified code.
* Exposes untouched private repository code to external LLM APIs unnecessarily.

## Decision
We implement a **Targeted AST Slicing and Context Extraction Engine**:
1. **Diff Hunk Bounding**: Extract only the added/modified lines plus immediate Git context lines (`-U5`).
2. **AST Node Enclosure**:
   * Parse the target file into a Python AST.
   * Locate the immediate enclosing scope (e.g. function/method definition, enclosing class, decorators, type signatures).
   * Include the function signature and docstring even if only 2 lines within the function were changed.
3. **Import Graph Extraction**:
   * Extract module-level imports (`import os`, `from app.models import User`) to provide context on types and external libraries used.
4. **Token Budgeting**:
   * Cap context per file to 2,000 tokens.
   * Exclude lockfiles, vendor directories, and minified bundles.

## Consequences

### Positive
* Predictable and low LLM token costs per review (typically < 3,000 tokens for average PR).
* Drastically reduced latency: Prompts are compact and fast to process.
* Enhanced focus: The LLM pays maximum attention to the modified logic within its surrounding functional scope.

### Negative / Trade-offs
* Complex inter-file architectural regressions (e.g., modifying an interface in `module_a` that subtly breaks `module_b` without modifying `module_b`) will not be caught in single-file focused context. (Can be addressed in future phases via repository-wide symbol indexing or LSIF/SCIP).
