# Phase 2 — Pre-Compute

Detect codebase standards, build the utility index, compute the import
graph, extract suppression entries from the diff, pre-read changed
files, and stamp out the conditional-agent tasks for Phase 3.

## Contents

- [Preconditions](#preconditions)
- [Cache check first (PR-level shortcut)](#cache-check-first-pr-level-shortcut) — consult `references/per-project-cache.md`; short-circuit on hit
- [Detect codebase standards](#detect-codebase-standards) — config + naming/import inference
- [Pre-compute Utility Index](#pre-compute-utility-index) — exports under utils/helpers/lib/shared/common/hooks + barrels
- [Pre-compute Import Graph (for Agent 4)](#pre-compute-import-graph-for-agent-4) — single repo-scan + awk bucketing (the bash block lives here, not in the cache reference)
- [Categorize Changed Files](#categorize-changed-files) — Component / Hook / Utility / Style / Test / Config / Other
- [Filter Trivial Files](#filter-trivial-files) — drop <5-line changes from primary lists
- [Strip Noise from Diff](#strip-noise-from-diff) — collapse pure renames, drop pure deletes
- [Pre-read Changed Files (speed optimization)](#pre-read-changed-files-speed-optimization) — single parallel batch of Reads
- [Pre-compute Suppression List (for Agent 6)](#pre-compute-suppression-list-for-agent-6) — awk over added diff lines + rule-config summary
- [End-of-phase: create conditional-agent tasks](#end-of-phase-create-conditional-agent-tasks) — Agent 5 / 6 / 7 tasks gated on inputs
- [Cache write](#cache-write) — snapshot the whole phase output into the PR-level slot
- [Next](#next-read-phases03-analyzemd)

## Preconditions

Assumes the following state is set:
- `repo_path` — set by `phases/01-setup.md`.
- `output_mode` — set by `phases/01-setup.md`.

If any required precondition is absent or empty, STOP and re-read
`phases/01-setup.md` (the producing phase). Do NOT proceed with
defaults, do NOT improvise, do NOT silently continue.

## Cache check first (PR-level shortcut)

Consult `references/per-project-cache.md` for the full cache mechanics
before doing anything else here — the cache section there is the source
of truth for layout, fingerprint computation, GC, and bypass rules.

Then, for this phase specifically:

1. Build the `pr-{number}-{headSha}.json` slot key per the cache section
   (skip this step in local-review mode — there is no PR head SHA).
2. **If hit**: load it, touch the file, and skip the rest of Phase 2.
   Proceed directly to Phase 3 (analyze) with the cached pre-compute
   output. Note in the user-facing progress: `"Phase 2: cache hit (PR
   head unchanged) — skipping pre-compute."`.
3. **If miss**: continue with the sub-steps below, **and at the end of
   the phase write the combined output** (standards summary + utility
   index + barrels + naming + import graph + suppression list + pre-read
   file content references) to that slot.

Run the per-repo GC sweep at this same point — see "Garbage collection"
in `references/per-project-cache.md`. One-shot, throttled by
`.gc.lastrun`.

### What slots get written when

For quick reference (full details in `references/per-project-cache.md`):

- `standards-{contentSha}.json` → written after the standards sub-step below.
- `utility-index-{treeSha}.json`, `barrels-{treeSha}.json`, `naming-{treeSha}.json` → written after the utility-index / barrel sub-step.
- `import-graph-{baseSha}-{namesSha}.json` → written after the import-graph sub-step.
- `pr-{number}-{headSha}.json` → written at the end of the phase, snapshotting all of the above plus pre-read content references + suppression list.

## Detect codebase standards

**Cached.** Key: `standards-{contentSha}.json` per
`references/per-project-cache.md`. On hit, load and skip the file reads
below. On miss, read these files if they exist:

1. **CLAUDE.md** or **AGENTS.md** in the repo root — project conventions
2. **.eslintrc** / **.eslintrc.js** / **.eslintrc.json** / **eslint.config.js** — linting rules
3. **tsconfig.json** — TypeScript configuration
4. **.prettierrc** / **.prettierrc.js** — formatting rules
5. **package.json** — dependencies, scripts, React version
6. **biome.json** / **biome.jsonc** — Biome configuration when present

Issue these reads as a **parallel batch** in a single message — six
independent file reads complete in roughly the time of the slowest one.

Also scan 3-5 existing component files near the changed files to infer:
- Naming conventions (PascalCase components, camelCase hooks, etc.)
- Import patterns (absolute vs relative, barrel exports)
- State management approach (Context, Zustand, Redux, etc.)
- Testing patterns (if test files exist nearby)

This naming/import inference has its own cache slot:
`naming-{treeSha}.json` keyed on the tree-SHA of `src/`. On hit, skip
the component-file scan entirely. On miss, do the scan, then write the
result.

Compile a concise `standards_summary` (bullet points) to pass to each
agent. Write it to the `standards-{contentSha}.json` cache slot.

## Pre-compute Utility Index

To save agents from redundantly grepping the codebase, build a
**utility index** during this step. **Cached.** Key:
`utility-index-{treeSha}.json`. On hit, load and skip the grep below.

```bash
# Find all exports from common utility/shared directories
grep -rn "export " {repo_path}/src/utils/ {repo_path}/src/helpers/ {repo_path}/src/lib/ {repo_path}/src/shared/ {repo_path}/src/common/ {repo_path}/src/hooks/ 2>/dev/null | head -200
```

Barrel export files have their own cache slot:
`barrels-{treeSha}.json`. On hit, skip the find. On miss:

```bash
find {repo_path}/src -name "index.ts" -o -name "index.tsx" | head -30
```

Compile the results into a `utility_index` — a list of available
utilities, hooks, and shared components with their file paths. This is
passed to all agents (especially Agent 2) so they can identify reuse
opportunities by scanning the index rather than grepping from scratch.
Write to the `utility-index-{treeSha}.json` and `barrels-{treeSha}.json`
slots before continuing.

## Pre-compute Import Graph (for Agent 4)

To prevent Agent 4 from running 70+ individual grep calls, pre-compute
the consumer map during this step. **Cached** — see
`references/per-project-cache.md` for the cache key derivation.
On hit, load and skip the grep step entirely.

On miss, do **one** repo-scan instead of N (one per changed file). Build
a single alternation regex over all changed-file basenames, grep once,
then post-process to bucket matches per-file. This is O(repo_size)
instead of O(repo_size × changed_files) — the difference is meaningful
on PRs with 30+ changed files in large repos.

```bash
# Use mktemp so parallel skill invocations don't collide on shared
# temp paths. mktemp respects $TMPDIR and works on git-bash / WSL /
# Linux / macOS without translation.
RAW_CONSUMERS=$(mktemp -t consumer-raw-XXXXXX)
CONSUMER_MAP=$(mktemp -t consumer-map-XXXXXX)
trap 'rm -f "$RAW_CONSUMERS" "$CONSUMER_MAP"' EXIT

# 1. Build the alternation. Strip extensions so `Foo.tsx` matches both
#    `from './Foo'` (extensionless) and `from './Foo.tsx'`. Quote each
#    name to escape regex metacharacters; basenames almost never contain
#    them, but a defensive escape costs nothing.
basenames=$(printf '%s\n' {changed_files} \
  | xargs -n1 basename \
  | sed -E 's/\.[^.]+$//' \
  | sort -u \
  | paste -sd '|' -)
# basenames now looks like: Foo|Bar|useThing|UserCard

# 2. ONE grep over the whole src/ tree. -E for the alternation. -H is
#    implicit with -r. -o so we get the matched basename back per line,
#    making the per-file bucketing trivial in step 3.
grep -rEHno "from\s+['\"][^'\"]*\b(${basenames})['\"]" \
  {repo_path}/src/ 2>/dev/null \
  > "$RAW_CONSUMERS"
# Each line: {consumer_path}:{line_no}:{matched_text}

# 3. Bucket per-changed-file in a single awk pass — extract the basename
#    out of the matched `from '...'` and emit `basename<TAB>consumer`.
awk -F: '
  {
    n = split($0, parts, ":")
    consumer = parts[1]
    # The matched text is the join of fields 3..n (in case the path itself
    # had colons — rare, but be safe).
    matched = parts[3]
    for (i = 4; i <= n; i++) matched = matched ":" parts[i]
    # Extract the basename from the matched `from '...'` text.
    if (match(matched, /['\''"]([^'\''"]+)['\''"]/)) {
      modpath = substr(matched, RSTART+1, RLENGTH-2)
      n2 = split(modpath, segs, "/")
      base = segs[n2]
      sub(/\.[^.]+$/, "", base)
      print base "\t" consumer
    }
  }
' "$RAW_CONSUMERS" | sort -u > "$CONSUMER_MAP"
```

Compile `$CONSUMER_MAP` into the `import_graph` — a mapping of
`changed_file → [consumer_file1, consumer_file2, ...]` keyed back from
basename. Pass this to Agent 4 so it skips the discovery step entirely
and goes straight to reading consumer files. Write the resulting JSON
to the cache slot before continuing. The `trap` cleans up the temp
files automatically when the bash invocation exits.

Also build a **changed exports summary** for Agent 4: for each changed
file, list only the exports that were added, removed, or modified (with
old/new signatures). Agent 4 doesn't need internal implementation
details — just the API surface changes and who consumes them.

## Categorize Changed Files

Before dispatching agents, classify each changed file:

| Category | Pattern | Primary Agent |
|----------|---------|---------------|
| Component | `*.tsx` with JSX return, `components/` | Agents 1, 2, 3; Agent 4 traces consumers |
| Hook | `use*.ts`, `hooks/` | Agent 1 (primary), Agent 3; Agent 4 traces consumers |
| Utility | `utils/`, `helpers/`, `lib/` | Agent 2 (primary); Agent 4 traces consumers |
| Style | `*.css`, `*.scss`, `*.module.*` | Skip — not reviewed |
| Test | `*.test.*`, `*.spec.*`, `__tests__/` | Agent 9 only (all other agents skip test files; Agent 9 reviews them for test-convention findings) |
| Config | `*.config.*`, `tsconfig`, `package.json` | Agent 2 |
| Other | Everything else | Agents 1, 2, 3; Agent 4 traces consumers |

Agent 4's primary files are not the changed files themselves but their
**consumers** — files that import or depend on the changed code.
Agent 4 builds this list dynamically from the `import_graph`.

Each agent receives the **full diff** but also a **primary file list** —
the files it should focus on first. This prevents agents from spending
equal time on files outside their focus area.

Store the categorization as `categorized_files`.

## Filter Trivial Files

Before building primary file lists, filter out files with fewer than 5
lines changed (additions + deletions) from all agents' primary file
lists. These are typically renames, single-line imports, or trivial
config changes that don't warrant individual analysis. The agent still
receives the full diff and may notice issues in these files while
scanning, but they should not appear in the primary file list.

To compute lines changed per file:
```bash
git diff {base}...{head} --stat | grep -E '^\s'
```

## Strip Noise from Diff

Before passing the diff to agents, remove noise that wastes tokens:

1. **Deleted-file hunks:** Remove hunks for files that are purely
   deleted (100% removal, 0 additions). In rename-heavy branches, the
   renamed file's hunks already contain the content.

2. **Pure-rename hunks:** Use `git diff -M90` (90% similarity threshold)
   to collapse renames into one-line notices. This filters out hunks
   where the content is identical or near-identical and only the path
   changed. In rename-heavy branches this can eliminate 50-60% of the
   diff.

```bash
# Generate diff with rename detection, excluding pure deletes
git diff {base}...{head} -M90 --diff-filter=d -- '*.ts' '*.tsx' '*.js' '*.jsx'
```

If the diff is still over 5000 lines after filtering, note the size in
each agent's prompt so they can prioritize primary files and avoid
getting bogged down in low-signal hunks.

## Pre-read Changed Files (speed optimization)

The biggest speed bottleneck is agents making redundant file-read tool
calls. Each agent independently reads the same changed files, costing
5-20 tool round-trips per agent. Eliminate this by pre-reading files
during this phase and embedding the content directly in agent prompts.

**Issue all primary-file Read calls in a SINGLE message of parallel
tool calls** — do not loop with one Read per turn. The Read tool runs
each call independently, so N Reads in one message complete in roughly
the time of the slowest one, not the sum. On a 30-file pre-read this is
the difference between ~15 seconds and ~1 second of wall time.

For each non-trivial changed file (on the primary file list):
1. Issue the Read tool call as part of the parallel batch.
2. Include the returned content in the agent prompt under a
   `## FILE CONTENTS` section.

```
## FILE CONTENTS (pre-loaded — do NOT re-read these files)
### {file_path} ({N} lines)
```{lang}
{full file content}
```
```

This trades larger prompts for fewer tool calls. A typical 10-file PR
adds ~3000 tokens of file content but saves 40+ tool calls across 4
agents (10 reads × 4 agents), which is ~60 seconds of wall time.

**Cap at 30 files / 20000 lines total** for Opus 4.7's 1M-token
context window. (The earlier 15/8000 cap was sized for 200K windows.)
If the PR exceeds this, pre-read only the primary files for each agent
and let them read the rest on demand. Include a note: "Additional files
are available at {repo_path} — read only if needed for cross-referencing."

If the orchestrator is running on a smaller-context model (Sonnet,
Haiku) — judge from your own model identity, not from anything in the
environment — halve the cap to 15 files / 10000 lines. Most invocations
are on Opus 4.7 1M, where 30 / 20K is the right setting. When
uncertain, prefer the smaller cap; falling back to on-demand file
reads inside agents is slower than a slightly-trimmed pre-read list.

Also pre-compute and include any grep results agents would need:
- For Agent 4: the import graph consumers (already pre-computed)
- For Agent 2: the utility index matches relevant to changed files
- For all agents: nearby test file paths (so they know test coverage exists without grepping)

Store the result as `pre_read_contents`.

## Pre-compute Suppression List (for Agent 6)

Extract every **newly-added** lint/type/formatter suppression comment
from the diff so Agent 6 can assess removability without re-scanning
the diff itself. Only added lines (`^+` in the diff, excluding the
`+++` file headers) count — pre-existing suppressions are out of scope
per the DIFF-ONLY RULE.

```bash
# Match added lines containing any suppression directive. Tune the path filter
# to whatever languages the repo uses.
git diff {base}...{head} -M90 --diff-filter=d -- '*.ts' '*.tsx' '*.js' '*.jsx' '*.py' \
  | awk '
      /^diff --git/ { next }
      /^\+\+\+ / { sub(/^\+\+\+ b\//, "", $0); file = $0; next }
      /^@@/ {
        match($0, /\+[0-9]+/); hunk_start = substr($0, RSTART+1, RLENGTH-1) + 0
        line_in_hunk = 0; next
      }
      /^\+/ && !/^\+\+\+/ {
        line = hunk_start + line_in_hunk
        if ($0 ~ /eslint-disable|@ts-ignore|@ts-expect-error|@ts-nocheck|tslint:disable|biome-ignore|noqa|prettier-ignore/) {
          print file "\t" line "\t" substr($0, 2)
        }
        line_in_hunk++; next
      }
      /^[ ]/ { line_in_hunk++ }
    '
```

For each match, record a `suppression_list` entry with:
- `file` — path relative to repo root
- `line` — 1-based line number in the post-diff file
- `directive` — which rule family fired the match (`eslint-disable`, `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, `tslint:disable`, `biome-ignore`, `noqa`, `prettier-ignore`)
- `rule` — the specific rule name where present (e.g. `react-hooks/exhaustive-deps`, `@typescript-eslint/no-explicit-any`, `lint/suspicious/noExplicitAny` for Biome). Null when the directive suppresses everything (`@ts-ignore`, `@ts-nocheck`, bare `eslint-disable-next-line`).
- `surrounding_code_snippet` — the suppression comment plus ~5 lines before and after from the pre-read file content, so Agent 6 can reason about what the suppression is hiding without re-reading the file

If `suppression_list` is empty, skip dispatching Agent 6 entirely in
Phase 3 — same pattern Agent 5 uses when the PR body is empty. Also
omit the conditional task creation for Agent 6 (see below).

Also, when the repo has configs that shape what each rule means,
pre-read them once so Agent 6 can judge whether a suppression is
justified by the active configuration:
- `tsconfig.json` (look for `strict`, `noImplicitAny`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`)
- `.eslintrc*` / `eslint.config.*`
- `biome.json` / `biome.jsonc`

Pass these as a compact `rule_config_summary` alongside the
`suppression_list`.

## End-of-phase: create conditional-agent tasks

Now that the gating inputs are known, create the conditional-agent
tasks for Phase 3. Skipped conditionals get no task at all — no
dangling state.

- **Agent 5 — Scope match**: create the task `"Analyze: Scope match"`
  (activeForm: `"Analyzing scope match"`) ONLY when the PR body
  (`pr_body` from Phase 1) is non-empty. Skip the task entirely in
  local-review mode or when the PR body is empty.

- **Agent 6 — Suppression removability**: create the task `"Analyze:
  Suppression removability"` (activeForm: `"Analyzing suppression
  removability"`) ONLY when `suppression_list` is non-empty. Skip
  otherwise.

- **Agent 7 — Abstraction & coupling boundaries**: create the task
  `"Analyze: Abstraction & coupling boundaries"` (activeForm:
  `"Analyzing abstraction & coupling boundaries"`) ONLY when
  `categorized_files` contains at least one Component-category file.
  Skip otherwise (utility-only and config-only PRs have no render
  surfaces to examine).

## Cache write

Before exiting this phase, write the combined output to the
`pr-{number}-{headSha}.json` slot per `references/per-project-cache.md`
(skip in local-review mode — no PR head SHA). Include:

- `standards_summary`
- `utility_index`, barrels list, `naming` inference
- `import_graph` + changed-exports summary
- `categorized_files`, filtered primary file lists
- `pre_read_contents` (or references to them)
- `suppression_list` + `rule_config_summary`

## Next: read `phases/03-analyze.md`.
