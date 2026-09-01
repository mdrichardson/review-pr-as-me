# Auto-Scorer for Review Findings

Adversarial evaluation of review findings. The scorer is INDEPENDENT from the reviewer — it tries to DISPROVE each finding, not confirm it.

Scoring is organized around 6 specific concerns. Each finding is evaluated against ALL of them.

## The 6 Concerns

| # | Concern | Question per finding | Penalty/Reward |
|---|---------|---------------------|----------------|
| C1 | **Catches actual issues** | Is this a real bug, crash, security hole, or wrong behavior? | +10 per real issue, +15 for security |
| C2 | **Not too nitpicky** | Is this about style, naming, reuse, preferences, or "you could also do X"? | -3 per nitpick |
| C3 | **Not unrelated** | Is this about pre-existing code not introduced by the diff? | -4 per unrelated finding |
| C4 | **Not fabricated** | Is this actually an issue, or is the code correct? | -5 per false positive |
| C5 | **Silent when clean** | If zero findings, was the PR actually clean? | +8 for correct zero |
| C6 | **Fast** | Wall-clock time from invocation to output | Tracked, not scored |

## Process

For each finding in the review output, run through these checks IN ORDER. Stop at the first classification that applies.

### Check 1: Diff origin (C3 — unrelated)
Is the flagged code actually introduced or modified by this diff? Check the raw diff hunks.
- If the code existed before this PR and the finding is NOT about a security/data-loss issue → **`unrelated`**
- "While you're here" suggestions → **`unrelated`**
- Suggestions about surrounding context the PR didn't touch → **`unrelated`**

### Check 2: Try to disprove (C4 — false positive)
Actively seek reasons the finding is WRONG:
- Is there a guard/check the reviewer missed?
- Is this pattern used safely elsewhere in the same codebase?
- Does the type system prevent the claimed failure?
- Is the "trigger condition" actually impossible given the data flow?
- Would the claimed bad behavior actually occur in practice?
- Did the reviewer misread the code?

If you find counter-evidence → **`false_positive`**

### Check 3: Check if it's a nitpick (C2 — nitpicky)
Even if technically accurate, is this a style/preference issue?
- Is it about naming, formatting, import order, or code organization?
- Is it "you could also do X" when the current approach works?
- Is it about reuse when the current code is correct and functional?
- Is it about component extraction, abstraction, or file splitting?
- Is it suggesting a "better" pattern when the current one isn't wrong?
- Is it about missing types, docs, or comments where code is clear?
- Would a busy senior engineer skip this comment in a real review?

If it's a preference that doesn't affect correctness or production behavior → **`nitpick`**

### Check 4: Confirm real issue (C1 — catches actual issues)
If the finding survived checks 1-3, it's real:
- **`security_issue`** (+15): XSS, injection, auth bypass, data exposure
- **`real_issue`** (+10): genuine bug, crash, wrong behavior, data loss, race condition, missed error handling that would break the user experience

## Per-Finding Output Format

```
FINDING: {finding title from review}
FILE: {file:line}
CONCERN CHECKS:
  C3 unrelated?:  {yes/no — is this about pre-existing code?}
  C4 false positive?: {yes/no — is the code actually correct?}
  C2 nitpick?:    {yes/no — is this style/preference, not behavior?}
  C1 real issue?:  {yes/no — genuine bug or security issue?}
CLASSIFICATION: {real_issue|security_issue|false_positive|nitpick|unrelated}
REASONING: {1-2 sentences}
COUNTER_EVIDENCE: {what disproof was attempted and result}
```

## Aggregate Output

```
=== CONCERN SCORECARD ===

C1 Catches actual issues:
  real_issues: {n} × 10 = {subtotal}
  security_issues: {n} × 15 = {subtotal}
  missed_issues: {list any real bugs the review failed to catch}

C2 Not too nitpicky:
  nitpicks: {n} × -3 = {subtotal}
  nitpick_rate: {nitpicks}/{total_findings} = {percent}%

C3 Not unrelated:
  unrelated: {n} × -4 = {subtotal}
  unrelated_rate: {unrelated}/{total_findings} = {percent}%

C4 Not fabricated:
  false_positives: {n} × -5 = {subtotal}
  false_positive_rate: {fps}/{total_findings} = {percent}%

C5 Silent when clean:
  correct_zero: {0 or 1} × 8 = {subtotal}
  (award +8 if review reported 0 findings AND PR was actually clean)
  (award 0 if review reported 0 findings but missed real issues)
  (N/A if review reported findings)

C6 Fast:
  wall_time_sec: {seconds from invocation to output}

COMPOSITE SCORE: {sum of all subtotals}
TOTAL FINDINGS: {n}
FINDING BREAKDOWN: {real}R / {fp}FP / {nit}N / {unr}U
```
