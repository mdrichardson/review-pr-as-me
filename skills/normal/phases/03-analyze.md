# Phase 3 — Analyze

Dispatch up to 9 parallel analysis agents over the pre-computed inputs
from Phase 2. Each agent returns a list of findings keyed to file + line.

> **Fast-mode override.** If you loaded `../fast/SKILL.md`, apply its
> model-stepdown override table (and Agent 5 skip rule) before using
> the model assignments below. The fast skill enumerates which phase
> files its overrides target.

## Preconditions

Assumes the following state is set:
- `repo_path`
- `output_mode`
- `standards_summary`
- `utility_index`
- `import_graph`
- `pre_read_contents`
- `categorized_files`
- `suppression_list` (may be empty array)

If any required precondition is absent or empty, STOP and re-read
`phases/02-pre-compute.md` (the producing phase). Do NOT proceed with
defaults, do NOT improvise, do NOT silently continue.

## Checklist

Read `references/react-review-checklist.md` (relative to this skill's
directory: `~/.claude/plugins/mdrichardson/review-pr-as-me/skills/normal/references/`)
for the detailed checklist that agents reference.

## Dispatch

Launch agents in a SINGLE message (parallel dispatch). The agents receive:
- The PR diff (changed files with hunks)
- **Pre-read file contents** from Phase 2 (agents should NOT re-read these)
- The `standards_summary` from Phase 2
- The `utility_index` from Phase 2
- Their specific checklist section
- Their **primary file list** and the full changed file list

### Prompt-cache-friendly structure

Each agent prompt is assembled in this fixed top-to-bottom order. The
goal is a byte-identical shared prefix across the parallel dispatch,
so Anthropic's automatic prompt cache covers everything except each
agent's tail. Diverging from the order — e.g. inlining boilerplate
per-agent — fragments the prefix and breaks caching, so dispatching
9 agents costs ~9× tokens instead of ~1×. Stick to the order:

```
[A] SHARED CONTRACT BLOCK            ← identical across all 9 agents
    - DIFF-ONLY RULE
    - EFFICIENCY RULES
    - ZERO FINDINGS IS VALID
    - BEFORE REPORTING: DISPROVE FIRST
    - FALSE POSITIVE WARNINGS
    - PROGRESS reporting protocol

[B] SHARED CONTEXT BLOCK             ← identical across all 9 agents
    - Codebase standards summary
    - Utility index
    - Barrel exports list
    - Pre-read FILE CONTENTS section
    - PR diff (with rename collapse / delete-filter applied)
    - Import graph + changed-exports summary
    - Suppression list
    - Rule-config summary
    - Categorize-changed-files table for this PR
    - review_instructions (if any)

[C] AGENT-SPECIFIC TAIL              ← unique per agent
    - Agent name + ownership statement (YOU OWN: …)
    - Primary file list for this agent
    - Agent-specific WHAT TO FLAG / WHAT NOT TO FLAG
    - Severity calibration overrides for this agent
    - Output format expectations
```

The model's caching layer keys on the longest stable prefix it sees in
a short window. When [A] + [B] are identical bytes across the 9
dispatched prompts in the same parent message, agents 2-9 hit cache
for everything except their own tail. On a typical 10-file PR this is
~30K tokens of cached prefix per agent, so 9 agents pay ~30K + 9 ×
tail_size ≈ 55K billed tokens instead of 9 × 35K = 315K.

This rule applies to:
- The analysis agents in this phase.
- The per-finding verification agents in Phase 4 (verify) — same shared
  structure (standards summary, file contents, diff context), with only
  the single-finding tail varying.

Do NOT inline the DIFF-ONLY / EFFICIENCY / DISPROVE-FIRST blocks into
each agent's tail individually — that fragments the prefix and breaks
caching. They go in [A], once, and apply to every agent unless the
agent description explicitly opts out (Agents 5, 6 opt out of subsets;
note their opt-outs in [C], not by removing them from [A]).

### Conditional dispatch rules

**Agent 5 (Scope Match)** runs only when the PR has a non-empty
`body`/description. Skip dispatching Agent 5 in local-review mode (no
PR) or when the PR body is empty — there is no stated scope to compare
against. (Phase 2 already gated the task creation on the same
condition, so the task list reflects the dispatch decision.)

**Agent 6 (Suppression Removability)** runs only when the pre-computed
`suppression_list` from Phase 2 is non-empty. Skip dispatching when no
new suppressions were added.

**Agent 7 (Abstraction & Coupling Boundaries)** runs only when
`categorized_files` contains at least one Component-category file. Skip
dispatch when no component files changed — utility-only and config-only
PRs have no render surfaces to examine.

## Model and Effort Assignment

Choose models based on the analysis depth each agent needs. The goal is
fast, focused analysis — not exhaustive exploration.

