# Plan-Fixes Walkthrough (Step 6b)

When `output_mode = "plan-fixes"`, replace Step 6 entirely with this interactive flow.

**Reuses from review-comments:** the shared header, the "Findings at a glance" table, and the unified finding block (five labeled sections) from `references/output-template.md`.

**Skips:** VOICE.md, voice comments (the **Comment (in my voice)** block), overall-review summary prose, positive callouts.

## Phase 1 — Summary

First, output a concise overview of ALL findings so the user sees the full picture before making any decisions. Use the shared header + "Findings at a glance" table shape (matches the review-comments rendering):

```markdown
# {Review title — "Local Code Review" or "Review Complete: {PR title}"}
**{Branch or PR link}** | **Files changed:** {N} | **Lines:** +{adds} / -{dels}
**Findings:** {N total} ({must-fix} must-fix, {should-fix} should-fix, {suggestion} suggestions)

## Findings at a glance

| # | File | Line | Severity | Issue | Effort |
|---|------|------|----------|-------|--------|
| 1 | `ComponentA.tsx` | 42 | must-fix | Missing null check on user prop | trivial |
| 2 | `ComponentA.tsx` | 117 | should-fix | Effect missing cleanup | small |
| 3 | `useData.ts` | 15 | should-fix | Stale closure in callback | small |

I'll walk through each finding for your decision, then apply all fixes at the end. You'll get a button picker per finding — usually Fix / Skip / Discuss / Fix all remaining, and when a finding has multiple suggested fixes you'll see Fix A / Fix B (/ Fix C) instead. The **Other** field handles anything else, like typing `stop` to jump straight to the apply phase.
```

No voice prose, no positive callout — Phase 1 is just the header and the table.

## Task Tracking for Walkthrough

Before starting Phase 2, create one task per finding using `TaskCreate` so the user can see progress at a glance. **Order tasks file-by-file, line-sorted within each file** (matches Phase 2 walkthrough order). Numbering stays global so the progress indicator works across the whole session:

```
Example for 4 findings across 3 files (file-by-file, line-sorted):
  "Finding 1/4 in ComponentA.tsx: Missing null check on user prop (must-fix)"      activeForm: "Reviewing finding 1/4"
  "Finding 2/4 in ComponentA.tsx: Effect cleanup missing (should-fix)"              activeForm: "Reviewing finding 2/4"
  "Finding 3/4 in useData.ts: Stale closure in callback (should-fix)"               activeForm: "Reviewing finding 3/4"
  "Finding 4/4 in helpers.ts: Could extract shared formatter (suggestion)"          activeForm: "Reviewing finding 4/4"
```

Mark each task `in_progress` when presenting it to the user, and `completed` when the user makes a decision (fix, skip, or discuss → resolved). During Phase 3 (Apply), update the task name to include the decision — e.g. append `→ fixed` or `→ skipped`.

## Phase 2 — Decisions

Walk through findings **one at a time**, grouped **file-by-file and sorted by line number within each file** (matches how the author will scan the code and matches the review-comments rendering). **Every per-finding decision MUST be collected via the `AskUserQuestion` tool — never via free-form text prompts.** This is plan-fixes mode: the user is reviewing their own code, and the button picker is how they want to drive the walkthrough. **Do NOT apply any edits during this phase** — only collect decisions.

For each finding, first emit the unified finding block from `references/output-template.md` (four labeled sections — Severity, Description, Diff, Effort — no Comment) as plain output so the user has the full context to read. Use **single-fix shape** when the verifier emitted no `fix_alternatives` and **multi-fix shape** (Suggested fix A / B / C — each its own diff block) when it did. See `references/output-template.md` for both layouts.

Then immediately follow with a single `AskUserQuestion` call. The option set is **shape-dependent** — exactly one of the three layouts below, picked from the count of available fix options (`primary + fix_alternatives`, dropping any whose snippet is null):

**Layout 1 — single fix (1 option):** the default. Used when there's only one concrete fix snippet.
- `question`: `"Decision for finding {n}/{total} — {file}:{line}?"`
- `header`: `"Decision"` (max 12 chars)
- `multiSelect`: `false`
- `options` (exactly four, in this order):
  1. `{ label: "Fix", description: "Apply the suggested fix during Phase 3." }` — prepend `"(Recommended) "` to the label when severity is `must-fix`.
  2. `{ label: "Skip", description: "Leave the code unchanged." }`
  3. `{ label: "Discuss", description: "Talk it through before deciding — no edits yet." }`
  4. `{ label: "Fix all remaining", description: "Mark this and every later finding as Fix and jump to Phase 3." }`

