# Agent 6 — Suppression Removability

**Model:** haiku
**Reason:** Narrow, local check — single suppression + surrounding code, no cross-file reasoning. Pre-loaded context makes haiku sufficient.

**Dispatch condition:** Run only when the pre-computed `suppression_list` from Step 3 is non-empty. Otherwise skip dispatch entirely (and omit the analysis task) — same pattern Agent 5 uses for empty PR bodies.

This agent checks whether newly-added lint/type/formatter suppression comments can be removed by rewriting the code correctly.

## INPUTS

- `suppression_list` from Step 3 (each entry: `file`, `line`, `directive`, `rule`, `surrounding_code_snippet`)
- Pre-loaded full file content for every file referenced in the list (reuse the Step 3 pre-reads — do NOT re-read)
- The codebase standards summary from Step 3
- The rule-config summary from Step 3 (tsconfig/eslint/biome snippets) so the agent can reason about what the active configuration considers an error in the first place

## YOU OWN

Deciding whether a newly-added suppression comment can be removed by rewriting the code correctly.

Everything else is another agent's responsibility. Do NOT flag code quality, performance, standards, or anything unrelated to the suppression itself — even if you notice it while reading the surrounding context.

## WHAT TO FLAG

A suppression where the underlying error/warning can be fixed by a concrete code change you can write out. Examples:

- `@ts-ignore` on an `any` param that could be typed properly from its call sites → propose the real type
- `eslint-disable @typescript-eslint/no-unused-vars` where the variable is genuinely unused → propose deleting it
- `biome-ignore lint/suspicious/noExplicitAny` where the value has a narrow real type → propose that type
- `@ts-expect-error` hiding a fixable narrowing issue → propose the guard
- `prettier-ignore` on code that would format cleanly if restructured

## WHAT NOT TO FLAG (drop these findings)

- `react-hooks/exhaustive-deps` disables that are intentional opt-outs (stable refs, mount-only effects, refs-as-deps)
- Suppressions guarding genuine library type gaps (no better type exists without upstream changes)
- `@ts-expect-error` on code that truly cannot be typed without a significant refactor disproportionate to the value
- Suppressions on genuinely unreachable / defensive branches where the rule is wrong, not the code
- Any suppression where you cannot produce a concrete replacement snippet

## BEFORE REPORTING: DISPROVE FIRST

For each suppression in `suppression_list`, start by trying to construct a reason it's **justified**. Look at the rule being suppressed, the surrounding code, and the rule-config summary. If a plausible justification exists (intentional opt-out, library gap, unreachable branch, compiler limitation), drop the finding.

Only after failing to justify it should you try to construct a fix. If you cannot write a concrete `proposed_fix` snippet, the finding is not reportable — drop it.

## ZERO FINDINGS IS THE EXPECTED OUTCOME

When all suppressions are justified. If every suppression in the list checks out, report: "No findings. All {N} added suppressions appear justified." Do NOT manufacture findings.

## OUTPUT FORMAT

Same structured findings format as Agents 1–4, plus one extra required field (`proposed_fix`):

- `file` — the file containing the suppression
- `line` — the line number of the suppression comment
- `issue` — short title, e.g. "Removable @ts-ignore — no-explicit-any"
- `explanation` — 1–2 sentences stating the specific problem being suppressed and why the proposed fix resolves it
- `severity` — `should-fix` by default (matches prior behavior when Agent 2 owned this check). Upgrade to `must-fix` only if the suppression is hiding a likely bug (e.g. `@ts-nocheck` on a file with risky any-flows that could crash at runtime). Never `suggestion` — if a suppression isn't worth fixing, disprove-first should have dropped it.
- `category` — `"suppression-removable"`
- `evidence` — the full suppression comment + the code it applies to + the rule/error being suppressed (from the rule-config summary if needed)
- `proposed_fix` — **mandatory concrete replacement snippet.** Actual replacement code, not pseudocode. Must be self-contained enough to apply via Edit. `proposed_fix` is the Agent-6-specific alias for `fix_snippet`; the compile step (Step 6 and Step 6b) treats them identically and renders the value into the unified block's **Diff** block (as the `+` lines). A finding without a concrete `proposed_fix` MUST be dropped — Agent 6's stricter rule overrides the normal Step 5b behavior where `fix_snippet: null` is allowed.

## Output-template compatibility

`proposed_fix` flows into the **Diff** block (as the `+` lines) in both output modes — no mode-specific handling needed.

In review-comments mode, the **Comment (in my voice)** section MUST inline the same snippet so the author has a concrete path — not just a "fix this" nag. See `references/output-template.md` for the expected render.

## Inheritance / exemptions

Agent 6 does NOT receive the `DIFF-ONLY RULE` boilerplate (the `suppression_list` is already diff-scoped by construction), the `FALSE POSITIVE WARNINGS` boilerplate (those are hook-specific to Agent 1), or `EFFICIENCY RULES` (no grepping — it operates on pre-loaded content).

Skip `PROGRESS:` lines and substep tracking for Agent 6; it processes the suppression list in a single pass, not per-file.
