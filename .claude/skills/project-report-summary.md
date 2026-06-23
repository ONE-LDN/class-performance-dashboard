---
name: project-report-summary
description: |
  Produces a short, scannable project SUMMARY report as a .docx file — a condensed sibling of the project-report skill. Use when Saffron asks for a 'summary report', 'project summary', 'short report', 'one-pager', a 'condensed' or 'exec' version of a project report, or names project-report-summary. Five fixed sections only: Executive Summary (Objective + Key Outcomes), The Plan / Target Structure (tabs table + per-tab key elements in point form), Research & Benchmark Criteria, Next Steps, and an Appendix. Same ONE LDN house style as project-report. Output is always a .docx delivered to /mnt/user-data/outputs/. Does NOT replace the full project-report (full narrative arc + background + audit + key decisions), progress updates (progress-update-writer), dashboard docs (dashboard-doc-pair-updater), or session logging (docs-manager).
---

# ONE LDN Project Report — Summary

Produces a short, management-readable **summary** of a project as a `.docx` file. This is the condensed counterpart to the `project-report` skill: where the full report builds the whole argument (background → benchmark → audit → design → decisions), the summary strips to what a reader needs to act — the objective, the structure being built, the standard it is held to, and what happens next, with an appendix for everything else.

**Use this when the reader wants the shape of the thing fast.** If they want the reasoning, the audit, or the decision log, use `project-report` instead. When in doubt which is wanted, ask.

**Point form over prose.** Unlike the full report (which is prose-led), the summary leans on tight bullets — especially in the per-tab breakdowns and Next Steps. Keep sentences short; cut filler.

---

## Structure (Fixed — Five Sections, Always This Order)

### 1. EXECUTIVE SUMMARY — Objective + Key Outcomes only
No long framing. One optional lead line, then the two labelled sub-sections:
- **Objective** — render in the gold-left-border callout box. A single sharp statement: who it is for, the question it answers, the baseline/comparison if relevant, and the threshold of success. State plainly what it is *not* meant to do.
- **Key Outcomes** — bullets, each opening with a bold lead phrase; the concrete, where-possible quantified results the work delivers or the plan locks in.

### 2. THE PLAN / TARGET STRUCTURE
Adapt the title to the project ("The Plan", "Target Structure", "The Redesign — Four-Tab Structure").
- **If it is a dashboard or any multi-view/multi-section tool:** open with the **Organising Principle table** at the top — `# | Tab/View | Question Answered | Status/Replaces`. Then one Title Case sub-heading per tab. Under each tab:
  - a **1–2 sentence summary** of the tab's purpose, then
  - **`Key elements:`** as a **bulleted list — point form, NOT prose.** Each bullet is a short element (a control, a card, a table, a filter), not a paragraph.
- **If it is not a dashboard:** describe the structure in point form under appropriate sub-headings — still bullets over prose.

### 3. RESEARCH & BENCHMARK CRITERIA
Kept tight:
- **Benchmark Metrics** table — `Metric | Priority | Notes` (Priority = Core / High / Medium).
- **Practical First-Version Requirements** — a short numbered list.
- A one-line note on what the benchmark is drawn from (e.g. sector norms, best-practice design principles). If the benchmark frame is not obvious, ask Saffron before drafting.

### 4. NEXT STEPS
Tiered bullets: **Immediate / Short-Term / Longer Term.** Add a short **Dependencies & Blockers** sub-heading only if there are live ones.

### 5. APPENDIX
The catch-all for relevant detail that would bloat the main flow. Include whatever applies, each under its own Title Case sub-heading, as bullets or compact tables — e.g. data sources and field names, exclusion rules, key decisions in brief, a coverage-against-benchmark snapshot, glossary, links, open questions and caveats. Omit the appendix only if there is genuinely nothing to add.

---

## Document Format — ONE LDN house style (identical to project-report)

Reproduce the house style exactly so the summary is visually indistinguishable from the full report. Generate with `docx-js`; sizes are in half-points, spacing in twips.

