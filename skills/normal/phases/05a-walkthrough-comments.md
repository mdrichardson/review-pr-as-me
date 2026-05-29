# Phase 5a — Walkthrough Comments (review-comments mode)

Walk the verified findings one at a time. For each finding, render the
unified finding block (description + voice comment), then ask the user
to **Post comment**, **I will comment**, or **Skip**. Post inline PR
comments via `gh` / `az` only when the user picks **Post comment**.
**I will comment** is a manual hand-off — the skill posts nothing and
the user handles posting in the PR UI. Nothing is posted in bulk and
nothing is posted without explicit per-finding approval.

## Preconditions

Assumes the following state is set:
- `findings[]` — the verified list set by `phases/04-verify.md`.
- `pr_comments_list` (may be empty) — set by `phases/01-setup.md`.
- `classified_previous_comments` (may be empty) — set by `phases/04-verify.md` Substep 4d.
- For PR mode: `pr_url`, `pr_number`, `owner`, `repo` (GitHub) or
  `id`, `org`, `project` (ADO), `headRefOid` (GitHub head SHA), and
  optional `gh_host` for GitHub Enterprise.

If any required precondition is absent or empty, STOP and re-read
`phases/04-verify.md` (the producing phase). Do NOT proceed with
defaults, do NOT improvise, do NOT silently continue.

## Empty-findings gate

If `findings[]` is empty, emit
`"No confirmed findings — nothing to post. Done."` and stop. Do not
render the header, do not enter the loop.

## Local-mode short-circuit

