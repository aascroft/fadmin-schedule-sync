# fadmin-schedule-sync — Project Context

## What This Is

A browser-based tool that converts FAdmin schedule exports (CSV) into ClickUp-ready import files (CSV). Entirely client-side — no server, no data leaves the user's browser.

**Live URL:** https://aascroft.github.io/fadmin-schedule-sync
**Repo:** https://github.com/aascroft/fadmin-schedule-sync
**Slack channel:** #fadmin-schedule-sync

---

## For Feedback Triage Sessions — Start Here

The tool is in V1 team testing. The team submits feedback via a Google Sheet — one row per issue, with: process (PS / FMQ / Both), issue type, affected column, retailer / merchant, Flyer Run URL, what the tool generated, what it should have generated, source of truth, scope, notes.

**Two fields drive the fix path:**

- **Source of truth** — if it's "ClickUp context export" and the value reported as wrong actually IS that value in ClickUp, the bug is **data-side** (fix in ClickUp). If the source is correct but the tool output is wrong, the bug is **code-side**.
- **Scope** — "one cell" or "one retailer" → almost always data-side. "Whole column" / "every row" → almost always code-side.

**Before changing any code:**
1. Read **Things That Look Like Bugs But Aren't** below to filter false positives
2. Use the **Feedback Triage Guide** to map symptom → likely cause → code area
3. Use the **Code Reference Map** to find the exact line(s) to read

**After any code change:** push to `main`. The user re-tests with their real FAdmin + ClickUp exports. That re-test IS the verification — there are no example files and no automated test suite in the repo.

---

## Code Reference Map

Line numbers in `index.html` (approx. 900 lines, single-file app):

| Concept | Function / Block | Lines |
|---|---|---|
| State & mode tracking | `state`, `selectMode` | 404-433 |
| File upload + `.csv` extension check | `setFile` | 453-473 |
| Column existence validation | `validateColumns` | 481-486 |
| FAdmin date parsing (MM/DD/YYYY) | `parseFAdminDate` | 500-507 |
| Date math + normalize to M/D/YYYY | `addDays`, `normDate` | 508-520 |
| Dedupe Set construction | `buildDedupeSet` | 524-531 |
| PS context map (join: Flyer Type ID) | `buildContextMapPS` | 533-540 |
| FMQ context map (filters Task Type = "Task", collects Flyer Subtask names) | `buildContextMapFMQ` | 543-566 |
| **PS output headers (19 cols, fixed order)** | `PS_HEADERS` constant | 569-589 |
| PS transform main loop | `transformPS` | 591-647 |
| **PS row construction (output object)** | inside `transformPS` | 621-643 |
| **FMQ output headers (22 cols, fixed order)** | `FMQ_HEADERS` constant | 650-673 |
| FMQ transform main loop | `transformFMQ` | 675-737 |
| **FMQ row construction (output object)** | inside `transformFMQ` | 710-733 |
| CSV unparse + download trigger | `downloadCSV` | 739-746 |
| Validation + transform orchestration | `runTransform` | 753-855 |
| User-facing status / error display | `showStatus` | 858-875 |

**Critical footgun:** the `*_HEADERS` constant and the row-construction object inside the transform are **separate blocks**. Adding, removing, or renaming a column requires editing **both**. If only one changes, output gets a missing column or a mismatched header. The two blocks are 35-50 lines apart for each mode.

---

## Feedback Triage Guide

| Feedback symptom | Most likely cause | Investigate |
|---|---|---|
| One cell blank for one retailer | ClickUp context row has it blank (data-side) | Open the actual ClickUp context export, find that retailer's row, check the column. If blank in source → data fix in ClickUp; if populated in source → code bug. |
| Whole column blank in every row | Code reads a column name that doesn't exist | Find column in `PS_HEADERS` / `FMQ_HEADERS`, find its row-construction line, verify the source column name string matches the ClickUp export header exactly (incl. trailing " (short text)" / " (drop down)" / parentheses / casing) |
| One cell wrong for one retailer | Same as blank-for-one — data-side most likely | Same investigation |
| Whole column wrong in every row | Wrong source column referenced in row construction | Same as whole-column-blank |
| Wrong date in any date field | Date formula incorrect, or Live Date wrong upstream | Date math at lines 508-520; cross-check formulas against "Date Calculation Logic" section |
| Wrong FMQ task-name format | One of the 3 branches in `formatFMQTaskName` chose incorrectly | Lines 525-550. Check live + end date months/years against the rules table |
| Missing rows (expected but not in output) | Failed context lookup OR was deduped | Check the join key exists in context file (PS: Flyer Type ID; FMQ: Merchant ID + Flyer Type ID, **Task Type = "Task" only**). Check the constructed Flyer Run URL is not in the dedupe file. |
| Extra rows (should have been excluded) | Dedupe URL didn't match exactly | Compare `https://fadmin.flippback.com/flyer_runs/{Flyer Run ID}` against dedupe file's `Flyer Run (url)` column character-for-character |
| Duplicate rows in output | FMQ context has duplicate Task-type rows for same Merchant + Flyer Type | Check FMQ context file for accidental duplicates |
| Wrong column order in output | `PS_HEADERS` / `FMQ_HEADERS` array was reordered | NEVER reorder these arrays — ClickUp expects exact order |
| Encoding / character issues (accents, em-dashes) | PapaParse unparse charset / quoting | `downloadCSV` lines 736-743; may need explicit BOM or quote settings |