**Layout 2 — two fix alternatives:** used when there are exactly two concrete fix snippets (primary + 1 alternative). `Fix all remaining` moves to the auto-provided `Other` field.
- `question`: same shape, with body `"This finding has two reasonable fixes — pick one or skip/discuss. {file}:{line}"`
- `header`: `"Decision"`
- `multiSelect`: `false`
- `options`:
  1. `{ label: "Fix A — {label_A}", description: "{description_A — 1-line summary}", preview: "{snippet_A}" }`
  2. `{ label: "Fix B — {label_B}", description: "{description_B — 1-line summary}", preview: "{snippet_B}" }`
  3. `{ label: "Skip", description: "Leave the code unchanged." }`
  4. `{ label: "Discuss", description: "Talk it through before deciding — no edits yet." }`

The `preview` field on each Fix option carries the fenced code (without the surrounding markdown fences — pass the raw snippet body) so the picker UI renders the alternatives side-by-side. Truncate previews longer than ~30 lines and add a trailing `// …` comment so the panel stays readable; the full snippet still exists in the rendered finding block above.

**Layout 3 — three fix alternatives:** used when there are exactly three concrete fix snippets. Both `Fix all remaining` and `Discuss` move to `Other`.
- `options`:
  1. `{ label: "Fix A — {label_A}", description: "...", preview: "{snippet_A}" }`
  2. `{ label: "Fix B — {label_B}", description: "...", preview: "{snippet_B}" }`
  3. `{ label: "Fix C — {label_C}", description: "...", preview: "{snippet_C}" }`
  4. `{ label: "Skip", description: "Leave the code unchanged." }`

Don't try to fit more than three fix variants into a single picker — Step 5b caps `fix_alternatives` at 2 entries (so primary + alternatives ≤ 3) for exactly this reason. If the verifier somehow returned more, drop the weakest before rendering.

If every snippet on the finding is null (design-level finding with no concrete patch), use **Layout 1** unchanged — the `Fix` button there means "I'll work this fix into the discussion-led implementation" rather than applying a snippet. The Diff block in the rendered finding above will already be omitted per the Diff block rule.

Do NOT add a fifth option in any layout — the auto-provided **Other** path handles anything else (commonly `"stop"`, plus `Fix all remaining` / `Discuss` overflow in Layouts 2 and 3).

### Handle Responses (Phase 2)

Map the `AskUserQuestion` answer to a decision:

