---
name: review-pr-as-me:fast
description: "Fast variant of review-pr-as-me:normal — same workflow but with every model stepped down one tier and Agent 5 (scope match) skipped. Use on subsequent runs after an initial review-pr-as-me:normal pass, when you're re-verifying fixes rather than reviewing a PR fresh. Triggers on /review-pr-as-me:fast, 'fast review', 'quick re-review', 're-review after fixes'."
allowed-tools: [Read, Agent, Glob, Grep, Bash, Edit, AskUserQuestion, Skill, ExitPlanMode]
---

# Review PR As Me — Fast

## Step 0 — Load the normal workflow first

Read `../normal/SKILL.md` (relative to this file's directory) in full
before doing anything else. If the relative path fails because the
harness resolves the skill from a different working directory, fall
back to the absolute path
`~/.claude/plugins/mdrichardson/review-pr-as-me/skills/normal/SKILL.md`.

If neither resolves, halt and report the error — fast mode only
documents the deltas from normal, so it can't run a review without
loading normal first.

**Important:** the normal skill is now a thin router. Reading it gives
you the phase index, invocation contract, hard rules, and cross-cutting
invariants — but the phase implementations live in
`../normal/phases/*.md` and are loaded JIT as you enter each phase.
The deltas below are scoped per phase file so you know exactly where to
apply them when those files are loaded.

## Deltas to apply when reading phase files

| Delta | Applies in |
|-------|------------|
| Model stepdown table (below) | `../normal/phases/03-analyze.md` (analysis dispatch) and `../normal/phases/04-verify.md` (per-finding verification) |
| Skip Agent 5 (Scope Match) rule | `../normal/phases/03-analyze.md` (dispatch) and the conditional-task creation at the end of `../normal/phases/02-pre-compute.md` |
| Task-list adjustments (verification task wording, omit Scope-match) | `../normal/phases/01-setup.md` (Required Tasks block) and end of `../normal/phases/02-pre-compute.md` (conditional tasks) |
| Cache-hit shortcut announcement | `../normal/phases/02-pre-compute.md` (PR-level slot check at the top of the phase) |

Both Phase 3 (analyze) and Phase 4 (verify) open with a fast-mode
override callout — when you see it, return here to apply the
stepdowns.

## 1. Model stepdown (Phase 3 dispatch and Phase 4 per-finding verification)

| Agent | Normal | Fast |
|-------|--------|------|
| Agent 1 — Hooks & React Patterns | opus | **sonnet** |
| Agent 2 — Reuse & Standards | sonnet | **haiku** |
| Agent 3 — Performance & Edge Cases | sonnet | **haiku** |
| Agent 4 — Cross-Cutting Impact | sonnet | **haiku** |
| Agent 6 — Suppression Removability | haiku | **haiku** |
| Agent 7 — Abstraction & Coupling Boundaries | opus | **sonnet** |
| Agent 8 — Comment Hygiene | haiku | **haiku** |
| Agent 9 — Other | sonnet | **haiku** |
| Phase 4 per-finding verification | opus | **sonnet** |

Agent 6 already runs at haiku — there is no lower tier to step down to.
The row is listed for documentation: fast mode still dispatches Agent 6
when the `suppression_list` is non-empty (same condition as normal
mode).

Agent 8 also runs at haiku in normal mode — no lower tier to step down
to. Fast mode dispatches Agent 8 unconditionally, same as normal.

## 2. Skip Agent 5 (Scope Match) — but keep Agent 6

Do not dispatch Agent 5. Rationale: PR title/body rarely changes between
runs, so scope-match output is the most stable across iterations —
dropping it is the cleanest "75% of subagents" cut, and it's already the
only haiku agent so there's no lower tier to step down to.

**Do NOT also skip Agent 6.** Fast mode exists for re-reviewing after
fixes, and fixes are the single most likely moment new suppressions get
introduced (authors silence a lint/type error while patching something
else). Dropping the suppression check in the mode specifically designed
to catch post-fix regressions defeats its purpose. Cost is negligible
since Agent 6 is already haiku.

## Everything else is unchanged

The rest of the normal skill — its Phase 1 (setup, clone, local-vs-PR
detection, **PR-comment fetching** in PR mode), Phase 2 (pre-compute:
standards detection, utility index, import graph, suppression list,
pre-reads, **per-project cache lookups**), Phase 3 parallel dispatch of
Agents 1–4 + 7–9 (with the **prompt-cache-friendly structure**
[A]+[B]+[C]), all of Phase 4 (dedup + **silent dedup-against-existing-PR-comments**
+ per-finding verification with **per-file batching when ≥3 findings
cluster** + compile + **previous-comment status classification**),
Phase 5a (review-comments per-finding posting walkthrough — see
`phases/05a-walkthrough-comments.md` — including the header,
"My previous comments on this PR" section, suppressed-count header
suffix, and per-finding Post comment / I will comment / Skip prompt), and
the plan-fixes walkthrough under `phases/walkthrough/` — applies verbatim,
with the model assignments above substituted and Agent 5 omitted. (You
already loaded the router in Step 0; the phase files are loaded JIT
as you enter each phase.)

**Cache-hit shortcut for fast mode.** The PR-level cache slot
(`pr-{number}-{headSha}.json` from `references/per-project-cache.md`)
is the dominant fast-mode optimization. When fast mode runs after a
normal-mode review with no new commits since, Phase 2 (pre-compute)
short-circuits entirely. Always announce a Phase 2 cache hit
explicitly to the user: `"Phase 2: cache hit (PR head unchanged) —
skipping pre-compute."` so they know fast mode is genuinely fast and
not waiting on a re-scan.

**PR-comment reading is orthogonal to agent dispatch.** The fetch is a
single `gh api` / `az repos pr thread list` call, dedup is a
file+line-range comparison, and status classification is a deterministic
lookup against GitHub's `isResolved`/`isOutdated` or ADO's `status`
field. None of it goes through a subagent, so there is no model tier
to step down. Fast mode does the same work as normal mode here.

Task-list adjustments (apply at the points the normal flow creates tasks):
- Omit any "Scope match" analysis task (Phase 2 end-of-phase conditional task creation).
- Keep the "Suppression removability" analysis task (no change from normal — still conditional on a non-empty `suppression_list`).
- Keep the "Fetch existing PR comments" setup task and the "Classify previous comments" wrap-up task unchanged from normal mode (both conditional on PR URL mode + non-empty comment list).
- The wrap-up verification task becomes `"Verify findings ({N} parallel Sonnet)"`.

## When to use fast vs normal

- **`/review-pr-as-me:normal`** — first-pass review, unfamiliar PR, high-stakes
  merge, or whenever scope-match against the PR description matters.
- **`/review-pr-as-me:fast`** — re-review after fixes have landed, iterating
  on a PR, or any subsequent pass where scope has already been verified.
