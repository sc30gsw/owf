---
name: owf-rubric
description: Display the owf scoring rubric — 0-100 scale, Red/Yellow/Green bands, verdict format, and iteration caps. Use when you need to understand how outlines and implementations are evaluated.
---

# OWF Scoring Rubric

## Score: 0–100 (4 axes × 25 pts each)

### Phase 1 — Outline axes

| Axis | What it measures | Max |
|---|---|---|
| `clarity` | Goal, scope, acceptance criteria readable in one sentence | 25 |
| `decomposition` | Steps are appropriate granularity, logically ordered, independently testable | 25 |
| `risk` | Edge cases, assumptions, rollback plan explicitly stated | 25 |
| `reuse` | Existing code / patterns referenced, no NIH | 25 |

### Phase 3 — Implementation axes

| Axis | What it measures | Max |
|---|---|---|
| `fidelity` | Implementation matches outline 1:1, no scope drift | 25 |
| `tests` | TDD followed, all tests pass, ≥80% coverage, edge cases tested | 25 |
| `simplify` | Reuse of existing utils/hooks/types; quality (file size, immutability); efficiency (no redundant loops, unused exports) | 25 |
| `maintain` | Naming, file size (≤800 lines), immutability, Result pattern, no console.log | 25 |

## Band thresholds

| Score | Band | Action |
|---|---|---|
| 100 | — | Perfect — no notes required |
| 80–99 | 🟢 GREEN | Accept. Append unresolved points to artifact as `## Remaining Risks`. |
| 75–79 | 🟡 YELLOW | Stop loop immediately. Generate PR note file. Escalate to user. |
| < 75 | 🔴 RED | Continue iteration (up to max). |

After max iterations with score < 75:
- Append `## Score Improvement Suggestions` with concrete proposals (split the task, reference a specific existing skill, point to similar implementation in codebase).

## Verdict block format (mandatory for critics and reviewers)

```
### Verdict
score: <0-100>
band: <RED|YELLOW|GREEN>
iteration: <N>/<MAX>

### Dimensions
- <axis>: <score>/25 — <one-line rationale>

### Findings
- [HIGH|MED|LOW] <location>: <specific problem> → <concrete fix>

### Routing
to: <owf-outliner|owf-fixer>
sections: ["<section or file>", ...]
```

Parse failure (malformed verdict): treat as score=0, band=RED, continue loop.

## Iteration caps

- Phase 1 (outline): default 3. Override: `OWF_MAX_ITER_OUTLINE` env var.
- Phase 3 (review): default 3. Override: `OWF_MAX_ITER_REVIEW` env var.
