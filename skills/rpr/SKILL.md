---
name: rpr
description: "Review React PRs and local code changes. Use this skill whenever the user provides a GitHub or ADO pull request URL, says 'review this PR', 'review my changes', 'review this branch', 'look at this diff', 'code review', or invokes /rpr. Also trigger when the user pastes a github.com/*/pull/* or dev.azure.com/*/pullrequest/* link. When no URL is provided, detects local git state and offers to review uncommitted changes or branch diff. Generates copy-paste-ready PR review comments in the user's personal voice. Never posts automatically."
---

# Review PR As Me

## Host compatibility

The phase and reference files use Claude Code terminology as their
portable baseline. When running under GitHub Copilot CLI, apply these
mappings throughout the workflow:

- Prefer the current Copilot workspace/session checkout when it matches
  the target repository. If a separate clone is needed, use
  `~/.copilot/repos/review-pr-as-me/{owner}/{repo}` instead of
  `~/.claude/repos/{owner}/{repo}`.
- Use `~/.copilot/.cache/review-pr-as-me/v1/` as the cache root and
  `~/.copilot/VOICE.md` as the voice file.
- Map legacy tools as follows: `Read` -> `view`, `Glob` -> `glob`,
  `Grep` -> `rg`, `Bash` -> `powershell` on Windows, `Agent` -> `task`,
  `AskUserQuestion` -> `ask_user`, and `Edit` -> `apply_patch`.
- Use Copilot progress reporting plus SQL `todos` where the workflow
  calls for `TaskCreate` or `TaskUpdate`.
- Map model tiers to available Copilot models: `opus` ->
  `claude-opus-4.8`, `sonnet` -> `claude-sonnet-4.6`, and `haiku` ->
  `claude-haiku-4.5`.
- On Windows, translate shell snippets to PowerShell or use Copilot's
  built-in file/search tools rather than running Unix snippets verbatim.

These mappings are host adaptations, not workflow changes. Other hosts
should continue using their native tool names, model aliases, paths, and
progress mechanisms.

Generate React PR review comments in your voice. Dispatches parallel
analysis agents, then compiles findings into copy-paste-ready output
organized by file, OR walks the user through fixes interactively when
the PR is theirs.

## Hard rules

This skill is a *reviewer*, not a fixer. The reviewer edits nothing
without explicit per-finding approval, and posts nothing in bulk —
review output is for the user to direct, not for the skill to apply
unilaterally. Concretely:

- Never use Edit or Write tools on repo files — **except** in
  plan-fixes mode (`output_mode = "plan-fixes"`), where Edit is
  allowed but only for findings the user explicitly approves with
  "fix" or "fix all". Never auto-apply without user confirmation;
  the walkthrough exists specifically so the user has the wheel.
- Never post comments in bulk and never post without explicit
  per-finding approval. The review-comments walkthrough is the ONLY
  place writes against the PR are permitted (`gh api … /comments`,
  `gh pr comment`, `az repos pr thread create`) — and only after the
  user picks "Post comment" for that specific finding. "Skip" and
  "I will comment" both post nothing — "I will comment" is a manual
  hand-off where the user posts in the PR UI themselves. No batch
  posting, no "post all", no implicit defaults.

## Invocation

```
/rpr [url] [instructions]
```

Arguments are optional and parsed left-to-right:
1. **Check for a URL first:** If any token starts with `http://`, `https://`, or matches a known PR URL pattern (`github.com/…/pull/`, `dev.azure.com/…/pullrequest/`), extract it as the URL. Everything after the URL is treated as instructions.
2. **Remaining text = instructions:** If no URL was found, the entire argument is free-text instructions.
3. **No argument at all:** Proceed to Step 0 (in `phases/01-setup.md`) with no instructions.

**Examples:**
- `/rpr` — no URL, no instructions → Step 0
- `/rpr focus on hooks` — no URL, instructions → Step 0 with `review_instructions`
- `/rpr https://github.com/org/repo/pull/42` — URL, no instructions → Step 1
- `/rpr https://github.com/org/repo/pull/42 focus on hooks` — URL + instructions → Step 1 with `review_instructions`

If a URL is present, jump to Step 1 inside `phases/01-setup.md`. If
instructions are present, store them as `review_instructions`.
Otherwise, proceed to Step 0 in `phases/01-setup.md`.

**The invocation is a contract.** When a URL is provided, every
decision the skill would otherwise prompt for has already been made or
is auto-detected: a separate clone goes under the host-specific review
clone root described above, the output mode is auto-detected from PR authorship
(review-comments for others' PRs, plan-fixes when the PR is yours —
see `phases/01-setup.md` "Detect PR Ownership → Select Output Mode"),
and any review focus rides in via the `[instructions]` slot as
`review_instructions`. Re-prompting the user for these things forces
them to repeat decisions they already encoded in the command, and
breaks the one-shot flow that URL-with-instructions invocation is
specifically designed for. The only user prompt permitted before Phase
2 is the Step-2 "clone not found at {path} — shall I create it?"
question, and only when that directory is genuinely missing. If you
feel tempted to ask anything else, re-read the invocation — the answer
is already there.

## Progress tracking

Use the host's progress/task mechanism up front and as phases run — mark
each task `in_progress` when starting and `completed` when done. The
detailed task list per mode (setup, always-on analysis agents,
conditional agents, wrap-up) lives in `phases/01-setup.md`.

## Phase Index (read JIT)

Each phase file is self-contained and opens with its own preconditions
+ re-entry guard. Read them only when entering the phase.

1. After invocation parsing → read `phases/01-setup.md` (Steps 0/1/2: local-vs-PR detection, URL parsing, clone management).
2. After setup completes (mode known, repo ready) → read `phases/02-pre-compute.md` (standards, utility index, import graph, suppressions, pre-read, categorize, filter, strip).
3. After pre-compute → read `phases/03-analyze.md` (dispatch up to 9 parallel analysis agents).
4. After all analysis agents return → read `phases/04-verify.md` (dedup, per-finding verify, classify previous comments).
5. After verification, if `output_mode == "review-comments"` → read `phases/05a-walkthrough-comments.md` (interactive per-finding walkthrough that posts approved comments via `gh`/`az`).
6. After verification, if `output_mode == "plan-fixes"` → read `phases/walkthrough/README.md` (entry point), then each sub-phase file as you enter it (`01-summary.md` → `02-decisions.md` → `03-apply.md` → `04-verify.md` → `05-rescan.md`).

## Cross-cutting invariants

### Render-Once Rule

Finding text is emitted EXACTLY ONCE — per finding, when that finding
is presented to the user. In review-comments mode that's the
per-finding step inside `phases/05a-walkthrough-comments.md`; in
plan-fixes mode it's Phase 1 of the walkthrough. The intermediate
"Report:" lines in Phase 4 (dedup, verify, classify) are status lines
— counts only, no finding bodies. Do not stream finding text during
dedup reasoning, verification dispatch, verification compile, or
comment classification. If you find yourself emitting finding text
outside the render-as-presented step, stop and fold it back in.

### Anti-Nitpick Filter

Every finding must pass the 4-question test defined in
`references/output-template.md`. See that file for the full filter and
severity definitions.

### Per-Project Cache

Consult `references/per-project-cache.md` before any Phase 2
(pre-compute) work. It is the source of truth for cache layout,
fingerprint keys, read-then-write pattern, garbage collection, and
bypass rules. Phase 2 references it but does not redefine the
mechanics.
