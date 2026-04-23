---
name: owf-review
description: Review an outline.md (adversarial only) OR an implementation against outline.md (adversarial + simplify). Auto-detects mode from git diff. Outline mode runs 2 parallel outline-critics. Implementation mode runs /simplify + 2 parallel reviewers + fixer loop (max 3 iterations) and produces ./outlines/<slug>/pr.md. Invoke as `/owf:review <path-to-outline.md>`.
---

# OWF Review — Dual-Mode Orchestrator

This skill has **two clearly separated modes**:

| モード | 対象 | 使用エージェント | 特徴 |
|---|---|---|---|
| **Outline Review** | `outline.md` 単体の品質確認 | `owf-outline-critic` × 2 並列 | 敵対的のみ（`/simplify` なし、fixer なし、1ラウンド） |
| **Implementation Review** | 実装コードの `outline.md` への整合性・品質検証 | `owf-reviewer` × 2 並列 + `owf-fixer` | 敵対的 + `/simplify` 前処理 + 修正ループ（最大3回） |

## Usage

```
/owf:review ./outlines/<slug>/outline.md
```

`--outline-only` flag（または対応する実装 diff がない場合）で Outline Review モードが自動選択されます。

## Execution steps

### Step 0 — Detect output language

Determine `DETECTED_LANG` by evaluating signals in priority order (first confident match wins):

**Priority 1 — Explicit override**: `OWF_LANG` env var (e.g., `ja`, `en`, `zh`, `ko`, `de`, `fr`, `es`). If set, use verbatim.

**Priority 2 — User conversation language**: Inspect the user's most recent prompt / active conversation turn. If the user is clearly writing in a specific language, use that. This is the strongest real-time signal.

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

Store as `DETECTED_LANG`. **Output skeletons in this skill show English labels as the canonical reference.** Translate every label and narrative line into `DETECTED_LANG`, and keep unchanged: markdown structure, file paths, slash commands, `@agent-name` references, shell commands, and score/band tokens (GREEN / YELLOW / RED / 🟢🟡🔴).

### Step 1 — Gather context and detect review mode

Read `./outlines/<slug>/outline.md`.

Collect the implementation diff:
```bash
# try in order until one works:
git diff main...HEAD 2>/dev/null || git diff master...HEAD 2>/dev/null || git diff HEAD~1 HEAD 2>/dev/null
```

List the modified files (excluding outline files):
```bash
git diff --name-only main...HEAD 2>/dev/null || git diff --name-only HEAD~1 HEAD 2>/dev/null | grep -v "^outlines/"
```

**Detect review mode:**
- If the user explicitly passes `--outline-only` → **Outline Review mode**
- Else if the modified-files list (excluding `outlines/**`) is empty → **Outline Review mode**
- Otherwise → **Implementation Review mode**

Announce the detected mode in chat (in `DETECTED_LANG`):
```
Detected review mode: <Outline Review | Implementation Review>
```

Then branch: Outline Review → Step 2-O → Step 3-O → final output.
Implementation Review → Step 2-I → Step 2.5-I → Step 3-I → ... → Step 7-I.

---

## Outline Review Mode (adversarial only — no /simplify, no fixer)

### Step 2-O — Spawn 2 parallel owf-outline-critic

Spawn **two `owf-outline-critic` agents in parallel** (single message, two Agent tool calls):

**Critic A** — axis_focus: clarity + decomposition (structure):
- The full content of outline.md
- Iteration: `1/1` (outline review is a one-pass confirmation — no loop here; use `/owf:outline` if iterative revision is needed)
- Instruction: `axis_focus: ["clarity", "decomposition"]` — score only these 2 axes (total 0–50)

**Critic B** — axis_focus: risk + reuse (context):
- The full content of outline.md
- Iteration: `1/1`
- Instruction: `axis_focus: ["risk", "reuse"]` — score only these 2 axes (total 0–50)

Wait for **both** critics to return partial Verdict blocks.

### Step 3-O — Merge and output

**Merge strategy** (same as `/owf:outline` Step 5):
1. Dimensions: combine all 4 axis lines.
2. Score: sum both `score:` values (0–100).
3. Findings: union, dedupe by `(section, severity-category)` keeping highest severity.
4. Routing sections: union.
5. Band: derive from merged score (GREEN ≥80, YELLOW 75–79, RED <75).

