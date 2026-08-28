# React PR Review Checklist

Use this checklist when analyzing React PRs. Each section maps to one analysis agent.

---

## Section 1: Hooks & React Patterns

### Rules of Hooks Violations
- Hooks called conditionally (`if (x) { useState(...) }`)
- Hooks called inside loops (`for (...) { useEffect(...) }`)
- Hooks called in nested functions (not at top level of component/custom hook)
- Hooks called outside React components or custom hooks

### useEffect Issues
- **Missing dependencies:** Variables used inside effect but not in dep array — effect won't re-run when they change
- **No cleanup function:** Subscriptions, event listeners, timers, or intervals without return cleanup — memory leak on unmount
- **Async directly:** `useEffect(async () => ...)` — must create inner async function instead
- **Infinite loops:** Setting state inside effect that triggers re-render which re-triggers effect
- **Overuse:** Using useEffect for what should be an event handler or derived state
- **Empty deps with references:** `useEffect(() => { doSomething(value) }, [])` where `value` changes — stale closure

### State Mutation
- `array.push(item)` instead of `[...array, item]` — React can't detect the update, won't re-render
- `object.key = value` instead of `{ ...object, key: value }` — same problem
- Nested object mutation: changing `state.nested.field` directly without spreading at each level
- `sort()`, `reverse()`, `splice()` mutate in place — use `toSorted()`, `toReversed()`, or spread first

### Conditional Rendering Gotchas
- `items.length && <Component />` renders the number "0" when array is empty — use `items.length > 0 &&` or ternary
- `value && <Component />` where value could be `0` or `""` — falsy but valid values render as text

### State Initialized from Props
- `useState(props.value)` captures initial value only — won't update when prop changes
- Fix: use `key` prop to reset component, or `useEffect` to sync (with caution)

### React 19+ Considerations
- If project uses React Compiler: flag unnecessary manual `useMemo`/`useCallback`/`React.memo` — compiler handles this
- `useFormStatus` must be used inside a `<form>` action context
- `useOptimistic` should have proper rollback handling
- `use()` hook: check that promise/context usage follows rules

### Custom Hooks
- Not following `use` prefix convention
- Custom hook doing too much — should have single responsibility
- Returning too many values (more than 3-4) — consider returning an object instead

---

## Section 2: Reuse & Standards

### Existing Utility Detection
Search these directories for functions that duplicate new code:
- `utils/`, `helpers/`, `lib/`, `shared/`, `common/`, `hooks/`
- Also check barrel files (`index.ts`) in these directories
- Grep for function names similar to what the PR introduces
- Check if the project has a shared component library or design system

### Component Design
- **Single responsibility:** Component handles data fetching + validation + state management + rendering — split the fetch/derive logic into a `use…()` hook; the component handles rendering only
- **Too many props:** >7-8 props suggests the component is doing too much or needs composition
- **Prop drilling:** Passing props through 3+ levels — consider Context or composition pattern
- **Hardcoded values:** Magic numbers/strings that should be constants or config
- **Overly specific:** Component built for one exact use case but could be generalized with minor changes (only flag if a second use case is visible in the codebase)
- **Premature abstraction (AHA):** New shared component, hook, or utility created for <3 call sites where the differences between current uses aren't clearly the same kind of variation — prefer duplication until the essential interface is clear; two uses don't tell you which differences are essential

### Codebase Convention Violations
- File naming doesn't match existing pattern (PascalCase vs kebab-case for components)
- Hook naming doesn't follow `use` prefix
- Export style doesn't match (default vs named exports)
- Directory placement doesn't follow existing structure
- Missing or inconsistent TypeScript types (overly broad `any`, missing interfaces)
- Import style mismatch (absolute vs relative, barrel vs direct)
- Boolean variable/prop/state missing `is`/`has`/`should`/`can`/`will`/`did` prefix (`isLoading` not `loading`, `hasPublisher` not `publisher`-as-boolean, `shouldShowTools` not `showTools`) — only fire when sibling files in the codebase follow the rule
- Abbreviations in identifiers (`dq` → `displayQuery`, `usr` → `user`, `idx` → `index`) — drop when the abbreviation is a codebase standing convention or a domain-standard acronym (`url`, `id`, `db`, `api`)
- Boolean name shape collides with a function-call shape — `showTools` reads as an imperative ("show the tools") but is actually a boolean; prefer `shouldShowTools` / `toolsVisible`
- In-diff DRY: the same 3+ lines appear 3+ times **within the diff itself** — flag as a suggestion to extract a helper. Distinct from the utility-index reuse check (that one catches duplicates of pre-existing utilities).

