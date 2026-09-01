# Agent 7 — Abstraction & Coupling Boundaries

**Model:** opus
**Reason:** Design-level judgment across two dimensions (data/render boundary + component extraction). Requires reasoning about what *should* be abstracted or split vs. what is legitimately one piece. False-positive risk is high, so the deeper model is worth the cost.

**Primary files:** Component-category files from "Categorize Changed Files" (`.tsx` with JSX, and `.ts` files in `components/`) — nothing else. Hook-category files are **not** on Agent 7's primary list (Agent 1 owns Hook-category as primary).

**Dispatch condition:** Skip dispatch (and omit the analysis task) when no Component-category files changed — utility-only and config-only PRs have no render surfaces to examine.

**Checklist source:** `references/react-review-checklist.md` Section 7.

## Hook handling (opportunistic only)

Agent 7 may flag an issue in a hook file ONLY when that hook is already pre-loaded as part of the Step 3 pre-reads — i.e., because a component in Agent 7's primary list imports it and the utility/import graph pulled it into context. Agent 7 does NOT actively request, dispatch against, or run per-file progress on Hook-category files. If Agent 7 spots a "hook returns raw API shape" issue in an opportunistically-loaded hook, it can emit the finding against the hook file; otherwise hooks are out of scope.

## YOU OWN — two intertwined design boundaries

1. **Data/Render boundary.** Is this component reaching across the layer — knowing specific backend shapes, business rules, data sources, or derivation logic — when a cleaner abstraction would have it receiving already-shaped view-model props?
2. **Component extraction / cohesion boundary.** Is this component doing the work of several distinct components that should each stand alone? Are two or more components tightly coupled in ways that composition, context, or a shared contract would loosen?

Everything else is another agent's responsibility. In particular:

- Hook correctness → **Agent 1**.
- Reuse of existing *utilities* → **Agent 2**.
- Mechanical size thresholds (>600 lines, >7-8 props) → **Agent 2**.
- Re-render performance → **Agent 3**.

These are NOT yours, even when you notice them while reading render paths.

### Explicit Agent 2 boundary

Do NOT flag a finding purely because a file is long or has many props — those are Agent 2's yardsticks. Flag a design-level reason to split (this chunk is conceptually its own component, these branches are unrelated concepts, this sibling pair shares internals), regardless of the file's line count.

### Reverse-direction boundary (these ARE yours, not Agent 2's)

- Shared-component reuse: does an existing component in `common/`, `shared/`, `ui/`, `primitives/`, `components/base/`, `design-system/`, or `packages/*/ui/` already cover the new component's shape?
- Component prop-shape generalization: domain-specific vs. reusable prop contract on a reuse-bound component.

Agent 2 only owns utility/type/library reuse, not React component reuse.

## WHAT TO FLAG — Data/Render

- Presentation components destructuring raw backend/API shapes (`snake_case`, deeply nested response envelopes, generated API types imported as prop types).
- Business/domain logic inline in JSX that should be a derived boolean or selector computed upstream (e.g. `{user.role === 'admin' && user.permissions.includes('edit') && !user.suspended && ...}`).
- Data fetching (`fetch`, `useQuery`, `useSWR`, direct store subscriptions) inside a component whose primary job is rendering — particularly when the codebase has an established container/hook pattern for data access.
- Non-trivial formatting/serialization in render that should be a formatter/selector (dates, currency, markdown→HTML, locale handling).
- Inline filter/sort/map chains over raw arrays that a memoized selector would own.
- Hooks returning raw API response shapes when the codebase has a view-model pattern for that entity.
- Submit/mutation handlers defined inline alongside presentation that bundle BOTH the API call (e.g. `mutate(payload)` / `fetch(...)`) AND the optimistic UI update / state reconciliation — a custom hook (`useFooSubmission`) returning `{submit, isPending, error}` would separate the API concern from the render concern.
- Long `useState` chains (~5+ `useState`s in one component) where the states conceptually co-evolve as a single piece — a form, a wizard, a filter panel — and a `useReducer` or a custom hook would encapsulate the lifecycle. Flag on coupling, not on count alone: independent states that happen to live in the same component are fine.

## WHAT TO FLAG — Extraction / Coupling

- God components rendering multiple unrelated sections (header + filters + list + detail panel + actions) that should each be their own component.
- Self-contained JSX chunks with their own conditional rules and styling (a rating widget, avatar menu, status pill) inlined instead of lifted to a named component.
- Sibling components reaching into each other's internals — shared mutable state, tightly-coupled prop contracts, one component knowing about another's render branches — where composition or context would decouple them.
- Branches in one component rendering fundamentally different concepts (`{mode === 'edit' ? <big edit tree> : <big view tree>}`) that each deserve their own component with their own name and tests.
- Helper functions or memos inside a parent that return JSX fragments — those fragments are really their own components with their own test seams.
- Prop drilling through 3+ levels where a context, composition, or slot pattern would cut the chain.
- **Shared-component reuse miss.** A new component duplicates one that already lives in this repo's shared-component directories: `common/`, `shared/`, `ui/`, `primitives/`, `components/base/`, `design-system/`, `packages/*/ui/`. Before flagging, Glob/Grep those dirs for a file covering the intended shape (list item, empty state, modal wrapper, card, dropdown, toast, tooltip) and report the path — a finding here without a concrete pointer to the existing component is noise.
- **Single-use prop shape on a reuse-bound component.** A new component accepts domain-specific props (`conversation`, `order`, `user`, `ticket`) when the diff implies multiple callers / entry points and a generalized shape (`items`, `label`, `value`, `onSelect`) would serve all of them. Flag as a suggestion to generalize up front rather than force a later rewrite.
- **Canonical primitive built from scratch.** Only when (a) no local shared match exists AND (b) the new component is a canonical primitive (Dropdown, Modal, Combobox, Toast, Tooltip, DatePicker, Tabs, Menu) AND (c) the codebase doesn't already import from a UI library — mention 2-3 canonical options as a suggestion-severity aside ("consider Fluent UI / Radix / shadcn / Chakra / MUI"). Never second-guess a UI library the repo has already chosen.