Output in chat (in `DETECTED_LANG` — canonical English skeleton):
```
========================================
OWF Outline Review: ./outlines/<slug>/outline.md
========================================
Score: <N> / 100
Band: 🟢 GREEN | 🟡 YELLOW | 🔴 RED
----------------------------------------
Rationale:
  clarity        <N>/25: <one-line>
  decomposition  <N>/25: <one-line>
  risk           <N>/25: <one-line>
  reuse          <N>/25: <one-line>
----------------------------------------
Findings:
  [HIGH] ...
  [MED] ...
  [LOW] ...
----------------------------------------
Next action:
  <if GREEN>: proceed to `/owf:implement`.
  <if YELLOW/RED>: re-run `/owf:outline` to improve the outline via the critic loop.
========================================
```

**Outline Review は単発**（ループなし、fixer なし、pr.md 生成なし）。結果を出力して終了。

If `OWF_TRACE=1`: append to `./outlines/<slug>/.owf-trace.log`:
`<ISO timestamp> | review-outline | score=<N> | band=<BAND> | <slug>`

---

## Implementation Review Mode (adversarial + simplify)

### Step 2-I — Set up adversarial Agent Team

The team consists of:

1. **owf-reviewer** (×2, parallel) — READ-ONLY adversarial critics. Spawned in parallel with different `axis_focus` assignments: Reviewer A covers `fidelity` + `tests`, Reviewer B covers `simplify` + `maintain`.
2. **owf-fixer** — implementation fixer (tools: Read, Write, Edit, Bash, Grep, Glob; outline.md excluded from write scope)

Initialize: `iter = 0`, `max_iter = 3` (or `OWF_MAX_ITER_REVIEW` env var if set).

Read the content of all modified source files.

### Step 2.5-I — Pre-review simplify pass (first iteration only)

Before spawning reviewers, invoke the Claude Code `simplify` skill once to eliminate low-hanging mechanical issues:

```
Invoke Skill: simplify
Scope instruction to pass: "Review and fix ONLY the following files. Do NOT touch outline.md or any file under ./outlines/. Files: <list from git diff --name-only>"
```

- Run this step **only on the first iteration** (`iter == 0`). Skip on subsequent iterations — owf-fixer handles cleanup from iteration 2 onward.
- If `/simplify` fails or is unavailable: log a warning and continue. Do NOT abort the review.
- After `/simplify` completes, re-read the modified files to get their updated content for the reviewer prompts.

### Step 3-I — Reviewer turn (2 parallel spawns)

Spawn **two `owf-reviewer` agents in parallel** (single message, two Agent tool calls):

**Reviewer A** — axis_focus: fidelity + tests:
- Full content of outline.md (labeled as "GROUND TRUTH — do not change this")
- Full git diff of implementation
- Content of all modified files (post-simplify)
- Current iteration number
- Instruction: `axis_focus: ["fidelity", "tests"]` — score only these 2 axes (0–50 each, total 0–50)

**Reviewer B** — axis_focus: simplify + maintain:
- Full content of outline.md (labeled as "GROUND TRUTH — do not change this")
- Full git diff of implementation
- Content of all modified files (post-simplify)
- Current iteration number
- Instruction: `axis_focus: ["simplify", "maintain"]` — score only these 2 axes (0–50 each, total 0–50)

Wait for **both** owf-reviewer agents to return their partial Verdict blocks before proceeding.

### Step 4-I — Merge partial Verdicts, then parse and branch

**Merge the two partial Verdicts from Reviewer A and B into one combined Verdict:**

1. **Dimensions**: combine all 4 axis lines (A's fidelity + tests, B's simplify + maintain).
2. **Score**: sum both `score:` values (A's 0–50 + B's 0–50 = merged 0–100).
3. **Findings**: union of both Findings sections. Dedupe by `(file, line, severity-category)` — on collision keep the higher severity.
4. **Routing files**: union of both `files:` lists.
5. **Band**: derive from merged score using thresholds (GREEN ≥80, YELLOW 75–79, RED <75).

If either partial Verdict is missing or malformed: treat that reviewer's contribution as score=0 for its 2 axes, and continue with the available findings. If both fail: treat merged score=0, band=RED.

Parse merged `score:` and derived `band:`. Proceed to branching.

**score == 100 (perfect):**
→ Proceed to Step 6-I (generate pr.md and final output).

**score ≥ 80 (GREEN):**
→ Record findings for pr.md `## Remaining Risks` section.
→ Proceed to Step 6-I.

**score ≥ 75 (YELLOW) — stop loop:**
→ Proceed to Step 6-I with YELLOW status.

**score < 75 (RED) AND iter < max_iter:**
→ Increment `iter`.
→ Proceed to Step 5-I (fixer turn).
→ After fixer completes, return to Step 3-I.

**score < 75 (RED) AND iter == max_iter:**
→ Record final findings for pr.md `## Next Action Proposals`.
→ Proceed to Step 6-I with RED status.

### Step 5-I — Fixer turn

