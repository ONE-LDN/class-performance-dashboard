# Class Performance Dashboard — System Document

> Last updated: 2026-07-29

---

## 1. Purpose

A single-page web application that gives fitness studio managers real-time insight into how their classes and coaches are performing. It ingests booking data exported from WodBoard (a gym management platform), persists it in a Supabase database, and renders it across four analytical views.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla JavaScript, HTML, CSS (no framework, no build step) |
| Charting | Chart.js |
| CSV Parsing | PapaParse |
| Database | Supabase (PostgreSQL) |
| Hosting | Static file (single `index.html`) |

The entire application is contained in **`index.html`** — one file with embedded CSS and JavaScript.

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                       Browser                           │
│                                                         │
│  ┌───────────┐    ┌────────────┐    ┌───────────────┐   │
│  │ File Drop │───▶│  PapaParse │───▶│ processRaw    │   │
│  │  Upload   │    │ CSV Parser │    │ Rows()        │   │
│  └───────────┘    └────────────┘    └──────┬────────┘   │
│                                            │             │
│  ┌─────────────────────────────────────────▼──────────┐  │
│  │               In-Memory Cache  (data[])            │  │
│  └──────────┬──────────────────────────────┬──────────┘  │
│             │                              │             │
│  ┌──────────▼──────────┐       ┌───────────▼──────────┐  │
│  │  saveDataToStorage  │       │  4 Tab Renderers      │  │
│  │  (UPSERT → Supabase)│       │  renderFlags()        │  │
│  └──────────┬──────────┘       │  renderMonthly()      │  │
│             │                  │  renderWeekly()        │  │
│  ┌──────────▼──────────┐       │  renderMatrix()       │  │
│  │ loadDataFromStorage │       └───────────────────────┘  │
│  │ (paginated fetch)   │                                  │
│  └─────────────────────┘                                  │
└─────────────────────────────────────────────────────────┘
                      │ Supabase JS Client
                      ▼
              ┌──────────────────┐
              │  Supabase DB     │
              │  class_performance│
              │  csv_uploads     │
              └──────────────────┘
