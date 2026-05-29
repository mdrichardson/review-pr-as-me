# Walkthrough Phase 1 — Summary

## Preconditions

Assumes the following state is set:
- `findings[]` — non-empty list of verified findings, set by `phases/04-verify.md`. The empty-findings gate in `phases/walkthrough/README.md` handles the empty case before entry, so by the time this file runs, `findings[]` is guaranteed to have at least one entry.

If any required precondition is absent or empty, STOP and re-read
`phases/04-verify.md` (the producing phase). Do NOT proceed with
defaults, do NOT improvise, do NOT silently continue.

## Re-entry guard

If `findings[]` already has a corresponding task list created in this
conversation, skip re-rendering the summary table — only re-emit the
per-finding tasks if any are still uncreated.

## Render the summary

Output a concise overview of ALL findings so the user sees the full
picture before making any decisions. Use the shared header + "Findings
at a glance" table shape (matches the review-comments rendering):

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

Before moving to Phase 2, create one task per finding using `TaskCreate`
so the user can see progress at a glance. **Order tasks file-by-file,
line-sorted within each file** (matches Phase 2 walkthrough order).
Numbering stays global so the progress indicator works across the whole
session:

```
Example for 4 findings across 3 files (file-by-file, line-sorted):
  "Finding 1/4 in ComponentA.tsx: Missing null check on user prop (must-fix)"      activeForm: "Reviewing finding 1/4"
  "Finding 2/4 in ComponentA.tsx: Effect cleanup missing (should-fix)"              activeForm: "Reviewing finding 2/4"
  "Finding 3/4 in useData.ts: Stale closure in callback (should-fix)"               activeForm: "Reviewing finding 3/4"
  "Finding 4/4 in helpers.ts: Could extract shared formatter (suggestion)"          activeForm: "Reviewing finding 4/4"
```

Mark each task `in_progress` when presenting it to the user, and
`completed` when the user makes a decision (fix, skip, or discuss →
resolved). During Phase 3 (Apply), update the task name to include the
decision — e.g. append `→ fixed` or `→ skipped`.

## Next: read `phases/walkthrough/02-decisions.md`.