---

## Things That Look Like Bugs But Aren't

Filter these out before reaching for a fix.

1. **`Coordinator (drop down)` → `Processor (drop down)`** is intentional. The PS context file has "Coordinator (drop down)"; the output column is "Processor (drop down)". They're the same field with different names. Don't "fix" the mapping.
2. **`End date` with lowercase 'd'** is intentional. FAdmin's column header literally has lowercase d. Don't change to "End Date".
3. **`Valid Date` differs from `Live Date`** is expected. They're different columns in FAdmin and can have different values for the same row. Don't combine or substitute.
4. **Output dates have no leading zeros** (`4/3/2026`, not `04/03/2026`). Intentional — this is what ClickUp accepts. Don't reformat.
5. **Rows that fail context lookup are skipped.** The "noContext" warning count is informational — many FAdmin rows are for retailers in the OTHER workflow, that's expected.
6. **`PS_HEADERS` / `FMQ_HEADERS` arrays and the row-construction objects are separate blocks.** Both must be edited together when changing columns.
7. **FMQ filters `Task Type = "Task"` only.** Subtasks and section rows are excluded by design. If FMQ output is missing a merchant, this filter is the first thing to check (the merchant might exist in the context file as a subtask row).
8. **FMQ `Preview Date` blank for some merchants** is intentional. When `Preview Days` is 0 or unset in the FMQ context file, `Preview Date` outputs blank. (`Due Date` is **not** blanked — it falls back to Live Date − 1; see the FMQ Due Date rule.) Don't treat a blank Preview Date as a bug — check the context file for that merchant's Preview Days value.
9. **FMQ `FADMIN Merchant Page (url)` / `FTP Path (url)` are constructed from Merchant ID, not read from context.** They will be populated on every FMQ row even when the Merchant Information Database leaves those fields blank. This is intentional (round 2 feedback fix). Renaming those context columns does not affect FMQ output. PS mode still reads them from its context file.

---

## Verifying Changes

No example files exist in the repo. The verification path:

1. **Trace the change mentally first.** Find the line(s) being changed. Read surrounding logic. Does it touch only the symptom column, or a shared path (e.g., date math used by every row)?
2. **Consider blast radius.**
   - PS uses one context row per `Flyer Type ID` — a code change at a row-construction line affects every retailer using that flyer type.
   - FMQ uses one context row per `Merchant ID + Flyer Type ID` — narrower scope.
   - Always ask: "could this fix break things for retailer Y while fixing retailer X?"
3. **For data-side issues, do not change code.** Push back: "ClickUp's context export shows value X for retailer Y. If that's wrong, fix it in ClickUp. The code is reading correctly."
4. **Push the fix to `main`.** The user re-runs the tool with their real exports and verifies the specific cells/rows from feedback. The user's re-test is the verification.

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

**Dedupe logic:** The tool constructs a Flyer Run URL for each FAdmin row (`https://fadmin.flippback.com/flyer_runs/{Flyer Run ID}`). If that URL already exists in the `Flyer Run (url)` column of the dedupe export, the row is skipped.

**Output:** One import-ready CSV per run. Column headers and column order must exactly match the authoritative formats (documented below).

---

## FAdmin Export — Columns Used

| Column Name | Purpose |
|---|---|
| `Flyer Run ID` | Constructs the Flyer Run URL; used as dedupe key |
| `Merchant ID` | FMQ join key; FMQ also constructs `FADMIN Merchant Page (url)` + `FTP Path (url)` from it |
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
| Preview Date | `Live Date − Preview Days` (FMQ: blank when Preview Days is 0/unset) |
| Due Date | `Live Date − Preview Days − 1` (i.e., Preview Date − 1; when Preview Days is 0/unset this is Live Date − 1). **FMQ Due Date is never blank.** |
| Valid Date | FAdmin `Valid Date` column — passthrough only |
| Live/Preview Date (FMQ) | FAdmin `Live Date` column — passthrough only |
| End Date (FMQ) | FAdmin `End date` column — passthrough only |

**Verified example:** Fresh Thyme Market — Preview Days=1, Live Date=Feb 3 → Preview Date=Feb 2, Due Date=Feb 1.

---

## Processing Support (PS) — Transformation Rules

### Join Key
FAdmin `Flyer Type ID` → PS Context `Flyer Type ID (short text)`

One context row per Flyer Type. If no match is found, the FAdmin row is skipped (expected for retailers in other workflows).

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