```

### Data Flow Summary

1. User drops one or more WodBoard CSV files onto the upload zone.
2. PapaParse converts each CSV into raw row objects.
3. `processRawRows()` normalises the raw data into the canonical session record shape.
4. Duplicate sessions (same class + start time) are removed in memory.
5. All rows are upserted into the `class_performance` Supabase table.
6. The app re-fetches the full authoritative dataset from Supabase (paginated).
7. The in-memory `data[]` array is replaced and all active tab views re-render.

---

## 4. Data Model

### Session Record (in-memory shape)

```javascript
{
  Class:          string,   // "WOD" | "HYROX - PEAK" | "Mat Pilates (Dynamic)" | ...
  'Start time':   string,   // "06/01/2025 10:30" — UTC, exactly as WodBoard exported it
  Capacity:       number,   // Maximum bookable places (rows with 0 are dropped)
  Bookings:       number,   // Confirmed bookings
  Cancelled:      number,   // Cancellations
  Week_Start:     string,   // "06/01/2025"  (Monday of the London week)
  Attendance_Pct: number,   // Attended / Capacity × 100, taken from the export
  Booking_Rate:   number,   // Bookings / Capacity × 100
  Is_Peak:        boolean,  // Convenience flag: Time_Block !== 'off'
  Time_Block:     string,   // 'am' | 'lunch' | 'pm' | 'off'
  Coach_Label:    string,   // "Jane Smith" or "Jane Smith + Bob Jones"
  Day_Name:       string,   // "Monday" … "Sunday"   (London)
  Time_Str:       string,   // "10:30"               (London)
  Day_Time:       string,   // "Monday 10:30"        (London)
  Area:           string,   // 'The Playground' | 'The Studio' | 'Off-site' | '' if unmapped
  Is_Routine:     boolean   // false for events and short courses
}
```

**`Start time` is UTC; every other time field is London.** WodBoard exports `Start time` in
UTC, so through British Summer Time it reads an hour early. `deriveTimeFields()` converts to
Europe/London and is the sole producer of `Week_Start`, `Day_Name`, `Time_Str`, `Day_Time` and
`Time_Block`. `Start time` is deliberately left as exported so the Supabase unique constraint
stays stable and re-uploads cannot duplicate rows.

### Supabase Table: `class_performance`

| Column | Type | Notes |
|---|---|---|
| `class_name` | text | Maps from `Class` |
| `start_time` | timestamptz | ISO UTC stored with Z suffix |
| `capacity` | integer | |
| `bookings` | integer | |
| `cancelled` | integer | |
| `week_start` | date | YYYY-MM-DD |
| `attendance_pct` | float | |
| `booking_rate` | float | |
| `is_peak` | boolean | Written for continuity; **recomputed on read, not trusted** |
| `coach_label` | text | |
| `day_name` | text | Written for continuity; **recomputed on read, not trusted** |
| `time_str` | text | Written for continuity; **recomputed on read, not trusted** |
| `day_time` | text | Written for continuity; **recomputed on read, not trusted** |

**Unique constraint:** `(class_name, start_time, week_start)` — enables idempotent upserts.

Rows written before the UTC correction carry the wrong `day_name`, `time_str`, `day_time` and
`is_peak`. Rather than migrate them, `loadDataFromStorage()` re-derives all four from
`start_time`, so the entire history reads correctly with no backfill and no risk of duplicating
rows against the unique constraint. Newly saved rows carry corrected values, so the table
self-heals as uploads happen. `Area` and `Is_Routine` are not stored at all — they come from the
in-code class map on every load.

### Supabase Table: `csv_uploads`

Stores a timestamp row on each upload so the UI can display "last updated" info.

---

## 5. Excluded Classes

These class types are filtered out on ingestion **and on read**, so they never appear even if
older rows exist in Supabase:

```
'Gym Time'
'Private Event'
'404'
'FSC Thursday Throwdown'    // how the export actually spells it
'FCS Thursday Throwdown'    // kept: the original entry, which never matched
'Unbeatable Hybrid Race'    // an event, not a class
'C4 Energy Run Club'        // one-off
'Hyrox Recovery Pass'       // a pass, not a class
```

The list previously contained only the `FCS` spelling, which matches nothing — the export says
`FSC`. Those rows had therefore been ingesting all along, roughly 18 a year at capacity 1 with
zero bookings, pulling averages down. Both spellings are kept deliberately.

Two further filters apply alongside the list:

- **Capacity 0** rows are dropped. They yield no computable booking rate and previously produced
  `NaN` averages.
- **`CLASS_ALIASES`** merges names that differ only by casing, so `COMP CLASS` and `Comp class`
  are one class rather than two.

---

## 6. Key Configuration Constants

```javascript
DAY_ORDER    = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
WEEKEND_DAYS = ['Saturday', 'Sunday']

// KPI targets — hardcoded, not configurable through the UI
BOOKING_RATE_TARGET = 70   // %
ATTENDANCE_TARGET   = 70   // %
CANCELLATION_TARGET = 15   // % (below is good)

// Absolute headcount targets, set independently per area.
// 13 is 72% of an 18-cap Playground class; 10 is 71% of a 14-cap Studio class.
AREA_HEAD_TARGET = { 'The Playground': 13, 'The Studio': 10 }

