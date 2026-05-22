# Class Performance Dashboard — System Document

> Last updated: 2026-05-22

---

## 1. Purpose

A single-page web application that gives fitness studio managers real-time insight into how their classes and coaches are performing. It ingests booking data exported from WodBoard (a gym management platform), persists it in a Supabase database, and renders it across five analytical views.

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
│  │  saveDataToStorage  │       │  5 Tab Renderers      │  │
│  │  (UPSERT → Supabase)│       │  renderFlags()        │  │
│  └──────────┬──────────┘       │  renderMonthly()      │  │
│             │                  │  renderWeekly()        │  │
│  ┌──────────▼──────────┐       │  renderMatrix()       │  │
│  │ loadDataFromStorage │       │  renderTrends()       │  │
│  │ (paginated fetch)   │       └───────────────────────┘  │
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
  Class:          string,   // "Pilates" | "Yoga" | "HIIT" | "Weights" | ...
  'Start time':   string,   // ISO 8601 UTC  "2025-01-06T10:30:00Z"
  Capacity:       number,   // Maximum bookable places
  Bookings:       number,   // Confirmed bookings
  Cancelled:      number,   // Cancellations
  Week_Start:     string,   // "06/01/2025"  (Monday of that week)
  Attendance_Pct: number,   // Bookings / Capacity × 100
  Booking_Rate:   number,   // Same as Attendance_Pct (used interchangeably)
  Is_Peak:        boolean,  // True if class falls in a peak time window
  Coach_Label:    string,   // "Jane Smith" or "Jane Smith + Bob Jones"
  Day_Name:       string,   // "Monday" … "Sunday"
  Time_Str:       string,   // "10:30"
  Day_Time:       string    // "Monday 10:30"
}
```

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
| `is_peak` | boolean | |
| `coach_label` | text | |
| `day_name` | text | |
| `time_str` | text | |
| `day_time` | text | |

**Unique constraint:** `(class_name, start_time, week_start)` — enables idempotent upserts.

### Supabase Table: `csv_uploads`

Stores a timestamp row on each upload so the UI can display "last updated" info.

---

## 5. Excluded Classes

These class types are filtered out during CSV processing and never stored or shown:

```
'Gym Time'
'Private Event'
'FCS Thursday Throwdown'
```

---

## 6. Key Configuration Constants

```javascript
DAY_ORDER = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']

// KPI targets
BOOKING_RATE_TARGET  = 70   // %
PEAK_BOOKING_TARGET  = 80   // %
CANCELLATION_TARGET  = 15   // % (threshold below is good)

// Trend / chart classes (restricted set)
TREND_CLASSES = ['Pilates', 'Yoga', 'HIIT', 'Weights']
```

---

## 7. Dashboard Views

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

**KPI cards (per month):**

| KPI | Target | Colour logic |
|---|---|---|
| Avg Booking Rate | ≥ 70% | Green / Amber / Red |
| Avg Attendance % | ≥ 70% | Green / Amber / Red |
| Avg Peak Booking Rate | ≥ 80% | Green / Amber / Red |
| Total Sessions | — | Neutral |
| Cancellation Rate | < 15% | Green / Amber / Red (inverted) |

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

**KPI cards:** Same five as Quarterly, but for the selected week.

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

### 7.5 Trends Tab

**Goal:** Track class popularity over time as weekly line chart.

**Classes shown:** Pilates, Yoga, HIIT, Weights (fixed set).

**Interactions:** Click a class name to toggle its line on/off.

**Chart:**
- X-axis: Week start dates ("W/C 6 Jan", "W/C 13 Jan", …)
- Y-axis: Booking % (0–100%)
- One coloured line per active class; gaps in data are bridged (`spanGaps: true`).

**Key function:** `renderTrends()`

---

## 8. Critical Processing Functions

### Date / Time

| Function | Input | Output |
|---|---|---|
| `ddmmyyyyToISO(str)` | `"06/01/2025 10:30"` | `"2025-01-06T10:30:00Z"` |
| `isoToStartTime(isoStr)` | ISO string | `"06/01/2025 10:30"` |
| `isoToWeekStart(isoStr)` | ISO string | `"06/01/2025"` |
| `parseDate(dateStr)` | date string | `Date` object |

All dates are stored as **UTC with Z suffix** to prevent BST/DST drift. Conversion only happens at the I/O boundary.

### Metrics

| Function | Description |
|---|---|
| `wbr(rows)` | Weighted booking rate = `mean(Booking_Rate)` across rows |
| `cancellationRate(rows)` | Mean cancellation percentage |
| `kpiColor(val, target)` | Returns `'green'` / `'amber'` / `'red'` CSS class |
| `heatColor(pct)` | Returns RGB string for heat map gradient |
| `formatPct(val)` | Number → formatted percentage string |

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
| `processRawRows(rows)` | Normalise raw CSV rows to session records |
| `handleFiles(files)` | Full upload pipeline: parse → dedup → save → reload → render |

---

## 9. Global State

```javascript
let data = []               // Source of truth; replaced on each load
let currentTab = 'flags'    // Active tab name

// Tab-specific state
let quarterlyState = { quarter: '', months: new Set() }
let weeklyState    = { week: '' }
let matrixState    = { periodType: 'monthly', period: '' }
```

State is ephemeral (in-memory); the persistent record lives in Supabase.

---

## 10. Design Decisions & Patterns

### Single-file SPA
No build toolchain. The entire app ships as one HTML file — no npm, no bundler, no deployment pipeline beyond static hosting. This keeps the project accessible to non-engineers who may manage it.

### Vanilla JavaScript
No React, Vue, or similar. Direct DOM manipulation keeps the bundle at zero and makes debugging straightforward in any browser DevTools.

### UTC-first storage
WodBoard timestamps can be ambiguous around UK DST transitions. Appending `Z` when writing to Supabase, and stripping it on read, ensures stored values are timezone-stable.

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
| Amber | `#fbbf24` | Warning / moderate performance |
| Red | `#f87171` | Poor performance / alert |

---

## 12. Known Limitations

- **No authentication:** The Supabase anon key is embedded in the HTML. The anon key grants read/write via the row-level-security policy; it should not be considered secret, but schema-level admin operations are still protected.
- **Hard-coded KPI targets:** 70% booking rate, 80% peak, 15% cancellation are not configurable via UI.
- **No data validation:** The app assumes well-formed WodBoard CSV input.
- **Flags limited to 3 weeks:** Older persistent issues are not surfaced on the Flags tab.
- **No export:** Processed data and charts cannot be downloaded.
- **Trend classes fixed:** Only Pilates, Yoga, HIIT, Weights appear on the Trends tab.
