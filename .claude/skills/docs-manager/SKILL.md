---
name: docs-manager
description: |
  Orchestrates ONE LDN's contractual documentation suite at session end. Chains progress-update-writer, save-conversation, and conditionally dashboard-doc-pair-updater in the correct sequence, then handles the Change Log, Software Register, Credentials Register, and System Documentation updates itself. All documents live in Notion. Trigger whenever Saffron says "update the docs", "end of session", "log this change", "document this system", or at the end of any session where something was built, deployed, or changed.
---

# docs-manager

**Orchestrator skill.** Chains the full end-of-session documentation sequence in order. Owns the Change Log, registers, and system docs. Delegates progress writing to `progress-update-writer` and conversation saving to `save-conversation`.

---

## Notion page IDs

| Document | Page ID | URL |
|---|---|---|
| Progress Document | `3660164f-7fc6-81d7-9e14-e2085e8c6ed6` | https://www.notion.so/3660164f7fc681d79e14e2085e8c6ed6 |
| Change Log | `3660164f-7fc6-8116-8a1e-ed0735dcbdcb` | https://www.notion.so/3660164f7fc681168a1eed0735dcbdcb |
| Software Register | `3660164f-7fc6-81dd-8e48-ea7539200286` | https://www.notion.so/3660164f7fc681dd8e48ea7539200286 |
| Credentials Register | `3660164f-7fc6-8185-9340-f7b762c36304` | https://www.notion.so/3660164f7fc681859340f7b762c36304 |
| System Documentation (folder) | `3660164f-7fc6-81ad-b143-f19f80e8a727` | https://www.notion.so/3660164f7fc681adb143f19f80e8a727 |
| Dashboard Manuals (folder) | `3660164f-7fc6-81fe-8740-f408beb7086d` | https://www.notion.so/3660164f7fc681fe8740f408beb7086d |

**Dashboard Manual page IDs:**
| Dashboard | Manual page ID |
|---|---|
| PT & Coach Review Dashboard | `3660164f-7fc6-81df-ad12-fb614004eab1` |
| Shop & Cafe Dashboard | `3660164f-7fc6-8156-bf88-dda9915eae99` |
| Class Performance Dashboard | `3660164f-7fc6-81f7-b1dd-d583a41fdfdb` |
| Consumables Dashboard | `3660164f-7fc6-81d5-a084-cad809a54e28` |

All reads and writes use the Notion MCP (`Notion:notion-fetch`, `Notion:notion-update-page`, `Notion:notion-create-pages`, `Notion:notion-search`).

---

## Skill chain — run in this order

```
1. progress-update-writer      ← drafts + posts weekly/daily entries to Notion
2. save-conversation           ← saves session log
3. docs-manager (this)         ← Change Log, registers, system docs
4. dashboard-doc-pair-updater  ← only if a dashboard changed architecturally this session
```

Announce the sequence at the start:

> "Running end-of-session docs. Sequence: progress update → save conversation → change log + registers → system docs → dashboard docs if needed. Starting with the progress entry now."

Do not skip steps. Do not run steps in parallel — Saffron may need to confirm details between steps.

---

## Step 1 — Trigger progress-update-writer

Run `progress-update-writer`. It will draft the weekly and daily Notion pages, show them to Saffron for confirmation, then post.

**Do not proceed to Step 2 until progress-update-writer has confirmed it posted to Notion.**

---

## Step 2 — Trigger save-conversation

Run `save-conversation`. Proceed to Step 3 immediately after — no confirmation needed.

---

## Step 3 — Classify the session

Identify which of the following apply. A session can belong to multiple types.

| Session type | docs-manager action |
|---|---|
| Production system modified or deployed | Append to Change Log |
| New tool, platform, or subscription | Prompt for Software Register entry |
| New credential or system access | Prompt for Credentials Register entry |
| New system / dashboard / automation / script / skill / agent built | Generate system doc (Step 5) |
| Existing system / dashboard / script / skill / agent meaningfully changed | Update system doc (Step 5) |
| Analysis / reporting / planning only | No further docs-manager action |

---

## Step 4 — Update the Change Log (if a production change occurred)

A production change is: any push to GitHub Pages, any Supabase upsert or schema change, any GAS deployment, any WodBoard configuration change, or any change to a live system.

Ask Saffron to confirm: approver name, and planned vs emergency. Everything else inferred from session.

Append a new row to the Change Log page using `Notion:notion-update-page`:

```
| [DD/MM/YYYY] | [System name] | [What changed — one sentence] | [Why — one sentence] | [Planned / Emergency] | [Approved by] | [Notes] |
```

---

## Step 5 — Update registers (if new tools or credentials appeared)

**New software/tool/subscription:**
Ask for: account holder, cost, billing frequency, renewal date, notes. Append to Software Register:
```
| [Date] | [Platform] | [Account holder] | [Purpose] | [Cost] | [Billing] | [Renewal date] | [Notes] |
```

