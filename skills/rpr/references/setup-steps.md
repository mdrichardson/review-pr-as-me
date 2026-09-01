# Setup Steps Reference

Detailed procedures for Steps 0, 1, and 2. Read this file when executing those steps.

## Contents

- [Step 0: No-URL Local Review Mode](#step-0-no-url-local-review-mode) — git-state detection, build available options, output mode selection, agent context for local reviews, passing user instructions
- [Step 1: Parse URL and Fetch PR](#step-1-parse-url-and-fetch-pr) — GitHub/ADO URL parsing, metadata + diff fetch, existing-comment fetch, building `pr_comments_list`, PR ownership detection → output mode
- [Step 2: Select a Repo Source and Detached Review Worktree](#step-2-select-a-repo-source-and-detached-review-worktree) — prefer a matching workspace, clone only as fallback, fetch exact refs, isolate the PR head

---

## Step 0: No-URL Local Review Mode

When no URL is provided (or only instructions are provided), detect the git state of the **current working directory** and present the user with relevant options. If the user provided instructions, store them as `review_instructions` — they will be passed to all analysis agents in Steps 4 and 5 as additional focus guidance.

### Acknowledge Instructions

If `review_instructions` is set, confirm them back to the user before presenting options. Include this in the `AskUserQuestion` prompt:

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

Based on the detection results, build a list of options. **Only show options that are actually available:**

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

Include brief context counts (files modified, commits ahead) when available to help the user decide.

### Handle Selection

**Option: Review uncommitted changes**
1. Generate the diff: `git diff` (unstaged) combined with `git diff --cached` (staged)
2. Get the list of changed files: `git diff --name-only` and `git diff --cached --name-only`
3. Set `repo_path` to the current working directory
4. Set `review_title` to "Uncommitted changes in `{repo_name}`" (derive repo name from directory)
5. **Skip Steps 1 and 2** — proceed directly to Step 3 (Detect Codebase Standards)

**Option: Review branch vs main**
1. Determine the default branch (`main` or `master` — whichever exists)
2. Generate the diff: `git diff {default_branch}...HEAD`
3. Get the list of changed files: `git diff --name-only {default_branch}...HEAD`
4. Get branch summary: `git log --oneline {default_branch}..HEAD`
5. Set `repo_path` to the current working directory
6. Set `review_title` to "Branch `{branch_name}` vs `{default_branch}`"
7. **Skip Steps 1 and 2** — proceed directly to Step 3 (Detect Codebase Standards)

**Option: Paste a PR link**
1. Ask the user for the URL via `AskUserQuestion`
2. Once provided, proceed to Step 1 as normal

### Output Mode Selection

After the user picks what to review (uncommitted changes or branch vs main), ask a follow-up question to determine the output mode. Use `AskUserQuestion`:

> **What should I do with findings?**
> 1. **Generate review comments** — copy-paste-ready comments in your voice
> 2. **Plan fixes** — walk through each finding and fix the ones you approve

Store the selection as `output_mode`:
- `"review-comments"` → existing Step 6 (compile comments in your voice)
- `"plan-fixes"` → Step 6b (interactive fix walkthrough)

**Skip this question when:**
- User selected "Paste a PR link" → `output_mode` is auto-detected in Step 1 based on PR authorship (review-comments if the PR is someone else's, plan-fixes if it's yours). See "Detect PR Ownership → Select Output Mode" at the end of Step 1.
- This question only applies to local review options where there's no PR metadata to infer from.

Now create the appropriate task list (review-comments or plan-fixes variant) based on the user's selections.

### Agent Context for Local Reviews

When running in local review mode (no PR), the agent prompts in Steps 4-5 should replace PR-specific language:
- Instead of "analyzing a React PR", say "analyzing local code changes"
- Instead of PR title/description, provide the `review_title` and branch name
- The `repo_path`, diff format, and everything else remains the same

### Passing User Instructions

If `review_instructions` is set (the user provided free-text instructions), append the following block to **every** agent prompt — the 4 Sonnet analysis agents (Step 4) AND the Opus verification agent (Step 5):

```
ADDITIONAL REVIEWER INSTRUCTIONS (from the user):
{review_instructions}

Prioritize these instructions. They may narrow your focus to specific areas,
ask you to ignore certain files, or emphasize particular concerns. Treat them
as top-level guidance for this review.
```

This applies in both local review mode and PR URL mode — the invocation syntax supports instructions in either case.

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
az repos pr show --id {id} \
  --org "https://dev.azure.com/{org}" \
  --output json
```

`az repos pr show` does not accept `--project`; the PR URL's project is
validated against the returned `repository.project` instead. Record
`repository.id`, `repository.remoteUrl`, and `repository.sshUrl` for
the thread request and local-checkout matching in Step 2.

Also fetch the GitHub diff with:
`gh pr diff {number} --repo {owner}/{repo}`.

Azure DevOps CLI has no `az repos pr diff` command. For ADO, record
`sourceRefName` and `targetRefName` from the metadata response; Step 2
fetches those exact refs and generates `git diff {baseSha}...{headSha}`
from the detached review worktree.

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
   existing thread (any author) is dropped in Step 5a. No point raising
   what's already on the PR.
2. **Previous-comments status section** — threads authored by the user
   are reported back in Step 6 with their resolution status, so the
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
az devops invoke \
  --area git \
  --resource pullRequestThreads \
  --route-parameters \
    project="{project}" \
    repositoryId="{repository.id}" \
    pullRequestId="{id}" \
  --org "https://dev.azure.com/{org}" \
  --api-version "7.1" \
  --http-method GET \
  --output json
```

The route parameters are required by the
`{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/threads`
resource route. `--api-version "7.1"` supplies its required
`api-version` query parameter. The response object contains the threads
in `value`; iterate over `response.value`, not the response object
itself.

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

Normalize into a single structured list used by Steps 5 and 6:

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
  > apply to the detached review worktree selected in Step 2. If you'd
  > rather edit your existing working copy, stop here and re-run `/rpr`
  > from that directory with no URL.
- Someone else's PR:
  > Reviewing {author}'s PR — using review-comments mode.

This detection is automatic. Do NOT prompt the user to confirm the
mode — the invocation contract still holds; announcing the pick is
enough. The only case the announcement is skipped is when a prior
decision already set `output_mode` (local mode's Step 0 question).

---

## Step 2: Select a Repo Source and Detached Review Worktree

The skill needs full file context to find existing utilities, understand
patterns, and verify findings. A local repository source and an isolated
review worktree are **required** — do not proceed without them.

### Prefer a Matching Copilot Workspace

Before looking for or creating a review clone, inspect the current
Copilot workspace/session checkout:

1. Confirm it is a Git worktree with
   `git -C "{workspace_path}" rev-parse --is-inside-work-tree`.
2. Read every configured remote URL, not just the directory name:

```bash
git -C "{workspace_path}" remote
git -C "{workspace_path}" remote get-url --all {remote}
```

3. Normalize each URL by removing credentials, a trailing slash, and a
   trailing `.git`, URL-decoding path segments, and comparing
   case-insensitively. Accept the workspace only when a remote resolves
   to the exact target identity:
   - GitHub/GHE: `{host}/{owner}/{repo}`.
   - ADO HTTPS or SSH: `{org}/{project}/{repo}`. Compare against the PR
     URL plus `repository.remoteUrl` and `repository.sshUrl` returned by
     `az repos pr show`; support both
     `dev.azure.com/{org}/{project}/_git/{repo}` and
     `ssh.dev.azure.com:v3/{org}/{project}/{repo}` forms.

Do not accept a checkout based only on its folder name or repo basename.
When a remote matches, set `source_repo` to the workspace path and
`source_remote` to that remote. Do not clone and do not switch or modify
the workspace's current branch or working tree.

### Fall Back to an Existing or New Review Clone

If the current workspace does not match:

1. Check `~/.claude/repos/` for an existing clone at
   `~/.claude/repos/{owner}/{repo}` (GitHub) or
   `~/.claude/repos/{org}/{project}/{repo}` (ADO), and validate its
   remote identity with the same rules.
2. Only when neither the workspace nor an existing clone matches, STOP
   and ask:

> "I need a local repository for **{path-to-repo}** to do a thorough review -- without it I can't search for existing utilities or understand codebase patterns. Want me to clone it to `~/.claude/repos/{path-to-repo}`?"

3. If the user agrees, clone it:

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

Set `source_repo` to the validated or new clone and `source_remote` to
its matching remote (normally `origin`).

### Fetch Exact PR Refs Without Switching the Source

Fetch only the base and head refs into namespaced refs. This must not
checkout, reset, or update a branch in `source_repo`.
Set `{prId}` to the parsed `{number}` for GitHub or `{id}` for ADO and
use that same value in every namespaced ref and worktree path below.

**GitHub/GHE:**

```bash
git -C "{source_repo}" fetch {source_remote} \
  "+refs/heads/{baseRefName}:refs/rpr/{prId}/base" \
  "+refs/pull/{prId}/head:refs/rpr/{prId}/head"
```

For GitHub Enterprise, run the fetch against the remote that matched the
enterprise host.

**ADO:**

```bash
git -C "{source_repo}" fetch {source_remote} \
  "+{targetRefName}:refs/rpr/{prId}/base" \
  "+{sourceRefName}:refs/rpr/{prId}/head"
```

Use the full `refs/heads/...` values returned by `az repos pr show`.
Then resolve `baseSha` and `headSha` from the two namespaced refs.

### Create or Reuse a Safe Detached Worktree

Use a path keyed by provider repository, PR ID, and fetched head SHA:

```bash
baseSha=$(git -C "{source_repo}" rev-parse "refs/rpr/{prId}/base")
headSha=$(git -C "{source_repo}" rev-parse "refs/rpr/{prId}/head")
worktree_path=~/.claude/worktrees/review-pr-as-me/{path-to-repo}/pr-{prId}-${headSha}
mkdir -p "$(dirname "$worktree_path")"
git -C "{source_repo}" worktree add --detach "$worktree_path" "$headSha"
```

If that path already exists, reuse it only when it is a registered
worktree at exactly `headSha` and `git status --porcelain` is empty.
Otherwise choose a new unique sibling path. Never reset, clean, remove,
or overwrite an existing or dirty worktree.

For ADO, now generate the review diff from the fetched commits:

```bash
git -C "$worktree_path" diff --find-renames "$baseSha...$headSha"
git -C "$worktree_path" diff --name-only "$baseSha...$headSha"
```

### Set Working Directory

All subsequent steps (standards detection, agent dispatch) operate from
the detached review worktree. Set `repo_path` to `worktree_path` and
pass the full path to each agent.
