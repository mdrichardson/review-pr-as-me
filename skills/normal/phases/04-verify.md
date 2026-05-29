# Phase 4 — Verify

Deduplicate findings, verify each with a per-finding (or per-file
batched) Opus agent, and classify the user's previous PR comments.

> **Fast-mode override.** If you loaded `../fast/SKILL.md`, apply its
> model-stepdown override table before using the model assignments
> below. The fast skill enumerates which phase files its overrides
> target.

## Contents

- [Preconditions](#preconditions)
- [Substep 4a: Deduplicate](#substep-4a-deduplicate-orchestrator-no-agent) — inter-agent dedup + silent dedup against existing PR comments (review-comments mode only)
- [Substep 4b: Per-Finding Verification](#substep-4b-per-finding-verification-parallel-opus-pre-loaded-context) — parallel Opus, per-finding default, per-file batching when ≥3 cluster
- [Additional Output for Unified Finding Block](#additional-output-for-unified-finding-block) — `fix_description` / `fix_snippet` / `fix_effort` / `fix_alternatives` schema; Agent 6 parity rules; Diff-source rule
- [Substep 4c: Compile Verification Results](#substep-4c-compile-verification-results-orchestrator-no-agent) — confirmed → `findings[]`, drop false positives, apply downgrades
- [Substep 4d: Classify User's Previous-Comment Status](#substep-4d-classify-users-previous-comment-status-orchestrator-no-agent) — bucket "mine" threads into Resolved / Outdated / Still-open / Unclear
- [Render-Once Rule (orchestrator discipline)](#render-once-rule-orchestrator-discipline)
- [Next](#next)

## Preconditions

Assumes the following state is set:
- `agent_findings[]` — flat list of finding records returned by all
  dispatched agents in `phases/03-analyze.md`.

If any required precondition is absent or empty, STOP and re-read
`phases/03-analyze.md` (the producing phase). Do NOT proceed with
defaults, do NOT improvise, do NOT silently continue.

The analysis agents cast a wide net. This phase deduplicates and
verifies with higher rigor using parallel per-finding Opus agents,
then produces the final `findings[]` list for output (Phase 5a) or
walkthrough (`phases/walkthrough/`).

## Substep 4a: Deduplicate (orchestrator, no agent)

Before dispatching verification, merge duplicate findings:

1. Group findings into dedup candidates by EITHER:
   a. Same file AND line numbers within ±5 lines, OR
   b. Same file AND issue-title fingerprint match (case-insensitive; ignore line-number tokens; treat synonyms — "missing cleanup" / "no cleanup" / "leak" — as equivalent), regardless of line distance.
2. If two candidates describe the same root cause (even from different agents with different framing, or at different lines on the same root issue), merge them:
   - Keep the more detailed explanation
   - Keep the higher severity
   - If the lines differ, list both as a line range (e.g. "42, 88")
   - Note both source agents
3. Report: `"Merged {N} findings into {M} unique findings ({N-M} duplicates removed)"` — counts only; do not list the merged findings inline. See the Render-Once Rule below.

Agent 5's `scope-match` category findings are PR-scoped rather than
line-scoped and almost never overlap with other agents' findings. Pass
them through dedup untouched.

### Silent dedup against existing PR comments

**Skip this substep entirely when `output_mode == "plan-fixes"`.**
Plan-fixes mode is specifically for walking through every finding and
fixing them — dropping items that reviewers already raised would leave
the user unable to address those issues in the walkthrough. On own-PR +
plan-fixes mode, dedup is actively counter-productive. Review-comments
mode keeps the dedup because its goal (avoid generating duplicate voice
comments) is the opposite.

After inter-agent dedup, drop any finding already covered by an
existing comment on the PR. Using `pr_comments_list` from Phase 1:

1. For each finding, check every entry in `pr_comments_list` where
   `status == "open"` and `file` matches the finding's file path.
2. If such an entry exists and its `line` is within ±3 lines of the
   finding's line (or line range), **drop the finding silently**. No
   output, no annotation — the goal is to avoid re-raising what's
   already on the PR.
3. Do not dedup against `"resolved"` or `"outdated"` threads. If the
   author marked a thread resolved but the code still has the issue,
   raising it fresh is correct. If the thread is outdated because the
   line moved, the line positions won't match anyway.
4. Do not dedup on file alone — only file + overlapping line range.
   Two distinct bugs in the same file at different lines are distinct
   findings.
5. `scope-match` findings (Agent 5) have `line: 0` and never match a
   line-scoped PR comment. Pass them through.
6. Track the count of dropped findings. Report:
   `"Dropped {N} findings already covered by existing PR comments."`
   This count surfaces in the review header in Phase 5a.

When `pr_comments_list` is empty (first review pass, or local mode),
this substep is a silent no-op.

## Substep 4b: Per-Finding Verification (parallel Opus, pre-loaded context)

**Default:** Dispatch ONE Opus agent per unique finding, all in a SINGLE
message (parallel). Each agent gets the relevant pre-read file content
from Phase 2 embedded directly — no tool calls needed. This gives each
agent deep focus on a single finding while running in parallel.

A/B testing showed per-finding agents produce more thorough analysis
than batched agents (which rush later findings). The wall-clock cost
is comparable since they run in parallel and each has a small, focused
prompt.

**Per-file batching when ≥3 findings cluster in one file.** When three
or more unique findings (after Substep 4a dedup) all reference the same
file, group them into a single per-file verification call instead of N
independent ones. Rationale: each per-finding agent ships the same
file's pre-read content, so N agents with 3+ findings on the same file
send 3+× the same content. One agent with the file's content embedded
once + an array of findings to verify is significantly cheaper in
tokens, and the "rush later findings" failure mode that motivated
per-finding agents only manifests when a single agent juggles findings
across many different files. Same-file batching keeps the agent's
focus tight (one file's data flow) while collapsing the redundancy.

The batching decision is per-file, not global. Mix-and-match is
expected on a typical review:

```
file A — 1 finding   → 1 per-finding agent
file B — 2 findings  → 2 per-finding agents
file C — 4 findings  → 1 per-file batched agent (covers all 4)
file D — 1 finding   → 1 per-finding agent
file E — 6 findings  → 1 per-file batched agent (covers all 6)
                      Total: 4 per-finding + 2 per-file = 6 dispatches
                      (instead of 14)
```

All dispatches still go in a SINGLE message for parallel execution.

The per-file batched prompt extends the per-finding prompt template:

```
You are a senior React engineer verifying MULTIPLE code review findings
that all reference the same file. The file's contents are pre-loaded
below — use them directly instead of reading from disk.

FILE: {path}

FILE CONTENTS:
{full file content}

RELEVANT DIFF SECTION FOR THIS FILE:
{diff hunks for this file only}

CODEBASE STANDARDS SUMMARY:
{standards summary}

FINDINGS TO VERIFY (process each one independently):
1. {finding 1 — file, line, issue, explanation, severity, evidence}
2. {finding 2 — file, line, issue, explanation, severity, evidence}
...
N. {finding N — file, line, issue, explanation, severity, evidence}

For EACH finding, separately:
1. Check the evidence using the pre-loaded file content.
2. Trace the data flow to confirm the issue manifests in practice.
3. Verify the flagged code was introduced by this diff, not pre-existing.
4. Output a verdict block:

   FINDING {N} VERDICT: "confirmed" | "false-positive" | "downgrade"
   {then the ADDITIONAL OUTPUT block — fix_description, fix_snippet,
    fix_effort, fix_alternatives — same schema as per-finding agents}

Treat each finding independently. A false-positive on finding 2 must
not influence your verdict on finding 3. Do NOT skim later findings.
The "rush later findings" anti-pattern is the #1 risk of batched
verification — counter it by physically separating each finding's
verdict block in your output.

Rejecting false positives is valuable — it protects the PR author
from noise. Don't feel pressure to confirm findings.
```

The orchestrator parses each `FINDING {N} VERDICT` block separately
and feeds them through Substep 4c exactly as if they had come from N
independent agents. There's no merge step; a per-file agent just emits
N verdict blocks instead of one.

Each verification agent uses **model: "opus"** and receives:
- Its single finding (with evidence snippet)
- Pre-read content of the file(s) referenced by the finding
- The relevant diff section
- The codebase standards summary

```
You are a senior React engineer verifying a SINGLE code review finding.
The file contents are pre-loaded below — use them directly instead of
reading from disk. The full repo is at {repo_path} if you need files
beyond what's provided.

FINDING TO VERIFY:
{finding — file, line, issue, explanation, severity, evidence}

Your job:
1. Check the evidence using the pre-loaded file content.
2. Trace the data flow to confirm the issue manifests in practice.
3. Verify the flagged code was introduced by this diff, not pre-existing.
4. VERDICT: "confirmed", "false-positive", or "downgrade"

Rejecting false positives is valuable — it protects the PR author from
noise. Don't feel pressure to confirm findings.

Common false positives to watch for:
- useCallback dependency arrays that ARE correct but look suspicious
- Memo "broken" claims where object identity IS actually preserved
- "Unstable callback" claims where the callback IS memoized upstream
- Missing cleanup claims for code that has no side effects to clean up
- Pre-existing issues in surrounding context not introduced by this diff
```

## Additional Output for Unified Finding Block

Both output modes now render findings with the same unified finding
block (Severity, Description, Diff, Effort — plus the voice Comment
only in review-comments mode). That means Substep 4b must produce the
same three extra fields regardless of `output_mode`. Append this to
every per-finding verification prompt:

```
ADDITIONAL OUTPUT — UNIFIED FINDING BLOCK:
If CONFIRMED, also provide:
- fix_description: 1-2 sentences describing the specific code change needed.
  MANDATORY. This populates the Description field in the unified block,
  including for findings with no concrete fix snippet.
- fix_snippet: the corrected code for the PRIMARY suggestion, complete
  enough to apply via Edit tool. OPTIONAL — emit null when no concrete
  replacement snippet makes sense (file-too-long, missing-test-coverage,
  design-level refactor, and similar design-level findings). The compile
  step omits the "Diff" block for findings with fix_snippet: null.
- fix_effort: "trivial" (1-line change), "small" (< 10 lines), or "medium"
  (refactor needed) — for the primary suggestion. MANDATORY in every case.
- fix_alternatives: OPTIONAL array of additional, distinctly-different
  approaches when more than one reasonable fix exists. Cap at 2 entries
  (so the total — primary + alternatives — never exceeds 3, which is the
  max the plan-fixes Phase 2 picker can render as buttons). Each entry:
    {
      label:       "1-5 word summary used as the AskUserQuestion button"
      description: "1-2 sentences — what this approach does and why a
                    reviewer would pick it over the primary"
      snippet:     "concrete replacement code (same rules as fix_snippet
                    — null is allowed only if the alternative is a
                    design-level direction with no one-shot patch)"
      effort:      "trivial | small | medium"
    }
  Only include alternatives when they're genuinely distinct paths
  (different abstraction, different tradeoff). Do NOT pad with minor
  stylistic variants — one strong fix beats two near-duplicates. If only
  one approach makes sense, omit fix_alternatives entirely (or emit []).

Be precise — every snippet, when present, must be actual replacement code,
not pseudocode. Include enough surrounding context that the Edit tool
can locate the old_string. If you cannot write a concrete replacement,
emit null rather than forcing pseudocode.
```

**Agent 6 parity:** Agent 6's `proposed_fix` field is the
suppression-removable alias for `fix_snippet` — the compile step treats
them identically. Agent 6 may also emit `fix_alternatives` (e.g. a
narrow type-guard fix vs. a broader signature change for the same
removable suppression) under the same 2-entry cap. Agent 6's stricter
rule still applies: a suppression-removable finding without a concrete
`proposed_fix` is dropped entirely, not emitted with a null slot.

**Diff source:** the compile step extracts the flagged line(s) from
the pre-read file content (`pre_read_contents` from Phase 2) and emits
them as the `-` lines of the **Diff** block; the verification agent's
`fix_snippet` (or Agent 6's `proposed_fix`) becomes the `+` lines. For
a single `line: 42`, take line 42 verbatim. For a range `line:
"42-47"`, take lines 42 through 47. No surrounding context, no
trimming of whitespace. The fenced block's language tag is always
`diff` so the `-` / `+` markers render correctly; no
per-file-extension language derivation is needed.

## Substep 4c: Compile Verification Results (orchestrator, no agent)

After all per-finding agents return:
1. Collect all "confirmed" findings — these become `findings[]`, the final list.
2. Discard "false-positive" findings.
3. Apply any "downgrade" severity changes.
4. Report: `"Verified {N} findings: {confirmed} confirmed, {rejected} false positives, {downgraded} downgraded"`.

## Substep 4d: Classify User's Previous-Comment Status (orchestrator, no agent)

Review-comments mode, PR-URL mode only. Skip entirely when
`pr_comments_list` is empty or the review is local.

For each entry in `pr_comments_list` where `is_mine == true`, produce a
classified row for the "My previous comments on this PR" section in
Phase 5a. The goal is to show the user how each of their prior comments
was handled since it was posted.

Classify each "mine" thread into one of four buckets:

| Bucket | Criteria |
|--------|----------|
| **Resolved** | `status == "resolved"` (author marked fixed / closed / byDesign / wontFix on ADO, or review thread resolved on GitHub). |
| **Outdated** | `status == "outdated"` on GitHub (the line no longer exists in the diff) — author changed something, thread auto-collapsed. |
| **Still open — line unchanged** | `status == "open"` AND the code at `file:line` in the current PR head matches the code the comment was originally posted against. Use the pre-read file contents from Phase 2; for GitHub, compare `line` vs `originalLine` (equal = unchanged position). For ADO, fall back to a content check: hash the line's text at post-time vs now — if unavailable, treat as "open — unclear". |
| **Still open — line changed** | `status == "open"` AND the line moved or was modified since the comment was posted. Author partially addressed it or touched nearby code. |
| **Open — unclear** | Any remaining cases (PR-wide / issue-level comments with no `file`/`line`, or status heuristics couldn't be computed). Default bucket. |

For each "mine" entry, emit a classified row:

```
{
  file:         {file or "(PR-wide)" for issue comments}
  line:         {line or "—"}
  body_excerpt: {from pr_comments_list}
  status:       {"Resolved" | "Outdated" | "Still open — line unchanged" | "Still open — line changed" | "Open — unclear"}
  url:          {from pr_comments_list}
}
```

Sort rows: **Still open (any flavor) first, then Unclear, then Outdated,
then Resolved** — the things the user most likely wants to re-check
float to the top.

If the user authored zero comments, skip the section entirely (don't
render an empty table).

Store the classified rows as `classified_previous_comments`.

## Render-Once Rule (orchestrator discipline)

The Substep 4a / 4c / 4d "Report:" lines are STATUS lines — counts
only, no finding bodies. Do not stream finding text during dedup,
verification dispatch, verification compile, or comment classification.
The full finding block (header, severity, description, diff, effort,
voice comment) is rendered EXACTLY ONCE — in Phase 5a (render comments)
or Phase 1 of the walkthrough — against the output-template.md spec.
Never inline during reasoning, never duplicated as "preview" before
the final compile.

If you find yourself emitting finding text outside the render phase,
stop and fold it into the render phase instead. Duplicate finding
bodies in the visible output almost always trace back to a violation
of this rule, not to a dedup miss.

## Next

- If `output_mode == "review-comments"` → read `phases/05a-walkthrough-comments.md`.
- If `output_mode == "plan-fixes"` → read `phases/walkthrough/README.md`.