- **`Fix`** (Layout 1) → Record `fix` with `chosen_option = "primary"`. Output `Marked for fix.` and move to next finding. Do NOT apply the edit yet.
- **`Fix A — …` / `Fix B — …` / `Fix C — …`** (Layouts 2-3) → Record `fix` with `chosen_option = "A"` (primary) / `"B"` / `"C"`. Output `Marked for fix (option {A|B|C}).` and move to next finding. Phase 3 applies that specific snippet.
- **`Skip`** → Record `skip`. Output `Skipped.` and move to next finding.
- **`Discuss`** (Layout 1 or 2) → Engage in conversation about the finding. The user may ask questions, propose new fix variants, or merge two of the alternatives. Once resolved, re-issue the same `AskUserQuestion` call — with whatever option set now reflects the post-discussion fix list (the verifier's output is mutable here; if discussion produced a third alternative, switch to Layout 3 on the re-ask). Still do NOT apply edits.
- **`Fix all remaining`** (Layout 1) → Mark this finding AND every remaining finding as `fix` (each with `chosen_option = "primary"` since the user didn't see a picker). Output `Marked all remaining ({N}) for fix (primary suggestion each).` and proceed directly to Phase 3.
- **`Other` (free text)** — interpret the text:
  - `"stop"` (or equivalent like `"that's enough"`, `"skip the rest"`) → Stop collecting decisions and proceed directly to Phase 3 with decisions made so far. Unreviewed findings are recorded as `skip`.
  - `"discuss"` (Layout 3 fallback) or `"fix all remaining"` (Layouts 2-3 fallback) → execute the corresponding action above.
  - `"A"` / `"B"` / `"C"` shorthand → record `fix` with that `chosen_option`.
  - anything else → treat as `Discuss` and engage in conversation, then re-issue the `AskUserQuestion` call with whatever option set the discussion produced.

### Verification Preferences

After collecting all decisions (or when the user picked `Fix all remaining` / typed `stop`), ask about verification depth before applying. Use a single `AskUserQuestion` call with exactly:

- `question`: `"How should I verify the fixes? TypeScript, linter, and the test suite run automatically. For the {N} must-fix finding(s) marked for fix, I can also:"`
- `header`: `"Verify"` (max 12 chars)
- `multiSelect`: `false`
- `options`:
  1. `{ label: "Regression tests", description: "After applying, write tests that catch each must-fix issue so it can't regress." }`
  2. `{ label: "TDD validation", description: "Before each must-fix fix, write a failing test, then confirm the fix makes it pass." }`
  3. `{ label: "Both", description: "TDD for must-fix + regression tests for should-fix findings." }`
  4. `{ label: "None", description: "Just the standard TypeScript / lint / test checks." }`

Map the answer to `verification_level`: `Regression tests` → `"regression"`, `TDD validation` → `"tdd"`, `Both` → `"both"`, `None` → `"none"`. If the user answers via `Other`, interpret the free text and pick the closest of the four; only re-ask when genuinely ambiguous.

**Skip this question** if no must-fix findings were marked for fix — default to `"none"` (Level 1 standard checks still run automatically).

## Phase 3 — Apply

After all decisions are collected (or the user says "fix all" / "stop"), show a recap grouped by file (fixed/skipped sections each sorted by file, then by line within file) and then apply:

```markdown
# Decision Recap

**Fixing ({N}):**
- `ComponentA.tsx`
  - Line 42 — Missing null check on user prop
  - Line 117 — Effect missing cleanup
- `useData.ts`
  - Line 15 — Stale closure in callback

**Skipped ({N}):**
- `helpers.ts`
  - Line 88 — Could extract shared formatter

Applying {N} fixes now...
```

Then apply all `fix`-marked findings using the Edit tool. **For multi-fix findings, apply the snippet that matches `chosen_option`** (`"primary"` → `fix_snippet`; `"A"` is identical to `"primary"`; `"B"` → `fix_alternatives[0].snippet`; `"C"` → `fix_alternatives[1].snippet`). Show a one-line confirmation for each, including the chosen option when there were alternatives:
```
Applied: ComponentA.tsx:42 — Missing null check on user prop
Applied: ComponentA.tsx:117 — Effect missing cleanup (option B — useReducer extraction)
Applied: useData.ts:15 — Stale closure in callback
```

For multi-fix Phase 3 recap rendering (the recap block above): when a finding had alternatives and the user picked one, append `→ option {A|B|C}` to that line (e.g. `Line 117 — Effect missing cleanup → option B`). Lines for single-fix findings are unchanged.

**When `verification_level` is `"tdd"` or `"both"`**, modify the apply step for each **must-fix** finding marked for fix:

1. Identify the appropriate test file (co-located `*.test.tsx` or `__tests__/` directory — match the project's existing test convention)
2. Write a test that targets the specific bug described in the finding
3. Run the test — confirm it **fails** (proving the issue is real)
4. If the test **passes** before the fix: flag as potential false positive. Use a single `AskUserQuestion` call:
   - `question`: `"Test passes without the fix at {file}:{line} — this finding may not be a real bug. Apply anyway?"`
   - `header`: `"Apply fix?"` (max 12 chars)
   - `multiSelect`: `false`
   - `options`:
     1. `{ label: "Apply anyway", description: "Apply the fix despite the test passing — author judges it's still worth it." }`
     2. `{ label: "Skip this fix", description: "Drop this finding — likely false positive." }`
   `Apply anyway` → continue to step 5; `Skip this fix` → record as `skip` and move on.
5. Apply the fix via Edit tool
6. Run the test — confirm it **passes**
7. Report: `TDD validated: {file}:{line} — {issue title}`

For non-must-fix findings, or when TDD is not active: apply via Edit as before (no test step).

## Phase 4 — Verify

Runs automatically after Phase 3. This phase only exists in plan-fixes mode — used when reviewing your own code, whether that's local changes or your own PR.

### Level 1 — Standard Checks (always runs)

Detect available tools from `package.json` scripts and project config:

1. **TypeScript:** `npx tsc --noEmit` (or the project's type-check script if defined). Report new errors only.
2. **Linter:** `npx eslint --no-warn-ignored {changed files}` (or the project's lint script). Report new warnings/errors only.
3. **Test suite:** Project's test command (`npm test`, `npx jest --bail`, etc.). Report failures only.

If a tool isn't configured in the project (no tsconfig, no eslint config, no test script), skip it silently — don't install or configure anything.

If any check fails:
- Show the failure and identify which applied fix likely caused it
- Use a single `AskUserQuestion` call:
  - `question`: `"Check failure looks related to the fix at {file}:{line}. What should I do?"`
  - `header`: `"Revert?"` (max 12 chars)
  - `multiSelect`: `false`
  - `options`:
    1. `{ label: "Revert the fix", description: "Undo the Edit and mark this finding as reverted in the summary." }`
    2. `{ label: "Keep as-is", description: "Leave the fix in place; note the failure in the summary for the author." }`
    3. `{ label: "Adjust", description: "Discuss the failure and re-apply a revised fix." }`
- `Revert the fix` → revert via Edit, update summary; `Keep as-is` → note in summary; `Adjust` → discuss with the user, then re-apply a revised fix.

### Level 2 — Regression Tests (when `verification_level` is `"regression"` or `"both"`)

For each must-fix finding that was applied and NOT already covered by TDD in Phase 3:
1. Identify the test file (same convention detection as TDD — co-located or `__tests__/`)
2. Write a test that would catch the original bug if it regressed
3. Run the test to confirm it passes
4. Report: `Regression test added: {test file} — {test description}`

When `verification_level = "both"`, also write regression tests for applied **should-fix** findings.

### Level 3 — TDD Validation

Handled inline during Phase 3's apply step (see above). By Phase 4, TDD findings are already validated. Phase 4 only reports the TDD summary.

### Verification Summary

Output after all checks complete:

```markdown
## Verification Results
**TypeScript:** {pass or N new errors}
**Linter:** {pass or N new warnings/errors}
**Tests:** {all passing or N failures}
**Regression tests written:** {N} (or: skipped)
**TDD validations:** {N} passed, {N} false-positives flagged (or: skipped)
```

## Phase 5 — Optional Re-scan (plan-fixes mode only)

After Phase 4, if any fixes were applied, use a single `AskUserQuestion` call:

- `question`: `"Fixes changed {N} files. Re-scan those files for issues introduced by the fixes?"`
- `header`: `"Re-scan?"` (max 12 chars)
- `multiSelect`: `false`
- `options`:
  1. `{ label: "Re-scan", description: "Re-run Agents 1 and 3 on just the modified files (must-fix / should-fix only — no suggestions)." }`
  2. `{ label: "Skip re-scan", description: "Go straight to the walkthrough summary." }`

**If `Re-scan`:**
1. Re-run **only Agents 1 and 3** (Hooks & Performance — the agents most likely to catch fix-induced regressions) on **only the files modified by fixes**
2. These agents use a **stricter threshold**: only report must-fix or should-fix findings. Suggestions are suppressed. Append this to their prompts:
   ```
   RESCAN MODE: You are re-scanning files that were just modified by automated fixes.
   Only report must-fix or should-fix severity findings. Do NOT report suggestions —
   this is a targeted check for issues introduced by the fixes, not a full review.
   ```
3. Send new findings through Opus verification (same Step 5 prompt)
4. If confirmed findings exist, present them using the Phase 2 → Phase 3 flow (decisions then apply). Phase 4 (Verify) runs again after any new fixes.
5. **No third pass.** After one re-scan, the loop ends regardless of results.

**If `Skip re-scan` (or an `Other` answer interpreted as no):** Proceed directly to the Walkthrough Summary.

## Walkthrough Summary

After the final phase completes, output:

```markdown
# Fix Summary
- **Fixed:** {N} findings
- **Skipped:** {N} findings
- **Verification:** {TypeScript pass/fail} | {Linter pass/fail} | {Tests pass/fail}
- **Regression tests added:** {N} (or "none")
- **TDD validations:** {N} passed (or "skipped")

**Files modified:**
{list of modified files, including new test files}
```
