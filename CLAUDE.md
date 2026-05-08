# fadmin-schedule-sync — Project Context

## What This Is

A browser-based tool that converts FAdmin schedule exports (CSV) into ClickUp-ready import files (CSV). The entire transformation runs client-side — no server, no data ever leaves the user's browser.

**Live URL:** https://aascroft.github.io/fadmin-schedule-sync
**Repo:** https://github.com/aascroft/fadmin-schedule-sync
**Slack channel:** #fadmin-schedule-sync

---

## Current State — V1

- [x] `index.html` built and deployed via GitHub Pages
- [x] Processing Support (PS) mode implemented
- [x] Flyer Management Queue (FMQ) mode implemented
- [x] Dedupe logic implemented
- [x] Failure mode hardening (column validation, file extension check, actionable error messages)
- [x] Built-in step-by-step instructions panel
- [ ] V1 team testing in progress
- [ ] Part 5 (ClickUp import steps) to be written after testing confirms import workflow

**Next updates** will most likely come from team testing feedback — specific rows, columns, retailers, or output cells identified as incorrect.

---

## Background

Flipp's operations team manages retailer flyer campaigns in FAdmin. The team exports the upcoming schedule as a CSV, manually transforms it (currently 1–2 days of work), and imports the result into ClickUp for task management. This tool eliminates that manual step.

---

## Architecture

Single `index.html` — all HTML, CSS, JS inline. No build step, no server, no framework. PapaParse v5.4.1 via CDN for CSV parsing.

```
index.html
  ├── Mode selector (Processing Support / Flyer Management Queue)
  ├── Three upload zones: FAdmin Export | Context File | Dedupe File
  ├── Generate CSV button
  ├── Status area (row counts, dedupe counts, warnings, errors)
  ├── Download button
  └── Instructions panel (collapsible, tabbed per mode)
```

Deploy = push to `main`. GitHub Pages serves within ~60 seconds.

---

## How the Tool Works

The user selects a mode, uploads three CSV files, clicks Generate, and downloads the output.

| Upload Zone | Processing Support | Flyer Management Queue |
|---|---|---|
| FAdmin RAW Export | Same file for both modes | Same file for both modes |
| Context File | PS Retailer Hub (ClickUp) | Merchant Information Database (ClickUp) |
| Dedupe File | Processing Support Board (ClickUp) | Flyer Management Queue (ClickUp) |

**Dedupe logic:** The tool constructs a Flyer Run URL for each FAdmin row (`https://fadmin.flippback.com/flyer_runs/{Flyer Run ID}`). If that URL already exists in the `Flyer Run (url)` column of the dedupe export, the row is skipped. This prevents creating duplicate ClickUp tasks.

**Output:** One import-ready CSV per run. Column headers and column order must exactly match the authoritative examples (documented below).

---

## FAdmin Export — Columns Used

The FAdmin export contains many columns. Only these are used by the tool:

| Column Name | Purpose |
|---|---|
| `Flyer Run ID` | Constructs the Flyer Run URL; used as dedupe key |
| `Merchant ID` | FMQ join key (not output) |
| `Merchant Name` | Output (both modes) |
| `Flyer Type ID` | PS join key; FMQ join key |
| `Flyer Run Name` | PS Task Name (second half of concatenation) |
| `Live Date` | Source for all date calculations |
| `Valid Date` | PS and FMQ output — direct passthrough |
| `End date` | FMQ output — direct passthrough (note: lowercase 'd') |

**Date format in FAdmin:** `MM/DD/YYYY` with leading zeros (e.g., `04/03/2026`). The tool normalizes all dates to `M/D/YYYY` without leading zeros in the output (e.g., `4/3/2026`).

---

## Date Calculation Logic

Both context files have `Assets Check Days (short text)` and `Preview Days (short text)`. Blank or null values are treated as 0.

| Output Field | Formula |
|---|---|
| Start Date | `Live Date − Assets Check Days` |
| Preview Date | `Live Date − Preview Days` |
| Due Date | `Live Date − Preview Days − 1` (i.e., Preview Date − 1) |
| Valid Date | FAdmin `Valid Date` column — passthrough only |
| Live/Preview Date (FMQ) | FAdmin `Live Date` column — passthrough only |
| End Date (FMQ) | FAdmin `End date` column — passthrough only |

**Verified example:** Fresh Thyme Market — Preview Days=1, Live Date=Feb 3 → Preview Date=Feb 2, Due Date=Feb 1 ✓

---

## Processing Support (PS) — Transformation Rules

### Join Key
FAdmin `Flyer Type ID` → PS Context `Flyer Type ID (short text)`

One context row per Flyer Type. If no match is found, the FAdmin row is skipped (expected for retailers that belong to other workflows).

### Task Name
`{Merchant Name} - {Flyer Run Name}`

