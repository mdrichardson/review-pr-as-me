# Phase 1 — Setup

Entry point for the skill after invocation parsing. Sets `repo_path`,
`output_mode`, the diff to review, and the base task list. Subsequent
phases assume these are in place.

## Contents

- [Preconditions](#preconditions) — empty (entry point)
- [Re-entry note](#re-entry-note) — avoid re-cloning / re-prompting if `repo_path` is already set
- [Required Tasks (base list)](#required-tasks-base-list) — setup tasks, always-on analysis agents, wrap-up tasks; conditional agents created in Phase 2
- [Step 0: No-URL Local Review Mode](#step-0-no-url-local-review-mode) — git-state detection, available-options builder, output-mode prompt, agent-context overrides for local reviews
- [Step 1: Parse URL and Fetch PR](#step-1-parse-url-and-fetch-pr) — GitHub/ADO URL parsing, metadata + diff + existing-comment fetch, `pr_comments_list` build, PR-ownership → `output_mode`
- [Step 2: Ensure Local Repo Clone](#step-2-ensure-local-repo-clone) — locate or clone, worktree-safe targeted fetch, PR-branch checkout
- [Next](#next-read-phases02-pre-computemd)

## Preconditions

`01-setup.md` has no preconditions — it's the entry point.

## Re-entry note

If `repo_path` is already set in this conversation, Steps 1–2 are done
— only re-read this file for the Required Tasks reference table or to
handle a mode-switch. Do NOT re-clone, do NOT re-checkout, do NOT
re-prompt for the local-vs-PR option.

## Required Tasks (base list)

Create the base task list up front. Conditional-agent tasks (Agent 5
Scope-match, Agent 6 Suppression-removability, Agent 7
Abstraction-coupling) are NOT created here — they're created at the
end of Phase 2 (pre-compute) once their gating inputs are known.

The setup tasks vary by mode; the always-on analysis core and wrap-up
tasks are mode-dependent in shape but not in agent count.

**Setup tasks** (mode-dependent):
- PR URL mode (create immediately): `"Parse PR URL and fetch metadata"` → `"Fetch existing PR comments"` → `"Ensure local repo clone is up to date"` → `"Detect codebase standards"`
- Local mode (create after Step 0 selection): `"Detect local git state and gather diff"` → `"Detect codebase standards"`

Omit `"Fetch existing PR comments"` when the PR has no comments (first
review pass) — the fetch step itself runs unconditionally in PR mode,
but if the response is empty there's no meaningful progress to track
and the subsequent dedup/status steps are silent no-ops.

**Always-on analysis tasks** (create after setup tasks):
```
"Analyze: Hooks & React patterns"               activeForm: "Analyzing hooks & React patterns"
"Analyze: Reuse & standards"                    activeForm: "Analyzing reuse & standards"
"Analyze: Performance & edge cases"             activeForm: "Analyzing performance & edge cases"
"Analyze: Cross-cutting impact"                 activeForm: "Analyzing cross-cutting impact"
"Analyze: Comment hygiene"                      activeForm: "Analyzing comment hygiene"
"Analyze: Other patterns"                       activeForm: "Analyzing other patterns"
```

These six always-on agents (1, 2, 3, 4, 8, 9) run unconditionally. The
three conditional agents (5, 6, 7) are tasked from Phase 2 — do not
create their tasks here.

**Wrap-up task** (mode-dependent):
- review-comments mode (PR URL): `"Deduplicate findings"` → `"Verify findings ({N} parallel Opus)"` → `"Classify previous comments"` → `"Walk through findings (post per approval)"`
- review-comments mode (local): `"Deduplicate findings"` → `"Verify findings ({N} parallel Opus)"` → `"Walk through findings (copy/paste output)"` (no PR to post to; output rendered to terminal)
- plan-fixes mode: `"Deduplicate findings"` → `"Verify findings ({N} parallel Opus)"` → `"Walk through fixes"` → `"Verify applied fixes"`

Omit `"Classify previous comments"` when the user authored zero
comments on the PR (nothing to classify).

The analysis tasks run in parallel during Phase 3 (analyze). Mark them
all `in_progress` at the same time when dispatching, and `completed`
as each returns.

---

## Step 0: No-URL Local Review Mode

When no URL is provided (or only instructions are provided), detect the
git state of the **current working directory** and present the user
with relevant options. If the user provided instructions, store them
as `review_instructions` — they will be passed to all analysis agents
in Phases 3 and 4 as additional focus guidance.

### Acknowledge Instructions

If `review_instructions` is set, confirm them back to the user before
presenting options. Include this in the `AskUserQuestion` prompt:

> **Review focus:** "focus on the custom hooks and ignore styling changes"

This lets the user catch typos or rephrase before the review runs.

### Git State Detection

Run all four checks **in a single parallel batch** — do not chain them
sequentially. Each is independent; sequential execution wastes wall time
on what should be a one-shot detection. Issue them as four parallel
Bash tool calls in a single message.

```bash
# Call 1: are we in a git repo?
git rev-parse --is-inside-work-tree 2>/dev/null
# Call 2: current branch name
git branch --show-current 2>/dev/null
# Call 3: uncommitted changes
git status --short 2>/dev/null
# Call 4: commits ahead of default branch (try main, fall back to master)
git rev-parse --verify main >/dev/null 2>&1 && \
  git log --oneline main..HEAD 2>/dev/null || \
  git log --oneline master..HEAD 2>/dev/null
```

### Build Available Options

Based on the detection results, build a list of options. **Only show
options that are actually available:**

| Condition | Option |
|-----------|--------|
| `git status --short` has output (staged or unstaged changes exist) | **Review my uncommitted changes** — review all staged and unstaged modifications in the working tree |
| Current branch is NOT `main`/`master` AND there are commits ahead of the default branch | **Review this branch vs main** — review all changes in this branch compared to the remote default branch |
| Always available | **Paste a PR link** — provide a GitHub or ADO pull request URL to review |

**Edge cases:**
- **Not in a git repo:** Show only "Paste a PR link" with a note that the current directory is not a git repository.
- **On main/master with no changes:** Show only "Paste a PR link" with a note that there are no local changes to review.
- **On a branch with uncommitted changes AND commits ahead:** Show all three options.

### Present Options

Use `AskUserQuestion` to present the available options. Example:

> I can review code in a few ways. What would you like?
>
> 1. **Review my uncommitted changes** — 5 files modified, 2 staged
> 2. **Review this branch (`feature/auth`) vs main** — 12 commits ahead
> 3. **Paste a PR link** — provide a GitHub or ADO URL
>
> Pick a number or paste a URL:

Include brief context counts (files modified, commits ahead) when
available to help the user decide.

### Handle Selection

**Option: Review uncommitted changes**
1. Generate the diff: `git diff` (unstaged) combined with `git diff --cached` (staged)
2. Get the list of changed files: `git diff --name-only` and `git diff --cached --name-only`
3. Set `repo_path` to the current working directory
4. Set `review_title` to "Uncommitted changes in `{repo_name}`" (derive repo name from directory)
5. **Skip Steps 1 and 2** — proceed directly to Phase 2 (pre-compute).

**Option: Review branch vs main**
1. Determine the default branch (`main` or `master` — whichever exists)
2. Generate the diff: `git diff {default_branch}...HEAD`
3. Get the list of changed files: `git diff --name-only {default_branch}...HEAD`
4. Get branch summary: `git log --oneline {default_branch}..HEAD`
5. Set `repo_path` to the current working directory
6. Set `review_title` to "Branch `{branch_name}` vs `{default_branch}`"
7. **Skip Steps 1 and 2** — proceed directly to Phase 2 (pre-compute).

**Option: Paste a PR link**
1. Ask the user for the URL via `AskUserQuestion`.
2. Once provided, proceed to Step 1 as normal.

### Output Mode Selection

After the user picks what to review (uncommitted changes or branch vs
main), ask a follow-up question to determine the output mode. Use
`AskUserQuestion`:

> **What should I do with findings?**
> 1. **Generate review comments** — copy-paste-ready comments in your voice
> 2. **Plan fixes** — walk through each finding and fix the ones you approve

Store the selection as `output_mode`:
- `"review-comments"` → runs the per-finding posting walkthrough via `phases/05a-walkthrough-comments.md`. Local-mode reviews still go through this file, but the posting step is replaced with a copy/paste render since there's no PR to post to.
- `"plan-fixes"` → runs the interactive walkthrough via `phases/walkthrough/`.

**Skip this question when:**
- User selected "Paste a PR link" → `output_mode` is auto-detected in Step 1 based on PR authorship (review-comments if the PR is someone else's, plan-fixes if it's yours). See "Detect PR Ownership → Select Output Mode" at the end of Step 1.
- This question only applies to local review options where there's no PR metadata to infer from.

Now create the appropriate task list (review-comments or plan-fixes
variant) based on the user's selections.

### Agent Context for Local Reviews

When running in local review mode (no PR), the agent prompts in Phase 3
(analyze) and Phase 4 (verify) should replace PR-specific language:
- Instead of "analyzing a React PR", say "analyzing local code changes"
- Instead of PR title/description, provide the `review_title` and branch name
- The `repo_path`, diff format, and everything else remains the same

### Passing User Instructions

If `review_instructions` is set (the user provided free-text instructions),
append the following block to **every** agent prompt — the analysis
agents (Phase 3) AND the verification agents (Phase 4):

```
ADDITIONAL REVIEWER INSTRUCTIONS (from the user):
{review_instructions}

Prioritize these instructions. They may narrow your focus to specific areas,
ask you to ignore certain files, or emphasize particular concerns. Treat them
as top-level guidance for this review.
```

This applies in both local review mode and PR URL mode — the invocation
syntax supports instructions in either case.

---

## Step 1: Parse URL and Fetch PR

### URL Detection

Parse the provided URL to determine the source and extract identifiers:

**GitHub:**
- Pattern: `github.com/{owner}/{repo}/pull/{number}`
- Extract: `owner`, `repo`, `number`

**ADO:**
- Pattern: `dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{id}`
- Legacy: `{org}.visualstudio.com/{project}/_git/{repo}/pullrequest/{id}`
- Extract: `org`, `project`, `repo`, `id`

### Fetch PR Metadata

**GitHub:**
```bash
gh pr view {number} --repo {owner}/{repo} --json title,body,author,files,additions,deletions,baseRefName,headRefName,url
```

**ADO:**
```bash
az repos pr show --id {id} --org "https://dev.azure.com/{org}" --project "{project}" --output json
```

Also fetch the diff:
- GitHub: `gh pr diff {number} --repo {owner}/{repo}`
- ADO: `az repos pr diff --id {id} --org "https://dev.azure.com/{org}" --project "{project}"`

If the ADO diff command is unavailable, fetch the diff via:
`az repos pr list --id {id}` to get source/target branches, then
`git diff {target}...{source}`.

Record from the metadata:
- PR title, description, URL
- PR author (GitHub `author.login` / ADO `createdBy.uniqueName` + `createdBy.mailAddress`)
- Files changed count, additions, deletions
- Base and head branch names
- List of changed file paths

### Fetch Existing PR Comments

After metadata, pull every existing comment/thread on the PR. These feed
two features in later steps:

1. **Silent dedup** — any new finding whose file + line overlaps an
   existing thread (any author) is dropped in Phase 4 dedup. No point
   raising what's already on the PR.
2. **Previous-comments status section** — threads authored by the user
   are reported back in Phase 5a with their resolution status, so the
   user can see at a glance how each prior comment was handled.

Fetch each thread's author, file path, line, body, and resolution
status. Prefer thread-level APIs (they expose `isResolved` / status
cleanly and collapse reply chains into single entities). Pull issue-
level (PR-wide) comments too — they have no file/line but still count
for status reporting.

**GitHub** (review threads via GraphQL, then top-level issue comments):

```bash
# Review threads — includes resolution + outdated flags
gh api graphql -f query='{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 100) {
        nodes {
          isResolved
          isOutdated
          comments(first: 50) {
            nodes {
              author { login }
              body
              path
              line
              originalLine
              url
              createdAt
            }
          }
        }
      }
    }
  }
}'

# Top-level PR issue comments — no file/line, status-report only
gh api "repos/{owner}/{repo}/issues/{number}/comments" --paginate \
  --jq '[.[] | {author: .user.login, body, url: .html_url, createdAt: .created_at}]'

# Current GitHub user (to tag "mine" vs "others")
gh api user --jq .login
```

For GitHub Enterprise, prefix with `GH_HOST={host}`.

**ADO** (threads expose status + file/line in a single call):

```bash
az repos pr thread list --id {id} \
  --org "https://dev.azure.com/{org}" --project "{project}" \
  --output json
```

Each thread has `status` (`active`, `fixed`, `closed`, `wontFix`,
`pending`, `byDesign`, `unknown`), `threadContext.filePath`,
`threadContext.rightFileStart.line`, a `comments[]` array, and
`isDeleted`. Skip threads where `isDeleted: true` or `threadContext` is
null (PR-wide / system threads — keep those only if they contain a
user comment worth status-tracking).

For ADO, "mine" = any thread whose first non-system comment's
`author.uniqueName` (or `author.mailAddress`) matches
`git config user.email`. GitHub uses the login from `gh api user`.

### Build `pr_comments_list`

Normalize into a single structured list used by Phases 4 and 5a:

```
pr_comments_list: [
  {
    id:         {platform-specific thread id, for linking in the status section}
    author:     {github login or ADO unique name}
    is_mine:    {true if author matches the current user}
    file:       {path relative to repo root, or null for PR-wide comments}
    line:       {line number in the PR's head commit, or null}
    body:       {first comment in the thread — full text}
    body_excerpt: {first ~100 chars of body, single-line, for table render}
    status:     {"open" | "resolved" | "outdated"}
    url:        {deep link to the thread/comment}
    created_at: {ISO timestamp}
  },
  ...
]
```

Status mapping:
- GitHub: `isResolved: true` → `"resolved"`; `isOutdated: true` (and
  not resolved) → `"outdated"`; else `"open"`.
- ADO: `fixed`/`closed`/`byDesign` → `"resolved"`; `wontFix` →
  `"resolved"` (author decided not to fix, not pending); `active`/
  `pending` → `"open"`; `unknown` → `"open"` (conservative default).
- Top-level issue comments (GitHub) and PR-wide ADO threads with no
  file/line always resolve to `"open"` — there's no resolve flag on
  them, and they're informational.

If the PR has zero comments (common on first review), `pr_comments_list`
is empty and the dedup + status-section steps silently no-op.

**Local review mode has no PR**, so `pr_comments_list = []`. The
comment-fetch step is skipped entirely when `repo_path` was set from a
local git state instead of a PR URL.

### Detect PR Ownership → Select Output Mode

After metadata and comments are fetched, decide `output_mode` by
comparing PR author against the current user. Reviewing your own PR in
review-comments mode makes no sense — generating copy-paste voice
comments to leave on your own PR is busywork. Switch to **plan-fixes**
mode and walk through the findings interactively instead.

Rule:

- **PR author == current user** → `output_mode = "plan-fixes"`
- **PR author != current user** → `output_mode = "review-comments"` (default)

Identity comparison:

- **GitHub**: PR `author.login` vs `gh api user --jq .login` (already
  fetched above for comment ingestion — reuse that value, don't re-call).
- **ADO**: PR `createdBy.uniqueName` OR `createdBy.mailAddress` vs
  `git config user.email`. Match on either field — ADO's `uniqueName`
  is often the email itself, but not always.

Announce the detected mode before proceeding (one line, not a prompt —
the user can course-correct after the fact by stopping and re-invoking):

- Own PR:
  > Detected this PR is yours — using plan-fixes mode to walk through
  > fixes interactively instead of generating voice comments. Edits will
  > apply to the local clone at `~/.claude/repos/{owner}/{repo}`. If
  > you'd rather edit your own working copy, stop here and re-run
  > `/review-pr-as-me` from that directory with no URL.
- Someone else's PR:
  > Reviewing {author}'s PR — using review-comments mode.

This detection is automatic. Do NOT prompt the user to confirm the
mode — the invocation contract still holds; announcing the pick is
enough. The only case the announcement is skipped is when a prior
decision already set `output_mode` (local mode's Step 0 question).

---

## Step 2: Ensure Local Repo Clone

The skill needs full file context to find existing utilities, understand
patterns, and verify findings. A local clone is **required** — do not
proceed without one.

### Locate or Clone the Repo

1. **Check `~/.claude/repos/`** for an existing clone at `~/.claude/repos/{owner}/{repo}` (GitHub) or `~/.claude/repos/{org}/{project}/{repo}` (ADO).
2. **If not found**, STOP and ask the user:

> "I need a local clone of **{owner}/{repo}** to do a thorough review -- without it I can't search for existing utilities or understand codebase patterns. Want me to clone it to `~/.claude/repos/{owner}/{repo}`?"

3. **If the user agrees**, clone it:

**GitHub:**
```bash
mkdir -p ~/.claude/repos/{owner}
gh repo clone {owner}/{repo} ~/.claude/repos/{owner}/{repo}
```

**ADO:**
```bash
mkdir -p ~/.claude/repos/{org}/{project}
git clone https://dev.azure.com/{org}/{project}/_git/{repo} ~/.claude/repos/{org}/{project}/{repo}
```

**GitHub Enterprise** (e.g. `microsoft.ghe.com`):
```bash
mkdir -p ~/.claude/repos/{owner}
GH_HOST={host} gh repo clone {owner}/{repo} ~/.claude/repos/{owner}/{repo}
```

### Update the Clone (worktree-safe, targeted fetch)

**Always** ensure the clone is current before scanning, even if it
already existed — but do NOT use `git fetch --all --prune` or
`git checkout {baseRefName} && git pull`. Both assume the local clone
is on a specific branch and that all remotes need updating; on
multi-remote enterprise repos that's wasted network, and on a clone
that's mid-checkout from a previous PR review, the `git checkout` step
can fail or move the wrong ref.

Instead, fetch only the two refs we actually need, and update the local
`{baseRefName}` ref **without changing what's checked out**:

```bash
cd ~/.claude/repos/{path-to-repo}

# Fetch only the base and head refs. Update local {baseRefName} ref
# even when not checked out (the `:{baseRefName}` is fast-forward-only —
# safe). The head fetch puts the PR's tip in FETCH_HEAD for `gh pr
# checkout` / `git checkout FETCH_HEAD` to use.
git fetch origin {baseRefName}:{baseRefName} {headRefName}
```

This works regardless of:
- Which branch is currently checked out in the clone (no implicit
  checkout)
- Whether the user is running `/review-pr-as-me` from a worktree of
  another repo (the clone at `~/.claude/repos/...` is independent)
- Whether `{baseRefName}` is `main`, `master`, `develop`, a release
  branch, or anything else (it's just a ref name to fetch)

If the `{baseRefName}:{baseRefName}` form fails because the local ref
diverged (rare — only happens if a previous review left local commits
on the base branch), fall back to a non-fast-forward update:
```bash
git fetch origin {baseRefName} && git update-ref refs/heads/{baseRefName} FETCH_HEAD
```

### Checkout the PR Branch

From the local clone directory:

**GitHub:**
```bash
gh pr checkout {number}
```
For GitHub Enterprise, prefix with `GH_HOST={host}`. (`gh pr checkout`
performs its own targeted fetch of the PR head, so the prior step's
`{headRefName}` fetch is redundant only on the GitHub path — keep it
anyway; an extra fetched ref is cheaper than a branch in the wrong
state.)

**ADO:**
```bash
git checkout FETCH_HEAD
```
The `{headRefName}` was already fetched by the previous step, so its
tip is in `FETCH_HEAD` — no second `git fetch` needed.

### Set Working Directory

All subsequent phases (standards detection, agent dispatch) operate
from this local clone path. Pass the full path to each phase as
`repo_path`.

---

## Next: read `phases/02-pre-compute.md`.
