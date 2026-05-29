# Output Template

Use this exact format when outputting the PR review. All voice comments must be written in the user's voice per `~/.claude/VOICE.md`.

## Contents

- [Format](#format) — top-level review structure (header, sections, file groups)
- [Summary](#summary) — overall summary section rules
- [My previous comments on this PR](#my-previous-comments-on-this-pr) — table of the user's prior PR comments + status
- [Findings at a glance](#findings-at-a-glance) — single-row-per-finding overview table
- [`{file/path/Component.tsx}`](#filepathcomponenttsx) — per-file finding-block layout
- [Unified Finding Block](#unified-finding-block) — Severity / Description / Diff / Effort / voice Comment template (the canonical block both output modes share)
- [Severity Icons](#severity-icons) — `must-fix` / `should-fix` / `suggestion` glyphs
- [Voice Rules (Quick Reference)](#voice-rules-quick-reference) — voice-comment style cheatsheet
- [Severity-specific phrasing](#severity-specific-phrasing) — voice-comment phrasing per severity tier
- [Examples by Severity](#examples-by-severity) — concrete before/after voice examples
- [Anti-Nitpick Filter](#anti-nitpick-filter) — the 4-question test every finding must pass

---

## Format

```markdown
# PR Review: {PR title}
**PR:** {link} | **Files changed:** {N} | **Lines:** +{additions} / -{deletions}
**Findings:** {N total} ({must-fix} must-fix, {should-fix} should-fix, {suggestion} suggestions){optional: " — {M} suppressed by existing comments" when Step 5a dropped findings against existing PR comments}

## Summary
{Optional 2-3 sentence overall assessment in the user's voice. Be direct, state the main observation first, mention any themes across the findings. Omit if there's nothing useful to say.}

## My previous comments on this PR
{Render this section ONLY when Step 5d produced classified rows — i.e. review-comments output mode AND PR URL mode AND the user has ≥1 prior comment on the PR. Omit entirely otherwise (first-pass review, local review, plan-fixes mode, or the user hasn't commented yet).}

| File | Line | Comment | Status |
|------|------|---------|--------|
| `ComponentA.tsx` | 42 | "can you add a cleanup to this effect?" | Still open — line unchanged |
| `useData.ts` | 15 | "this cast bypasses the type checker" | Still open — line changed |
| `Foo.tsx` | 99 | "rename to isLoading" | Resolved |
| _(PR-wide)_ | — | "overall looks good, a few inline notes" | Open — unclear |

Sort rows: **Still open** (any flavor) first, then **Open — unclear**, then **Outdated**, then **Resolved**. The things the user most likely wants to re-check float to the top. Comment column shows the first ~100 chars of the thread's first comment on one line (truncate with `…` if longer). File cell shows `_(PR-wide)_` for issue-level comments with no file/line.

## Findings at a glance

| # | File | Line | Severity | Issue | Effort |
|---|------|------|----------|-------|--------|
| 1 | `ComponentA.tsx` | 42 | must-fix | Missing null check on user prop | trivial |
| 2 | `ComponentA.tsx` | 117 | should-fix | Effect missing cleanup | small |
| 3 | `useData.ts` | 15 | should-fix | Stale closure in callback | small |

===

## `{file/path/Component.tsx}`

{unified finding block for first finding in this file}

===

{unified finding block for next finding in this file, sorted by line}

===

## `{next/file/path.ts}`

{unified finding block}
```

Use `===` between every issue boundary — between consecutive findings within a file AND between the last finding of one file and the next file's `## file/path` header. The only `---` rules that survive are the section breaks (PR header → My-previous-comments / Findings-at-a-glance, Summary, etc.); finding boundaries always use `===`.

## Unified Finding Block

Every finding in every mode uses exactly this shape. The **Comment (in my voice)** section is present only in review-comments mode. The **Diff** block has two flavors — single-fix (default, one diff block) and multi-fix (one diff block per labeled alternative, when the verifier emitted `fix_alternatives`).

### Single-fix shape (default)

Use this when the verifier emitted a primary `fix_snippet` and no `fix_alternatives` (or an empty array):

```markdown
### `{filepath}:{line}` — {issue title}

**Severity:** {must-fix | should-fix | suggestion}

**Description:** {1–2 sentence technical explanation of the specific problem. Not a restatement of the title.}

**Diff:**
```diff
- {only the flagged line(s) — no surrounding context}
+ {concrete replacement code — from Step 5b fix_snippet or Agent 6 proposed_fix}
```

**Effort:** {trivial | small | medium}

**Comment (in my voice):**
```
{voice-mode copy-paste-ready comment for GitHub/ADO. One logical line per paragraph — no soft wraps. Embedded code blocks may still be multi-line. See VOICE.md.}
```
```

### Multi-fix shape (when `fix_alternatives` is non-empty)

When the verifier emitted 1+ entries in `fix_alternatives`, render the primary as **Suggested fix A** and each alternative as **B**, **C** (cap at C — Step 5b limits alternatives to 2). Each option carries its own description and effort; drop the standalone bottom-line `**Effort:**` field — per-option effort replaces it:

```markdown
### `{filepath}:{line}` — {issue title}

**Severity:** {must-fix | should-fix | suggestion}

**Description:** {1–2 sentence technical explanation. May also note that there are multiple reasonable fixes — the per-option descriptions explain each.}

**Suggested fix A — {label_A}** (effort: {effort_A})
{description_A — what this approach does and why pick it}
```diff
- {flagged line(s)}
+ {snippet_A — the primary fix_snippet}
```

**Suggested fix B — {label_B}** (effort: {effort_B})
{description_B}
```diff
- {flagged line(s)}
+ {snippet_B — fix_alternatives[0].snippet}
```

{repeat for C if fix_alternatives has 2 entries}

**Comment (in my voice):**
```
{voice-mode copy-paste-ready comment. One logical line per paragraph — no soft wraps. Embedded code blocks may still be multi-line. When suggesting alternatives in a review comment, present both inline (e.g. "you could either pull this into a hook, or just memo the callback — happy with either"). See VOICE.md.}
```
```

### Diff block rule

Include the **Diff** block only when the verification agent produced a concrete replacement snippet. Some findings (file too long, missing test coverage, design-level refactor) don't map to a one-shot patch — in those cases, omit the **Diff** block entirely and let the **Description** carry the fix direction in prose. Do not force pseudocode into the block; an empty slot is better than a mushy one. Agent 6 already drops findings without a `proposed_fix` — that stricter rule for suppression-removable findings still stands.

When `fix_alternatives` exists but every snippet (primary + alternatives) is null, fall back to single-fix shape with the **Diff** block omitted — multi-fix rendering only kicks in when at least 2 of the options have concrete snippets to compare. If only the primary has a snippet and the alternatives are all null, render single-fix and mention the alternative directions in **Description** instead.

## Severity Icons

| Level | Icon + Label | Use When |
|-------|-------------|----------|
| Must fix | must-fix | Would break in production — bugs, security issues, data loss |
| Should fix | should-fix | Could cause problems over time — perf issues, missing cleanup, stale closures |
| Suggestion | suggestion | Would improve quality — reuse opportunities, design improvements, minor edge cases |

## Voice Rules (Quick Reference)

When writing the **Comment (in my voice)** section, follow these rules from VOICE.md:

**DO:**
- Dive in with the point — no greeting on individual comments
- State the issue in the first sentence
- Use contractions: I'm, don't, we're, can't, shouldn't, I'll, I've
- Use "Can you..." framing — collaborative, not imperative
- Use abbreviations naturally: "FYI", "np", "NBD"
- Say "please" when asking someone to change something
- Offer help: "let me know if it doesn't fit", "happy to chat about this"
- Keep it short — most comments should be 1-3 sentences
- End without ceremony — no sign-off, just stop when you're done
- Be genuine — occasional empathy where it fits
- Use "--" sparingly — once per comment max, not in every sentence
- Prose collapses to one line per paragraph — no soft wrapping. If you want a paragraph break, use a blank line. Embedded code blocks (rare) keep their natural newlines.

**DON'T:**
- Sign off with name ("Thanks, Michael")
- Use formal language ("Per the documentation", "As per best practices")
- Use corporate buzzwords ("leverage", "synergy")
- Use passive voice when direct works
- Hedge excessively — state things confidently, add caveats only when needed
- Use emoji in voice comments (professional context)
- Write walls of text for simple observations
- Hard-wrap prose at ~70 chars — let the renderer wrap. Hard wraps break triple-click select-line copy-paste.

## Severity-specific phrasing

The voice differs per severity. Tune the comment to match.

### `suggestion` — lead with `nit:` when truly optional

For truly optional / cosmetic / "while you're in here" findings, prefix the comment with `nit: `. This is the single most common marker in real review comments. Use `nit:` whenever the finding is genuinely take-it-or-leave-it — token swaps, naming, small reuse opportunities, micro-extractions.

Soft-opinion openers also belong on suggestions when stating a preference rather than reporting a fact:
- "I lean towards..."
- "My instinct on this is..."
- "Consider..."
- "Worth..."

When the suggestion is non-blocking and the reviewee shouldn't feel compelled to act, **say so explicitly**:
- "Np if not."
- "No need to block on it for now, though."
- "Not something for now, but keep an eye out..."
- "I'm _okay_ with this for now, so long as [condition]."

End suggestions with an invitation to push back when there's any judgment call:
- "Thoughts?"
- "Open to input"
- "Feel free to push back."
- "Your call."

Skip `nit:` when the suggestion is more substantive (design-level extraction, reuse miss with real downstream impact) — those land closer to should-fix tone even if labeled suggestion.

### `should-fix` — direct, with offered help

State the issue, propose the fix, offer help. Don't `nit:` these — they're not optional.

- "[problem]. Can you [fix]? Let me know if you need a hand with it"
- "[problem] — [proposed fix]. Let me know if a caller I missed [edge case]."

### `must-fix` — concise, confident, no hedging

Direct statement of the bug and the fix. No `nit:`, no soft openers, no "if you have time". Still collaborative ("Can you swap..."), but no caveat softeners.

### CAPS for emphasis

Capitalize a single word for soft emphasis instead of using exclamation marks. Sparingly — once per comment max. Examples from real comments: "I'm **VERY** open to other ideas", "**LOTS** of overlap", "**BIG** changes". Render as plain caps in the comment (no markdown bold) — the comment is copy-pasted into GitHub/ADO.

## Examples by Severity

### Must Fix Example

```markdown
### `src/components/UserList.tsx:42` — State mutated directly

**Severity:** must-fix

**Description:** `users.push(newUser)` mutates the array in place. React won't detect the change because the reference is the same, so the UI won't update.

**Diff:**
```diff
- users.push(newUser);
+ setUsers([...users, newUser]);
```

**Effort:** trivial

**Comment (in my voice):**
```
I think this might be a state mutation issue. users.push() mutates the array directly so React won't pick up the change. Can you try swapping it for setUsers([...users, newUser])? That way React sees a new reference and re-renders.
```
```

### Should Fix Example

```markdown
### `src/components/Poller.tsx:87` — useEffect missing cleanup

**Severity:** should-fix

**Description:** The `setInterval` is never cleared. If this component unmounts and remounts, intervals will stack and degrade performance.

**Diff:**
```diff
- useEffect(() => {
-   setInterval(tick, 1000);
- }, []);
+ useEffect(() => {
+   const id = setInterval(tick, 1000);
+   return () => clearInterval(id);
+ }, []);
```

**Effort:** trivial

**Comment (in my voice):**
```
The setInterval here doesn't have a cleanup, so if the component unmounts it'll keep running. Can you return a clearInterval from the useEffect? Let me know if you need a hand with it
```
```

### Suggestion Example

```markdown
### `src/components/Timestamp.tsx:15` — Existing utility available

**Severity:** suggestion

**Description:** `formatTimestamp()` is defined here but `formatDate()` in `src/utils/dates.ts` does the same thing with timezone support.

**Diff:**
```diff
- const formatted = formatTimestamp(date);
+ import { formatDate } from '@/utils/dates';
+ const formatted = formatDate(date);
```

**Effort:** small

**Comment (in my voice):**
```
nit: we already have formatDate in utils/dates.ts that handles this. Should be able to drop it in here, but let me know if it doesn't quite fit your use case. Np if not.
```
```

### Suppression-Removable Example (Agent 6)

Findings with `category: "suppression-removable"` always carry a concrete `proposed_fix` — Agent 6 drops any finding without one. The `proposed_fix` flows into the `+` lines of the **Diff** block, and the **Comment (in my voice)** must inline the fix so the author has a concrete path rather than a generic "fix this" nag.

```markdown
### `src/rows/renderRow.tsx:47` — Removable @ts-ignore — no-explicit-any

**Severity:** should-fix

**Description:** The `@ts-ignore` hides an `any` param that can be typed from its call sites — all three callers pass `UserProfile`, so widening the signature removes the need to suppress.

**Diff:**
```diff
- // @ts-ignore
- function renderRow(user) {
-   return <Row name={user.name} />;
- }
+ function renderRow(user: UserProfile) {
+   return <Row name={user.name} />;
+ }
```

**Effort:** trivial

**Comment (in my voice):**
```
This @ts-ignore looks removable — the callers all pass UserProfile, so typing the param directly should work:

    function renderRow(user: UserProfile) {
      return <Row name={user.name} />
    }

Can you drop the suppression and use that? Let me know if a caller I missed passes something else
```
```

## Anti-Nitpick Filter

Before including a finding, it must pass this test. The finding must answer YES to at least one:

1. Would this cause a bug in production?
2. Could this hurt performance under load or over time?
3. Would this confuse a future reader trying to understand the code?
4. Does this miss a reuse opportunity that already exists in the codebase?

If the answer is NO to all four, drop the finding. Do NOT include it.

**Explicitly skip:**
- Formatting/style issues (ESLint/Prettier handles these)
- Adding docstrings where code is self-explanatory
- Trivial naming preferences
- Type annotations on obvious types
- Import ordering
- "You could also do X" when current approach is fine
- Single-use abstractions / premature DRY
- Missing tests (unless PR claims test coverage)
