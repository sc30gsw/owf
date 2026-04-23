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

### Step 0 — Detect output language

Determine `DETECTED_LANG` by evaluating the following signals in priority order (first confident match wins):

**Priority 1 — Explicit override**
If `OWF_LANG` environment variable is set (e.g., `ja`, `en`, `zh`, `ko`, `de`, `fr`, `es`), use it verbatim and skip all further detection.

**Priority 2 — User conversation language**
Inspect the user's most recent prompt / the active conversation turn. If the user is clearly writing in a specific natural language, use that as `DETECTED_LANG`. This is the strongest real-time signal — someone chatting in Korean wants Korean output even if the project's README is in English. Treat mixed-language prompts where technical terms / code are in English as still indicating the non-English language.

**Priority 3 — Project signals (aggregated)**
Sample multiple project artefacts and identify the dominant natural language across all of them:
```bash
{
  head -30 README.md 2>/dev/null
  head -30 readme.md 2>/dev/null
  head -30 CLAUDE.md 2>/dev/null
  cat .github/ISSUE_TEMPLATE/*.md 2>/dev/null | head -30
  git log --oneline -50 2>/dev/null
} 2>/dev/null | head -200
```

Identify the dominant language using these signals (supported codes):

| Code | Language | Detection hint |
|---|---|---|
| `ja` | Japanese | Hiragana/Katakana (U+3040–U+30FF) present |
| `zh` | Chinese | CJK ideographs (U+4E00–U+9FFF) dominant, no kana |
| `ko` | Korean | Hangul (U+AC00–U+D7AF) present |
| `de` | German | Frequent `ä ö ü ß` and common words (`der die das und nicht ist`) |
| `fr` | French | Frequent `à â ç é è ê ë î ï ô ù û` and common words (`le la les de des un une est`) |
| `es` | Spanish | Frequent `ñ ¿ ¡ á é í ó ú` and common words (`el la los las de un una es`) |
| `en` | English | Predominantly ASCII with no dominant marker above |

Pick the language with the strongest signal across the aggregated sample. Ignore isolated tokens (e.g., a single Japanese word in an otherwise English README does not make it `ja`).

**Priority 4 — Fallback**
If no confident signal: default to `en`.

Store the result as `DETECTED_LANG` and pass it to every agent invocation in subsequent steps. **All "Output in chat" skeletons below show English labels as the canonical reference** — translate labels and narrative into `DETECTED_LANG`, and keep unchanged: markdown structure, file paths, slash commands, `@agent-name` references, shell commands, and score/band tokens (GREEN / YELLOW / RED / 🟢🟡🔴).

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

### Step 4 — Run owf-outline-critic (2 parallel critics)

Read the current `./outlines/<slug>/outline.md` content.

Spawn **two `owf-outline-critic` agents in parallel** (single message, two Agent tool calls):

**Critic A** — axis_focus: clarity + decomposition (structure):
- The full content of outline.md
- Current iteration number (start at 1)
- Instruction: `axis_focus: ["clarity", "decomposition"]` — score only these 2 axes (total 0–50)

**Critic B** — axis_focus: risk + reuse (context):
- The full content of outline.md
- Current iteration number (start at 1)
- Instruction: `axis_focus: ["risk", "reuse"]` — score only these 2 axes (total 0–50)

Wait for **both** critics to return their partial Verdict blocks before proceeding.

### Step 5 — Merge partial Verdicts, then parse and branch

**Merge the two partial Verdicts from Critic A and B into one combined Verdict:**

1. **Dimensions**: combine all 4 axis lines (A's clarity + decomposition, B's risk + reuse).
2. **Score**: sum both `score:` values (A's 0–50 + B's 0–50 = merged 0–100).
3. **Findings**: union of both Findings sections. Dedupe by `(section, severity-category)` — on collision keep the higher severity.
4. **Routing sections**: union of both `sections:` lists.
5. **Band**: derive from merged score using thresholds (GREEN ≥80, YELLOW 75–79, RED <75).

If either partial Verdict is missing or malformed: treat that critic's contribution as score=0 for its 2 axes, and continue with the available findings. If both fail: treat merged score=0, band=RED.

Parse merged `score:` and derived `band:`. Proceed to branching.

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
→ Output in chat (in `DETECTED_LANG` — canonical English):
```
⚠️ Outline score: <N>/100 🟡 YELLOW (iteration <N>/3)
Review the outline; if acceptable, run `/owf:implement ./outlines/<slug>/outline.md`.
Reason for YELLOW: <list MED/HIGH findings>
Details: ./outlines/<slug>/outline.md, ./outlines/<slug>/outline-pr.md
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
→ Output in chat (in `DETECTED_LANG` — canonical English):
```
❌ Outline score: <N>/100 🔴 RED (3/3 iterations reached)
Maximum iterations reached. Improvement suggestions have been appended to outline.md.
Review them, then either split the task or update the outline per the suggestions.
Details: ./outlines/<slug>/outline.md
```
→ Stop.

### Step 6 — (loop back to Step 4 if RED and iter < 3)

### Step 7 — Final output (GREEN or 100)

Output in chat (in `DETECTED_LANG` — canonical English):
```
✅ Outline ready: ./outlines/<slug>/outline.md
Score: <N>/100 🟢 GREEN (<N> iterations)

<if score < 100:>
Remaining risks / unresolved points are listed at the end of outline.md under "## Remaining Risks".

Next steps:
- Review outline.md
- If acceptable: `/owf:implement ./outlines/<slug>/outline.md`
- Direct agent route: invoke `@owf-implementer` with the outline.md path
```

## Iteration cap

Default: 3 iterations. Override with `OWF_MAX_ITER_OUTLINE` environment variable.

## Terminology constraint (CRITICAL — prevents hallucinated commands)

When generating any user-facing chat output (final output in Step 7, YELLOW stop message in Step 5, RED max-iteration message in Step 5):

- **NEVER** suggest `/owf:<agent-name>` style commands. Agents (`owf-outliner`, `owf-outline-critic`, `owf-implementer`, `owf-reviewer`, `owf-fixer`) are NOT slash commands. Writing `/owf:owf-outline-critic` or `/owf:owf-outliner` is a hallucination — those commands do not exist and will fail with "Unknown command".
- **Valid OWF slash commands**, the only ones that may appear after `/`:
  - `/owf:outline`
  - `/owf:implement`
  - `/owf:review`
  - `/owf:rubric`
- When referring to agents in chat output, **always use `@agent-name` prefix** (e.g., `@owf-outline-critic`, `@owf-implementer`). The `@` form is how a user invokes an agent directly; the `/owf:` form is reserved for the four skills listed above.
- If the user asks "how do I re-run the critic", the correct answer is either:
  1. `/owf:review ./outlines/<slug>/outline.md` (skill route — auto-selects Outline Review mode when no implementation diff exists), OR
  2. `@owf-outline-critic` with `axis_focus` and outline path in the prompt (direct agent route).

This rule overrides any pattern-completion instinct that might produce `/owf:<agent>`.