Direct concatenation with ` - ` separator. No date parsing. The Flyer Run Name in the FAdmin export already contains the date range as a string (e.g., "Apr 10").

### Processor Field
The output column is `Processor (drop down)`. The source in the PS context file is `Coordinator (drop down)` — these are the same field with different names. The tool maps `Coordinator (drop down)` → `Processor (drop down)`.

### Output — 19 Columns (exact order)

| # | Output Header | Source |
|---|---|---|
| 1 | `Task Name` | `{Merchant Name} - {Flyer Run Name}` |
| 2 | `Task Content` | PS Context `Task Content` (may contain retailer processing notes; leading/trailing newlines stripped) |
| 3 | `Due Date` | Live Date − Preview Days − 1 → M/D/YYYY |
| 4 | `Start Date` | Live Date − Assets Check Days → M/D/YYYY |
| 5 | `Ext. Comms (drop down)` | PS Context `External Comms (Form Entry) (drop down)` |
| 6 | `FADMIN Merchant Page (url)` | PS Context `FADMIN Merchant Page (url)` |
| 7 | `FQC (drop down)` | PS Context `FQC (drop down)` |
| 8 | `FTP Path (url)` | PS Context `FTP Path (url)` |
| 9 | `Flyer Cadence (drop down)` | PS Context `Flyer Cadence (drop down)` |
| 10 | `Flyer Review (drop down)` | PS Context `Flyer Review (drop down)` |
| 11 | `Flyer Review Guide (url)` | PS Context `Flyer Review Guide (url)` |
| 12 | `Flyer Run (url)` | `https://fadmin.flippback.com/flyer_runs/{Flyer Run ID}` |
| 13 | `Lead (drop down)` | PS Context `Lead (drop down)` |
| 14 | `OneGuide (url)` | PS Context `OneGuide (url)` |
| 15 | `Page Swaps & Revisions (drop down)` | PS Context `Page Swaps & Revisions (drop down)` |
| 16 | `Preview Date` | Live Date − Preview Days → M/D/YYYY |
| 17 | `Processor (drop down)` | PS Context `Coordinator (drop down)` |
| 18 | `Upload (drop down)` | PS Context `Upload (drop down)` |
| 19 | `Valid Date` | FAdmin `Valid Date` → M/D/YYYY (may differ from Live Date) |

### Required Columns — PS Context File
`Flyer Type ID (short text)`, `Preview Days (short text)`, `Assets Check Days (short text)`

---

## Flyer Management Queue (FMQ) — Transformation Rules

### Join Key
FAdmin `Merchant ID` + `Flyer Type ID` → FMQ Context `Merchant ID (short text)` + `Flyer Type ID (short text)`

**Critical:** Only rows in the FMQ context file where `Task Type` = `Task` are used to build the lookup map. Subtask rows and section rows are excluded.

### Task Name — Date Range Format

The FMQ task name is a date range built from the FAdmin `Live Date` and `End date` columns. Three format rules apply:

| Condition | Format | Example |
|---|---|---|
| Live and End in same month and year | `{Merchant} - {Full Month} {start day} to {end day}, {year}` | `Accès pharma - January 15 to 28, 2026` |
| Different months, same year | `{Merchant} - {Full Month} {start day} to {Full Month} {end day}, {year}` | `Accès pharma - January 29 to February 11, 2026` |
| Different months AND different years (cross-year) | `{Merchant} - {Short Month} {start day} to {Short Month} {end day}, {end year}` | `Accès pharma - Dec 18 to Jan 14, 2026` |

If either date is missing or unparseable, the task name falls back to the merchant name only.

### Output — 14 Columns (exact order)

| # | Output Header | Source |
|---|---|---|
| 1 | `Merchant Name` | FAdmin `Merchant Name` |
| 2 | `Subtasks` | *(empty)* |
| 3 | `Flyer Run ID` | FAdmin `Flyer Run ID` |
| 4 | `Flyer Type ID` | FAdmin `Flyer Type ID` |
| 5 | `Flyer Run Link` | `https://fadmin.flippback.com/flyer_runs/{Flyer Run ID}` |
| 6 | `Task Name` | Date range format (see rules above) |
| 7 | `Start Date` | Live Date − Assets Check Days → M/D/YYYY |
| 8 | `Live/Preview Date` | FAdmin `Live Date` → M/D/YYYY (passthrough) |
| 9 | `Valid Date` | FAdmin `Valid Date` → M/D/YYYY (passthrough) |
| 10 | `Due Date` | Live Date − Preview Days − 1 → M/D/YYYY |
| 11 | `End Date` | FAdmin `End date` → M/D/YYYY (passthrough) |
| 12 | `Oneguide` | FMQ Context `OneGuide (url)` |
| 13 | `Task Description` | *(empty)* |
| 14 | `Time Estimate` | FMQ Context `Time Estimate` |

