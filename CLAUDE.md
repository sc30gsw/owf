# OWF Plugin — Internal Guide

## Plugin philosophy

**Outline is 90% of the work.** The quality of `outline.md` determines the quality of everything downstream. This plugin enforces a quality gate on outlines before any implementation starts.

## Architecture

```
/owf:outline  → owf-outliner (opus, effort:xhigh) ↔ owf-outline-critic (opus, effort:xhigh)
/owf:implement → owf-implementer (sonnet)
/owf:review   → owf-reviewer (opus, effort:xhigh) ↔ owf-fixer (sonnet)
```

All agents are self-contained — no dependency on external plugins or user-scope skills.

## Output artifacts (minimal by design)

Per feature (slug), at most 3 files:
- `./outlines/<slug>/outline.md` — the single source of truth (never modified after Phase 1)
- `./outlines/<slug>/outline-pr.md` — review request notes (only if score < 100 in Phase 1)
- `./outlines/<slug>/pr.md` — PR template (created in Phase 3)

No runtime state directories. No verdict JSON files. No per-iteration archives.

## Reviewer READ-ONLY enforcement

`owf-outline-critic` and `owf-reviewer` have `tools: Read, Grep, Glob` only.
Without Write/Edit tools, they physically cannot modify files regardless of prompt.
`owf-reviewer` additionally has `Bash` but only for read-only commands (git diff, npm test).

## Scoring

Both phases use: 4 axes × 25 points = 100 total.
- 80+ = 🟢 GREEN (accept, note gaps)
- 75-79 = 🟡 YELLOW (stop loop, escalate to user)
- <75 = 🔴 RED (continue iteration, max 3)

See `/owf:rubric` for full details.

## Environment variables

| Variable | Default | Effect |
|---|---|---|
| `OWF_MAX_ITER_OUTLINE` | 3 | Max iterations for Phase 1 critic loop |
| `OWF_MAX_ITER_REVIEW` | 3 | Max iterations for Phase 3 review loop |
| `OWF_TRACE` | (off) | If set to 1, append score log to `.owf-trace.log` |

## Templates

Templates in `./templates/` are reference designs — the actual outline content is generated inline by the skill. Edit template files to customize the default structure.

## `@` agent usage (manual control)

Each agent can be called directly without going through the skill orchestrator:

- `@owf-outliner` — fill/revise an outline.md directly
- `@owf-outline-critic` — review an existing outline.md and return Verdict
- `@owf-implementer` — implement from outline.md (pass full content in prompt)
- `@owf-reviewer` — review implementation and return Verdict
- `@owf-fixer` — apply fixes from a reviewer Verdict

This is useful when you want to intervene mid-loop or run a single phase step manually.