**New credential or system access:**
Ask for: username/account name only. Append to Credentials Register:
```
| [Platform] | [Purpose] | [URL] | [Username] | See secure location | [Access level] | [Permissions] | [Created] | [Updated] | [Notes] |
```
Remind Saffron: "Add the credential value to the designated secure location — not here."

---

## Step 6 — System Documentation

### What triggers a system doc

Generate or update a system doc whenever **any of the following were built or meaningfully changed** this session:

- Dashboard (new build or tab/feature/data source added)
- Data pipeline or cleaning script (e.g. `payg_clean_and_combine.py`)
- Automation or agent (e.g. Daily Ops Agent, cron routines)
- Skill (new SKILL.md or significant update to an existing one)
- Standalone script that meaningfully changed the workflow
- Any integration between two systems

**Do not generate a system doc for:** one-off analyses, reports, Excel outputs, planning documents, or minor config tweaks that don't affect how the system works.

### New system — create two Notion pages

Create both as children of the System Documentation folder (`3660164f-7fc6-81ad-b143-f19f80e8a727`) using `Notion:notion-create-pages`.

**Page A — Technical (`[System Name] — Technical`)**

Writing style: Saffron's working language — precise, operational shorthand. Technical terminology without definition (RLS, Supabase, anon key, GitHub Pages, GAS, SKILL.md, cron, etc.). Architecture rationale, dependencies, config values, layer references. Assume reader has full ONE LDN stack context.

Must include:
- Purpose
- Tools and Platforms Used
- Data Flow / Architecture
- Configuration Details
- Access and Credentials
- Known Limitations and Risks
- Ongoing Maintenance
- Change History

**Page B — Plain English (`[System Name] — Plain English`)**

Writing style: Plain spoken English. No acronyms without explanation. Short sentences. Explain what it does and why it exists — omit implementation detail unless it affects a non-technical user. Aimed at Evgenia Koroleva and ONE LDN ops staff.

Active rewriting rule: Don't strip jargon — rewrite from scratch. "RLS policies restrict row-level access by user JWT claims" → "The dashboard only shows each user their own data — access is controlled automatically."

Must include (simplified):
- What does it do
- Why does it exist
- What does [relevant person] need to do day-to-day
- What tools are involved (described in plain terms)
- What happens if something goes wrong
- Who should be told if this changes

### Existing system — update the relevant pages

Fetch the existing technical and plain English pages. Identify which sections are affected by this session's changes. Update those sections only — don't overwrite unchanged content.

Always append to the Change History table in the technical doc:
```
| [DD/MM/YYYY] | [What changed] | [Why] | [Version bump] |
```

If the system is a dashboard, also trigger `dashboard-doc-pair-updater` (Step 7) to update the user manual in Dashboard Manuals.

### After creating or updating system docs

Update the Documentation Log in the Progress Document with Notion page links:
```
| [System name] | [DD/MM/YYYY] | [v1.0 or updated version] | [link — technical] | [link — plain English] |
```

---

## Step 7 — Trigger dashboard-doc-pair-updater (if a dashboard changed)

If this session involved a material change to a live dashboard (new tab, new data source, RLS change, deployment change, new feature), trigger `dashboard-doc-pair-updater` after Step 6.

Pass it:
- The change description
- The Notion page ID of the technical system doc
- The Notion page ID of the corresponding Dashboard Manual page (see IDs at top of this skill)

---

## Document standards (from Schedule 1)

Every system documentation page must include:
- Purpose — what problem this solves, why it exists
- Tools and platforms — named specifically (e.g. "Supabase project `ljjwssicvvyyueyznmou`")
- Configuration details — how it's set up, key settings
- Data flows and dependencies — what goes in, what comes out, what it depends on
- Access and credential management — who has access, where credentials are stored
- Known limitations or risks — what it can't do, what could break
- Ongoing maintenance instructions — step-by-step for a non-technical person

The plain English version satisfies the Schedule 1 "non-technical person" requirement. The technical version is the internal working reference.

---

## Handover summary (on request)

When Saffron asks to generate a handover summary, fetch all four living documents from Notion and use `templates/handover_summary_template.md` as the structure. Create the summary as a new Notion page under the hub titled `Handover Summary — [DD/MM/YYYY]`. Include the signed confirmation per Schedule 1 clause 4. Share the link with Saffron and Evgenia Koroleva.

---

## Fallback

If the Notion MCP is not available, save all updates as local Markdown files to `~/Documents/Claude/WORK/ONE_LDN/Documentation/` and flag clearly to Saffron that manual upload is needed.

---

## Skill relationships

| Skill | Role | When |
|---|---|---|
| `progress-update-writer` | Writes weekly/daily Notion pages | Step 1 — always |
| `save-conversation` | Saves session log | Step 2 — always |
| `dashboard-doc-pair-updater` | Updates system doc + dashboard manual | Step 7 — if dashboard changed |
| `stakeholder-update-writer` | Drafts Evgenia email from day pages | End of day — separate trigger |
| `one-ldn-context` | Reference for system names, IDs, stakeholders | Load as needed |
