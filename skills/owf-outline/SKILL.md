---
name: owf-outline
description: Phase 1 — Create a development outline from a task description. Runs an adversarial critic loop (max 3 iterations) and produces ./outlines/<slug>/outline.md. Use when starting any development task. Invoke as `/owf:outline <task description>`.
---

# OWF Outline — Phase 1 Orchestrator

## Usage

```
/owf:outline <task description>
```

Example:
```
/owf:outline Add a useDebounce hook to the search input in the user list feature
```

## Execution steps

Execute the following steps in order. Do not skip steps.

### Step 0 — Detect project language

Determine the language to use for all outline.md text (section headers, content, placeholder comments):

1. Check environment variable `OWF_LANG` (e.g., `ja`, `en`, `zh`). If set, use it directly.
2. Otherwise, sample the project README:
   ```bash
   head -30 README.md 2>/dev/null || head -30 readme.md 2>/dev/null || echo ""
   ```
   If the output contains Japanese characters (hiragana/katakana/kanji — Unicode range \\u3040–\\u9FFF), use `ja`.
   If it contains predominantly CJK characters of another type, use `zh`.
   Otherwise, use `en`.
3. If no README exists: default to `en`.

Store the result as `DETECTED_LANG` and pass it to every agent invocation in subsequent steps.

### Step 1 — Generate feature slug

Convert the task description to a kebab-case slug (max 5 words, lowercase):
- "Add useDebounce hook to search" → `add-use-debounce-search`
- "Fix login redirect on token expiry" → `fix-login-redirect-token-expiry`

Check that `./outlines/<slug>/` does not already exist. If it does, append a short disambiguator (e.g., `-2`).

### Step 2 — Scaffold the outline file

Create the directory and outline.md:

```bash
mkdir -p ./outlines/<slug>
```

Create `./outlines/<slug>/outline.md` with the skeleton below (fill in `<TITLE>` and `<SLUG>`). Section headers and placeholder text use English here, but owf-outliner will rewrite them in `DETECTED_LANG` when filling content.

```markdown
# Outline: <TITLE>

<!-- slug: <SLUG> | created: <ISO timestamp> -->

## Context
<!-- Background and motivation. Why is this task needed? -->

## Goal / Acceptance Criteria
<!-- What "done" looks like. Each item must have a clear pass/fail definition. -->
- [ ] 

## Out of Scope
<!-- Explicitly list what this change does NOT include. -->
- 

## Change Surface
<!-- Expected files / feature directories to change. -->
```
src/
├── features/
│   └── <feature-name>/
│       ├── api/
│       ├── components/
│       ├── hooks/
│       └── __tests__/
```

## Steps (TDD: RED → GREEN → REFACTOR)
<!-- Each step should be independently testable. Order matters. -->

1. **Write failing test(s) for** ...
   - Test: ...
2. **Implement** ...
3. **Refactor**: ...

## Existing Code to Reuse
<!-- Specific file paths found in the codebase. No NIH. -->
- 

## Risks / Assumptions / Rollback
<!-- All assumptions made. Edge cases. What to do if this breaks. -->
- Assumption: ...
- Risk: ...
- Rollback: ...

## Verification
<!-- Concrete commands + manual steps to confirm completion. -->
- [ ] `npm test` — all pass
- [ ] `tsc --noEmit` — clean
- [ ] Manual: ...

## Questions for Reviewer
<!-- Design decisions, trade-offs, or specific concerns for code review. -->
- 
```

### Step 3 — Run owf-outliner (initial creation)

Spawn the `owf-outliner` agent with:
- The original task description
- The path to the newly created outline.md
- `lang=<DETECTED_LANG>` (from Step 0)
- Instruction: "Fill in outline.md completely in the specified language. Research the codebase for existing code to reuse before writing."

Wait for owf-outliner to complete. It will have written the filled outline.md.

### Step 4 — Run owf-outline-critic (review)

Read the current `./outlines/<slug>/outline.md` content.

Spawn the `owf-outline-critic` agent with:
- The full content of outline.md
- Current iteration number (start at 1)

Wait for owf-outline-critic to return. Its entire response is a Verdict block.

### Step 5 — Parse verdict and branch

Parse the Verdict block from the critic's response using these exact markers:
- `score:` followed by an integer
- `band:` followed by `RED`, `YELLOW`, or `GREEN`

If parsing fails: treat score=0, band=RED.

**Branch by band:**

**score == 100 (perfect):**
→ Proceed to Step 7 (final output).

**score ≥ 80 (GREEN):**
→ Append the following to `./outlines/<slug>/outline.md`:
```markdown

## Remaining Risks / Unresolved Points
<!-- Added by owf-outline-critic (score: <N>/100, <BAND>) -->
<paste the Findings section here>
```
→ Proceed to Step 7 (final output).

**score ≥ 75 (YELLOW) — stop loop:**
→ Create `./outlines/<slug>/outline-pr.md` using the template at `owf/templates/outline-pr.md.tmpl` as structure.
→ Pre-fill "Questions for Reviewer" with the critic's [HIGH] and [MED] findings.
→ Output in chat (Japanese):
```
⚠️ Outline score: <N>/100 🟡 YELLOW (iteration <N>/3)
アウトラインのレビューを確認し、OK であれば `/owf:implement ./outlines/<slug>/outline.md` を実行してください。
Yellow の原因: <list MED/HIGH findings in Japanese>
詳細: ./outlines/<slug>/outline.md, ./outlines/<slug>/outline-pr.md
```
→ Stop.

**score < 75 (RED) AND iteration < 3:**
→ Increment iteration counter.
→ Spawn `owf-outliner` again in "Mode B: Revision" with:
  - The full Findings section from the critic's Verdict
  - The Routing section (which sections to revise)
  - Current outline.md content
  - `lang=<DETECTED_LANG>`
→ Wait for owf-outliner to complete revisions.
→ Return to Step 4.

**score < 75 (RED) AND iteration == 3 (max reached):**
→ Append to `./outlines/<slug>/outline.md`:
```markdown

## Score Improvement Suggestions
<!-- owf-outline-critic reached max iterations (score: <N>/100 after 3 rounds) -->
The following suggestions may help resolve remaining issues:

<for each unresolved HIGH finding, suggest one of:>
- **Split the task**: This outline covers multiple feature domains. Consider splitting into separate outlines: 1) [scope A], 2) [scope B].
- **Reference existing implementation**: Similar pattern already exists at [file path] — use it as a template.
- **Clarify requirement**: "[ambiguous criteria]" needs a concrete definition. Discuss with stakeholders before implementing.
```
→ Output in chat (Japanese):
```
❌ Outline score: <N>/100 🔴 RED (3/3 イテレーション完了)
最大イテレーション数に達しました。改善提案を outline.md に追記しました。
確認後、タスクを分割するか、提案に従って outline を更新してください。
詳細: ./outlines/<slug>/outline.md
```
→ Stop.

### Step 6 — (loop back to Step 4 if RED and iter < 3)

### Step 7 — Final output (GREEN or 100)

Output in chat (Japanese):
```
✅ Outline 完成: ./outlines/<slug>/outline.md
スコア: <N>/100 🟢 GREEN (<N> イテレーション)

<if score < 100:>
残リスク・未充足点は outline.md の末尾 "## Remaining Risks" を確認してください。

次のステップ:
- outline.md を確認してください
- 問題なければ: `/owf:implement ./outlines/<slug>/outline.md`
- エージェントに直接依頼する場合: `@owf-implementer` に outline.md のパスを渡してください
```

## Iteration cap

Default: 3 iterations. Override with `OWF_MAX_ITER_OUTLINE` environment variable.
