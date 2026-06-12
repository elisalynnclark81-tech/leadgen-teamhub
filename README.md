# CGM Lead Generation Hub

Salesforce Lightning-style lead generation dashboard for the CompuGroup Medical lead generation team — single-file HTML app, hosted free on GitHub Pages, backed by a live two-way Google Sheets database.

**Live URL pattern:** `https://elisalynnclark81-tech.github.io/<repo-name>/`

---

## What's in this repo

| File | What it is |
|---|---|
| `index.html` | The entire dashboard — UI, logic, and SheetJS import engine in one file. No build step, no dependencies. |
| `Code.gs` | Google Apps Script backend. Paste into the Google Sheet (Extensions → Apps Script). This is the bridge between the dashboard and the Sheet. |
| `CGM_LeadGen_Backend.xlsx` | Starter workbook. Upload to Google Drive → open as Google Sheets → this becomes the live database (LEADS, DAILY_ACTIVITY, COACHING_NOTES, LISTS tabs). |
| `README.md` | This file. |

Guides (distributed separately, not required in the repo):
- **CGM Rep Quick Start Guide.docx** — for LGS / BDR / SDR reps
- **CGM Manager Setup Guide.docx** — full deployment + administration walkthrough

---

## Architecture (60-second version)

```
Reps' browsers                     GitHub Pages                Google
┌──────────────┐    loads     ┌──────────────────┐   POST   ┌──────────────────┐
│  Rep opens    │ ──────────► │   index.html      │ ───────► │  Apps Script      │
│  personal link│              │  (the dashboard)  │ ◄─────── │  Web App (/exec)  │
└──────────────┘              └──────────────────┘   JSON   └────────┬─────────┘
                                                                      │ reads/writes
                                                             ┌────────▼─────────┐
                                                             │   Google Sheet    │
                                                             │ LEADS · ACTIVITY  │
                                                             │ COACHING · LISTS  │
                                                             └──────────────────┘
```

- **GitHub Pages** serves the static dashboard. It stores no data.
- **The Google Sheet** is the single source of truth for leads, activity, and coaching notes.
- **Apps Script** exposes the Sheet as a tiny JSON API protected by a team key.
- **Two-way sync:** manager imports/assignments flow down to reps; rep dispositions, status changes, and notes flow back up — typically within seconds.
- **Offline-friendly:** with Cloud Sync off, the dashboard still works standalone using browser storage (original behavior, fully preserved).

---

## Deploy to GitHub Pages (first time, ~5 minutes)

1. **Create the repository**
   - Go to https://github.com/new (logged in as `elisalynnclark81-tech`)
   - Repository name: `leadgen-hub` (or any name — it becomes the URL path)
   - Visibility: **Public** (required for free GitHub Pages)
   - Click **Create repository**

2. **Upload the dashboard**
   - On the new repo page: **uploading an existing file** link (or Add file → Upload files)
   - Drag in `index.html` (and optionally this `README.md`)
   - Commit message: `Initial dashboard deploy` → **Commit changes**

3. **Turn on GitHub Pages**
   - Repo → **Settings** → **Pages** (left sidebar)
   - Source: **Deploy from a branch**
   - Branch: `main` · Folder: `/ (root)` → **Save**
   - Wait 1–2 minutes. Your live URL appears at the top:
     `https://elisalynnclark81-tech.github.io/leadgen-hub/`

4. **Verify** — open the URL in a private/incognito window. You should see the CGM login screen.

### Updating the dashboard later
Repo → open `index.html` → pencil icon (Edit) → paste the new version → **Commit changes**. Pages redeploys automatically in ~1 minute. Reps just refresh — no reinstall, links stay the same.

---

## Connect the Google Sheets backend (one time, ~10 minutes)

Full illustrated walkthrough in the **Manager Setup Guide**. Summary:

1. Upload `CGM_LeadGen_Backend.xlsx` to Google Drive → open it → **File → Save as Google Sheets**.
2. In the Google Sheet: **Extensions → Apps Script** → delete the placeholder → paste all of `Code.gs` → set `TEAM_KEY` to a password of your choice → **Save**.
3. **Deploy → New deployment** → type **Web app** → Execute as: **Me** → Who has access: **Anyone** → **Deploy** → authorize when prompted.
4. Copy the **Web app URL** (ends in `/exec`).
5. In the dashboard (logged in as Manager): **Admin / Content → Settings → Cloud Sync** → paste URL + team key → **Test Connection** → **Save**.
6. **Lead Assignment → Rep Links** → copy each rep's personal link and send via Teams.

> Rep links carry the sync URL and key, so each rep's first click auto-connects them — zero setup on their side.

### Redeploying after editing Code.gs
**Deploy → Manage deployments → ✏️ → Version: New version → Deploy.** The `/exec` URL does not change, so nothing needs to be re-sent to reps.

---

## Standard lead fields

| Field | Values |
|---|---|
| **Lead Status** | Open · Call Back · Not Interested · Left Voicemail · No Answer · Do Not Call · LinkedIn Outreach · Webinar Invite Sent · Webinar Registered · Email Sent · Qualified |
| **Lead Source** | Net New Cold Call · Customer Based Calling · CMS NPI · CMS CLIA · Webinar · Other |

Call dispositions automatically set the unified Lead Status (mapping table on the LISTS tab of the backend Sheet, and editable in `index.html` → `DISP_TO_STATUS`). `Bad Phone Number` keeps the status but raises a red **Bad Phone** data-quality flag.

## Duplicate management

**Duplicate Manager** (Data & Ops section) finds duplicate records with Salesforce-style matching rules, in priority order:

1. **Email** — exact, case-insensitive
2. **Phone** — digits-only comparison (formatting ignored)
3. **Name + Company** — fuzzy match (strips LLC/Inc/Clinic/etc.)

Chained matches are clustered together (A=B by email, B=C by phone → one group). Pick the master record → **Merge**: empty master fields are filled from duplicates, notes are appended, the most-advanced status wins (Do Not Call always wins), duplicate rows are deleted from the Sheet. Imports also pre-check new rows against existing leads and skip duplicates by default.

## Per-rep links

Each rep gets `...?rep=<firstname>` (e.g. `?rep=misty`). Their link:
- skips the login screen and signs them in,
- lands on **My Leads**,
- auto-configures Cloud Sync (when generated while sync is on).

Generate and copy links from **Lead Assignment → Rep Links**.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Bad team key" on Test Connection | `TEAM_KEY` in Code.gs must exactly match the key in Admin → Settings. |
| Test Connection fails entirely | Re-deploy the web app; confirm access is **Anyone** and you copied the `/exec` URL (not the editor URL). |
| Rep link opens login screen instead of auto-login | Slug must be the rep's first name in lowercase (`?rep=honelyn`). Regenerate from Lead Assignment → Rep Links. |
| Changes to Code.gs not taking effect | You must create a **New version** under Manage deployments — saving alone doesn't republish. |
| Storage-full warnings in the dashboard | That's local mode. Turn on Cloud Sync; leads then live in the Sheet with no browser limit. |
| Sheet edits not showing for reps | Reps sync on view changes and the Sync pill (top bar). Have them tap **Sync** or refresh. |

---

## Security notes

- The Apps Script URL + team key together gate all reads/writes. Treat rep links like internal credentials — share via Teams/Outlook only.
- The Sheet itself stays private to your Google account; "Anyone" access on the *web app* means anyone **with the URL and the key** can call the API, not view the Sheet.
- No lead data is stored on GitHub — `index.html` is logic only.

---

CompuGroup Medical — internal sales enablement tool. Maintained by Lisa Clark, Lead Generation Sales Manager.
