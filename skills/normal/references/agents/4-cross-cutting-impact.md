# Agent 4 — Cross-Cutting Impact

**Model:** sonnet
**Reason:** Consumer tracing is mostly grep + read. The pre-computed import graph eliminates the hard part (discovery), so the remaining work is well-suited to sonnet.

**Primary files:** **NOT** the changed files — Agent 4's primary list is the **consumers** of the changed files. Agent 4 receives the pre-computed import graph + changed-exports summary from Step 3 instead of the full diff.

This agent looks **outward** from the diff — Agents 1, 2, 3 look inward at the changed code itself.

## YOU OWN

Cross-cutting impact: how the changed code affects files OUTSIDE the diff that consume it.

## Scope rule (agent-specific)

```
SCOPE RULE: Your only job is finding issues in files OUTSIDE the diff
that are affected BY the diff — Agents 1, 2, 3 already cover inward-
looking analysis of the changed files themselves, so anything you flag
inside a changed file would be a duplicate of their work. Every finding
you report must reference a file that is NOT in the diff's changed file
list. If the issue is inside a changed file, skip it.

DIFF SCOPE: The changed file list you receive covers ALL files in the diff —
whether from a PR, a branch vs main (`git diff main...HEAD`), or uncommitted
changes. ALL listed files are in scope. Do not limit yourself to working-tree
changes when the diff is branch-scoped. If you see files in the list that don't
appear in your local `git diff`, it's because they were changed in earlier commits
on the branch — they are still in scope.
```

## What to look for (per changed file)

1. **Find consumers:** Use the pre-computed import graph. Read files that consume changed components, hooks, or functions.
2. **Check prop/API contract changes:** If a component's props, a hook's return type, or a function's signature changed, verify all call sites still work. Flag breaking changes that aren't updated everywhere.
3. **Trace data flow across boundaries:** If the diff changes how data is shaped, filtered, or passed — follow it upstream (who provides it?) and downstream (who consumes it?). Flag mismatches.
4. **Detect missing co-changes:** If a shared type, constant, or config was updated, check if all files referencing it were also updated. Flag files that reference the old shape/value.
5. **Check context/provider impact:** If a context value or provider changed, find all `useContext` consumers and verify they handle the new shape.

## Efficiency rules (Agent 4 specific)

```
EFFICIENCY RULES specific to Agent 4:
- Use the PRE-COMPUTED IMPORT GRAPH provided below — do NOT grep for consumers
  from scratch. The graph maps each changed file to its consumer files.
- Read the FULL content of consumer files, not just the import line — you need
  to see how the consumed export is actually used.
- Only report findings where the interaction is actually broken or risky.
  Do NOT flag consumers that are unaffected by the change.
- You MUST report at least 1 consumer grep per changed export. If no issues
  are found for an export, say "verified clean" for that export.
```

## Required output: Consumer Verification Log

Agent 4 MUST include this section at the end of its output, after the findings list:

```
## Consumer Verification Log
For each changed export/component/hook, list:
- Export: {name} from {file}
  Consumers found: {count} ({file1}, {file2}, ...)
  Status: {affected — see finding X / verified clean}
```

This log is mandatory even when no issues are found. It proves Agent 4 actually traced consumers rather than only analyzing the diff in isolation. If the log is missing, the output is considered incomplete — the orchestrator may re-dispatch.

## Output

Standard structured-finding schema from the shared contract, **plus** the Consumer Verification Log block above. Category: `"cross-cutting"`.

## Inheritance

Receives the full shared contract (`_shared-contract.md`).
