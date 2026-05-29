# Plan-Fixes Walkthrough (entry point)

When `output_mode == "plan-fixes"`, this directory replaces Step 6
entirely with an interactive fix walkthrough split into five sub-phases.

**Reuses from review-comments:** the shared header, the "Findings at a
glance" table, and the unified finding block (five labeled sections)
from `references/output-template.md`.

**Skips:** VOICE.md, voice comments (the **Comment (in my voice)**
block), overall-review summary prose, positive callouts.

## Empty-findings gate

If `findings[]` is empty, emit `"No confirmed findings — walkthrough
complete."` and stop. Do NOT enter `01-summary.md` — there is nothing to
walk through.

## Sub-phase index

| Sub-phase | File | What it does |
|-----------|------|--------------|
| 1 — Summary | `01-summary.md` | Render header + "Findings at a glance" table; create per-finding tasks. |
| 2 — Decisions | `02-decisions.md` | Walk findings one at a time via `AskUserQuestion`; collect `decisions[]`. Ask for `verification_level` at the end. |
| 3 — Apply | `03-apply.md` | Recap, then Edit-apply fix-marked findings. TDD loop inline for must-fix when `verification_level in ("tdd", "both")`. |
| 4 — Verify | `04-verify.md` | Level 1 standard checks (TS / lint / tests). Level 2 regression tests when `verification_level in ("regression", "both")`. Level 3 TDD reporting only (ran inline in Phase 3). |
| 5 — Rescan | `05-rescan.md` | Optional one-shot re-scan of modified files (Agents 1 & 3 only). Loops back to Phase 2 once if it produces new findings, else emits the Walkthrough Summary. |

## Loop edges

```
empty findings[] → Walkthrough complete, stop (gate above)
01-summary    → 02-decisions (always, after task creation)
02-decisions  → 03-apply     (after all decisions collected, or Fix-all / stop)
                              First pass: iterates findings[], writes decisions[].
                              Second pass (when rescan_count == 1): iterates
                              findings_rescan[], writes decisions_rescan[].
03-apply      → 04-verify    (always)
                              First pass: applies decisions[]. Re-entry pass
                              (when decisions_rescan[] is set): applies
                              decisions_rescan[] only.
04-verify     → 05-rescan    (always, if any fixes applied)
05-rescan     → 02-decisions (ONCE, if re-scan confirmed new findings into
                              findings_rescan[]; sets rescan_count = 1)
05-rescan     → Walkthrough Summary (otherwise, or after the one allowed loop)
```

## State carried across sub-phases

This table is the **only** place the state shape is defined. Sub-phase
files reference it by name — they must not redefine fields.

| Variable | Type | Set in | Mutable during | Read in |
|----------|------|--------|----------------|---------|
| `findings[]` | Array of verified finding records (file, line, severity, fix_snippet, fix_alternatives, etc.) | Inherited from Phase 4 (verify) of main flow | `02-decisions.md` Discuss flow may augment `fix_alternatives` on a single finding | All sub-phases |
| `decisions[]` | Array aligned with `findings[]`. Each entry: `{ index, action: "fix" \| "skip", chosen_option?: "primary" \| "A" \| "B" \| "C" }` | `02-decisions.md` | (immutable after Phase 2 ends) | `03-apply.md` (drives Edit calls), recap rendering in 03/04 |
| `verification_level` | One of `"none"`, `"regression"`, `"tdd"`, `"both"` | End of `02-decisions.md` (skipped → `"none"` when no must-fix marked for fix) | (immutable after Phase 2) | `03-apply.md` (TDD branch), `04-verify.md` (regression branch) |
| `files_modified` | Array of file paths actually edited during Phase 3 (excludes reverted) | `03-apply.md` | `04-verify.md` removes entries on revert | `05-rescan.md` (re-scan target set), Walkthrough Summary |
| `rescan_count` | Integer, starts at `0` | `05-rescan.md` increments on entry | `05-rescan.md` only | `05-rescan.md` re-entry guard (one allowed loop) |
| `findings_rescan[]` | Array of NEW verified findings produced by the re-scan agents (same record shape as `findings[]`). Empty when no re-scan occurred or re-scan produced nothing | `05-rescan.md` writes once after re-scan agents return | (immutable after Phase 5 writes it) | `02-decisions.md` (second pass — when `rescan_count == 1`, iterate `findings_rescan[]` instead of `findings[]`), Walkthrough Summary |
| `decisions_rescan[]` | Array aligned with `findings_rescan[]`, same record shape as `decisions[]` | `02-decisions.md` on second pass (when `rescan_count == 1`) | (immutable after second-pass Phase 2 ends) | `03-apply.md` on re-entry — applies ONLY `decisions_rescan[]`, never re-touches original `decisions[]`. Walkthrough Summary aggregates both. |
| `verification_results` | Record: `{ ts_errors, lint_warnings, test_failures, regression_tests_written, tdd_validations_passed, tdd_false_positives, notes }` | `04-verify.md` | `03-apply.md` writes tdd_* slots; `04-verify.md` writes everything else | Walkthrough Summary |

## Skip rules

- Empty `findings[]` → walkthrough does not enter at all (gate above).
- No must-fix findings marked for fix at end of Phase 2 → skip the
  Verification Preferences question; default `verification_level =
  "none"`.
- `files_modified` empty at start of Phase 5 → skip the re-scan question
  entirely; go straight to the Walkthrough Summary.
- `rescan_count >= 1` on entering Phase 5 → skip the re-scan question;
  emit Walkthrough Summary.

## Next: read `phases/walkthrough/01-summary.md`.
