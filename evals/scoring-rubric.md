# Scoring Rubric for Review Findings

Use this rubric to classify each finding from a review run. Every finding gets exactly ONE label.

## Labels

### real_issue (+10 points)
The finding identifies a genuine problem that would cause incorrect behavior, a crash, data loss, or a security vulnerability. You would leave this comment on the PR yourself.

**Examples:**
- State mutated directly (React won't re-render)
- useEffect missing cleanup for a subscription
- Stale closure capturing old state in async callback
- Unhandled promise rejection that breaks the page
- XSS vector via dangerouslySetInnerHTML

### security_issue (+15 points)
A real_issue specifically about security — XSS, injection, auth bypass, data exposure. Score these separately for tracking.

### false_positive (-5 points)
The finding flags something that isn't actually a problem. The code is correct; the reviewer got it wrong.

**Examples:**
- Claims a dependency is missing but it's correctly excluded
- Claims a callback is unstable but it's memoized upstream
- Claims a ref-based effect is wrong but the pattern is standard
- Claims optional chaining in deps is a bug but the behavior is intentional
- Claims missing cleanup but the code has no side effects to clean up

### nitpick (-3 points)
The finding is technically accurate but is a style/preference issue you wouldn't bother commenting on. It doesn't affect correctness or performance.

**Examples:**
- "You could use an existing utility for this" (when the current code is fine)
- Naming suggestion ("consider renaming X to Y")
- "This file is getting long, consider splitting"
- "Consider extracting this into a custom hook"
- "You could also do X" when the current approach works
- Import ordering or style preferences
- Missing TypeScript type annotations where types are obvious
- Suggesting a different component pattern when the current one is correct

### unrelated (-4 points)
The finding is about code that existed before this diff. It may even be a valid observation, but it's not about the PR author's changes.

**Examples:**
- Flagging a bug in a function that wasn't modified by this PR
- Suggesting improvements to surrounding code not in the diff
- "While you're here, you should also fix..."
- Pre-existing pattern that the PR didn't introduce or worsen

### correct_zero (+8 points, per PR)
Award this bonus when the skill correctly reports zero findings on a PR that genuinely has no issues. Only applicable to clean PRs — don't award this on PRs that have real issues the skill missed.

## Scoring Formula

```
review_score = (real_issues * 10)
             + (security_issues * 15)
             - (false_positives * 5)
             - (nitpicks * 3)
             - (unrelated * 4)
             + (correct_zero * 8)
```

## How to Score

1. Read each finding in the review output
2. For each finding, assign exactly one label: `real_issue`, `security_issue`, `false_positive`, `nitpick`, or `unrelated`
3. Count totals per label
4. Also note any real issues the review MISSED (for tracking detection quality — not scored but logged in notes)
5. Record wall-clock time from invocation to final output
6. Compute the score and add a row to `improvement-log.tsv`

## Edge Cases

- **Real issue but wrong severity**: Still counts as `real_issue`. Severity accuracy is tracked separately in notes.
- **Real issue but about unrelated code**: Counts as `unrelated`. The issue is real but out of scope for this PR.
- **Nitpick disguised as should-fix**: If the reviewer inflated severity on a style preference, count as `nitpick`.
- **Partially correct**: If the core observation is valid but the explanation is wrong or exaggerated, count as `real_issue` with a note about the inaccuracy.