> Shared-component reuse and component prop-shape generalization are owned by Section 7 (Agent 7), not Section 2. Agent 2 owns reuse of utilities, helpers, lib functions, shared types, and npm libraries only.

---

## Section 3: Performance & Edge Cases

### Re-render Issues
- Parent component re-renders causing expensive children to re-render unnecessarily
- Missing `React.memo` on pure components that receive complex props and render expensively
- New object/array/function created on every render passed as prop to memoized child (breaks memo)
- State stored too high in the tree — lifting state up further than needed causes broad re-renders

### Memoization (Only When Impactful)
- `useMemo` missing for expensive computations (filtering/sorting large arrays, complex calculations)
- `useCallback` missing for function props passed to `React.memo` children
- Do NOT flag missing memo for trivial computations — only when there's a measurable cost

### Error Boundaries
- Components that fetch data or do complex rendering without error boundary protection
- Entire page crashes on single component error — need granular boundaries
- Missing fallback UI for error states

### Effect Cleanup
- `setTimeout`/`setInterval` without `clearTimeout`/`clearInterval` in cleanup
- `addEventListener` without `removeEventListener` in cleanup
- WebSocket/EventSource connections without close in cleanup
- `AbortController` not used for fetch requests — can't cancel on unmount
- Subscriptions (RxJS, pub/sub) without unsubscribe

### Async & Race Conditions
- Component state updated after unmount (`setState` on unmounted component)
- No `AbortController` for fetch — multiple rapid re-renders cause out-of-order responses
- Stale closure: async callback references old state value instead of current
- Missing loading/disabled state during async operations — user can trigger multiple times

### Missing UI States
- **Loading:** No loading indicator while data is being fetched
- **Error:** No error handling/display when fetch fails or operation errors
- **Empty:** No empty state when data array is empty (just blank screen)
- **Offline:** No handling for network failures (where applicable)

### Security
- `dangerouslySetInnerHTML` with unsanitized user input — XSS vector
- User input rendered without escaping
- Sensitive data in component state that persists after logout

### Large Data
- Rendering 50+ items without virtualization (react-window, react-virtualized, tanstack-virtual)
- Loading all data at once without pagination or infinite scroll
- Large images without lazy loading or responsive sizing

---

## Section 4: Architecture & Module Boundaries

Only apply these checks when the project uses a feature-folder structure
(`src/features/`, `src/shared/`, `src/app/`). Skip for flat or layer-based
layouts unless the PR explicitly introduces feature-folder boundaries.

### Feature Folder Violations
- **Cross-feature import:** Feature A imports directly from a file inside Feature B's folder — cross-feature dependencies should compose at the `app/` level or go through each feature's public barrel
- **`shared/` imports `features/`:** A module under `shared/` imports from any feature — inverts the required direction (`shared → features → app`)
- **Leaking barrel:** A feature's `index.ts` re-exports internal modules that should stay private; only export what external callers legitimately need
- **Missing barrel / no public surface:** Feature has no `index.ts`, letting callers path-hack into internals — no enforced boundary

### Dependency Direction Violations
- **Wrong-way import:** A stable/general module imports a volatile/specific one (e.g. a `shared/` utility pulling from a feature, a low-level service importing a high-level orchestrator)
- **Circular imports:** Module A imports from B which imports from A — confirm with `madge` or `eslint-plugin-import/no-cycle`
- **No enforced boundary:** Project uses feature-folder structure but has no `eslint-plugin-import/no-restricted-paths` (or equivalent) rule — the architecture constraint is doc-only with no runnable check

### Dependency Inversion Opportunities
- **Stable module imports volatile:** A shared/inner module takes a direct import from a feature/outer module when it could receive the dependency via a callback, prop, render-prop, or context value — the inner module defines the contract; the outer implements it
- **Hard-coded cross-boundary import where injection would do:** A hook or handler reaches directly into a sibling feature; passing the dependency in from the call site removes the coupling without ceremony

