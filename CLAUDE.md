# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A class/coach performance dashboard for a fitness studio. It ingests booking data exported from **WodBoard** (CSV), persists it in **Supabase** (PostgreSQL), and renders analytical views with **Chart.js**.

The **entire application is one file: `index.html`** — embedded CSS + vanilla JavaScript, no framework, no build step, no npm, no bundler. It runs as a static file and is meant to stay accessible to non-engineers.

## Working in this repo

- **Run it:** open `index.html` in a browser. With an empty Supabase it falls back to the embedded `EMBEDDED_DATA` sample dataset, so the dashboard is fully explorable offline.
- **There is no build, lint, or test suite.** To sanity-check JS changes, extract the main inline script and parse it, e.g.:
  ```bash
  awk '/<script>/{f=1; next} /<\/script>/{f=0} f' index.html > /tmp/dash.js && node --check /tmp/dash.js
  ```
  (The earlier `<script src=...>` CDN tags don't match `/<script>/`, so this grabs the main block.)
- **`index.html` is ~1MB** — most of the size is the embedded sample data on one line near the top (`EMBEDDED_DATA`). Use `Grep`/`Read` with offsets rather than reading the whole file; edit by targeting the function/constant you need.

## Architecture (the parts that span files/concepts)

The app is **data-driven from the CSV** — class names, coaches, days, and times are read straight from the WodBoard export, normalized in `processRawRows()`, upserted to Supabase, then re-fetched as the source of truth. **A timetable change (new classes/coaches/times) generally needs no code change**; it flows in when new CSVs are uploaded. Only *hardcoded assumptions* break — see below.

Pipeline: drop CSV → `handleFiles()` → PapaParse → `processRawRows()` (normalize to the session-record shape) → dedup in memory → `saveDataToStorage()` (UPSERT) → `loadDataFromStorage()` (re-fetch) → re-render the active tab via `renderCurrentTab()`.

Key non-obvious invariants:
- **UTC-first storage.** Timestamps are written to Supabase with a `Z` suffix and stripped on read, to avoid UK BST/DST drift. Conversion only happens at the I/O boundary.
- **Idempotent uploads.** Supabase unique constraint `(class_name, start_time, week_start)` makes re-uploading the same CSV safe (update-in-place, no duplicates).
- **Paginated fetch.** `loadDataFromStorage()` loops with `.range()` in 1000-row batches because Supabase caps responses at 1000 rows; a single fetch would silently truncate large datasets.
- **Post-upload re-fetch** (rather than merging the local upload) keeps state correct when multiple people upload.

State is in-memory and ephemeral (`data[]`, `currentTab`, plus per-tab `quarterlyState`/`weeklyState`/`matrixState`); Supabase holds the persistent record.

### Hardcoded assumptions to watch
These are the things that *don't* auto-adapt and are common sources of "the dashboard ignores X":
- `EXCLUDE_CLASSES` — class names dropped during ingestion (never stored/shown).
- `csvCheckIsPeak()` — peak-time windows are hardcoded minute ranges (morning + evening). New off-window class times won't count as "peak" until this is updated.
- Tab wiring lives in three places that must stay in sync: the nav `<button data-tab=...>` list, the matching `<div class="section" data-tab=...>` panes, the `switch` in `renderCurrentTab()`, and the `currentTab` default. The pane with class `active` is the one shown on load.
- KPI targets (booking rate, peak, cancellation) are hardcoded inside render functions, not configurable.

## Documentation & the doc-updater skill

`DASHBOARD.md` is a detailed system document (data model, every view's logic, function reference, design decisions). **`index.html` is always the source of truth — if `DASHBOARD.md` disagrees, the code wins**, and the doc can lag behind recent changes.

After any change to dashboard logic (tabs, KPI targets/thresholds, data model, ingestion, processing functions), run the **`/dashboard-doc-updater`** skill (`.claude/skills/dashboard-doc-updater.md`) to re-sync `DASHBOARD.md`. It rewrites only the affected sections and bumps the `Last updated` date; it intentionally does **not** commit.