### Required Columns — FMQ Context File
`Merchant ID (short text)`, `Flyer Type ID (short text)`, `Task Type`, `Preview Days (short text)`, `Assets Check Days (short text)`

---

## Failure Mode Hardening

The tool validates all three uploaded files before running the transform. Each failure produces a specific, actionable error message.

| Failure | Behaviour |
|---|---|
| Non-CSV file uploaded (.xlsx etc.) | Inline error on upload zone immediately |
| Wrong file in FAdmin zone (missing expected columns) | Hard stop — names the missing columns, tells user to re-upload the correct FAdmin file |
| Context file missing columns / "All Columns" not toggled | Hard stop — names the missing columns, tells user to re-export with All Columns enabled |
| Dedupe file missing `Flyer Run (url)` column | Hard stop — explains duplicate risk, tells user to re-export with All Columns enabled |
| 100% of rows fail context lookup | Distinct error — "No rows matched at all" (not the normal partial-miss message) |
| Some rows have blank Live Date | Non-blocking warning alongside the success message |

---

## Schema Assumptions and Known Limitations

These assumptions were confirmed by reverse-engineering the example files. If any of these change, the tool will need to be updated.

1. **FAdmin column names are stable.** The tool reads columns by name. Any rename in the FAdmin export format will break the affected transformation silently or cause a validation error.

2. **ClickUp column names are stable.** The context and dedupe files are read by column name. ClickUp occasionally changes export column names. The validation checks will catch changes to critical join-key columns; non-critical column renames will cause silent blank values in the output.

3. **ClickUp export must use "All Columns."** Exports without All Columns enabled will be missing the required join-key columns and will fail validation.

4. **FMQ context: Task-type rows only.** The FMQ Merchant Information Database export contains rows of multiple types (Task, subtask, section). Only rows where `Task Type` = `Task` are used. If ClickUp changes the Task Type values, the lookup map will be empty.

5. **FAdmin date format is MM/DD/YYYY.** The tool parses dates by splitting on `/` and assuming month/day/year order. If the FAdmin export format changes, all date calculations will break.

6. **PS Processor source field.** In the PS context export, the field is named `Coordinator (drop down)`. The output column is `Processor (drop down)`. These are the same field. If ClickUp renames `Coordinator (drop down)`, the Processor column in the PS output will be blank.

7. **Valid Date can differ from Live Date.** In PS output, `Valid Date` is a direct passthrough from FAdmin col `Valid Date` — it is not the same as Live Date. Do not substitute one for the other.

8. **FAdmin `End date` has a lowercase 'd'.** The column header is `End date`, not `End Date`. The tool reads it exactly as-is.

9. **Output column order is fixed.** ClickUp's CSV import maps columns by header name, but the order in the file should match the authoritative examples to avoid any import field-mapping issues.

---

## Example Files — Reference Only

The following files were used to reverse-engineer all transformation logic. They are **not needed at runtime** — the app is entirely client-side and does not reference these files.

| File | What it was used for |
|---|---|
| `example - FAdmin RAW Export.csv` | Confirmed FAdmin column names, date formats, and data structure |
| `example - ClickUp - PS - Context - PS Retailer Hub.csv` | Confirmed PS join key, confirmed all context column names |
| `example - ClickUp - FMQ - Context - Merchant Information Database.csv` | Confirmed FMQ join key, Task Type filter, context column names |
| `example - ClickUp - PS - Dedupe Tasks.csv` | Confirmed dedupe column name (`Flyer Run (url)`) for PS |
| `example- ClickUp - FMQ - Dedupe Tasks.csv` | Confirmed dedupe column name for FMQ |
| `example - PS - Import Ready.csv` | Authoritative PS output format — 19 columns, exact order, exact headers |
| `example - FMQ - Import Ready.csv` | Authoritative FMQ output format — 14 columns, exact order, exact headers |
| `Processing Support Scheduling Import Tool Sheet.xlsx` | Legacy Google Sheets tool — used to verify join key (Flyer Type ID), date formulas, and PS Task Name format |

**It is safe to delete these files from the repo root.** All schema, column mapping, transformation logic, and format decisions derived from them are documented in this file. If you ever need to re-verify the logic against raw data, you would need to re-export fresh examples from FAdmin and ClickUp at that time (the original examples may be from a different date range than current live data).

---

## Development Workflow

```bash
git clone https://github.com/aascroft/fadmin-schedule-sync
cd fadmin-schedule-sync

# Edit index.html directly
# Test by opening index.html in a browser (no server needed)

# Deploy
git add index.html
git commit -m "your message"
git push origin main
# GitHub Pages updates within ~60 seconds
```

---

## ClickUp Export Instructions (for reference)

Both context and dedupe files are exported the same way:
1. Open the correct ClickUp list
2. Make sure all statuses are expanded
3. Click the settings cog → Export view
4. Toggle on **All Columns**
5. Format: CSV
6. Click Download
