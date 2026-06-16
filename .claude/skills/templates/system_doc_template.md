# [System Name] — System Documentation
**Documented by:** Saffron Meltzer, Data Analyst
**Date created:** [DD/MM/YYYY]
**Last updated:** [DD/MM/YYYY]
**Version:** 1.0

---

## Purpose

*What problem does this system solve? Why does it exist? Write for a non-technical reader.*

[One paragraph explaining what this system does and why ONE LDN needs it.]

---

## Tools and Platforms Used

| Tool / Platform | Role in this system |
|---|---|
| [e.g., Supabase — project `ljjwssicvvyyueyznmou`] | [e.g., Stores and serves the sales data] |
| [e.g., GitHub Pages — repo `saffron-oneldn/shop-sales`] | [e.g., Hosts the dashboard website] |
| [e.g., WodBoard] | [e.g., Source of the raw sales data] |

---

## How It Works — Data Flow

*Step-by-step: where data comes from, what happens to it, where it ends up.*

1. [Step 1 — e.g., "Saffron exports the shop-sales CSV from WodBoard"]
2. [Step 2 — e.g., "The CSV is uploaded via the dashboard's upload button"]
3. [Step 3 — e.g., "The dashboard reads the file and saves it to Supabase"]
4. [Step 4 — e.g., "The dashboard displays the data from Supabase in charts and tables"]

**Data dependency diagram:**
```
[Source] → [Processing step] → [Storage] → [Output]
```

---

## Configuration Details

*How is it set up? Key settings, environment variables, or configuration choices.*

| Setting | Value | Notes |
|---|---|---|
| [e.g., Supabase project] | [project ID] | |
| [e.g., GitHub repo] | [repo URL] | |
| [e.g., Data refresh method] | [Manual upload / Auto] | |

---

## Access and Credential Management

*Who has access, what level, and where credentials are stored.*

| Access needed | Who has it | Credential location |
|---|---|---|
| [e.g., Supabase Admin] | Saffron | [Secure location] |
| [e.g., Dashboard (read-only)] | Any user with the URL | Public — no login required |

---

## Known Limitations and Risks

*What this system can't do, what could go wrong, what to watch out for.*

- [Limitation 1 — e.g., "Data is not updated in real time — requires manual CSV upload"]
- [Risk 1 — e.g., "If the Supabase free tier limit is reached, the dashboard will show no data"]
- [Risk 2 — e.g., "The GitHub Pages URL is public — do not display sensitive financial data"]

---

## Ongoing Maintenance

*Step-by-step instructions for a non-technical person to keep this running.*

### Regular tasks
| Task | Frequency | Who | How |
|---|---|---|---|
| [e.g., Upload new sales data] | [e.g., Weekly] | Saffron | [e.g., Export CSV from WodBoard → open dashboard → upload] |

### If something breaks
| Symptom | Likely cause | Fix |
|---|---|---|
| [e.g., Dashboard shows no data] | [e.g., Supabase connection issue] | [e.g., Check Supabase project is active at supabase.com] |

### Updating the code
- All code lives in the GitHub repository listed above
- Changes require editing `index.html` and pushing to GitHub
- Contact Saffron or a developer to make code changes

---

## Change History

| Date | Change | Reason | Version |
|---|---|---|---|
| [DD/MM/YYYY] | Initial build | New system | 1.0 |
