---
name: progress-update-writer
description: >
  Drafts the weekly progress entry for the ONE LDN Progress Document in Saffron's working style,
  then appends it directly to the Notion Progress Document page. Session-scoped only - covers what
  was built or changed in this session. Run alongside docs-manager at session end. Trigger when
  Saffron says "update progress", "log this session", "write up the weekly entry", or at the end
  of any session where something was built, changed, or decided.
---

# progress-update-writer

Drafts and posts the weekly progress entry for the ONE LDN Progress Document. Session-scoped — covers this session only. Run alongside `docs-manager` at session end.

---

## Notion targets

**Progress Document page ID:** `3660164f-7fc6-81d7-9e14-e2085e8c6ed6`

All weekly and daily entries are created as **sub-pages** of the Progress Document — not appended as inline content.

---

## Page structure

```
Progress Document
└── Week of [DD Mon YYYY]            ← weekly sub-page (created if it doesn't exist)
    └── [Weekday DD Mon YYYY]        ← daily sub-sub-page (created each session)
        ├── Plain English summary    ← top of page, 2-3 sentences, stakeholder level
        ├── ---
        └── Full technical detail    ← Saffron's working style
```

### Weekly sub-page

- Title: `Week of [Monday DD Mon YYYY]`
- Parent: Progress Document (`3660164f-7fc6-81d7-9e14-e2085e8c6ed6`)
- Contains: Completed / In Progress / Blockers / Upcoming tables (see format below)
- Acts as a container for the day pages — tables are updated each time a day page is added

**Check before creating:** Fetch the Progress Document and look for an existing child page with the current week's title. If it exists, use it. If not, create it.

### Daily sub-sub-page

- Title: `[Weekday DD Mon YYYY]` — e.g. `Wednesday 20 May 2026`
- Parent: the weekly sub-page
- Created fresh each session — one page per working day
- If a page for today already exists (multiple sessions in one day), append to it rather than creating a duplicate

---

## Daily page format

```
[Plain English summary — 2-3 sentences, stakeholder level]
What was worked on today in plain terms. No jargon, no acronyms. 
What it means for the business, not how it was built.

---

__SYSTEM NAME__

* Specific change — include tab names, field names, commit hashes, URLs where relevant
* Another change to the same system

__ANOTHER SYSTEM__

* What changed

**In progress:**
- [workstream] — [brief status]

**Blockers:**
- [what] — waiting on: [person]

**Upcoming:**
- [next concrete step]
```

### Plain English summary rules (top section)

- 2–3 sentences maximum
- Written for Evgenia / non-technical reader — same register as `stakeholder-update-writer`
- No acronyms, no system names in jargon form — "shop dashboard" not "shop-sales repo"
- Covers what was done and why it matters, not how
- Example: "Today I added new stock alert features to the shop dashboard — it will now flag when products need reordering urgently or soon, without needing to click through to find them. I also built a new consumables tracker dashboard for cafe stock like cups and packaging."

### Technical detail rules (below the divider)

- System headers in `__CAPS__` for dashboards/major systems
- Bullets are factual and specific — tab names, row counts, field names, commit hashes, URLs
- Past tense for completed, present continuous for in-progress
- No prose paragraphs, no passive voice, no filler
- If something is blocked or waiting, name who specifically

---

## Weekly sub-page format

```markdown
## Completed
[system headers + bullets — aggregated from all day pages this week]

## In Progress
[workstream — brief status]

## Blockers
- [what] — waiting on: [person]

## Upcoming
- [next concrete milestone]
```

Update these tables each time a new day page is added — pull the Completed items from today's session and merge into the weekly view. Don't duplicate — if an item already appears, update its status rather than adding a new row.

Also update the **Active Workstreams** table at the top of the Progress Document itself to reflect current status after each session.

---

## Step-by-step

### Step 1 — Draft both versions

Review the conversation and identify everything built, changed, or decided this session. Group by system.

Draft two versions simultaneously:
- **Plain English summary** (2–3 sentences, top of day page)
- **Technical detail** (system headers + bullets, below divider)

Show both to Saffron before posting. Ask: "Anything to add, correct, or remove?"

### Step 2 — Resolve the weekly sub-page

Fetch the Progress Document's child pages using `Notion:notion-fetch` or `Notion:notion-search`. Look for a page titled `Week of [Monday DD Mon YYYY]`.

- **Exists:** use its page ID
- **Doesn't exist:** create it under the Progress Document using `Notion:notion-create-pages`; add the weekly table structure (Completed, In Progress, Blockers, Upcoming)

### Step 3 — Create (or update) the daily sub-sub-page

Look for a child page of the weekly page titled `[Weekday DD Mon YYYY]`.

- **Doesn't exist:** create it under the weekly page; write the full content (plain English summary + divider + technical detail)
- **Exists (multiple sessions today):** fetch it and append today's session content below what's already there, with a thin divider between sessions

### Step 4 — Update the weekly sub-page tables

Merge today's completed items into the weekly Completed table. Update In Progress, Blockers, and Upcoming to reflect current state.

### Step 5 — Update the Active Workstreams table

Fetch the Progress Document and update the Active Workstreams table at the top:
- Status changes (e.g. Blocked → In Progress)
- Last Updated dates
- New workstreams if needed

### Step 6 — Confirm

Report back: "Posted — [link to weekly page] / [link to day page]."

---

## Relationship to other skills

- **`docs-manager`** — orchestrates this skill; run first in the chain
- **`stakeholder-update-writer`** — reads the day pages from Notion at end of day to generate the Evgenia email
- **`save-conversation`** — run alongside this at session end
