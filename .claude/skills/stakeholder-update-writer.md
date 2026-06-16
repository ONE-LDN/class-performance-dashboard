---
name: stakeholder-update-writer
description: >
  Produces a plain English end-of-day email summary for Evgenia Koroleva covering what was worked
  on that day across all ONE LDN sessions. Triggered manually at end of working day. Reads today's
  progress entries from the Notion Progress Document, distils them into a brief, jargon-free Gmail
  draft, and creates it via the Gmail MCP. Trigger when Saffron says "send Evgenia an update",
  "draft the daily email", "end of day update", or "stakeholder update".
---

# stakeholder-update-writer

Produces the end-of-day plain English email summary for Evgenia Koroleva. Triggered manually — run once at the end of the working day, after all sessions are complete.

## Output

A Gmail draft created via `Gmail:create_draft`:
- **To:** evgenia.koroleva@oneldn.com
- **Subject:** `ONE LDN — Daily Update [DD MMM YYYY]`
- **Body:** Plain text, no markdown, signed "Saffron"
- **Mode:** Draft only — Saffron reviews and sends manually

---

## Source material

Read today's completed work from the Notion Progress Document:

**Page ID:** `3660164f-7fc6-81d7-9e14-e2085e8c6ed6`

Fetch the page using `Notion:notion-fetch`, find the entry for the current week (`### Week of...`), and use the **Completed** section as the basis for the email. Cross-reference with the current session context for anything logged in this session that may not yet be posted.

---

## Writing style — the benchmark

This is what the output should read like:

> Hi Evgenia,
>
> Quick update on today's work. The shop dashboard has had a few new features added — there are now two new alert cards on the summary page that flag when stock needs ordering soon or urgently, and these link straight through to the order prediction tab. I've also built out a new stock history tab that shows a month-by-month breakdown per product including revenue, cost, and margin — though the margin figures will show as a dash until cost prices are filled in.
>
> On the consumables side, I've built a new dashboard for tracking cafe consumables (things like cups, bags, packaging). It's live at the usual dashboards page. I'm just waiting on invoices from Harry before I can finish the bit that reads supplier delivery notes automatically.
>
> Happy to walk through anything on a call.
>
> Saffron

**Rules:**
- Plain spoken English — no acronyms without explanation, no technical terms
- 3–6 sentences or short paragraphs; one paragraph per major area of work
- Explain the *what* and *why in plain terms* — what does this mean for the business or for Evgenia, not how it was built
- Name the system in plain terms: "shop dashboard", "PT dashboard", "the consumables tracker" — not "shop-sales repo" or "Supabase schema"
- If something is blocked or waiting on a person, mention it briefly and name who: "I'm waiting on Harry for the invoice files"
- Don't mention tools, code, databases, GitHub, Supabase, or anything technical unless it directly affects Evgenia
- Tone: direct, warm, professional — like a colleague giving a quick end-of-day verbal update
- Always end with an offer to discuss if needed
- Sign off as "Saffron"
- Never use bullet points in the email body — prose only

**Active rewriting rule:** Don't summarise the progress entry — translate it. "5 schema migrations applied to Supabase project ljjwssicvvyyueyznmou" becomes "I've updated the database to support the new features." The goal is that Evgenia understands what happened and why it matters without needing any technical context.

---

## Step-by-step

### Step 1 — Read today's work

Fetch the Progress Document from Notion. Find the current week's entry. Extract the Completed items. Cross-reference with the current conversation for anything not yet posted.

If nothing was completed today (e.g. planning-only session), note what was worked on and what the next step is — still worth sending.

### Step 2 — Group by area

Group completed items into 2–4 plain-English areas (e.g. "shop dashboard", "consumables tracker", "PT dashboard", "PAYG analysis"). Each area gets one paragraph in the email.

Don't list every bullet — synthesise. If 5 things happened to the shop dashboard, summarise them as "a few new features" and call out the 1–2 most visible ones.

### Step 3 — Draft the email

Write in plain English (see style guide above). Show the draft to Saffron before creating the Gmail draft. Ask: "Happy with this, or anything to adjust?"

### Step 4 — Create the Gmail draft

Once confirmed, call `Gmail:create_draft`:

```
to: evgenia.koroleva@oneldn.com
subject: ONE LDN — Daily Update [DD MMM YYYY]
body: [plain text email]
```

Confirm with: "Draft created — it's in your Gmail drafts folder ready to review and send."

---

## What NOT to include

- Anything that's in progress but not completed (unless it's a blocker worth flagging)
- Technical details: file names, repo names, database names, migration counts, commit hashes
- Jargon: RLS, Supabase, anon key, GitHub Pages, GAS, CSV parser, cron, etc.
- Anything Saffron explicitly says to leave out

---

## Relationship to other skills

- **`progress-update-writer`** — run this first in each session; stakeholder-update-writer reads what it posts
- **`docs-manager`** — run alongside progress-update-writer at session end; this skill runs separately at end of day
