---
name: dashboard-doc-pair-updater
description: >
  Keep dashboard documentation in sync. When a dashboard changes, updates both the technical system
  doc and the user-facing manual in Notion - producing two versions: a technical version in Saffron's
  working language and a plain-English version for non-technical stakeholders (Evgenia Koroleva).
  Saves all outputs to Notion. Provide a change summary, the system doc page ID, and the dashboard
  manual page ID. Use whenever a dashboard changes architecture, data access, setup, or troubleshooting
  guidance. Trigger when user says "update the docs", "the dashboard changed", "I just added X to
  the dashboard", or "docs are out of date".
---

# dashboard-doc-pair-updater

Updates a dashboard's technical system doc and user-facing manual in Notion when the dashboard changes. Called by `docs-manager` (Step 7) or triggered directly when a dashboard changes.

---

## Notion targets

**System Documentation folder:** `3660164f-7fc6-81ad-b143-f19f80e8a727`
**Dashboard Manuals folder:** `3660164f-7fc6-81fe-8740-f408beb7086d`

**Known page IDs:**

| Dashboard | Technical doc | User manual |
|---|---|---|
| PT & Coach Review Dashboard | `3660164f-7fc6-8197-ae31-d7d4f07acd31` | `3660164f-7fc6-81df-ad12-fb614004eab1` |
| Shop & Cafe Dashboard | — | `3660164f-7fc6-8156-bf88-dda9915eae99` |
| Class Performance Dashboard | — | `3660164f-7fc6-81f7-b1dd-d583a41fdfdb` |
| Consumables Dashboard | — | `3660164f-7fc6-81d5-a084-cad809a54e28` |
| ONE LDN Daily Ops Agent | `3660164f-7fc6-81e4-abf4-ce2df397139e` | `3660164f-7fc6-816e-809b-dd3362ebd203` |

If a dashboard doesn't have an existing page, create one under the appropriate folder using `Notion:notion-create-pages`.

---

## Inputs required

- **Change description** — 1–3 sentences describing what changed
- **System doc page ID** — the technical Notion page to update
- **Manual page ID** — the user-facing Notion page to update

If called from docs-manager, these will be passed directly. If triggered standalone, ask Saffron which dashboard changed.

---

## Step 1 — Read both pages

Fetch both pages using `Notion:notion-fetch`:
- Read the technical system doc to understand current state
- Read the user manual to understand current state

Identify which sections are affected by the change.

---

## Step 2 — Map the change to affected sections

| Change type | Technical doc sections | User manual sections |
|---|---|---|
| Permissions / RLS | Architecture, Data Flow, Troubleshooting | How it works, What happens if something goes wrong |
| Deployment method | Architecture, Configuration, Troubleshooting | Is it running yet, What happens if something goes wrong |
| New data source / table | Architecture, Data Flow, Configuration | How it works, What does it do |
| New feature / tab | Architecture, Data Flow | What does it do, What does [person] need to do |
| Bug fix | Configuration, Troubleshooting, Change History | What happens if something goes wrong |

Apply **coordinated but audience-appropriate updates**: technical doc gets the implementation detail; user manual gets the user-facing consequence. Same change, different angle.

---

## Step 3 — Update the technical system doc

Use `Notion:notion-update-page` with `command: update_content` to update the relevant sections.

**Writing style:**
- Saffron's working language — precise, operational shorthand
- Technical terminology without definition (RLS, Supabase, anon key, GitHub Pages, GAS, etc.)
- Include rationale, dependencies, config values, layer references
- Assume reader has full ONE LDN stack context

Always append a row to the Change History table:
```
| [DD/MM/YYYY] | [What changed] | [Why] | [new version] |
```

---

## Step 4 — Update the user manual

Use `Notion:notion-update-page` with `command: update_content` to update the relevant sections.

**Writing style:**
- Plain spoken English — no acronyms without explanation, no assumed technical context
- Short sentences, friendly tone
- Explain the what and why in plain terms — omit implementation detail unless it affects the user
- Aimed at Evgenia Koroleva and ONE LDN ops staff

**Active rewriting rule:** Don't just strip jargon — rewrite from scratch. "RLS policies restrict row-level access by user JWT claims" → "The dashboard only shows each user their own data — access is controlled automatically."

**Sections to always keep in the manual (simplified):**
- What does it do
- Why does it exist
- What does [person] need to do day-to-day
- What tools are involved
- What happens if something goes wrong
- Who should be told if this changes

---

## Step 5 — Verify

After updating, fetch both pages and confirm:
- Changed sections contain the new content
- Unchanged sections are untouched
- Change History row appended to technical doc
- Both pages have current Last Updated date

Report a brief summary of what changed in each page and confirm with Notion links.

---

## Common pitfalls

- **Don't overwrite whole pages** — use `update_content` with precise `old_str` / `new_str` pairs, not `replace_content`
- **Troubleshooting sections** often have multiple entries — append to the end, don't replace existing entries
- **Plain English version**: don't summarise the technical version — rewrite it for a non-technical reader from scratch
- **If a page doesn't exist yet**: create it under the correct folder, write full content, then confirm with Saffron