### Task Name
`{Merchant Name} - {Flyer Run Name}`

Direct concatenation with ` - ` separator. Source is FAdmin `Flyer Run Name` column — same pattern as PS mode. If `Flyer Run Name` is blank, output is just the merchant name.

### Subtasks
Comma-separated `Task Name` values of all `Flyer Subtask` rows in the context file whose `Parent ID` matches the matched Task row's `Task ID`. Empty if none.

### Output — 22 Columns (exact order)

| # | Output Header | Source |
|---|---|---|
| 1 | `Task Name` | `{Merchant Name} - {Flyer Run Name}` |
| 2 | `Merchant Name` | FAdmin `Merchant Name` |
| 3 | `Subtasks` | Comma-separated Flyer Subtask names from FMQ Context (see above) |
| 4 | `Flyer Run ID` | FAdmin `Flyer Run ID` |
| 5 | `Flyer Type ID` | FAdmin `Flyer Type ID` |
| 6 | `Flyer Run Link` | `https://fadmin.flippback.com/flyer_runs/{Flyer Run ID}` |
| 7 | `Start Date` | Live Date − Assets Check Days → M/D/YYYY |
| 8 | `Live Date` | FAdmin `Live Date` → M/D/YYYY (passthrough) |
| 9 | `Preview Date` | Live Date − Preview Days → M/D/YYYY; **blank if Preview Days is 0 or unset** |
| 10 | `Valid Date` | FAdmin `Valid Date` → M/D/YYYY (passthrough) |
| 11 | `Due Date` | Live Date − Preview Days − 1 → M/D/YYYY. **Never blank** (= Live Date − 1 when Preview Days is 0/unset; = Preview Date − 1 otherwise) |
| 12 | `End Date` | FAdmin `End date` → M/D/YYYY (passthrough) |
| 13 | `Oneguide` | FMQ Context `OneGuide (url)` |
| 14 | `Task Description` | *(empty)* |
| 15 | `Time Estimate` | FMQ Context `Time Estimate` |
| 16 | `Segment (drop down)` | FMQ Context `Segment (drop down)` |
| 17 | `Parent Banner (drop down)` | FMQ Context `Parent Banner (drop down)` |
| 18 | `Flyer Cadence (drop down)` | FMQ Context `Flyer Cadence (drop down)` |
| 19 | `Flyer Review Guide (url)` | FMQ Context `Flyer Review Guide (url)` |
| 20 | `Category (drop down)` | FMQ Context `Category (drop down)` |
| 21 | `FADMIN Merchant Page (url)` | **Constructed:** `https://fadmin.flippback.com/merchants/{Merchant ID}/dashboard` (FAdmin `Merchant ID`, not context — see note below) |
| 22 | `FTP Path (url)` | **Constructed:** `https://fadmin.flippback.com/merchants/{Merchant ID}/ftp_files` (FAdmin `Merchant ID`, not context) |

**Why constructed, not pulled from context:** the Merchant Information Database leaves these two fields blank on ~75% of Task rows, but both URLs are a fixed FAdmin URL scheme keyed by Merchant ID (verified 370/370 + 366/370 against populated rows; the 4 exceptions were data-entry errors in ClickUp). Constructing from FAdmin `Merchant ID` populates every row and is immune to ClickUp data hygiene. PS mode still pulls these from its context file — only FMQ constructs them.

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

These assumptions were confirmed by reverse-engineering the original FAdmin and ClickUp exports during V1 build. If any of these change in the upstream systems, the tool will need to be updated.

1. **FAdmin column names are stable.** The tool reads columns by name. Any rename in the FAdmin export format will break the affected transformation silently or cause a validation error.
2. **ClickUp column names are stable.** Context and dedupe files are read by column name. Validation catches changes to critical join-key columns; non-critical column renames cause silent blank values in the output.
3. **ClickUp export must use "All Columns."** Exports without this enabled will fail validation.
4. **FMQ context: Task-type rows only.** The Merchant Information Database export contains rows of multiple types (Task, subtask, section). Only `Task` rows are used.
5. **FAdmin date format is MM/DD/YYYY.** The tool parses by splitting on `/` and assuming month/day/year order. Format change → all date calculations break.
6. **PS Processor source field:** PS context has `Coordinator (drop down)`; output column is `Processor (drop down)`. Same field, different names.
7. **Valid Date can differ from Live Date.** PS output `Valid Date` is a passthrough from FAdmin `Valid Date` — not the same as Live Date.
8. **FAdmin `End date` has a lowercase 'd'.** Read exactly as-is.
9. **Output column order is fixed.** ClickUp's CSV import maps by header name, but the file order should match the authoritative format to avoid mapping issues.

---

## Development Workflow

```bash
git clone https://github.com/aascroft/fadmin-schedule-sync
cd fadmin-schedule-sync

# Edit index.html directly
# Test locally by opening index.html in a browser (no server needed)

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