| Agent | Model | Rationale |
|-------|-------|-----------|
| Agent 1 — Hooks & React | **model: "opus"** | Hook correctness requires deep reasoning about closures, dep arrays, render cycles |
| Agent 2 — Reuse & Standards | **model: "sonnet"** | Pattern matching against standards — mechanical, fast |
| Agent 3 — Performance & Edge Cases | **model: "sonnet"** | Edge case analysis is systematic — sonnet handles well with pre-loaded context |
| Agent 4 — Cross-Cutting Impact | **model: "sonnet"** | Consumer tracing is mostly grep + read — pre-computed import graph eliminates the hard part |
| Agent 7 — Abstraction & Coupling Boundaries | **model: "opus"** | Design-level judgment across two dimensions (data/render boundary + component extraction). Requires reasoning about what SHOULD be abstracted or split vs. what is legitimately one piece. False-positive risk is high, so the deeper model is worth the cost. |
| Agent 5 — Scope Match | **model: "haiku"** | Semantic comparison of PR title/body against diff summary — no code reasoning, no file reading |
| Agent 6 — Suppression Removability | **model: "haiku"** | Narrow, local check — single suppression + surrounding code, no cross-file reasoning; pre-loaded context makes haiku sufficient |
| Agent 8 — Comment Hygiene | **model: "haiku"** | Narrow text scan over pre-loaded file contents — no code reasoning, no cross-file tracing |
| Agent 9 — Other | **model: "sonnet"** | Pattern-matching catch-all for repeated-but-orthogonal review smells (redundant guards, telemetry gaps, test conventions, content/i18n) — sonnet handles the breadth well with pre-loaded context |

Since file contents are pre-loaded, agents spend their time reasoning
about the code rather than reading it. This means sonnet agents perform
closer to opus quality because the bottleneck (context gathering) is
removed.

## Agent Prompts

Each agent's full prompt content lives in `references/agents/`. Inline
content was extracted there to keep this phase navigable; the
orchestrator reads each file when assembling the corresponding agent
prompt.

Loading order when assembling a prompt for agent N (per
"Prompt-cache-friendly structure" above):

1. **[A] Shared contract block** — read `references/agents/_shared-contract.md` once and use it as the top-of-prompt block for every agent. Skip the sections listed under "Per-agent exemptions" for agents 5, 6, and 8.
2. **[B] Shared context block** — `standards_summary`, `utility_index`, barrels, pre-read FILE CONTENTS, PR diff, `import_graph`, `suppression_list`, `rule_config_summary`, `categorized_files` table, `review_instructions`. Same bytes for every agent.
3. **[C] Agent-specific tail** — read the matching agent reference file:

| Agent | Reference file | Dispatch condition |
|-------|----------------|--------------------|
| 1 — Hooks & React Patterns | `references/agents/1-hooks-react.md` | Always (when there are Component or Hook changed files) |
| 2 — Reuse & Standards | `references/agents/2-reuse-standards.md` | Always |
| 3 — Performance & Edge Cases | `references/agents/3-performance-edge-cases.md` | Always (when there are Component or Hook changed files) |
| 4 — Cross-Cutting Impact | `references/agents/4-cross-cutting-impact.md` | Always (when import graph has consumers) |
| 5 — Scope Match | `references/agents/5-scope-match.md` | Skip when PR body is empty / local-review mode / vague scope |
| 6 — Suppression Removability | `references/agents/6-suppression-removability.md` | Skip when `suppression_list` is empty |
| 7 — Abstraction & Coupling Boundaries | `references/agents/7-abstraction-coupling.md` | Skip when no Component-category files changed |
| 8 — Comment Hygiene | `references/agents/8-comment-hygiene.md` | Always |
| 9 — Other | `references/agents/9-other.md` | Always |

Each reference file is self-contained: it states the agent's model,
primary file rule, ownership, what to flag, what not to flag, output
schema, and any exemptions from the shared contract. The orchestrator
does not need to consult any other source when building [C] for that
agent.

## Substep Progress Tracking

When dispatching agents, also create **substep tasks** for visibility.
For each agent, create one child task per primary file it will analyze:

```
Example for Agent 1 with 4 primary files:
  4a. "Hooks: UserProfile.tsx"        activeForm: "Analyzing UserProfile.tsx"
  4b. "Hooks: useAuth.ts"             activeForm: "Analyzing useAuth.ts"
  4c. "Hooks: SettingsPanel.tsx"       activeForm: "Analyzing SettingsPanel.tsx"
  4d. "Hooks: usePermissions.ts"      activeForm: "Analyzing usePermissions.ts"
```

Create these substep tasks immediately when dispatching agents. Mark
each substep `completed` as its parent agent reports progress (the
"PROGRESS: done with {filename}" lines). When all substeps for an agent
are done, mark the parent agent task `completed`.

**Cap substeps at 5 per agent.** If an agent has more than 5 primary
files, group the extras under a single "N remaining files" substep.
This keeps the task list readable.

## Output

Collect each agent's findings into `agent_findings[]` — a flat array of
finding records that Phase 4 (verify) consumes. Each record carries the
source agent identifier so dedup can use it.

## Next: read `phases/04-verify.md`.
