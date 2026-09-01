# Walkthrough Phase 4 — Verify

## Preconditions

Assumes the following state is set:
- `findings[]` — set by `phases/04-verify.md` (or, on re-entry, `findings_rescan[]` from `phases/walkthrough/05-rescan.md`).
- `decisions[]` — set by `phases/walkthrough/02-decisions.md` (or `decisions_rescan[]` on re-entry).
- `verification_level` — set by `phases/walkthrough/02-decisions.md`.
- `files_modified` — set by `phases/walkthrough/03-apply.md`.
- `verification_results` — TDD slots (`tdd_validations_passed`,
  `tdd_false_positives`, `notes`) populated by Phase 3 when
  `verification_level` is `"tdd"` or `"both"`. Empty / partial otherwise.

If any required precondition is absent or empty, STOP and re-read
`phases/walkthrough/03-apply.md` (the producing phase). Do NOT proceed
with defaults, do NOT improvise, do NOT silently continue.

## Re-entry guard

If `verification_results` is already populated with all four core slots
(`ts_errors`, `lint_warnings`, `test_failures`, `regression_tests_written`),
skip the verification commands and read
`phases/walkthrough/05-rescan.md` next.

## Level 1 — Standard Checks (always runs)

Detect available tools from `package.json` scripts and project config:

1. **TypeScript:** `npx tsc --noEmit` (or the project's type-check script if defined). Report new errors only.
2. **Linter:** `npx eslint --no-warn-ignored {changed files}` (or the project's lint script). Report new warnings/errors only.
3. **Test suite:** Project's test command (`npm test`, `npx jest --bail`, etc.). Report failures only.

If a tool isn't configured in the project (no tsconfig, no eslint
config, no test script), skip it silently — don't install or configure
anything.

Write the results into `verification_results`:
- `ts_errors` — count of new TypeScript errors (or `null` if not run).
- `lint_warnings` — count of new lint warnings/errors (or `null` if not run).
- `test_failures` — count of test failures (or `null` if not run).

If any check fails:
- Show the failure and identify which applied fix likely caused it.
- Use a single `AskUserQuestion` call:
  - `question`: `"Check failure looks related to the fix at {file}:{line}. What should I do?"`
  - `header`: `"Revert?"` (max 12 chars)
  - `multiSelect`: `false`
  - `options`:
    1. `{ label: "Revert the fix", description: "Undo the Edit and mark this finding as reverted in the summary." }`
    2. `{ label: "Keep as-is", description: "Leave the fix in place; note the failure in the summary for the author." }`
    3. `{ label: "Adjust", description: "Discuss the failure and re-apply a revised fix." }`
- `Revert the fix` → revert via Edit, remove the entry from
  `files_modified`, update the summary.
- `Keep as-is` → note in the summary (write to `verification_results.notes`).
- `Adjust` → discuss with the user, then re-apply a revised fix
  (keep `files_modified` as-is; the file is still modified).

## Level 2 — Regression Tests (when `verification_level is "regression" or "both"`)

For each must-fix finding that was applied and NOT already covered by
TDD in Phase 3 (i.e. not in `verification_results.tdd_validations_passed`):

1. Identify the test file (same convention detection as TDD — co-located or `__tests__/`).
2. Write a test that would catch the original bug if it regressed.
3. Run the test to confirm it passes.
4. Increment `verification_results.regression_tests_written`.
5. Report: `Regression test added: {test file} — {test description}`.

When `verification_level = "both"`, also write regression tests for
applied **should-fix** findings.

If no test runner is detectable, skip Level 2 entirely and append `"no
test runner detected"` to `verification_results.notes` (only once per
verify phase, not per-finding).

## Level 3 — TDD Validation

TDD ran inline in Phase 3 — report `tdd_validations_passed` and
`tdd_false_positives` from `verification_results`, do not re-execute.

## Verification Summary

Output after all checks complete:

```markdown
## Verification Results
**TypeScript:** {pass or N new errors}
**Linter:** {pass or N new warnings/errors}
**Tests:** {all passing or N failures}
**Regression tests written:** {N} (or: skipped)
**TDD validations:** {N} passed, {N} false-positives flagged (or: skipped)
```

## Next: read `phases/walkthrough/05-rescan.md`.
