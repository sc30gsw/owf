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

### Step 1 — Read and validate outline.md

Read the outline.md at the given path.

Validate it has these required sections:
- `## Goal / Acceptance Criteria` — not empty
- `## Steps` — contains at least one numbered step
- `## Verification` — contains at least one check

If any required section is missing or empty: stop and report in chat (Japanese):
```
❌ outline.md が不完全です。以下のセクションが未記入です: [list]
`/owf:outline` でアウトラインを先に完成させてください。
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

Output in chat (Japanese):
```
✅ 実装完了: ./outlines/<slug>/outline.md

テスト: <N> pass, 0 fail
カバレッジ: <N>%<if <80: " ⚠️ 80%未満です">
型チェック: クリーン
変更ファイル:
  - <file1> (<new|modified>)
  - <file2> (<new|modified>)

次のステップ: `/owf:review ./outlines/<slug>/outline.md`
```
