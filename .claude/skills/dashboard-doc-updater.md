# Skill: dashboard-doc-updater

Keep `DASHBOARD.md` in sync with the current state of `index.html` whenever the dashboard logic changes.

## When to use

Run `/dashboard-doc-updater` after:
- Adding or changing a dashboard tab or view
- Modifying KPI targets, thresholds, or colour logic
- Changing the data model (new fields, renamed columns, removed fields)
- Updating the CSV ingestion or Supabase integration
- Adding, removing, or renaming processing functions
- Any other change that makes the current `DASHBOARD.md` inaccurate

## What this skill does

1. Reads `index.html` end-to-end to understand the current implementation.
2. Reads the existing `DASHBOARD.md`.
3. Identifies every section of `DASHBOARD.md` that is outdated, incomplete, or missing relative to the current code.
4. Rewrites only the affected sections — never removes accurate content.
5. Updates the `Last updated` date at the top of `DASHBOARD.md`.
6. Reports a brief summary of what changed.

## Execution instructions

Read `index.html` in full. Focus on:

- **Constants** (`DAY_ORDER`, `EXCLUDE_CLASSES`, KPI targets, `TREND_CLASSES`, colour tokens)
- **Data model** (the shape built by `processRawRows()` and the Supabase table columns)
- **Tab renderers** (`renderFlags`, `renderMonthly`, `renderQuarterlyChart`, `renderMonthlyCards`, `renderWeekly`, `renderMatrix`, `renderTrends`) — note any changed logic, new thresholds, or renamed functions
- **Processing functions** (date helpers, metric helpers, storage helpers, CSV helpers)
- **Global state** variables
- **UI/design tokens** (CSS variables or hardcoded hex values)

Then diff what you found against `DASHBOARD.md`. For each discrepancy:

- If a section is stale, rewrite it to match the code exactly.
- If a new feature exists that has no section, add one in the appropriate place.
- If a feature documented in `DASHBOARD.md` no longer exists in `index.html`, remove or clearly mark it as removed.

Do **not** change prose style, headings structure, or accurate content. Only update what the code contradicts.

After editing `DASHBOARD.md`, print a concise bullet list of every section that was modified and why.

## Notes

- The authoritative source of truth is always `index.html`. If `DASHBOARD.md` disagrees with `index.html`, `index.html` wins.
- Update the `Last updated` date at the top of `DASHBOARD.md` to today's date in YYYY-MM-DD format.
- Do not commit the changes — leave that to the user.