### State Kind & Placement
- **Server state in a client store:** Remote/async data held in `useState` or a global client store (Zustand, Redux, Context) instead of a query/cache layer (TanStack Query, SWR) — hand-rolled caching, stale-data bugs, and cache invalidation complexity follow
- **Feature-local state in global store:** State only read within one feature hoisted into a global store — same defect as exporting feature internals through a barrel; nothing external needs it
- **State lifted above the closest common parent:** State shared by two siblings lifted to a grandparent or app root when the direct parent covers all readers — broadens re-render blast radius unnecessarily (see also Section 3's "State stored too high" performance check)

### What NOT to flag
- Layer-based or flat projects without feature-folder conventions
- Cross-feature imports that go through the feature's own public barrel (that's the intended pattern)
- State genuinely shared across features that lives at `app/` level
- `shared/` modules importing other `shared/` modules (same layer is fine)
- Circular import findings when no clear call-chain is visible in the diff

---

## Section 7: Abstraction & Coupling Boundaries

### Raw Backend/API Shapes in Presentation
- Component destructures snake_case fields or deeply-nested API response envelopes directly in render
- Component imports types from `api/`, `types/api/`, `__generated__/`, `schema.*` and uses them as prop types
- No view-model / adapter layer between the data source and the render component

### Business/Domain Logic in JSX
- Complex conditions inline in render (`{user.role === 'admin' && user.permissions.includes(...) && user.active && ...}`)
- Permission / visibility / state-machine logic expressed as ad-hoc boolean chains in JSX
- Derived booleans (`canEdit`, `isVisible`, `shouldShowBadge`) computed inside render instead of passed as a prop or memoized upstream

### Data Fetching in Presentation Components
- `fetch` / `useQuery` / `useSWR` / direct store subscriptions called from a component whose primary role is rendering
- Codebase has an established container/hook pattern but this component bypasses it
- Same data fetched in multiple sibling components instead of lifted to a parent container

### Formatting / Derivation in Render
- Non-trivial date, currency, or locale formatting inline in JSX
- Markdown → HTML, sanitization, or serialization happening in render
- Inline `.filter(...).map(...)` chains over raw arrays where a memoized selector would own the derivation

### Hook Return Shapes
- Custom hooks that return the raw API response shape when the codebase has a view-model convention for that entity
- Hooks that mix data-fetching concerns with UI-state concerns (loading flags, error banners) in ways that force the consuming component to know too much

### Inline Mutation / Submit Handlers
- Handler defined inline in the component bundles the API call AND the optimistic UI update / state reconciliation — should be a custom hook (`useFooSubmission`) returning `{submit, isPending, error}` so the presentation can stay pure
- Render components doing their own retry / backoff / abort orchestration when a hook abstraction exists elsewhere in the codebase
- Drop for truly-trivial components (one API call, no optimistic state, no error reconciliation) — the hook would be overkill

### Long useState Chains
- 5+ `useState`s in one component whose states are conceptually a single piece (form, wizard, filter panel) — a `useReducer` or a custom hook would encapsulate the lifecycle
- Flag on coupling, not on count alone: independent UI states that happen to share a component are fine

### God Components & Missing Extractions
- One component rendering 5+ unrelated sections (header + filters + list + detail panel + actions) that should each be their own component
- Self-contained JSX chunks with their own rules — a rating widget, avatar menu, status pill with conditional styling — inlined instead of lifted to a named component
- Helper functions or memos inside a parent component that return JSX fragments (those are really their own components)
- Diverging branches in a single component rendering fundamentally different concepts (`{mode === 'edit' ? <big tree A> : <big tree B>}`) — each branch is its own component

### Inter-Component Coupling
- Sibling components reaching into each other's internals — shared mutable state, one knowing about the other's render branches, tightly-coupled prop contracts
- Prop drilling through 3+ levels where a context, composition, or slot pattern would cut the chain
- A parent component that exists purely to wire together two children that could compose directly

### Shared Component Reuse & Generalization
- New component duplicates one that already lives in this repo's shared-component dirs (`common/`, `shared/`, `ui/`, `primitives/`, `components/base/`, `design-system/`, `packages/*/ui/`). Glob/Grep those dirs first; only flag with a concrete pointer to the existing component (path + line).
- New component takes domain-specific props (`conversation`, `order`, `user`, `ticket`) when the diff suggests multiple callers / entry points and a generalized shape (`items`, `label`, `value`, `onSelect`) would serve all of them — flag as a suggestion to generalize up front.
- New component is a canonical primitive (Dropdown, Modal, Combobox, Toast, Tooltip, DatePicker, Tabs, Menu) built from scratch AND the repo doesn't already import from a UI library — mention 2-3 canonical options (Fluent UI / Radix / shadcn / Chakra / MUI) as a suggestion. Never second-guess a library the repo already uses.

