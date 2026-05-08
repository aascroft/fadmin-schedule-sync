# fadmin-schedule-sync — Project Context

## ⚠️ First Session Instruction

**Do not start writing code yet.** The transformation rules that define the core logic of this tool have not been documented yet. Before building anything, ask the user to provide:

1. A sample FAdmin export CSV (the raw input)
2. A sample ClickUp import CSV for **Processing Support** mode
3. A sample ClickUp import CSV for **Flyer Management Queue** mode
4. Any notes on logic that isn't obvious from the CSV columns alone

Once you have reviewed those files and confirmed you understand the full column mapping and transformation logic for both modes, then proceed with building.

---

## What This Is

A browser-based tool that converts FAdmin schedule exports (CSV) into ClickUp-ready import files (CSV). The entire transformation runs client-side — no server, no data ever leaves the user's browser.

**Live URL (once deployed):** https://aascroft.github.io/fadmin-schedule-sync  
**Repo:** https://github.com/aascroft/fadmin-schedule-sync  
**Slack channel:** #fadmin-schedule-sync

---

## Background

Flipp's operations team manages retailer flyer campaigns in an internal system called **FAdmin**. Each retailer has a pre-built schedule of empty flyer campaigns. To manage the processing workload, the team exports this schedule from FAdmin as a CSV, manually transforms it (currently 1–2 full days of work), and imports the result into **ClickUp** for task management.

This tool eliminates that manual transformation step.

---

## Decisions Already Made

| Decision | Choice | Reason |
|---|---|---|
| Tool type | Static web app | No server needed, works in any browser, free hosting |
| Hosting | GitHub Pages (`aascroft` org) | Free, version-controlled, instant deploys on push to main |
| Repo visibility | Public | No sensitive data in the code; required for free Pages |
| Input | CSV drag-and-drop | FAdmin exports as CSV |
| Output | CSV download | ClickUp accepts CSV import via its UI |
| Processing | 100% client-side JS | Data never leaves the browser — privacy safe |
| CSV parsing | PapaParse library | Standard, well-tested browser CSV library |

---

## User

The end user is a non-developer operations team member at Flipp. He is reasonably technically comfortable but the UI should be dead simple — drop file, select mode, download output, done. The tool should also include built-in step-by-step instructions so he can reference the full end-to-end process (FAdmin export settings → tool → ClickUp import) without needing a separate document.

---

## Two Output Modes

The FAdmin export is always identical regardless of which process it's for. However, there are **two different ClickUp import formats** depending on the destination workflow. The user selects which one they need before (or alongside) dropping in the file.

| Mode | ClickUp Destination |
|---|---|
| Processing Support | Processing Support workflow in ClickUp |
| Flyer Management Queue | Flyer Management Queue workflow in ClickUp |

**UI pattern:** A selector (dropdown or prominent toggle) presented before or alongside the file drop zone. Each mode runs a different transformer function against the same input CSV and produces a different output CSV.

Label direction: "Import Type" or "What are you importing into?" — finalize during build.

---

## Current State

- [x] Repo created: `aascroft/fadmin-schedule-sync`
- [x] GitHub Pages hosting planned
- [ ] Transformation rules documented (pending meeting with team member)
- [ ] `index.html` built
- [ ] GitHub Pages enabled in repo settings
- [ ] End-to-end tested with real data
- [ ] Instructions panel written

---

## What's Needed Before Building

The transformation rules that map FAdmin CSV columns → ClickUp CSV columns are being gathered in a meeting with the team member. Once available, they will be added to this file under a **Transformation Rules** section. Rules are needed for **both output modes separately**.

Expect to receive:
- A sample FAdmin export CSV (raw input — same for both modes)
- A sample ClickUp import CSV for **Processing Support** (desired output, mode 1)
- A sample ClickUp import CSV for **Flyer Management Queue** (desired output, mode 2)
- Notes on the logic/modifications made during the manual process for each

---

## Architecture Plan

```
index.html               ← Single file: all HTML, CSS, JS inline
  ├── UI layer           ← Mode selector, drag-and-drop zone, status, download button
  ├── Parser             ← PapaParse reads the uploaded CSV
  ├── Transformer        ← Two functions:
  │                           transformProcessingSupport(rows)
  │                           transformFlyerQueue(rows)
  │                         Selected at runtime based on user's mode choice
  ├── Writer             ← PapaParse unparses output, triggers CSV download
  └── Instructions       ← Built-in step-by-step guide panel (per-mode instructions)
```

Single `index.html` file. No build step, no dependencies to install, no framework. Push to `main` → GitHub Pages serves it immediately.

---

## Development Workflow

```bash
# Clone
git clone https://github.com/aascroft/fadmin-schedule-sync
cd fadmin-schedule-sync

# Work on index.html directly
# Test by opening index.html in a browser (no server needed for local dev)

# Deploy = just push to main
git add .
git commit -m "your message"
git push origin main
# GitHub Pages picks it up within ~60 seconds
```

---

## GitHub Pages Setup (one-time, do once index.html exists)

1. Go to repo → Settings → Pages
2. Source: Deploy from a branch
3. Branch: `main` / `/ (root)`
4. Save — URL will be `https://aascroft.github.io/fadmin-schedule-sync`

---

## Transformation Rules

*To be added after the team meeting. This section will contain the full column mapping and transformation logic for both modes.*

### Processing Support
*Pending*

### Flyer Management Queue
*Pending*

---

## Design Direction

- Clean, minimal UI — white/light background, single clear action
- Mode selector is prominent — user must choose before they can process
- Drag-and-drop zone as the hero element
- Status feedback inline (e.g. "✓ 2,847 rows processed — ready to download")
- Instructions section below the tool (collapsible or tabbed, one tab per mode)
- Mobile-friendly but desktop-optimized (ops team uses desktop)
- Should look polished enough to showcase internally as an AI-built tool
