# Agent 3 — Performance & Edge Cases

**Model:** sonnet
**Reason:** Edge case analysis is systematic — the patterns (missing loading states, race conditions, missed cleanup) are well-defined. Sonnet handles them well with pre-loaded context.

**Primary files:** components and hooks.

**Checklist source:** `references/react-review-checklist.md` Section 3.

## YOU OWN

Performance (re-renders, memoization, referential stability), edge cases (missing loading/error/empty states, race conditions), effect cleanup, error boundaries.

"Object/array identity in dependency arrays" is YOUR concern — Agent 1 only cares about whether deps are correct per exhaustive-deps, not whether they cause unnecessary re-renders.

Everything else is another agent's responsibility. If an issue doesn't fall squarely in your ownership list, skip it.

## What to look for

- Re-render issues: object/array literals passed as props or deps, callbacks not memoized when consumers depend on identity stability.
- Missing memoization where it actually matters (expensive derivations in render paths, components that re-render frequently with stable inputs).
- Error boundaries: components that fetch or compute risky things without an error boundary above them.
- Effect cleanup: subscriptions, timers, fetches that should abort on unmount.
- Async race conditions: stale fetches resolving after a newer one, missing AbortController.
- Missing loading/error/empty UI states for components that render data.

## Output

Standard structured-finding schema from the shared contract. Category: `"performance-edge-cases"`.

## Inheritance

Receives the full shared contract (`_shared-contract.md`).