### What NOT to flag
- Trivial prop use (`<div>{user.name}</div>`)
- Small one-off components where abstraction or extraction is over-engineering
- Codebases whose convention IS direct coupling or in-file composition — honor the codebase
- Genuine render concerns (className from status, aria from role, layout from item count)
- Named adapters (`UserCardFromApi`, `*Adapter.tsx`) where the coupling is intentional
- Mechanical "file over 600 lines" / "over 7-8 props" findings — those are Agent 2's yardstick territory; Agent 7 only flags extraction when there's a specific design-level reason beyond size
- Any "could be more abstract" / "could be split up" suggestion without a concrete cost (API blast radius, test seam impossible, inlined chunk has its own domain rules)
- Shared-component "reuse" findings without a concrete existing component to point at — if Glob/Grep of the shared dirs turned up nothing, drop it rather than invent a hypothetical base
- UI-library suggestions when the repo already imports from one (Fluent UI, Radix, shadcn, Chakra, MUI, Ant Design, Mantine, HeadlessUI, Base UI, etc.) — honor the existing choice
- Inline mutation-handler findings for truly-trivial components (one API call, no optimistic state, no error reconciliation)
- `useState`-chain findings on fewer than ~5 states or where the states are genuinely independent

---

## Section 8: Comment Hygiene

Default to writing no comments. Only flag a comment when it fails to
explain a non-obvious WHY, narrates ephemeral context (plans, PRs,
callers) that rots fast, or restates what the next line of code already
says. Honor the codebase's existing convention — if sibling files share
the same style, a new comment matching it isn't a finding.

### Plan / Phase / Spec narration
- `// Phase 5a/5b — pushier descriptions` — references to phases that
  mean nothing once the plan file is gone (should-fix)
- `// from .claude/plans/auth-refactor.md` — pointers to plan documents
  that won't survive (should-fix)
- `// polish pass`, `// cleanup pass`, `// per spec section 3.2` —
  process-stage narration with no semantic value (should-fix)
- Any `Phase \d+[a-z]?` reference, references to `.claude/plans/`, or
  numbered spec section markers

### PR / Review / Commit narration
- `// addresses review comment from @alice` — review-thread context that
  rots the moment the PR merges (should-fix)
- `// per @bob's feedback`, `// per code review` — reviewer-handle
  narration tied to a specific moment in time (should-fix)
- `// in this PR`, `// fixes #123 review`, `// see PR #456` —
  PR/issue-number references that mean less and less over time
  (should-fix)
