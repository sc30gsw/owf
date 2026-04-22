---
name: owf-review
description: Phase 3 — Review implementation against outline.md using an adversarial reviewer/fixer Agent Team (max 3 iterations). Produces ./outlines/<slug>/pr.md. Invoke as `/owf:review <path-to-outline.md>`.
---

# OWF Review — Phase 3 Orchestrator

## Usage

```
/owf:review ./outlines/<slug>/outline.md
```

## Execution steps

### Step 1 — Gather review context

Read `./outlines/<slug>/outline.md`.

Collect the implementation diff:
```bash
# try in order until one works:
git diff main...HEAD 2>/dev/null || git diff master...HEAD 2>/dev/null || git diff HEAD~1 HEAD 2>/dev/null
```

List the modified files:
```bash
git diff --name-only main...HEAD 2>/dev/null || git diff --name-only HEAD~1 HEAD 2>/dev/null
```

Read the content of all modified source files (not test files — read those separately if needed).

### Step 2 — Set up adversarial Agent Team

Create an adversarial review team using the Agent Teams mechanism (TeamCreate). The team consists of:

1. **owf-reviewer** — READ-ONLY adversarial critic (tools: Read, Grep, Glob, Bash for read-only commands only)
2. **owf-fixer** — implementation fixer (tools: Read, Write, Edit, Bash, Grep, Glob; outline.md excluded from write scope)

Initialize: `iter = 0`, `max_iter = 3` (or `OWF_MAX_ITER_REVIEW` env var if set).

### Step 3 — Reviewer turn

Spawn `owf-reviewer` agent with:
- Full content of outline.md (labeled as "GROUND TRUTH — do not change this")
- Full git diff of implementation
- Content of all modified files
- Current iteration number

Wait for owf-reviewer to return its Verdict block.

### Step 4 — Parse verdict and branch

Parse `score:` and `band:` from the Verdict block. If parsing fails: treat score=0, band=RED.

**score == 100 (perfect):**
→ Proceed to Step 6 (generate pr.md and final output).

**score ≥ 80 (GREEN):**
→ Record findings for pr.md `## Remaining Risks` section.
→ Proceed to Step 6.

**score ≥ 75 (YELLOW) — stop loop:**
→ Proceed to Step 6 with YELLOW status.

**score < 75 (RED) AND iter < max_iter:**
→ Increment `iter`.
→ Proceed to Step 5 (fixer turn).
→ After fixer completes, return to Step 3.

**score < 75 (RED) AND iter == max_iter:**
→ Record final findings for pr.md `## Next Action Proposals`.
→ Proceed to Step 6 with RED status.

### Step 5 — Fixer turn

Spawn `owf-fixer` agent with:
- Full Findings section from the reviewer's Verdict
- List of flagged files from the Routing section
- Reminder: "outline.md is ground truth — do not modify it"

Wait for owf-fixer to report completion. It will run tests internally and confirm they pass.

Return to Step 3 for next reviewer turn.

### Step 6 — Generate pr.md

Create `./outlines/<slug>/pr.md` using the following structure (adapt to actual content):

Fill in the template at `owf/templates/pr.md.tmpl` with:
- **Summary**: from outline.md `## Goal / Acceptance Criteria`
- **Changes**: list from git diff (grouped by feature)
- **What was NOT done**: from outline.md `## Out of Scope`
- **Questions for Reviewer**: from outline.md `## Questions for Reviewer` + any unresolved [MED] findings
- **Mermaid diagram**: if outline.md had one, adapt it; otherwise omit
- **Self-checklist results**: actual pass/fail status of each item based on review findings
- **Review temperature**: based on final score (< 80: High, 80-95: Middle, 100: Low)

If score < 100: append `## Remaining Risks` or `## Next Action Proposals` (from Step 4).

### Step 7 — Final CLI output

Output in chat (Japanese):

```
========================================
OWF Review: ./outlines/<slug>/outline.md
========================================
スコア: <N> / 100
バンド: 🟢 GREEN | 🟡 YELLOW | 🔴 RED
イテレーション: <N> / <MAX>
----------------------------------------
評価の根拠:
  fidelity   <N>/25: <one-line>
  tests      <N>/25: <one-line>
  simplify   <N>/25: <one-line>
  maintain   <N>/25: <one-line>
----------------------------------------
<if GREEN or YELLOW:>
次のアクション:
  1. pr.md を確認してください: ./outlines/<slug>/pr.md
  2. コミット後に PR を作成: gh pr create --body-file ./outlines/<slug>/pr.md
<if any LOW findings remain:>
  3. 任意の改善: <brief list of LOW findings>

<if RED:>
次のアクション:
  - 未解決の指摘事項を outline.md の `## Next Action Proposals` に記載しました。
  - outline を分割するか、手動で修正後、再度 `/owf:review` を実行してください。
========================================
```

## Notes

- The reviewer (owf-reviewer) is hardcoded READ-ONLY via its `tools:` frontmatter — no hook needed.
- The fixer (owf-fixer) excludes outline.md from all writes via instruction — enforce via prompt.
- If OWF_TRACE=1 is set in environment, append one line to `./outlines/<slug>/.owf-trace.log`:
  `<ISO timestamp> | review | iter=<N> | score=<N> | band=<BAND> | <slug>`