MIN_SAMPLE = 5   // below this a KPI card is greyed and labelled rather than read as reliable
```

### Areas

The WodBoard export carries no room or area field, so the mapping is hardcoded across four
lists: `PLAYGROUND_ROUTINE`, `PLAYGROUND_EVENT`, `STUDIO_ROUTINE`, `STUDIO_EVENT`, plus
`OFFSITE_CLASSES` for classes that happen away from the building.

- `classArea(name)` → `'The Playground'` | `'The Studio'` | `'Off-site'` | `''`
- `isRoutineClass(name)` → `false` for events and short courses

**Events are held apart from the routine timetable** because they fill far better than weekly
classes — folding them in flatters the timetable's own numbers.

A class in none of the lists still appears in the integrated totals but in neither area row, and
is counted in an "unmapped" notice under the KPI rows. This is the one hardcoded assumption most
likely to go stale: **a new class name needs adding here or it will show in that notice.**

### Time blocks

Blocks follow the day, because the weekday and weekend timetables barely overlap.

| Block | Weekdays (Mon–Fri) | Weekends |
|---|---|---|
| `am` — AM Peak | 06:00–08:59 | 09:00–11:00 |
| `lunch` — Lunch | 12:00–13:00 | *none* |
| `pm` — PM Peak | 17:00–19:30 | *none* |
| `off` — Off-peak | everything else | everything else |

`timeBlockFor(dayName, minutes)` is **computed on read, never stored**, so the windows can be
revised in one place without a database migration. The midday block is labelled simply "Lunch"
rather than a peak block: it fills at roughly 41% against 47% for off-peak, so calling it peak
would overstate it.

---

## 7. Dashboard Views

### Day and time-block filters (all four tabs)

Each tab carries a `.filter-slot` inline with its own period control — below the tabs, above the
KPI cards. Both filters apply to **everything on the tab**, including the quarterly trend chart,
so no figure on screen can disagree with another.

- **Day** — a day of the week. Only days that ran in the selected period are offered.
- **Time block** — All / AM Peak / Lunch / PM Peak / Off-peak. Options follow the selected day, so
  a block with no classes that day is **disabled** rather than offered and returning nothing.
  Selecting Saturday disables Lunch and PM, since no weekend classes run then.

`filterState = { day, block }` is shared by all tabs but **reset on every tab switch** — carrying
a narrow filter into another view is how an empty tab gets misread as missing data.

Key functions: `renderFilterControls(slotId, periodRows)`, `applySessionFilters(rows)`,
`resetFilters()`.

### KPI cards

All KPI rows are built by the same two functions, so the integrated and area rows can never drift
apart: `kpiCardsHtml(rows, headTarget)` and `renderAreaKpiRows(containerId, unassignedId, rows)`.

Every card states its verdict **in words** ("On target" / "Near" / "Below" / "Low n") as well as
colour. The existing green and amber fail colourblind separation — ΔE 5.7 under protanopia against
a target of 8 — so colour alone must never carry the verdict.

Cards computed from fewer than `MIN_SAMPLE` (5) sessions are greyed and labelled "Low n". This
applies to cards only; breakdown bars keep their colour and carry a marker, because at weekly
granularity almost every class runs fewer than five times and greying would wash out the panel.

### 7.1 Flags Tab

**Goal:** Surface underperforming classes that need immediate attention.

**Logic:**
- Scans the **3 most recent weeks** of data.
- Groups sessions by `Class + Coach_Label + Day_Time` (e.g. "Pilates · Jane Smith · Monday 10:30").
- A group is flagged when `Booking_Rate < 50%`.

**Severity:**
| Colour | Condition |
|---|---|
| Red | Below 50% in the latest week only |
| Purple | Below 50% for 3+ consecutive weeks |

Each flag card shows:
- Class name, coach, day/time slot
- Fill percentage for each week
- Consecutive weeks below threshold
- Cancellation rate
- Recommended action (Class design / Instructor coaching / Marketing)

**Key function:** `renderFlags()`

---

### 7.2 Quarterly Tab

**Goal:** Aggregate monthly performance across an entire quarter.

**Period selection:** Drop-down populated from available data (e.g. "2025 Q1").

**KPI cards — three rows.** The integrated row first, then a denser row per area:

| KPI | Target | Colour logic |
|---|---|---|
| Avg Booking Rate | ≥ 70% | Green / Amber / Red |
| Avg Attendance % | ≥ 70% | Green / Amber / Red |
| Avg Bookings | per area (13 / 10) | `kpiColorCount` — proportional band |
| Sessions | — | Neutral |
| Canx Rate | < 15% | Green / Amber / Red (inverted) |

The **Avg Peak Booking Rate** card was removed. Peak figures are now reachable directly through
the time-block filter, which also avoids the card simply duplicating the filtered rate.

On the integrated row, **Avg Bookings shows no verdict** — two areas with different targets have
no meaningful joint one, so it reads "Per-area targets below".

**Area rows** (`#quarterlyAreaKPI`): The Playground, then The Studio, each with a coloured rail,
session and event count, and its headcount target. **They need not sum to the integrated row** —
off-site classes belong to no area, and unmapped names are reported in `#quarterlyUnassigned`
rather than silently dropped.

