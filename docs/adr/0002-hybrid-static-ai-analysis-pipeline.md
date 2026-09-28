# ADR 0002: Hybrid Static and AI Analysis Pipeline

* **Status**: Accepted
* **Date**: 2026-09-28
* **Deciders**: Software Architect & Platform Lead

---

## Context
Code review platforms generally fall into two extremes:
1. **Pure Static Analyzers / Linters**: Fast, deterministic, and zero cost, but produce high false positives on semantic intent and cannot reason about concurrency, business logic, architectural regressions, or subtle API misuse.
2. **Pure LLM Reviewers**: Ask an LLM to "review the code", which frequently yields noisy stylistic complaints, missed CVEs, hallucinated non-existent lines, and high token costs.

## Decision
We adopt a **Hybrid Pipeline Architecture**:
1. **Deterministic Static Analysis First**:
   * Tools: **Ruff** (Python code hygiene, AST sanity), **Bandit** (AST-level security flaws), **Semgrep** (pattern-based CVE detection).
   * These run concurrently in worker sandboxes with strict execution timeouts.
2. **Context-Aware LLM Semantic Review**:
   * Uses the static findings as supplementary context (`static_findings_summary`).
   * Explicitly forbidden from commenting on style or formatting.
   * Focuses strictly on logic correctness, security bypasses, race conditions, error handling, and performance regressions.
3. **Deduplication & Confidence Fusion**:
   * If a static tool and the LLM flag the same code block, the finding is merged into a single `HYBRID` finding with elevated confidence.
   * Eliminates duplicate comments on the developer's PR.

## Consequences

### Positive
* High signal-to-noise ratio: Developers only see actionable, high-severity insights.
* LLM efficiency: The LLM does not waste output tokens explaining syntax errors or pep8 violations.
* Resilient degradation: If the LLM provider experiences latency or outages, static analysis findings can still be delivered.

### Negative / Trade-offs
* Requires local installation or containerized packaging of static analyzer binaries (Ruff, Bandit, Semgrep) in the worker environment.
* Adds finding normalization complexity to reconcile heterogeneous output formats into a unified model.
