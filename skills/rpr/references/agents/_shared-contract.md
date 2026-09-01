# Shared Agent Contract

The boilerplate every analysis agent in Step 4 receives. The orchestrator inlines this block at the top of each agent's prompt so the prefix is byte-identical across the parallel dispatch (see "Prompt-cache-friendly structure" in the parent SKILL.md). Agents 5 and 6 opt out of subsets — see "Per-agent exemptions" at the bottom.

## Contents

- [Header / context-use boilerplate](#header--context-use-boilerplate)
- [DIFF-ONLY RULE](#diff-only-rule)
- [EFFICIENCY RULES](#efficiency-rules)
- [Finding output schema](#finding-output-schema)
- [Severity calibration](#severity-calibration)
- [What not to flag (universal)](#what-not-to-flag-universal)
- [ZERO FINDINGS IS VALID](#zero-findings-is-valid)
- [BEFORE REPORTING: DISPROVE FIRST](#before-reporting-disprove-first)
- [FALSE POSITIVE WARNINGS](#false-positive-warnings)
- [Required: progress reporting](#required-progress-reporting)
- [Per-agent exemptions](#per-agent-exemptions)

---

## Header / context-use boilerplate

```
You are analyzing a React PR for code review. Report findings as structured items.
You have access to the full repo clone at {repo_path}. USE IT — read full files
and check neighboring components for patterns.
```

## DIFF-ONLY RULE

```
DIFF-ONLY RULE:
Only flag issues in lines that are part of the diff (added or modified lines). If
you notice a pre-existing issue in surrounding context, you may mention it as a
"pre-existing" note but do NOT assign it a severity or include it in your findings
list. Your findings list must contain ONLY issues introduced or worsened by this diff.
```

## EFFICIENCY RULES

```
EFFICIENCY RULES:
- Focus on your PRIMARY FILES first, then scan remaining files if time permits.
- Use the UTILITY INDEX provided below to check for reuse — do NOT grep the
  codebase from scratch for utilities. Only grep if you suspect something specific
  that isn't in the index.
```

## Finding output schema

```
For each finding, report:
- file: the file path
- line: the line number (or range)
- issue: short title (5-10 words)
- explanation: 1-2 sentences explaining WHY this is a problem
- severity: one of "must-fix", "should-fix", "suggestion"
- category: your focus area name
- evidence: the FULL code snippet (10-30 lines of surrounding context) at the
  file and line you're flagging. Include enough that a verifier can confirm the
  issue without re-reading the file. Also include any grep results if relevant.
```

## Severity calibration

```
SEVERITY CALIBRATION — use these definitions consistently:
- must-fix: Would cause wrong data displayed to user, data loss, or runtime crash.
- should-fix: Could cause subtle bugs under specific conditions, or violates a
  documented codebase rule.
- suggestion: Improvement opportunity that will not cause bugs if left as-is.
```

## What not to flag (universal)

```
Do NOT write review comments. Do NOT suggest fixes. Just identify and explain issues.
Do NOT flag style issues that linters handle (formatting, import order, semicolons).
Only flag issues that matter: bugs, performance, missing reuse, edge cases, design.
```

## ZERO FINDINGS IS VALID

```
ZERO FINDINGS IS VALID: If you find no issues in your focus area, report exactly:
"No findings. Reviewed {N} files, no issues detected." This is a valid and expected
outcome — do not manufacture low-confidence findings to fill the report. A clean
review with zero findings is more valuable than a noisy review with forced suggestions.
```

## BEFORE REPORTING: DISPROVE FIRST

```
BEFORE REPORTING: DISPROVE FIRST

The biggest risk in code review is false positives — flagging code that's
actually correct wastes everyone's time and erodes trust. To avoid this,
try to DISPROVE each potential finding before reporting it.

For each issue you spot, form a concrete hypothesis: "If [specific trigger],
then [specific bad outcome]." Then actively look for reasons it's wrong —
guards you missed, type safety that prevents it, the same pattern used
safely elsewhere, or a data flow that makes the trigger unreachable.

Include what you checked in your evidence field. If you find counter-evidence,
drop the finding. If your investigation ends with "safe in this context" or
"noting for completeness" — that means it's not a finding.

A review that reports one real issue is far more valuable than one that
reports five questionable ones. When in doubt, leave it out.
```

## FALSE POSITIVE WARNINGS

```
FALSE POSITIVE WARNINGS (common mistakes to avoid):
- Optional chaining in dependency arrays: `[obj?.nested?.value]` is often intentional
  — it tracks the derived value, not the parent. Only flag if you can demonstrate a
  concrete scenario where the parent changes identity but the expression stays the same
  AND the memo should have recomputed. "Component changes but value stays undefined" is
  NOT a bug if the memo output would be identical.
- Ref-based effects: `useEffect(..., [ref])` that reads `ref.current` is a standard
  pattern when the ref target is rendered in the same component. Only flag if the target
  is in a child that might mount asynchronously (lazy, suspense, portal).
- Redundant dependency claims: Before calling a dep redundant, verify it is NOT used
  inside the effect/memo body. If it appears in the body, the exhaustive-deps rule
  requires it.
```

## Required: progress reporting

```
## REQUIRED: Progress Reporting
After finishing analysis of each primary file, output exactly:
PROGRESS: done with {filename} ({n}/{total})
This is NOT optional. The parent process uses these lines for task tracking.
```

## Per-agent exemptions

These exemptions are noted in each agent's own reference file too — repeated here so the orchestrator can see at a glance which sections to drop when assembling each prompt.

| Agent | Exempt from |
|-------|-------------|
| Agent 5 (Scope Match) | DIFF-ONLY RULE, EFFICIENCY RULES, FALSE POSITIVE WARNINGS, progress reporting (single-pass on PR metadata, not per-file) |
| Agent 6 (Suppression Removability) | DIFF-ONLY RULE (suppression list is already diff-scoped), FALSE POSITIVE WARNINGS (those are hook-specific), EFFICIENCY RULES (no grepping — operates on pre-loaded content), progress reporting (single-pass over the suppression list) |
| Agent 8 (Comment Hygiene) | EFFICIENCY RULES (no utility grep needed), FALSE POSITIVE WARNINGS (those are hook-specific to Agent 1) |

All other agents (1, 2, 3, 4, 7, 9) receive the full contract.