**Quarterly line chart:** One line per month plotting booking rate over the months in the quarter.

**Monthly breakdown cards:** For each month:
- Summary KPI row
- Class breakdown: each class → avg booking %, session count, cancellation rate (horizontal bar)
- Coach breakdown: each coach → avg booking %, session count, cancellation rate (horizontal bar)

**Key functions:** `renderMonthly()`, `renderQuarterlyChart()`, `renderMonthlyCards()`

---

### 7.3 Weekly Tab

**Goal:** Deep dive into a single week.

**Period selection:** Drop-down populated from available weeks.

**KPI cards:** Same three rows as Quarterly (integrated, then The Playground and The Studio),
for the selected week. Containers: `#weeklyKPI`, `#weeklyAreaKPI`, `#weeklyUnassigned`.

At weekly granularity, narrowing by day and block can drop an area below five sessions, at which
point those cards grey out and read "Low n".

**Class breakdown:** Horizontal bar chart — one bar per class, showing booking %, session count, cancellation rate.

**Coach breakdown:** Horizontal bar chart — one bar per coach, same metrics.

**Key function:** `renderWeekly()`

---

### 7.4 Performance Matrix

**Goal:** Multi-dimensional heat map showing booking rates by Class × Coach × Day/Time.

**Period toggle:** Month or Quarter (button group, not a drop-down).

**Layout:** One table per class (sorted by avg booking rate, best first).

- **Rows:** Day/Time slots (sorted `DAY_ORDER` then chronologically).
- **Columns:** Coach names.
- **Cells:** Colour-coded booking % with sample size badge.

**Cell colour scale:**

| Booking % | Colour |
|---|---|
| < 30% | Dark red |
| 30–70% | Amber gradient |
| ≥ 70% | Green |

Peak-time cells carry a small indicator badge.

**Key function:** `renderMatrix()`

---

### 7.5 Trends Tab — **REMOVED**

This tab no longer exists. There is no `renderTrends()` function, no `TREND_CLASSES` constant and
no nav button for it; `renderCurrentTab()` switches over `flags`, `monthly`, `weekly` and `matrix`
only. The section is retained as a note so the absence reads as deliberate rather than an
oversight.

---

## 8. Critical Processing Functions

### Date / Time

| Function | Input | Output |
|---|---|---|
| `ddmmyyyyToISO(str)` | `"06/01/2025 10:30"` | `"2025-01-06T10:30:00Z"` |
| `isoToStartTime(isoStr)` | ISO string | `"06/01/2025 10:30"` |
| `isoToWeekStart(isoStr)` | ISO string | `"06/01/2025"` |
| `parseDate(dateStr)` | date string | `Date` object |
| `lastSundayUTC(year, monthIdx)` | year + month | last Sunday of that month, UTC |
| `isBritishSummerTime(utcDate)` | `Date` | `true` between the last Sunday in March 01:00 UTC and the last Sunday in October 02:00 UTC |
| `londonFromUtcString(startTime)` | `"06/01/2025 10:30"` UTC | `Date` whose `getUTC*` accessors read **London wall-clock** |
| **`deriveTimeFields(startTimeUtc)`** | UTC start-time string | `{ weekStart, dayName, timeStr, dayTime, block, isPeak }` |
| `normaliseSessionRows(rows)` | shaped session records | same records with every derived field recomputed |
| `csvGetWeekStart` / `csvGetDayName` | UTC start-time string | thin wrappers over `deriveTimeFields` |

`ddmmyyyyToISO` appends `Z` as a **label**, not a conversion, and `isoToStartTime` reads back with
`getUTC*` accessors, so the Supabase round-trip is lossless. It was never the source of the
BST drift — the drift is in WodBoard's export, which is UTC.

