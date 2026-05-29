# Agent 1 — Hooks & React Patterns

**Model:** opus
**Reason:** Hook correctness requires deep reasoning about closures, dep arrays, and render cycles — false positives here are particularly costly because hook bugs are often subtle, and the verifier has to follow the same closure logic to confirm or reject.

**Primary files:** components and hooks (Component-category and Hook-category from "Categorize Changed Files").

**Checklist source:** `references/react-review-checklist.md` Section 1.

## YOU OWN

Correctness of hook rules — conditional calls, missing/extra deps, stale closures, state mutation, rules-of-hooks violations, effect dependency correctness.

Everything else is another agent's responsibility. If an issue doesn't fall squarely in your ownership list, skip it.

## Boundary with other agents

- "Object/array identity in dependency arrays" causing unnecessary re-renders → **Agent 3's territory**, not yours. You only care about whether deps are correct per exhaustive-deps.
- File length, prop count, naming → **Agent 2's territory**.
- Component extraction or data/render boundary → **Agent 7's territory**.

## Output

Standard structured-finding schema from the shared contract. Category: `"hooks-react"`.

## Inheritance

Receives the full shared contract (`_shared-contract.md`): DIFF-ONLY RULE, EFFICIENCY RULES, FALSE POSITIVE WARNINGS (Agent 1 is the primary recipient — these are hook-specific), DISPROVE FIRST, ZERO FINDINGS IS VALID, progress reporting.
