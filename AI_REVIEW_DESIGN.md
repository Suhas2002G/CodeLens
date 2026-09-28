# AI Review Engine & Context Extraction Architecture

## 1. Design Philosophy: High Signal, Zero Noise

Conventional AI review tools fail because they treat an LLM as a magical "read the whole repo and critique it" machine. This leads to:
1. **Excessive Noise**: Hallucinated complaints about missing docstrings, formatting nitpicks, or variable naming preferences.
2. **Context Window Exhaustion & High Cost**: Sending thousands of lines of unmodified files wastes money and degrades reasoning performance.
3. **Out-of-Diff Hallucinations**: Recommending line edits on lines the PR never touched.

### Our Strategy: Hybrid Targeted Context
```
PR Unified Diff
      │
      ▼
Hunk & Line Mapper ──► AST Node Extractor (Enclosing function/class & imports)
      │                                     │
      ▼                                     ▼
Deterministic Linters (Ruff/Bandit)   Focused Context Envelope
      │                                     │
      └──────────────────┬──────────────────┘
                         ▼
             Structured Prompt Assembly
                         │
                         ▼
        LLM Provider (JSON Schema Enforcement)
                         │
                         ▼
           Pydantic Validation & Line Bounds Check
                         │
                         ▼
         Deduplication with Static Tool Findings
```

---

## 2. Context Extraction Engine

### 2.1 Unified Diff Parsing
For every modified file in the PR:
1. Parse the diff hunks:
   ```diff
   @@ -120,6 +120,12 @@ def calculate_discount(order: Order) -> Decimal:
        if order.customer.is_vip:
   +        if order.coupon_code == "SUMMER50":
   +            return order.total * Decimal("0.50")
            return order.total * Decimal("0.20")
   ```
2. Identify:
   * Modified line numbers in new file (`121`, `122`).
   * GitHub diff position index (1-based line offset in the unified diff hunk).
   * Context lines (unmodified lines provided by git around the change).

### 2.2 AST Scope Extraction
Using Python's native `ast` module:
1. Parse the modified file into an AST tree.
2. Locate the deepest AST node that encloses the modified lines:
   * **Enclosing Function/Method**: Signature, arguments, return type, and docstring.
   * **Enclosing Class**: Class name, base classes, class-level attributes.
   * **Decorators**: `@atomic`, `@cached`, `@login_required` (vital for concurrency and auth context).
3. Extract file-level imports (`import stripe`, `from models import Order`).
4. Result: Instead of sending 1,000 lines of `payment_service.py`, we send the 30-line `calculate_discount` method with its type signature and decorators.

### 2.3 Noise & File Filtering
The following files are automatically excluded from LLM analysis:
* Lockfiles (`poetry.lock`, `package-lock.json`, `pnpm-lock.yaml`, `Pipfile.lock`, `yarn.lock`).
* Auto-generated files (e.g. Alembic migration scripts, Protobuf generated `_pb2.py`, OpenAPI generated stubs).
* Minified files (`*.min.js`, `*.min.css`).
* Binary, image, and media files.
* Files exceeding 1,000 lines of diff in a single commit (flagged as bulk refactor).

---

## 3. Prompt Engineering & Structured Outputs

### 3.1 System Prompt
```markdown
You are a Principal Software Engineer and Staff Security Architect conducting an automated code review on a GitHub Pull Request.

Your goal is to identify high-severity, high-confidence issues that a human reviewer might miss:
- Critical logic bugs, off-by-one errors, state corruption, or unhandled exceptions.
- Security vulnerabilities (OWASP Top 10, injection, insecure deserialization, privilege escalation).
- Concurrency issues, race conditions, deadlocks, and unsafe shared mutable state.
- Performance pitfalls (N+1 database queries, catastrophic backtracking regex, unbounded memory growth).
- Resource leaks (unclosed sockets, connections, or file handles).
- Breaking API changes or broken interface contracts.

STRICT CONSTRAINTS:
1. DO NOT comment on code formatting, whitespace, line length, or style conventions (handled by Ruff/linters).
2. DO NOT suggest adding comments, type docstrings, or renaming variables unless the existing name introduces serious logic ambiguity.
3. Every finding MUST target an exact line or range within the provided modified diff.
4. Provide a concrete, copy-pasteable replacement snippet in `suggested_fix` whenever possible.
5. If the code is well-written and has no significant flaws, return an empty `findings` list.
```

### 3.2 User Prompt Construction
```markdown
Analyze the following pull request changes.

Repository: {{repo_name}}
PR Title: {{pr_title}}
PR Description: {{pr_description}}

Relevant Static Analysis Findings (Already detected by deterministic tools):
{{static_findings_summary}}

Modified Code Context:
<untrusted_code_diff>
File: {{file_path}}
Enclosing Symbol: {{enclosing_symbol}}
Imports: {{imported_modules}}

```diff
{{hunk_content}}
```
</untrusted_code_diff>

Output your review as a JSON object strictly matching the required schema.
```

---

## 4. Structured Output Schema (Pydantic v2)

```python
from enum import Enum
from typing import List, Optional
from pydantic import BaseModel, Field

class FindingCategory(str, Enum):
    SECURITY = "SECURITY"
    CORRECTNESS = "CORRECTNESS"
    PERFORMANCE = "PERFORMANCE"
    CONCURRENCY = "CONCURRENCY"
    ERROR_HANDLING = "ERROR_HANDLING"
    API_MISUSE = "API_MISUSE"
    RESOURCE_LEAK = "RESOURCE_LEAK"

class FindingSeverity(str, Enum):
    CRITICAL = "CRITICAL"
    HIGH = "HIGH"
    MEDIUM = "MEDIUM"
    LOW = "LOW"
    INFO = "INFO"

class AIFinding(BaseModel):
    file_path: str = Field(..., description="Relative path of the modified file")
    line_start: int = Field(..., description="Starting line number in the new file")
    line_end: int = Field(..., description="Ending line number in the new file")
    category: FindingCategory = Field(..., description="Issue classification")
    severity: FindingSeverity = Field(..., description="Impact severity")
    confidence: float = Field(..., ge=0.0, le=1.0, description="Confidence score between 0.0 and 1.0")
    title: str = Field(..., max_length=200, description="Short, descriptive title")
    description: str = Field(..., description="Explanation of why this is a bug and potential impact")
    suggested_fix: Optional[str] = Field(None, description="Exact replacement code for GitHub diff suggestion")

class AIReviewOutput(BaseModel):
    summary: str = Field(..., description="High-level assessment of the changes")
    verdict: str = Field(..., description="VERDICT: 'APPROVE', 'REQUEST_CHANGES', or 'COMMENT'")
    findings: List[AIFinding] = Field(default_factory=list, description="List of identified issues")
```

---

## 5. Post-Processing & Validation Guardrails

Before an AI finding is accepted into the database or posted to GitHub:
1. **Line Bounds Validation**:
   * Check whether `line_start` and `line_end` correspond to modified or context lines inside the PR diff hunk.
   * If a finding references an unchanged line outside the diff: convert it from an inline comment to a file-level note in the summary review.
2. **Confidence Filter**:
   * Findings with `confidence < 0.70` are discarded or logged as internal diagnostics to maintain high developer trust.
3. **Deduplication with Static Linters**:
   * If Ruff or Bandit already flagged an issue on line `L` (e.g. `B608`), and the LLM flags the same line with `SECURITY`, combine them into a single finding, elevating confidence and citing both engines.
4. **Token & Cost Accounting**:
   * Record `prompt_tokens`, `completion_tokens`, and estimated dollar cost on the `reviews` table.