If the review is local (no `pr_url` — set in Step 0 of
`phases/01-setup.md`), skip the per-finding posting prompt entirely
and render every finding via the bulk format described in
[Local-mode render](#local-mode-render) at the bottom of this file.
There is no PR to post to; the user copy/pastes whatever they want.

## Order of operations (PR mode)

1. [Render the header + overview](#step-1-render-the-header--overview).
2. [Cache the head commit SHA once](#step-2-cache-the-head-commit-sha).
3. [Loop findings one at a time](#step-3-per-finding-loop).
4. [Render the final summary](#step-4-final-summary).

---

## Step 1: Render the header + overview

Read `~/.claude/VOICE.md` (voice rules used in the per-finding
**Comment (in my voice)** block) and `references/output-template.md`
(format for the header, glance table, and previous-comments table).

Then emit, exactly once at the top of the walkthrough:

1. The **PR header line** — title, PR link, files / lines stats, and
   the **Findings:** count line. Include the
   `— {N} suppressed by existing comments` suffix if Phase 4 Substep 4a
   dropped any findings.
2. The **"My previous comments on this PR"** section — render only if
   `classified_previous_comments` is non-empty. Sort per the template
   (Still open → Open — unclear → Outdated → Resolved). Skip entirely
   if `pr_comments_list` is empty or the user has zero prior comments.
3. The **"Findings at a glance"** table — one row per finding,
   numbered, sorted by file then by line.
4. A one-line walkthrough hint:

   > Walking each finding now. For each one you'll see the description,
   > the comment in your voice, and three choices: Post comment /
   > I will comment / Skip.

Do not render the unified finding blocks here. They render JIT in
Step 3 as each finding comes up.

---

## Step 2: Cache the head commit SHA

Inline PR comments on GitHub require `commit_id`. Fetch the PR head
SHA once and cache it for the loop:

**GitHub** (including GHE — prefix with `GH_HOST={gh_host}` when set):
```bash
gh pr view {pr_number} --repo {owner}/{repo} --json headRefOid -q .headRefOid
```

**ADO**: thread-create takes `--path` plus `--right-file-start-line`
directly, no SHA needed.

Store the result in `head_sha`. If the fetch fails, fall back to
re-extracting it from `phases/01-setup.md`'s metadata fetch (the same
PR metadata call already returned `headRefOid` — reuse it instead of
re-calling).

---

## Step 3: Per-finding loop

Sort `findings[]` by `(file, line)`. For each finding, do the
following in order. Do not parallelize — each finding waits on user
input before moving on.

### 3a. Render the finding

Render the unified finding block from
`references/output-template.md` (single-fix or multi-fix shape, per
the verifier output). The block ALWAYS includes the
**Comment (in my voice)** section in this mode — the voice comment is
what gets posted if the user approves.

Follow the Diff block rule (omit the block when `fix_snippet: null`).

### 3b. Pick the default recommendation

Severity-driven, no exceptions:

| Severity      | Recommended option |
|---------------|--------------------|
| `must-fix`    | **Post comment**   |
| `should-fix` | **Post comment**   |
| `suggestion` | **Skip**           |

The recommended option is marked with `(Recommended)` in the
`AskUserQuestion` choice list, and is listed FIRST.

### 3c. Ask the user

Use `AskUserQuestion` with exactly three options. The question text
is short and references the finding by its glance-table number:

```
question: "Finding {N} of {total}: {file}:{line} — {severity}. What now?"
header: "Comment action"
options:
  - { label: "Post comment",    description: "Post the voice comment above to the PR as an inline review comment." }
  - { label: "I will comment",  description: "Don't post — I'll go to the PR and post a comment myself. The voice comment above is there to copy/adapt." }
  - { label: "Skip",            description: "Don't post anything for this finding. Move on." }
```

Reorder so that the recommended option (per 3b) is FIRST in the array
and append `(Recommended)` to its `label`. The other two follow in
the order shown above.

`AskUserQuestion` is the ONLY mechanism for the per-finding prompt —
do not fall back to free-text "type 1/2/3" prompts or yes/no
questions, even if the same finding cycles back after an error.

### 3d. Handle the answer

**Skip** → increment `skipped_count`. Move to the next finding.

**Post comment** →
- Use the **Comment (in my voice)** body verbatim from the rendered
  finding block as `body_text`.
- Post per the [Posting commands](#posting-commands) section below.
- On success, increment `posted_count` and record the returned
  comment URL.
- On failure, surface the error to the user and ask once whether to
  retry, skip, or abort the walkthrough.

**I will comment** →
- Post nothing. The voice comment is already rendered in 3a for the
  user to copy or adapt manually in the PR UI.
- Increment `manual_count`. Do NOT prompt the user for text and do
  NOT call any posting command.
- Move to the next finding.

After posting (or skipping / handing off manually), update the
in-progress TaskUpdate entry for this finding to `completed`, then
move to the next finding.

### 3e. Re-entry / interruption

If the user interrupts mid-loop (e.g. asks a clarifying question),
answer concisely and then resume at the same finding — do NOT
restart the loop or re-render the header. The walkthrough is a
single linear pass; track `current_finding_index` to support resume.

---

## Step 4: Final summary

After the loop completes, render one short summary line:

```
Walkthrough complete. Posted: {posted_count}. Will comment manually: {manual_count}. Skipped: {skipped_count}. Total findings: {N}.
```

Then list the comment URLs, one per line, for the ones that were
posted (these are what the user will click through to verify or
resolve threads later). No second header, no recap of finding bodies
— the bodies were already shown once in the loop.

If `posted_count == 0`, omit the URL list. Emit
`"No comments posted — you can re-run the skill anytime."` when
`manual_count == 0` too; if `manual_count > 0`, instead emit
`"No comments posted by the skill — {manual_count} flagged for manual posting. See the rendered comments above."`

---

## Posting commands

Pick the platform from the URL set in Phase 1.

### GitHub (and GitHub Enterprise)

Inline comment on a file/line — the common case:
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --method POST \
  -f body="{body_text}" \
  -f commit_id="{head_sha}" \
  -f path="{file}" \
  -F line={line} \
  -f side=RIGHT
```

GHE: prefix the command with `GH_HOST={gh_host}`.

PR-wide comment (only when the finding has no file/line):
```bash
gh pr comment {pr_number} --repo {owner}/{repo} --body "{body_text}"
```

Pass `body_text` via a heredoc to preserve newlines and quote-safe
content. Example (Bash):
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --method POST \
  -f body="$(cat <<'EOF'
{body_text}
EOF
)" \
  -f commit_id="{head_sha}" \
  -f path="{file}" \
  -F line={line} \
  -f side=RIGHT
```

Capture the returned JSON's `html_url` to show in the final summary.

### ADO

Inline comment on a file/line:
```bash
az repos pr thread create \
  --id {id} \
  --org "https://dev.azure.com/{org}" \
  --project "{project}" \
  --comments "{body_text}" \
  --status active \
  --path "{file}" \
  --right-file-start-line {line} \
  --right-file-end-line {line} \
  --right-file-start-offset 1 \
  --right-file-end-offset 1 \
  --output json
```

PR-wide comment (no `--path` / line flags):
```bash
az repos pr thread create \
  --id {id} \
  --org "https://dev.azure.com/{org}" \
  --project "{project}" \
  --comments "{body_text}" \
  --status active \
  --output json
```

Capture the returned thread's `_links.self.href` (or assemble the
deep link from `id` + thread id) for the final summary.

### Posting hard rules

- ONE comment per posting call. No batching.
- NEVER call a posting command without the user having just answered
  "Post comment" for THAT specific finding. "Skip" and
  "I will comment" both mean the skill posts nothing — no implicit
  post, no fallback to gh/az.
- NEVER use a "Post all" / "Approve all" shortcut. Each finding gets
  its own three-option prompt via `AskUserQuestion`.
- If a posting command fails, do NOT silently retry against a
  different endpoint or with different flags. Surface the error and
  let the user decide.

---

## Local-mode render

When there's no PR (local review), skip Steps 2–4 entirely and render
every finding in one shot using the original bulk format:

1. Header line + (if applicable) "Findings at a glance" table.
2. File-grouped unified finding blocks, sorted by `(file, line)`,
   separated by `===` per `references/output-template.md`.
3. Stop. There's nothing to post to; the user copy/pastes what they
   want.

No previous-comments section in local mode — there are no PR
comments to classify.

---

## Next: stop. Walkthrough complete.
