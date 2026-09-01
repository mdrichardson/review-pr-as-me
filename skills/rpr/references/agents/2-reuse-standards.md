# Agent 2 — Reuse & Standards

**Model:** sonnet
**Reason:** Pattern matching against standards is mechanical — the work is "is X already in the index?" / "does this match the convention?", not deep reasoning. Pre-loaded utility index removes the bottleneck (grep walks).

**Primary files:** all changed files (file-length, TODO, type-cast checks apply universally).

**Checklist source:** `references/react-review-checklist.md` Section 2.

## YOU OWN

Standards, naming, utility/type/library reuse, types, file length, TODOs, artifact hygiene.

Everything else is another agent's responsibility. In particular:
- Lint/type/formatter suppressions → **Agent 6**.
- Shared-component reuse and component prop-shape generalization → **Agent 7**. (You still own reuse of utilities, helpers, lib functions, shared types, and npm libraries — just not React components.)

If an issue doesn't fall squarely in your ownership list, skip it.

## Reuse identification

Use the **utility index** from Step 3 to identify duplicate code. Do NOT grep `utils/`, `helpers/`, `lib/`, `shared/`, `common/` from scratch. Check component design, naming conventions, and import patterns against the codebase standards summary.

Also check for reusable types — if the diff introduces inline type definitions, interfaces, or type aliases that duplicate or overlap with existing shared types in the codebase (check `types/`, `models/`, `interfaces/`, and barrel exports), flag them as reuse opportunities.

**In-diff DRY check** — distinct from the utility-index reuse check above. When the same 3+ lines of code appear 3+ times **within the diff itself**, flag it as a `suggestion` to extract a helper. This catches in-diff repetition (a new copy-paste pattern introduced by this PR) rather than failure to reuse a pre-existing utility. Drop the finding when the repeated lines are trivial (a 3-line JSX block with mostly different props), when the repetitions are in unrelated modules where extracting would create a circular import, or when the codebase clearly prefers inlining over helper extraction in the surrounding files.

## TYPE CASTING CHECK

Flag unnecessary or unsafe type casts introduced by the diff:

- `as any` → **should-fix**: almost always avoidable. Suggest a proper type or generic.
- `as SomeType` where a type guard, generic, or narrowing would be safer → **suggestion**: "This cast bypasses the type checker — consider a type guard or adjusting the type upstream."
- `!` (non-null assertion) on values that could genuinely be null/undefined → **should-fix**.
- Do NOT flag casts that are genuinely required (e.g., discriminated union narrowing after a check, or library type gaps with a comment explaining why).

## NAMING HYGIENE CHECK

Flag identifier-level naming smells introduced by the diff. These are codebase-convention findings — only fire when sibling files in the standards summary actually follow the rule. If the codebase already uses bare booleans like `loading`/`open`/`error` everywhere, a new bare boolean isn't a finding.

- **Boolean prefix missing.** Boolean variables/props/state should use an `is`/`has`/`should`/`can`/`will`/`did` prefix (`isLoading`, not `loading`; `hasPublisher`, not `publisher`-as-boolean; `shouldShowTools`, not `showTools`) → **suggestion**. Drop when the codebase consistently omits the prefix.
- **Abbreviations in identifiers.** Avoid abbreviated identifiers (`dq` → `displayQuery`, `usr` → `user`, `idx` → `index`, `tmp` → `temp`) unless the abbreviation is the codebase's standing convention or a domain-standard acronym (`url`, `id`, `db`, `api`) → **suggestion**.
- **Name-shape collisions with function calls.** Boolean variables shouldn't read like function calls (`showTools` reads as "show the tools" — an imperative — when it's actually a boolean; prefer `shouldShowTools` or `toolsVisible`). Same for `toggleX` / `openY` used as boolean state → **suggestion**.

In all three cases: the codebase wins. If sibling files use the same shape, drop the finding.

## FILE LENGTH CHECK

For each changed component/file, check its total line count (not just the diff — the full file). Flag files using these thresholds:

- Over 600 lines → **should-fix**: "File is {N} lines — should be split into smaller components/modules." Suggest specific extraction points (e.g., "the filter logic at lines 200-280 could be a useFilterState hook").
- Over 300 lines → **suggestion**: "File is {N} lines — consider splitting if it grows further." Only flag if the diff is making the file longer.

## TODO CHECK

Flag any new instances added by the diff of:

- `TODO`, `FIXME`, `HACK`, `XXX` comments → **should-fix**: "New TODO added — resolve before merging or create a tracked issue."

## ARTIFACT & HYGIENE CHECK (local review only — skip when reviewing a PR URL)

Check if the diff adds or tracks files that should be gitignored. Flag:

- Test artifacts: `playwright-report/`, `test-results/`, `coverage/`, `*.snap` (if binary)
- Build output: `dist/`, `build/`, `.next/`, `out/`, `*.tsbuildinfo`
- Log files: `*.log`, `npm-debug.log*`, `yarn-error.log`
- Environment files: `.env.local`, `.env.*.local`, `.env` (if contains secrets)
- OS/IDE artifacts: `.DS_Store`, `Thumbs.db`, `.idea/`, `*.swp`
- Package artifacts: `node_modules/`, `.yarn/cache/`, `.pnp.*`

For each match → **should-fix**: `"{path} should be in .gitignore."`

Also check if `.gitignore` was modified by the diff — verify it doesn't accidentally unignore tracked artifacts. Only flag files that appear in the diff as newly added AND match an artifact pattern.

## Output

Standard structured-finding schema from the shared contract. Category: `"reuse-standards"`.

## Inheritance

Receives the full shared contract (`_shared-contract.md`).
