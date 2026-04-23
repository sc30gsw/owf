---
name: owf-implement
description: Phase 2 — Implement a development task from outline.md using TDD. outline.md is the single source of truth and must not be modified during implementation. Invoke as `/owf:implement <path-to-outline.md>`.
---

# OWF Implement — Phase 2 Orchestrator

## Usage

```
/owf:implement ./outlines/<slug>/outline.md
```

## Execution steps

### Step 0 — Detect output language

Determine `DETECTED_LANG` by evaluating signals in priority order (first confident match wins):

**Priority 1 — Explicit override**: `OWF_LANG` env var (e.g., `ja`, `en`, `zh`, `ko`, `de`, `fr`, `es`). If set, use verbatim.

**Priority 2 — User conversation language**: Inspect the user's most recent prompt / active conversation turn. If the user is clearly writing in a specific language, use that. This is the strongest real-time signal — English code with Korean narrative still indicates Korean.

**Priority 3 — Project signals (aggregated)**:
```bash
{
  head -30 README.md 2>/dev/null
  head -30 readme.md 2>/dev/null
  head -30 CLAUDE.md 2>/dev/null
  cat .github/ISSUE_TEMPLATE/*.md 2>/dev/null | head -30
  git log --oneline -50 2>/dev/null
} 2>/dev/null | head -200
```
Identify the dominant language. Supported: `ja` (hiragana/katakana), `zh` (CJK ideographs, no kana), `ko` (hangul U+AC00–U+D7AF), `de` (`ä ö ü ß` + `der die das und`), `fr` (`à â ç é è ê` + `le la les de des`), `es` (`ñ ¿ ¡ á é í ó ú` + `el la los las`), `en` (predominantly ASCII, no dominant marker).

**Priority 4 — Fallback**: `en`.

Store as `DETECTED_LANG`. **Output skeletons in this skill show English labels as the canonical reference.** Translate every label and narrative line into `DETECTED_LANG`, and keep unchanged: markdown structure, file paths, slash commands, `@agent-name` references, shell commands, and score/band tokens.

### Step 1 — Read and validate outline.md

Read the outline.md at the given path.

Validate it has these required sections:
- `## Goal / Acceptance Criteria` — not empty
- `## Steps` — contains at least one numbered step
- `## Verification` — contains at least one check

If any required section is missing or empty: stop and report in chat (in `DETECTED_LANG`):
```
❌ outline.md is incomplete. Missing/empty sections: [list]
Run `/owf:outline` first to complete the outline.
```

### Step 2 — Fix the read-only constraint

Before proceeding: confirm to yourself that outline.md MUST NOT be modified during this phase. If you discover a conflict between the outline and the codebase, STOP and report to the user with:
- The specific conflict (what outline says vs what codebase shows)
- Two options: (a) update the outline first (`/owf:outline`), (b) implement with the conflict noted as a deviation

### Step 3 — Determine execution mode

Scan the `## Steps` and `## Change Surface` sections. Determine if parallel execution is safe:

**Sequential mode** (default): if steps share file paths or have sequential dependencies.

**Parallel mode**: if the outline describes independent feature groups (e.g., "Step 1-3: feature A in `src/features/a/`; Step 4-6: feature B in `src/features/b/`"). If parallel mode is clearly safe:
- Split steps into non-overlapping groups
- Spawn one `owf-implementer` agent per group via Agent Teams
  - Each receives: full outline.md content + only their assigned step range
  - No shared file writes between groups
- After all groups complete: proceed to Step 5 (test suite)

If there is any doubt about file path overlap: use sequential mode.

### Step 4 — Spawn owf-implementer

**Sequential mode**: spawn one `owf-implementer` agent with:
- Full outline.md content
- Instruction to follow TDD (RED → GREEN → REFACTOR) for each step
- Coverage target: ≥80%

Wait for owf-implementer to report completion. If it reports a conflict with outline.md (scope drift detected), stop and escalate to user.

**Parallel mode**: spawn multiple `owf-implementer` agents. Each receives their assigned step range. Wait for all to complete.

### Step 5 — Run full test suite

After owf-implementer reports done, run the project's test suite from the project root:

```bash
# detect test runner and run
npm test 2>&1 || yarn test 2>&1 || pnpm test 2>&1
```

If tests fail: provide the failure output to the `owf-implementer` agent with instruction "Fix the failing tests without changing outline.md." Repeat until all pass (up to 3 fix attempts; if still failing after 3 attempts, stop and report to user).

### Step 6 — Type check

```bash
npx tsc --noEmit 2>&1 || true
```

If type errors exist: provide them to `owf-implementer` for targeted fixes. Repeat until clean.

### Step 7 — Lint (if configured)

Check for lint config (`.eslintrc*`, `biome.json`, `.oxlintrc`, `oxlint.json`):
```bash
ls .eslintrc* biome.json .oxlintrc oxlint.json 2>/dev/null
```

If lint config found, run lint. Provide any errors to `owf-implementer` for fixes.

### Step 8 — Final output

Output in chat (in `DETECTED_LANG`):
```
✅ Implementation complete: ./outlines/<slug>/outline.md

Tests: <N> pass, 0 fail
Coverage: <N>%<if <80: " ⚠️ below 80%">
Type check: clean
Changed files:
  - <file1> (<new|modified>)
  - <file2> (<new|modified>)

Next step: `/owf:review ./outlines/<slug>/outline.md`
```

## Terminology constraint (CRITICAL — prevents hallucinated commands)

When generating any user-facing chat output:

- **NEVER** suggest `/owf:<agent-name>` style commands. Agents (`owf-outliner`, `owf-outline-critic`, `owf-implementer`, `owf-reviewer`, `owf-fixer`) are NOT slash commands. Writing `/owf:owf-implementer`, `/owf:owf-outline-critic`, etc. is a hallucination — those commands do not exist and will fail with "Unknown command".
- **Valid OWF slash commands**, the only ones that may appear after `/`:
  - `/owf:outline`
  - `/owf:implement`
  - `/owf:review`
  - `/owf:rubric`
- When referring to agents in chat output, **always use `@agent-name` prefix** (e.g., `@owf-implementer`, `@owf-reviewer`).

This rule overrides any pattern-completion instinct that might produce `/owf:<agent>`.