Spawn `owf-fixer` agent with:
- Full Findings section from the merged Verdict
- List of flagged files from the Routing section (union)
- Reminder: "outline.md is ground truth — do not modify it"

Wait for owf-fixer to report completion. It will run tests internally and confirm they pass.

Return to Step 3-I for next reviewer turn.

### Step 6-I — Generate pr.md

Create `./outlines/<slug>/pr.md` using the following structure (adapt to actual content):

Fill in the template at `owf/templates/pr.md.tmpl` with:
- **Summary**: from outline.md `## Goal / Acceptance Criteria`
- **Changes**: list from git diff (grouped by feature)
- **What was NOT done**: from outline.md `## Out of Scope`
- **Questions for Reviewer**: from outline.md `## Questions for Reviewer` + any unresolved [MED] findings
- **Mermaid diagram**: if outline.md had one, adapt it; otherwise omit
- **Self-checklist results**: actual pass/fail status of each item based on review findings
- **Review temperature**: based on final score (< 80: High, 80-95: Middle, 100: Low)

If score < 100: append `## Remaining Risks` or `## Next Action Proposals` (from Step 4-I).

### Step 7-I — Final CLI output

Output in chat (in `DETECTED_LANG` — canonical English skeleton):

```
========================================
OWF Implementation Review: ./outlines/<slug>/outline.md
========================================
Score: <N> / 100
Band: 🟢 GREEN | 🟡 YELLOW | 🔴 RED
Iteration: <N> / <MAX>
----------------------------------------
Rationale:
  fidelity   <N>/25: <one-line>
  tests      <N>/25: <one-line>
  simplify   <N>/25: <one-line>
  maintain   <N>/25: <one-line>
----------------------------------------
<if GREEN or YELLOW:>
Next action:
  1. Review pr.md: ./outlines/<slug>/pr.md
  2. After committing, open the PR: gh pr create --body-file ./outlines/<slug>/pr.md
<if any LOW findings remain:>
  3. Optional improvements: <brief list of LOW findings>

<if RED:>
Next action:
  - Unresolved findings are recorded in pr.md's `## Next Action Proposals`.
  - Split the outline or apply manual fixes, then re-run `/owf:review`.
========================================
```

## Notes

- **Mode separation is strict**: Outline Review never runs `/simplify` or the fixer loop. Implementation Review always runs `/simplify` first and uses the fixer loop for RED iterations.
- The reviewer (owf-reviewer) and critic (owf-outline-critic) are hardcoded READ-ONLY via their `tools:` frontmatter — no hook needed.
- The fixer (owf-fixer) excludes outline.md from all writes via instruction — enforce via prompt.
- Two agents run in parallel per review turn in both modes (critic A/B for outline, reviewer A/B for implementation). The orchestrator merges their partial Verdicts before branching.
- The `/simplify` skill runs once at Step 2.5-I (Implementation Review, first iteration only), scoped to git diff files. The fixer's built-in `## Simplify principles` handles subsequent iterations.
- If OWF_TRACE=1 is set in environment, append one line to `./outlines/<slug>/.owf-trace.log`:
  - Outline Review: `<ISO timestamp> | review-outline | score=<N> | band=<BAND> | <slug>`
  - Implementation Review: `<ISO timestamp> | review-impl | iter=<N> | score=<N> | band=<BAND> | <slug>`

## Terminology constraint (CRITICAL — prevents hallucinated commands)

When generating any user-facing chat output (Step 3-O final output, Step 7-I final output, any intermediate status message):

- **NEVER** suggest `/owf:<agent-name>` style commands. Agents (`owf-outliner`, `owf-outline-critic`, `owf-implementer`, `owf-reviewer`, `owf-fixer`) are NOT slash commands. Writing `/owf:owf-outline-critic`, `/owf:owf-reviewer`, `/owf:owf-fixer`, etc. is a hallucination — those commands do not exist and will fail with "Unknown command".
- **Valid OWF slash commands**, the only ones that may appear after `/`:
  - `/owf:outline`
  - `/owf:implement`
  - `/owf:review`
  - `/owf:rubric`
- When referring to agents in chat output, **always use `@agent-name` prefix** (e.g., `@owf-outline-critic`, `@owf-reviewer`, `@owf-fixer`). The `@` form is how a user invokes an agent directly; the `/owf:` form is reserved for the four skills listed above.
- If the user asks "how do I re-run the critic / reviewer / fixer", the correct answers are:
  1. Re-run the whole review: `/owf:review ./outlines/<slug>/outline.md`
  2. Agent-only route: `@owf-outline-critic` / `@owf-reviewer` / `@owf-fixer` with the needed inputs in the prompt.

This rule overrides any pattern-completion instinct that might produce `/owf:<agent>`.