- **Page:** US Letter — `size: { width: 12240, height: 15840 }`. Margins `{ top: 1417, right: 1417, bottom: 1417, left: 1701 }`. Content width = **9122 DXA** — all table widths sum to this.
- **Font:** **Cambria** throughout (`styles.default.document.run = { font: "Cambria", size: 21, color: "444444" }`).
- **Title block:** "ONE LDN" — bold `888888` size 22, `spacing.before: 1200`. Project title — bold `111111` size 56. Report-type subtitle (use "Summary Report" or "Project Summary") — `444444` size 32. Metadata lines — label bold `444444` size 20, value `888888` size 20.
- **Section headings (H1):** bold, **ALL CAPS** (`allCaps: true`), `888888`, size 20, with a bottom rule (single, size 4, colour `CCCCCC`, space 4). `spacing.before: 360, after: 160`.
- **Sub-headings:** Title Case, bold, `444444`, size 21 (10.5pt), `spacing.before: 240, after: 80`.
- **Body:** `444444`, size 21, **justified**, `line: 288` (1.2), `spacing.after: 120` (6pt after). (The summary uses little body prose — mostly bullets.)
- **Bullets / numbered lists:** same as body (justified, `line: 288`) but `spacing.after: 60` (3pt after). Indent `left: 360, hanging: 220`. Lead phrase bold `111111`; remaining text `444444`; both size 21.
- **Objective callout:** single-cell table — fill `F7F7F0`, left border single size 16 colour `E8C547`, other borders none, cell margins `{ top:120, bottom:120, left:200, right:160 }`, text `444444` size 20.
- **Tables:** header rows `111111` fill with white bold 9pt (size 18) text; body cells `444444` 9.5pt (size 19) with `F7F7F7` zebra on alternate rows. Coverage-status cells (if used in the appendix): Present `2E7D32` / Partial `B07A00` / Missing `B23B3B`.
- **Currency/dates:** GBP, DD/MM/YYYY.

### Metadata block (top of document, before Section 1)

```
ONE LDN
[Project Title] — Summary Report
Date:          [DD Month YYYY]
Version:       [x.x — Draft for review / Final]
Project:       [repo or system name]
Prepared for:  [audience]
```

### Critical docx-js pitfalls (must follow — these have bitten us)

- **Always define an explicit `Normal` paragraph style AND set `style: "Normal"` on every non-heading paragraph** — including paragraphs inside table cells and callout boxes. docx-js does not emit a default `Normal` style when you supply custom `paragraphStyles`; any unstyled paragraph then falls back to the only named style present (`Heading1`) in some readers (notably Apple Pages), making the whole body open as headings. Section headings use `HeadingLevel.HEADING_1` (style `Heading1`, `basedOn: "Normal"`); everything else is `Normal`. Verify after building: unpack the `.docx` and confirm `styles.xml` has `w:styleId="Normal"` and that no body paragraph has `pStyle=Heading1` or a missing `pStyle`.
- **Tables need dual widths** (table `width` + each cell `width`, summing to content width) and `ShadingType.CLEAR` (never SOLID) — per the docx skill.

---

## Execution Steps

1. **Read the context.** Understand the objective, the structure being built, and the benchmark. If a full `project-report` already exists for this project, draw the summary from it. Ask for missing inputs before starting.

2. **Confirm it's the summary that's wanted**, not the full report. If the reader needs reasoning/audit/decisions, redirect to `project-report`.

3. **Read `/mnt/skills/public/docx/SKILL.md`** before writing any code.

4. **Draft the content first** — Objective + Key Outcomes, the tabs table and per-tab key-element bullets, the benchmark table, tiered next steps, appendix. Keep it point-form and tight.

5. **Generate the `.docx`** with the `docx` npm package, applying the house style and the docx-js pitfalls above. Set `NODE_PATH` to the global modules path if requiring `docx` globally.

6. **Validate** with `python /mnt/skills/public/docx/scripts/office/validate.py [file]`, then unpack to confirm the `Normal`-style check above.

7. **Deliver** to `/mnt/user-data/outputs/[project-name]-summary-[YYYY-MM-DD].docx` and surface it to the user (`present_files`, or the session's file-delivery tool).

---

## Example Section Map — PAYG Revenue Health Dashboard (summary)

| Section | Content |
|---|---|
| 1. Executive Summary | Objective (callout): weekly ahead/behind read on PAYG revenue. Key Outcomes: demand-pool split stated; two views, two questions; sequenced path to production |
| 2. The Plan — Target Structure | Organising Principle table (Health Check / Revenue Trends). Per tab: 1–2 sentence summary + Key elements bullets (KPI cards, deltas, product matrix; adaptive trend table, view-mode control, YoY tinting) |
| 3. Research & Benchmark Criteria | Benchmark metrics table (revenue-pacing + fitness-sector) + five first-version requirements |
| 4. Next Steps | Immediate (ClassPass split, live data, avg rev/unit) / Short-Term (refund toggle, charts) / Longer Term (intro conversion, combined-demand dashboard) |
| 5. Appendix | Data sources & fields; exclusion rules; ClassPass Gym Time vs Class Attendances note; prototype coverage snapshot; open questions |
