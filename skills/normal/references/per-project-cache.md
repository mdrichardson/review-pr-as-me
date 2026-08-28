# Per-Project Cache

Most of Phase 2 (pre-compute) produces output that depends only on stable
inputs (config files, source-tree contents, PR head SHA). Recomputing it
on every invocation is the dominant wall-time cost on subsequent runs
against the same repo. The skill keeps a per-project cache so successive
runs (and especially `/review-pr-as-me-fast` re-reviews) skip work whose
inputs haven't changed.

## Layout

The cache mirrors the per-repo path convention used by the existing
clone directory at `~/.claude/repos/...` — slashes, not flattened
identifiers — so the same `{path-to-repo}` value can be reused as a
suffix in both places without translation.

```
~/.claude/.cache/review-pr-as-me/v1/
  {path-to-repo}/                      ← {owner}/{repo} for GitHub,
                                          {org}/{project}/{repo} for ADO
                                          (matches ~/.claude/repos/...)
    standards-{contentSha}.json        ← codebase standards summary
    utility-index-{treeSha}.json       ← exports under utils/helpers/lib/shared/common/hooks
    barrels-{treeSha}.json             ← list of barrel index files under src/
    naming-{treeSha}.json              ← inferred naming/import conventions
    import-graph-{baseSha}-{namesSha}.json ← changed-file → consumer map
    pr-{number}-{headSha}.json         ← entire Phase 2 output (the rprf killer)
    .gc.lastrun                        ← unix timestamp of last GC sweep
```

`v1` is the **cache-format version**. Bump to `v2` (and update both this
section and the cache-write code) any time the on-disk schema or the
fingerprint inputs change in a way that would make old entries
incorrect. The skill never reads from a different version directory.

## Fingerprints (cache keys)

The key IS the freshness check. A stale entry is unreachable, so no
read-time TTL is needed.

| Cache | Key derivation |
|-------|----------------|
| `standards` | `sha256(concat( CLAUDE.md ?? "", AGENTS.md ?? "", .eslintrc* ?? "", eslint.config.* ?? "", tsconfig.json ?? "", .prettierrc* ?? "", package.json ?? "", biome.json ?? "" ))` — concatenate the contents of every detection input that exists on disk in a stable order, then hash. |
| `utility-index` | `git rev-parse HEAD:src/utils HEAD:src/helpers HEAD:src/lib HEAD:src/shared HEAD:src/common HEAD:src/hooks 2>/dev/null` joined into one string and hashed. Missing dirs simply contribute nothing — the joined string still uniquely identifies the present-dir state. |
| `barrels` | `git rev-parse HEAD:src` (one tree-SHA covers the whole `src/` directory). |
| `naming` | Same key as `barrels` — naming is inferred from sibling files in `src/components/` and the relevant subset of that tree. |
| `import-graph` | `sha256(baseSha + "\n" + sorted(changedFileBasenames).join("\n"))` |
| `pr` | `pr-{number}-{headSha}` — the PR head SHA is the freshness signal. |

`HEAD` here is the PR head commit (after `gh pr checkout`), not whatever
branch was checked out before. Always run the `git rev-parse` calls
**after** Phase 1's `gh pr checkout {number}` / `git checkout
FETCH_HEAD` so the tree-SHAs reflect the PR state, not the base state.

## Read-then-write pattern

For each cache slot, before computing:

1. Build the key per the table above (the actual hashing is cheap —
   pure local SHA, no network).
2. Look for `~/.claude/.cache/review-pr-as-me/v1/{path-to-repo}/{slot}-{key}.json`.
3. If it exists, load it and `touch` the file (updates mtime — used by GC
   to track recency). Skip the compute step entirely.
4. If it does NOT exist, run the compute step, then write the result to
   that path before continuing.

The `pr-{number}-{headSha}.json` slot is special: when present, it
short-circuits all other Phase-2 cache lookups. Load it once and skip
straight to Phase 3 (analyze). This is the single biggest win for
`/rprf` re-runs where the user pushed nothing new.

## Garbage collection (size only — no read-time TTL)

A read against the cache never expires an entry by age. The fingerprint
already handles correctness — old entries are unreachable, not wrong.
The only reason to delete entries is to keep the cache from growing
unbounded.

On every invocation, before doing anything else in Phase 2:

1. Read `.gc.lastrun` for the current repo. If missing or older than
   **7 days**, run a sweep:
   - Delete any cache file with `mtime` older than **30 days** (catches
     drift from things outside the fingerprint, e.g. skill prose
     changes that didn't bump `v1`).
   - If the repo's cache directory is over **500 MB**, delete files in
     ascending `mtime` order until it's under the cap.
2. Write the current unix timestamp to `.gc.lastrun`.

The 7-day GC throttle keeps the per-invocation cost negligible (a single
`stat` call most of the time). The 30-day floor is the safety net: if
something goes wrong with fingerprinting, no cache entry survives a
month.

## When to bypass the cache

Skip the cache entirely (read AND write) when:
- The user passes a `--no-cache` flag in `review_instructions` (free-text
  match — `\bno[- ]?cache\b` case-insensitive).
- The repo is a shallow clone (`git rev-parse --is-shallow-repository`
  returns `true`) — tree-SHA-based keys would still be correct but
  shallow-clone weirdness around tree objects is not worth debugging on
  the cache path; just recompute.

## What gets a dedicated cache slot vs. only the PR snapshot

The `pr-{number}-{headSha}.json` slot is a snapshot of the **entire**
Phase 2 output — everything below is folded into it when present, so a
cache hit on the PR slot serves all downstream consumers without
re-reading individual slots. The list below describes which items
also get their **own** dedicated slot (worth caching across PRs)
versus which ride only inside the PR snapshot (cheap enough to
recompute when the PR slot misses).

- **No dedicated slot — pre-read file contents.** Cheap to re-read on
  every invocation; not worth a per-file cache. Folded into the PR
  snapshot.
- **Never cached anywhere — PR comments** (Phase 1's
  `pr_comments_list`). Comments arrive continuously while a review is
  in progress; caching them would risk acting on stale dedup data.
  Always fetch fresh, even on a PR-snapshot cache hit.
- **No dedicated slot — suppression list.** The awk pass over the
  diff is not a bottleneck. Folded into the PR snapshot so a cache
  hit there still gets it; on a PR snapshot miss, recompute from
  scratch.
