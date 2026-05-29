# Walkthrough Phase 2 — Decisions

## Preconditions

Assumes the following state is set:
- `findings[]` (non-empty) — set by `phases/04-verify.md`.

On second pass (when `rescan_count == 1` after `phases/walkthrough/05-rescan.md`
looped back), assumes additionally:
- `findings_rescan[]` (non-empty) — set by `phases/walkthrough/05-rescan.md`.

If any required precondition is absent or empty, STOP and re-read the
producing phase. Do NOT proceed with defaults, do NOT improvise, do NOT
silently continue.

## Re-entry guard

If `decisions[]` already has the same length as `findings[]`, Phase 2's
first pass is done — skip Phase 2 entirely and read
`phases/walkthrough/03-apply.md` next. Do NOT re-prompt the user for any
finding that already has a decision.

On second pass: if `decisions_rescan[]` already has the same length as
`findings_rescan[]`, skip Phase 2 and read
`phases/walkthrough/03-apply.md` next (which will apply
`decisions_rescan[]` only).

## Pass selection

This file is read once at the start of Phase 2 and walks through findings
in-file. Pick the iteration target based on `rescan_count`:

- **First pass** (`rescan_count == 0` or unset): iterate `findings[]`,
  write `decisions[]`.
- **Second pass** (`rescan_count == 1` after `05-rescan.md` looped back):
  iterate `findings_rescan[]`, write `decisions_rescan[]`. Everything
  else in this file is identical — only the input and output arrays
  change.

In the text below, "the array" means whichever array applies to the
current pass.

## Picker layouts (summary)

The Decision picker has three layouts. Pick the one that matches the
count of available fix options for the current finding (`primary +
fix_alternatives`, dropping any whose snippet is null):

- **Layout 1 — single fix:** one concrete fix snippet. Options: `Fix`,
  `Skip`, `Discuss`, `Fix all remaining`.
- **Layout 2 — two fix alternatives:** primary + 1 alternative. Options:
  `Fix A`, `Fix B`, `Skip`, `Discuss`. `Fix all remaining` moves to
  `Other`.
- **Layout 3 — three fix alternatives:** primary + 2 alternatives.
  Options: `Fix A`, `Fix B`, `Fix C`, `Skip`. Both `Fix all remaining`
  and `Discuss` move to `Other`.

Full picker schema for each layout is in the **Per-finding template**
below.

## Per-finding template (repeat for each finding in the array)

Walk through findings **one at a time**, grouped **file-by-file and
sorted by line number within each file** (matches how the author will
scan the code and matches the review-comments rendering). **Every
per-finding decision MUST be collected via the `AskUserQuestion` tool —
never via free-form text prompts.** This is plan-fixes mode: the user is
reviewing their own code, and the button picker is how they want to
drive the walkthrough. **Do NOT apply any edits during this phase** —
only collect decisions.

For each finding:

1. **Emit the unified finding block** from `references/output-template.md`
   (four labeled sections — Severity, Description, Diff, Effort — no
   Comment) as plain output so the user has the full context to read.
   Use **single-fix shape** when the verifier emitted no
   `fix_alternatives` and **multi-fix shape** (Suggested fix A / B / C
   — each its own diff block) when it did. See
   `references/output-template.md` for both layouts.

2. **Issue a single `AskUserQuestion` call** with the layout that
   matches the available fix-option count:

**Layout 1 — single fix (1 option):** the default. Used when there's only one concrete fix snippet.
- `question`: `"Decision for finding {n}/{total} — {file}:{line}?"`
- `header`: `"Decision"` (max 12 chars)
- `multiSelect`: `false`
- `options` (exactly four, in this order):
  1. `{ label: "Fix", description: "Apply the suggested fix during Phase 3." }` — prepend `"(Recommended) "` to the label when severity is `must-fix`.
  2. `{ label: "Skip", description: "Leave the code unchanged." }`
  3. `{ label: "Discuss", description: "Talk it through before deciding — no edits yet." }`
  4. `{ label: "Fix all remaining", description: "Mark this and every later finding as Fix and jump to Phase 3." }`

**Layout 2 — two fix alternatives:** used when there are exactly two concrete fix snippets (primary + 1 alternative). `Fix all remaining` moves to the auto-provided `Other` field.
- `question`: same shape, with body `"This finding has two reasonable fixes — pick one or skip/discuss. {file}:{line}"`
- `header`: `"Decision"`
- `multiSelect`: `false`
- `options`:
  1. `{ label: "Fix A — {label_A}", description: "{description_A — 1-line summary}", preview: "{snippet_A}" }`
  2. `{ label: "Fix B — {label_B}", description: "{description_B — 1-line summary}", preview: "{snippet_B}" }`
  3. `{ label: "Skip", description: "Leave the code unchanged." }`
  4. `{ label: "Discuss", description: "Talk it through before deciding — no edits yet." }`

The `preview` field on each Fix option carries the fenced code (without
the surrounding markdown fences — pass the raw snippet body) so the
picker UI renders the alternatives side-by-side. Truncate previews
longer than ~30 lines and add a trailing `// …` comment so the panel
stays readable; the full snippet still exists in the rendered finding
block above.

