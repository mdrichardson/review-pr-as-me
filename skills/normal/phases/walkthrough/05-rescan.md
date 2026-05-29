# Walkthrough Phase 5 — Optional Re-scan

## Preconditions

Assumes the following state is set:
- `findings[]` — set by `phases/04-verify.md`.
- `decisions[]` — set by `phases/walkthrough/02-decisions.md`.
- `files_modified` — set by `phases/walkthrough/03-apply.md`.
- `verification_results` — set by `phases/walkthrough/04-verify.md`.
- `rescan_count` — initialized to `0` if absent.

If any required precondition is absent or empty, STOP and re-read
`phases/walkthrough/04-verify.md` (the producing phase). Do NOT proceed
with defaults, do NOT improvise, do NOT silently continue.

## Hard cap on re-scan loops

If `rescan_count >= 1`, skip the re-scan question and emit the
Walkthrough Summary directly (no second loop).

Otherwise, increment `rescan_count` (set to `1`) and proceed below.

## Re-scan question

After Phase 4, if any fixes were applied (`files_modified` non-empty),
use a single `AskUserQuestion` call:

- `question`: `"Fixes changed {N} files. Re-scan those files for issues introduced by the fixes?"`
- `header`: `"Re-scan?"` (max 12 chars)
- `multiSelect`: `false`
- `options`:
  1. `{ label: "Re-scan", description: "Re-run Agents 1 and 3 on just the modified files (must-fix / should-fix only — no suggestions)." }`
  2. `{ label: "Skip re-scan", description: "Go straight to the walkthrough summary." }`

## If `Re-scan`

1. Re-run **only Agents 1 and 3** (Hooks & Performance — the agents
   most likely to catch fix-induced regressions) on **only the files
   modified by fixes** (i.e. the `files_modified` set).
2. These agents use a **stricter threshold**: only report must-fix or
   should-fix findings. Suggestions are suppressed. Append this to
   their prompts:

   ```
   RESCAN MODE: You are re-scanning files that were just modified by automated fixes.
   Only report must-fix or should-fix severity findings. Do NOT report suggestions —
   this is a targeted check for issues introduced by the fixes, not a full review.
   ```

3. Send new findings through Opus verification (same Phase 4 verify
   prompt — see `phases/04-verify.md`).
4. Write the verified results into `findings_rescan[]` (NOT into
   `findings[]` — the original list stays immutable).
5. If `findings_rescan[]` is non-empty, loop back to
   `phases/walkthrough/02-decisions.md` for the second pass.
   `02-decisions.md` detects `rescan_count == 1` and iterates
   `findings_rescan[]` instead of `findings[]`, writing
   `decisions_rescan[]`.
6. If `findings_rescan[]` is empty after verification, fall through to
   the Walkthrough Summary below.

**No third pass.** After one re-scan, the loop ends regardless of
results. The `rescan_count >= 1` guard at the top of this file enforces
this on any re-entry.

## If `Skip re-scan` (or an `Other` answer interpreted as no)

Proceed directly to the Walkthrough Summary below.

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

When a re-scan ran, aggregate across both passes:
- **Fixed** = entries in `decisions[]` where `action == "fix"` PLUS
  entries in `decisions_rescan[]` where `action == "fix"`.
- **Skipped** = entries in either array where `action == "skip"`.
- **Files modified** = the final `files_modified` set (already reflects
  both passes plus any Phase 4 reverts).

## Next: emit the Walkthrough Summary and stop.