- Commit-history narration ("originally implemented in", "added when we
  switched from X")

### Caller / Flow narration
- `// used by UserProfile` — coupling info that rots when callers move
  (should-fix)
- `// added for the checkout flow` — feature/flow narration that
  belongs in the PR description, not the source (should-fix)
- `// only called from the admin panel` — caller assumptions that
  silently break when a new caller appears (should-fix)

### WHAT-not-WHY (redundant with next line of code)
- `// increment counter` above `count++` (suggestion)
- `// loop through users` above `for (const user of users)` (suggestion)
- `// check if logged in` above `if (user.isAuthenticated)` (suggestion)
- Comment that restates what well-named identifiers already convey on
  the very next line

### Wordy / multi-line block comments
- Multi-paragraph block comments where one short line would suffice
  (suggestion)
- Multi-line `/* ... */` blocks narrating an obvious pattern (suggestion)
- Verbose JSDoc on internal functions whose signature already documents
  the contract — only flag when sibling files in the codebase don't
  share the same wordiness; honor the codebase convention

### What NOT to flag
- Comments that explain a non-obvious WHY: hidden constraints,
  invariants, workarounds for specific bugs, behavior that would
  surprise a reader (e.g. `// Required to work around Edge bug #1234 —
  flush before unmount`)
- JSDoc-style comments that match the codebase's documented convention
  (when sibling files use the same style, a new file matching it isn't
  a finding)
- License headers, `@generated` markers, copyright notices, lint-config
  comments at file top
- `@param`, `@returns`, `@throws`, `@deprecated`, `@see` JSDoc
  annotations — structured documentation, not narrative
- Justified algorithm / regex / math explanations where the code itself
  is genuinely opaque (regex captures, bitwise tricks, numerical
  stability notes)
- Comments upstream of this diff — only flag comments on lines added or
  modified by the diff
- `TODO`, `FIXME`, `HACK`, `XXX` markers — **owned by Section 2**
  (Reuse & Standards). Section 8 does not double-flag these.

---

## Section 9: Other

A deliberate grab-bag for small, recurring review patterns that don't
earn a dedicated lens. Designed to grow over time. Honor the codebase
in every category — if the codebase already breaks the convention, the
codebase wins.

Section 9 is the only section that reviews test files (`*.test.*`,
`*.spec.*`, `__tests__/`). Sections 1–8 skip test files; Section 9 picks
them up for the test-convention check below.

### Redundant guards / dead conditions
- `hasX && x !== undefined` when `hasX` already implies the right side
  (e.g. `hasPublisher && publisher` when `hasPublisher` is computed as
  `publisher != null`) — suggestion
- Optional chaining on values already narrowed by destructuring or
  earlier type guards (`const { user } = props; user?.name` where
  `user` is non-optional in `props`) — suggestion
- Ternaries `x ? y : null` that read more cleanly as `x && y` —
  suggestion
- `if (foo) { ... } else if (foo && bar) { ... }` — the second branch
  is unreachable — suggestion
- `Boolean(x) && x` and similar tautologies — suggestion
- Drop the finding if the second check is load-bearing for TypeScript
  narrowing (logically redundant but the type checker needs it)

### Telemetry symmetry & gaps
- Success path emits a telemetry event, error / cancel / abort path
  doesn't (e.g. `try { emit('saved'); } catch { /* nothing */ }`) —
  should-fix (error paths are the most valuable signal)
- Paired events (Start/Complete, Open/Close, Begin/End, Mount/Unmount)
  where one half is missing or one half lacks payload fields the other
  includes — suggestion
- New mutation / fetch / state-changing code path that touches
  user-visible state with no observability emit at all, when the
  codebase has an established telemetry pattern — suggestion
- Do NOT flag missing telemetry when the codebase has no telemetry
  system. Grep sibling files for `track*`/`emit*`/`log*`/`analytics.*`
  patterns first — if nothing, drop the finding. Agent 9 does not
  propose new systems.

### Test conventions
- `test('...', ...)` instead of `it('...', ...)` when sibling tests in
  the same file or repo use `it` (or vice versa) — suggestion
- Test titles not in third-person-singular-present form when sibling
  titles in the codebase use it ("should do X" / "does X" / "X-es when
  Y") — suggestion. The codebase's convention wins; never invent a
  style preference.
- Test title doesn't match what the body asserts (title says "renders
  empty state" but body asserts on a button click handler) —
  suggestion
- `describe(...)` block whose name doesn't describe a logical group of
  the tests inside it — suggestion
- Drop when the codebase convention is inconsistent within itself —
  Agent 9 enforces what's already there, not personal style

### Content / i18n consistency
- Ellipsis usage inconsistent within a single string family (some
  `"Loading…"`, some `"Loading..."`, some `"Loading"`) — suggestion
- Format params declared in the locale file but missing from a call
  site, or vice versa (`t('error.notFound', { resource })` where the
  message string doesn't reference `{resource}`; or the message string
  references `{count}` but the call site doesn't pass it) — should-fix
  when the result is broken UI, suggestion otherwise
- Localized message (`t('...')`) used in a log line where a plain
  English string would be appropriate; or — the reverse — a plain
  hardcoded string in a UI-facing spot where the rest of the file uses
  `t(...)` — suggestion
- Sentence-case vs Title-Case vs SCREAMING-CASE drift within the same
  string family or sibling buttons — suggestion

### What NOT to flag
- Codebase convention wins everywhere. If sibling files use the same
  shape, drop the finding.
- Blank-line / spacing / quote-style / trailing-comma issues — that's
  Prettier territory.
- Telemetry "gaps" where the project clearly has no telemetry system —
  Agent 9 does not invent systems.
- Test-title gripes when the codebase has no consistent convention.
- Redundant guards that exist for genuine type-narrowing reasons.
- Boolean / variable naming hygiene (`isLoading` vs `loading`,
  abbreviations like `dq`, shape-collisions like `showTools`) —
  **owned by Section 2** (Reuse & Standards).
- DRY findings (same 3+ lines repeating 3+ times) — **owned by
  Section 2**.
- Comment text quality — **owned by Section 8** (Comment Hygiene).
- Pre-existing issues in surrounding context — DIFF-ONLY RULE applies.