**`deriveTimeFields()` is the single source of truth** for every time-derived field, called by the
CSV path, the Supabase read path and `normaliseSessionRows()` alike, so the three can never
disagree. `normaliseSessionRows()` exists for inputs that arrive pre-shaped and carry values from
before the fix: the pre-processed master-sheet upload format and `EMBEDDED_DATA`.

### Metrics

| Function | Description |
|---|---|
| `wbr(rows)` | **Capacity-weighted** booking rate = `Σbookings ÷ Σcapacity × 100`. Not a mean of `Booking_Rate` — that would weight a 10-seat class the same as a 50-seat one |
| `cancellationRate(rows)` | `Σcancelled ÷ (Σbookings + Σcancelled) × 100` — a share of everything ever booked, not a mean of per-row percentages |
| `kpiColor(val, target)` | Green ≥ target, amber within **10 points**, else red. For percentages |
| `kpiColorCount(val, target)` | Green ≥ target, amber within **15%** of target, else red. For absolute counts, where a fixed 10-point band against a target of 13 could never read as red |
| `heatColor(pct)` | Returns RGB string for heat map gradient |
| `formatPct(val)` | Number → formatted percentage string |
| `kpiCard(label, value, target, state, thin)` | One card, including its state word |
| `kpiCardsHtml(rows, headTarget)` | The five cards for one row; `headTarget` null suppresses the headcount verdict |
| `renderAreaKpiRows(containerId, unassignedId, rows)` | Both area rows plus the unmapped-class notice |

### Storage

| Function | Description |
|---|---|
| `saveDataToStorage(dataArray)` | UPSERT all rows to Supabase |
| `loadDataFromStorage()` | Paginated fetch (1 000 rows/batch) to bypass Supabase row limit |
| `storageSavedDate()` | Fetch latest upload timestamp from `csv_uploads` |

### CSV Ingestion

| Function | Description |
|---|---|
| `parseCSVFile(file)` | PapaParse wrapper → Promise |
| `isRawFormat(headers)` | Detect WodBoard raw format vs pre-processed |
| `processRawRows(rows)` | Normalise raw CSV rows to session records — applies exclusions, `normaliseClassName`, the capacity-0 drop, `deriveTimeFields`, `classArea` and `isRoutineClass` |
| `normaliseClassName(name)` | Collapse casing variants via `CLASS_ALIASES` |
| `handleFiles(files)` | Full upload pipeline: parse → dedup → save → reload → render |

---

## 9. Global State

```javascript
let data = []                 // Source of truth; replaced on each load
let currentTab = 'weekly'     // Active tab name — matches the pane marked .active in the markup
let usingSampleData = false   // True when falling back to EMBEDDED_DATA

// Tab-specific state
let quarterlyState = { quarter: '', months: new Set() }
let weeklyState    = { week: '' }
let matrixState    = { periodType: 'monthly', period: '' }

// Shared by all tabs, reset on every tab switch
let filterState = { day: 'all', block: 'all' }
```

State is ephemeral (in-memory); the persistent record lives in Supabase.

`usingSampleData` exists because the embedded sample is now passed through
`normaliseSessionRows()` and so can no longer be recognised by identity (`data !== EMBEDDED_DATA`).
Without the flag, sample rows would be merged into a genuine upload.

---

## 10. Design Decisions & Patterns

### Single-file SPA
No build toolchain. The entire app ships as one HTML file — no npm, no bundler, no deployment pipeline beyond static hosting. This keeps the project accessible to non-engineers who may manage it.

### Vanilla JavaScript
No React, Vue, or similar. Direct DOM manipulation keeps the bundle at zero and makes debugging straightforward in any browser DevTools.

### UTC storage, London display
WodBoard exports `Start time` in **UTC**, so through British Summer Time every class reads an hour
early — 46–53% of any given dataset. The `Z` suffix on write and the `getUTC*` accessors on read
make that round-trip lossless, so storage was never the problem; the drift arrives in the export.

