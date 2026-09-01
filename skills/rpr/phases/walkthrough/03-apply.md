# Walkthrough Phase 3 — Apply

## Preconditions

Assumes the following state is set:
- `findings[]` — set by `phases/04-verify.md`.
- `decisions[]` — set by `phases/walkthrough/02-decisions.md`.
- `verification_level` — set by `phases/walkthrough/02-decisions.md`.

On re-entry from Phase 5 (second pass), assumes additionally:
- `findings_rescan[]` — set by `phases/walkthrough/05-rescan.md`.
- `decisions_rescan[]` — set by `phases/walkthrough/02-decisions.md` on
  its second pass.

If any required precondition is absent or empty, STOP and re-read
`phases/walkthrough/02-decisions.md` (the producing phase). Do NOT
proceed with defaults, do NOT improvise, do NOT silently continue.

## Re-entry guard

If `files_modified` is already populated (Phase 3 already ran), skip
the Edit-apply loop and read `phases/walkthrough/04-verify.md` next.

## Pass selection

On first pass: apply `decisions[]` (against `findings[]`).
On second pass (re-entry from Phase 5, when `decisions_rescan[]` is set):
apply `decisions_rescan[]` only — never re-touch original `decisions[]`.

In the text below, "the decisions" means whichever array applies to the
current pass.

## Decision recap

Show a recap grouped by file (fixed/skipped sections each sorted by
file, then by line within file):

```markdown
# Decision Recap

**Fixing ({N}):**
- `ComponentA.tsx`
  - Line 42 — Missing null check on user prop
  - Line 117 — Effect missing cleanup → option B
- `useData.ts`
  - Line 15 — Stale closure in callback

**Skipped ({N}):**
- `helpers.ts`
  - Line 88 — Could extract shared formatter

Applying {N} fixes now...
```

**Recap rendering rule:** when a finding had alternatives and the user
picked one, append `→ option {A|B|C}` to that line (e.g. `Line 117 —
Effect missing cleanup → option B`). Lines for single-fix findings are
unchanged.

## Apply

Apply all `fix`-marked findings using the Edit tool. **For multi-fix
findings, apply the snippet that matches `chosen_option`** (`"primary"`
→ `fix_snippet`; `"A"` is identical to `"primary"`; `"B"` →
`fix_alternatives[0].snippet`; `"C"` → `fix_alternatives[1].snippet`).

Show a one-line confirmation for each, including the chosen option when
there were alternatives:

```
Applied: ComponentA.tsx:42 — Missing null check on user prop
Applied: ComponentA.tsx:117 — Effect missing cleanup (option B — useReducer extraction)
Applied: useData.ts:15 — Stale closure in callback
```

For non-must-fix findings, or when `verification_level` is `"none"` or
`"regression"`: apply via Edit as above (no test step).

## TDD branch (when `verification_level in ("tdd", "both")`)

For each **must-fix** finding marked for fix, run this 7-step TDD loop
**before** the plain apply above:

1. **Identify the test file.** Co-located `*.test.tsx` / `*.test.ts`
   next to the source file, or `__tests__/` directory — match the
   project's existing test convention. If a test file does not yet
   exist for this source file but a test runner is present, create the
   test file using the project's naming convention.

2. **Write a test that targets the specific bug described in the
   finding.** Use the project's existing test framework and helpers
   (e.g. `describe` / `it` / `expect`, React Testing Library renders,
   Vitest, Jest — whichever is in use).

3. **Run the test.** Confirm it **fails** (proving the issue is real).

4. **Verify the failure mode.** If the test was expected to fail and
   does, continue.

5. **False-positive prompt.** If the test **passes** before the fix:
   flag as potential false positive. Use a single `AskUserQuestion`
   call:
   - `question`: `"Test passes without the fix at {file}:{line} — this finding may not be a real bug. Apply anyway?"`
   - `header`: `"Apply fix?"` (max 12 chars)
   - `multiSelect`: `false`
   - `options`:
     1. `{ label: "Apply anyway", description: "Apply the fix despite the test passing — author judges it's still worth it." }`
     2. `{ label: "Skip this fix", description: "Drop this finding — likely false positive." }`

   `Apply anyway` → record this finding's index in
   `verification_results.tdd_false_positives` and continue to step 6.
   `Skip this fix` → record as `skip` in this pass's decisions, append
   this finding's index to `verification_results.tdd_false_positives`,
   and move to the next finding.

6. **Apply the fix via Edit tool** (same as the plain apply above —
   `chosen_option` rules apply).

7. **Run the test again.** Confirm it **passes**. Append this finding's
   index to `verification_results.tdd_validations_passed`. Report:

   ```
   TDD validated: {file}:{line} — {issue title}
   ```

## Fallback when no test runner is detectable

When no test runner is detectable (no `test` script in `package.json`,
no `jest.config.*`, no `vitest.config.*`), skip the TDD loop for that
finding entirely:

- Set the finding's `tdd_validations_passed` contribution to `null`
  (do NOT add to the array).
- Note `"no test runner detected"` in `verification_results.notes`
  (append, comma-separated, once per missing-runner finding is fine).
- Do NOT install or configure a test framework.
- Fall through to the plain apply path (Edit only, no test step).

## State writes

Phase 3 writes / mutates:
- `files_modified` — append each successfully-edited file path. On a
  later revert (Phase 4), the reverting phase removes the entry.
- `verification_results.tdd_validations_passed` — indices of findings
  where the TDD test failed before the fix and passed after.
- `verification_results.tdd_false_positives` — indices of findings where
  the TDD test passed before the fix (whether the user chose Apply
  anyway or Skip this fix).
- `verification_results.notes` — `"no test runner detected"` entries
  when the TDD fallback kicked in.

All other slots in `verification_results` are written by Phase 4.

## Next: read `phases/walkthrough/04-verify.md`.