**Layout 3 — three fix alternatives:** used when there are exactly three concrete fix snippets. Both `Fix all remaining` and `Discuss` move to `Other`.
- `options`:
  1. `{ label: "Fix A — {label_A}", description: "...", preview: "{snippet_A}" }`
  2. `{ label: "Fix B — {label_B}", description: "...", preview: "{snippet_B}" }`
  3. `{ label: "Fix C — {label_C}", description: "...", preview: "{snippet_C}" }`
  4. `{ label: "Skip", description: "Leave the code unchanged." }`

Don't try to fit more than three fix variants into a single picker —
Phase 4 (verify) caps `fix_alternatives` at 2 entries (so primary +
alternatives ≤ 3) for exactly this reason. If the verifier somehow
returned more, drop the weakest before rendering.

If every snippet on the finding is null (design-level finding with no
concrete patch), use **Layout 1** unchanged — the `Fix` button there
means "I'll work this fix into the discussion-led implementation"
rather than applying a snippet. The Diff block in the rendered finding
above will already be omitted per the Diff block rule.

Do NOT add a fifth option in any layout — the auto-provided **Other**
path handles anything else (commonly `"stop"`, plus `Fix all remaining` /
`Discuss` overflow in Layouts 2 and 3).

### Handle Responses

Map the `AskUserQuestion` answer to a decision entry, then append to the
current pass's array (`decisions[]` or `decisions_rescan[]`):

- **`Fix`** (Layout 1) → Record `{ action: "fix", chosen_option: "primary" }`. Output `Marked for fix.` and move to next finding. Do NOT apply the edit yet.
- **`Fix A — …` / `Fix B — …` / `Fix C — …`** (Layouts 2-3) → Record `{ action: "fix", chosen_option: "A" }` (primary) / `"B"` / `"C"`. Output `Marked for fix (option {A|B|C}).` and move to next finding. Phase 3 applies that specific snippet.
- **`Skip`** → Record `{ action: "skip" }`. Output `Skipped.` and move to next finding.
- **`Discuss`** (Layout 1 or 2) → Engage in conversation about the finding. The user may ask questions, propose new fix variants, or merge two of the alternatives. Once resolved, re-issue the same `AskUserQuestion` call — with whatever option set now reflects the post-discussion fix list (the verifier's `fix_alternatives` on this finding is mutable here; if discussion produced a third alternative, switch to Layout 3 on the re-ask). Still do NOT apply edits.
- **`Fix all remaining`** (Layout 1) → Mark this finding AND every remaining finding as `{ action: "fix", chosen_option: "primary" }` (each with `chosen_option = "primary"` since the user didn't see a picker). Output `Marked all remaining ({N}) for fix (primary suggestion each).` and skip ahead to the Verification Preferences question, then Phase 3.
- **`Other` (free text)** — interpret the text:
  - `"stop"` (or equivalent like `"that's enough"`, `"skip the rest"`) → Stop collecting decisions. Unreviewed findings are recorded as `{ action: "skip" }`. Skip ahead to the Verification Preferences question, then Phase 3.
  - `"discuss"` (Layout 3 fallback) or `"fix all remaining"` (Layouts 2-3 fallback) → execute the corresponding action above.
  - `"A"` / `"B"` / `"C"` shorthand → record `{ action: "fix", chosen_option: that letter }`.
  - anything else → treat as `Discuss` and engage in conversation, then re-issue the `AskUserQuestion` call with whatever option set the discussion produced.

## Verification Preferences

After every finding has a decision (or the user typed `stop` / picked
`Fix all remaining`), ask about verification depth before applying.

**Skip this question** if no must-fix findings were marked for fix in
the current pass — default to `verification_level = "none"` (Level 1
standard checks still run automatically). On the second pass, the
verification level set in the first pass is preserved; re-ask only when
the first pass was `"none"` and the second pass has must-fix findings
marked for fix.

Use a single `AskUserQuestion` call with exactly:

- `question`: `"How should I verify the fixes? TypeScript, linter, and the test suite run automatically. For the {N} must-fix finding(s) marked for fix, I can also:"`
- `header`: `"Verify"` (max 12 chars)
- `multiSelect`: `false`
- `options`:
  1. `{ label: "Regression tests", description: "After applying, write tests that catch each must-fix issue so it can't regress." }`
  2. `{ label: "TDD validation", description: "Before each must-fix fix, write a failing test, then confirm the fix makes it pass." }`
  3. `{ label: "Both", description: "TDD for must-fix + regression tests for should-fix findings." }`
  4. `{ label: "None", description: "Just the standard TypeScript / lint / test checks." }`

Map the answer to `verification_level`:
- `Regression tests` → `"regression"`
- `TDD validation` → `"tdd"`
- `Both` → `"both"`
- `None` → `"none"`

If the user answers via `Other`, interpret the free text and pick the
closest of the four; only re-ask when genuinely ambiguous.

## Next

If more findings remain in the queue:
  - Move to the next finding (file-by-file, sorted by line within file).
  - Return to the "Per-finding template" block above and run it for
    that finding. Do NOT re-read this file.

After every finding has a decision (or the user typed `stop` / picked
`Fix all remaining`):
  - Run the "Verification Preferences" question once (per the rules above).
  - Then read `phases/walkthrough/03-apply.md`.