The correction is applied where the data is *interpreted*, not where it is stored. `start_time`
stays exactly as exported, which keeps `(class_name, start_time, week_start)` stable — change it
and re-uploading an old CSV would insert duplicates instead of updating in place. Day, time and
block are derived in London on both ingest and read.

Confirmed three independent ways against the May and June 2026 exports: the 06:15 HYROX reading
05:15, Built reading 08:30 against a known 09:30 start, and Gym Time spanning 05:00–19:00 where
opening hours are 06:00–20:00.

### Derived fields recomputed on read
`day_name`, `time_str`, `day_time` and `is_peak` are recomputed from `start_time` on every load
rather than trusted from the database. That corrected the whole history the moment the fix shipped
— no migration, no downtime, no chance of duplicating rows — and it means the time-block windows
can be revised by editing one function.

### Odd times left as recorded
After the UTC correction, a small number of sessions still sit off their usual slot. These are
**bank holiday timetables, not errors**: on the May 2026 bank holiday Mondays the whole timetable
slides later and runs fewer classes (Built at 10:00 and 11:00 against its usual 09:30). They are
left exactly as recorded rather than snapped to the nearest slot, because they are real scheduling
decisions worth being able to see.

### Idempotent uploads
The Supabase unique constraint `(class_name, start_time, week_start)` means re-uploading the same CSV is safe — rows are updated in-place, not duplicated.

### Paginated fetch
Supabase enforces a 1 000-row response cap by default. `loadDataFromStorage()` loops with `.range()` until a partial page is returned, ensuring large datasets load completely.

### Post-upload re-fetch
After saving, the app re-fetches from Supabase rather than merging the local upload into `data[]`. This prevents stale state if two users upload simultaneously.

---

## 11. Colour & UI System

| Token | Hex | Use |
|---|---|---|
| Background | `#0a0a0a` | Page background |
| Card | `#111` | Card surfaces |
| Border | `#1e1e1e` | Card borders |
| Text primary | `#f0f0f0` | Headings, values |
| Text muted | `#999` / `#666` | Labels, secondary info |
| Accent | `#e8c547` | Golden yellow — brand accent, tabs |
| Green | `#4ade80` | Good performance |
| Amber | `#e8c547` | Warning / moderate performance — **the same hex as the accent** |
| Red | `#f87171` | Poor performance / alert |
| Neutral rail | `#2a2825` | Top border of cards carrying no verdict |
| Playground rail | `#e8c547` | Area row marker |
| Studio rail | `#74b9ff` | Area row marker |

Two known weaknesses in this palette, both mitigated rather than fixed:

- **Green and amber fail colourblind separation** — ΔE 5.7 under protanopia against a target of 8.
  Every KPI card therefore states its verdict in words as well as colour (`.kpi-state`).
- **Amber and the brand accent are the same hex** (`#e8c547`), so "warning" and "brand" are
  indistinguishable. Changing it would mean revisiting the dashboard's visual identity, so it
  stands for now.

---

## 12. Known Limitations

- **No authentication:** The Supabase anon key is embedded in the HTML. The anon key grants read/write via the row-level-security policy; it should not be considered secret, but schema-level admin operations are still protected.
- **Hard-coded KPI targets:** 70% booking rate, 70% attendance, 15% cancellation, and headcount
  targets of 13 (Playground) and 10 (Studio) are not configurable via the UI.
- **Area map is hardcoded:** the export has no room field, so `classArea()` depends on maintained
  class-name lists. A new or renamed class shows in the "unmapped" notice until it is added. This
  is the assumption most likely to go stale.
- **Two classes can never hit their headcount target:** Weightlifting (capacity 10) and Built
  (capacity 12) cannot reach the Playground target of 13. Accepted deliberately — a single
  headcount target is a different percentage in every capacity tier, and the Playground spans
  10 to 51.
- **`Waitlists` is not captured.** The export carries a waitlist column that ingestion ignores, so
  a class that is full with people queuing looks identical to one that is merely full.
- **No data validation:** The app assumes well-formed WodBoard CSV input.
- **Flags limited to 3 weeks:** Older persistent issues are not surfaced on the Flags tab.
- **No export:** Processed data and charts cannot be downloaded.