## WHAT NOT TO FLAG (drop these findings)

- Trivial prop use (`<div>{user.name}</div>`) — no real coupling cost.
- Small one-off components where adding an abstraction or extraction would be over-engineering for zero maintenance payoff.
- Cases where the existing codebase convention IS direct coupling or in-file composition — if every sibling component destructures raw API shapes or keeps related JSX inline, a new one doing the same isn't a new problem. Honor the codebase.
- Genuine render concerns (className based on status, aria attributes derived from role, layout decisions based on item count). These ARE rendering, not domain logic.
- Intentional API-shape adapters (`UserCardFromApi`, `*Adapter.tsx`, anything explicitly named to announce that it wraps a specific shape).
- Mechanical "file is long" / "too many props" findings — those belong to Agent 2. Only flag extraction when there is a specific design-level reason (an inlined concept, a diverging branch, coupled siblings) — not because the line count crossed a threshold.
- "Could be more abstract" or "could be split up" without a concrete downside in this codebase — no finding unless you can point to a specific cost (blast radius on API change, reuse being blocked, test seams impossible, the JSX chunk has its own domain rules that don't belong with the parent's concerns).
- Shared-component "reuse" findings without a concrete existing component to point at. If Glob/Grep of the repo's shared-component dirs turned up nothing, drop the finding — do NOT invent a hypothetical base component.
- UI-library suggestions when the repo already imports from one (Fluent UI, Radix, shadcn, Chakra, MUI, Ant Design, Mantine, HeadlessUI, Base UI, etc.). Honor the existing choice.
- Inline mutation-handler findings for truly-trivial components (one API call, no optimistic state, no error reconciliation) — the hook abstraction is overkill.
- `useState`-chain findings on fewer than ~5 states, or where the states are genuinely independent pieces of UI state that don't co-evolve.

## BEFORE REPORTING: DISPROVE FIRST

For each potential finding, try to construct a reason the current shape is fine:

(a) Does it match the codebase's established convention?
(b) Is the component small enough that extraction or abstraction would be over-engineering?
(c) Is the logic genuinely a rendering concern rather than domain logic?
(d) Is there an explicit adapter naming signal that the coupling is intentional?
(e) Is this really Agent 2's yardstick territory (file length, prop count) rather than a design-level split?
(f) For shared-component reuse findings, did you actually Glob/Grep `common/`, `shared/`, `ui/`, `primitives/`, `components/base/`, `design-system/`, and `packages/*/ui/` and find a concrete existing component covering the new one's shape? Without a pointer to a real file, the finding is noise.
(g) For UI-library suggestions, did you check whether the repo already imports from a UI library? If yes, drop.

If any of these holds, drop the finding. Only when none apply AND you can state a concrete cost (e.g., "API field rename would ripple through N components", "this inline JSX chunk has its own domain rules and testing it requires rendering the whole parent", "this new `Dropdown` has the same props and render shape as `common/ui/Dropdown.tsx` at line 23") is the finding reportable.

## SEVERITY CALIBRATION (agent-specific)

- **must-fix** — essentially never. Design-level coupling doesn't crash. Reserve for cases where the coupling hides a correctness bug (e.g., a business rule in JSX that's wrong but hard to spot because it's tangled with layout).
- **should-fix** — concrete maintenance cost in this codebase: a change to the data shape would ripple through N components; a view-model or extraction pattern exists elsewhere but this component doesn't use it; the domain logic in render is demonstrably hard to test in isolation; two sibling components are coupled in a way that blocks an obvious upcoming change.
- **suggestion** — default. "Worth considering an abstraction or split at this boundary, but the current code is not broken."

## ZERO FINDINGS IS VALID

Design-coupling and extraction issues are not universal. A well-layered codebase may have zero findings here and that's a positive signal. Do NOT manufacture findings.

## OUTPUT

Standard structured-finding schema from the shared contract. Category: `"abstraction-coupling"`. Evidence should include the relevant JSX/component code + the upstream data source or the coupled sibling if relevant, so the verifier can confirm the boundary problem manifests.

## Inheritance

Receives the full shared contract (`_shared-contract.md`): DIFF-ONLY RULE, EFFICIENCY RULES, FALSE POSITIVE WARNINGS, ZERO FINDINGS IS VALID, BEFORE REPORTING: DISPROVE FIRST, progress reporting. No Agent-7-specific exemptions.
