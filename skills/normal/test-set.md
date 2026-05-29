# Review Skill Test Set

Ground truth PRs with known issues caught during human review. Use early commits (before review fixes) to test whether the skill catches the same issues.

## Primary Test Set (human-reviewer ground truth)

### PR #594 — feat(feature-settings): add extension install dialog wizard
- **Early commit:** `ddf2e8cc` (before any review fixes)
- **Reviewer:** cjennison (human)
- **Fix commits:** `55aa8089`, `a3e648da`
- **Ground truth (13 human-flagged issues):**
  1. Missing telemetry on InstallSummaryStep (install outcomes)
  2. Missing telemetry on ExtensionInstallDialog (dialog interactions)
  3. Input not wrapped in Fluent Field (EnvironmentVariableStep)
  4. Dropdown not wrapped in Fluent Field (ConnectionPicker)
  5. Wizard state not reset when dialog reopened
  6. Unsafe type assertion (manifest as PackageManifest)
  7. Missing error handling for connection fetch failure
  8. installStarted state never reset on remount
  9. Step indicator dots lack ARIA accessibility
  10. Duplicate env vars from flatMap (no dedup by schemaname)
  11. Inaccurate JSDoc comment on DeploymentSettings
  12. Missing outcome telemetry (success/failure events)
  13. Duplicate connection refs from flatMap (no dedup by logicalname)

### PR #620 — feat(feature-settings): add connection settings table UI
- **Early commit:** `c7bf1fc7` (main feature commit, before fixes)
- **Reviewer:** cjennison (human)
- **Fix commits:** `55f97e80`
- **Ground truth (9 human-flagged issues):**
  1. A11y: StatusIcon decorative icons missing aria-hidden="true"
  2. Bug: fallbackDisplayName doesn't handle empty string after prefix stripping
  3. Bug: empty displayName renders broken icon fallback
  4. A11y: redundant aria-label on Badge (ConnectorDetailsTab)
  5. Missing visibility gate (extensionsEnabled) on Connections section
  6. Dead code: unused TYPE_KEYS constant and associated localization keys
  7. A11y: incorrect Tooltip relationship="label" should be "description"
  8. A11y: redundant aria-label on Badge (ConnectionDetailsDialog)
  9. Missing telemetry instrumentation for user interactions

### PR #597 — feat: add agents list page in agents-list package
- **Early commit:** `b70f75d6` (initial feature commit)
- **Reviewers:** Lexi (human), sergioe (human)
- **Fix commits:** `be9269f2`, `aea5d224`, `2cb0e8a6`, `ee6d5d51`, `4e75e9ab`
- **Ground truth (8 human-flagged issues):**
  1. createBot mutation object used as callback dep — unstable across renders (Lexi)
  2. Missing error handling: createBot.mutate and deleteBot.mutate fire without onError (Lexi)
  3. Premature telemetry: trackEvent fires before mutation completes (Lexi)
  4. A11y: DataGridRow has onClick but no onKeyDown (Lexi)
  5. Naming mismatch: locale file vs package name (Lexi)
  6. DRY violation: emptyState and errorState styles identical (Lexi)
  7. Wrong template: employee agent should use 'gptagent-1.0.0' not 'gptagent' (sergioe)
  8. Inconsistent naming: "employee" vs "declarative" agent terminology (sergioe)

### PR #644 — feat: license extension reminder banner + dialog
- **Early commit:** `daed06a6` (initial feature commit)
- **Reviewers:** sergioe, alex-kwiatkowski (human)
- **Fix commits:** `9599f2b1`, `088b150e`, `f360e0f9`
- **Ground truth (4 human-flagged issues):**
  1. Pluralization for "1 day" display (sergioe)
  2. Telemetry should use scenario pattern (sergioe)
  3. Naming: LicenseExtensionBanner should be LicenseExpirationBanner (alex-kwiatkowski)
  4. Banner slot architecture suggestion (alex-kwiatkowski) — deferred

### PR #601 — Add DA overview page slotting with agent-type routing
- **Early commit:** `cd03d122` (last commit before any fix)
- **Reviewer:** sergioe (human)
- **Fix commits:** `e9d51215`, `fbf2696d`, `1f868fc2`
- **Ground truth (5 human-flagged issues):**
  1. Missing telemetry for tab interactions (sergioe)
  2. Missing test coverage for DaOverviewPage (sergioe)
  3. Premature HostSlotRegistry entry (sergioe)
  4. Inconsistent naming: "employee" vs "declarative" (sergioe)
  5. Use options object instead of positional args in onNavigate (sergioe)

## Scoring

For each test PR, run the review skill against the early commit diff. Score:
- **hit**: skill found an issue that matches a ground truth item (+10)
- **miss**: skill failed to find a ground truth issue (0, tracked for false negative rate)
- **false_positive**: skill flagged something that wasn't in ground truth AND isn't a real issue (-5)
- **nitpick**: skill flagged a style/preference issue not in ground truth (-3)
- **bonus_find**: skill found a REAL issue not in ground truth (reviewers missed it) (+15)

**Detection rate** = hits / total_ground_truth × 100%
