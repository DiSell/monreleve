# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**MonRélevé** (TimePro) is a client-side-only Progressive Web App for French interim workers to create, manage, and export weekly hour reports (relevés d'heures). There is no backend, no build step, and no package manager — the entire application is a single HTML file.

## Running the App

There is no build or install step. Serve the static files directly:

```bash
# Python (any machine with Python 3)
python -m http.server 8000
# Then open http://localhost:8000/monreleve/
```

Or open `monreleve/index.html` directly in a browser (some features like Web Share API require HTTPS/localhost).

## Architecture

### Single-file SPA

All HTML, CSS, and JavaScript live in [monreleve/index.html](monreleve/index.html) (~3,000 lines). There are no external JS/CSS modules — only SVG assets alongside the file.

External libraries are loaded via CDN at runtime:
- **PDF.js v3.11.174** — extracts text from imported interim contract PDFs
- **jsPDF v2.5.1 + jsPDF-AutoTable v3.5.28** — generates the final export PDF
- **Leaflet v1.9.4** — interactive map for calculating mileage (depot ↔ jobsite)

### Screen Flow

Navigation is screen-based: each "screen" is a `<div id="s_*">` toggled visible by `showScreen(id)`. The user flow is:

```
s_rgpd (first launch / RGPD consent)
  └─ s_upload (dashboard / PDF import)
       └─ s0 → s1_ag → s2_hr → s3_cl → s4_sig → s5_fb → s6_ok
s_month (monthly history, accessible from dashboard)
s_legal (legal/data deletion page)
```

### Key Subsystems

| Subsystem | Entry point(s) | Description |
|---|---|---|
| PDF import | `handlePDF()` | Uses PDF.js to extract text from Adecco/Manpower/Start People contracts via regex |
| Timesheet entry | Screen `s2_hr` | Daily AM/PM hour pairs, quick-fill templates, overtime at 35h/43h thresholds |
| Supplements | `calcSupplements()` | IKM mileage (URSSAF or BTP zone rates), meal allowances, flat bonuses |
| Mileage map | Leaflet integration in `s2_hr` | Depot ↔ jobsite address geocoding + distance calc |
| Signature | Screen `s4_sig` | Canvas-based freehand signature capture |
| PDF export | `exportPDF()` | Generates A4 PDF with hours table, supplements, salary estimate, and signature boxes |
| Data persistence | `localStorage` | `timepro_weeks` (all submissions), `timepro_draft` (auto-saved in-progress form, debounced 800ms) |
| Monthly history | `buildMonthView()` | Aggregates `timepro_weeks` by month with collapsible week breakdowns |

### Data Model (localStorage)

- `timepro_weeks`: JSON array of submitted week objects (hours, supplements, contract info, signature)
- `timepro_draft`: JSON of the current in-progress form, auto-saved on input change

### Optional Cloud Sync

A Supabase integration exists in the code but is disabled by default (commented config ~line 1216). It is not wired to the active data flow.

## Deployment

Copy `monreleve/` to any HTTP host. The app requires HTTPS in production for:
- `navigator.share` (Web Share API for PDF sharing on mobile)
- Consistent localStorage behavior on iOS Safari

GitHub remote: `https://github.com/DiSell/monreleve.git` (branch: `master`)

## Key Constraints

- **French locale throughout** — all UI text, legal disclosures, and business rules (URSSAF thresholds, BTP zones) are French-specific.
- **No framework** — DOM manipulation is entirely vanilla JS. Do not introduce a framework or bundler.
- **Privacy-first** — data never leaves the device by default. The RGPD screen and `s_legal` legal text are legally required disclosures (editor: Didier Sellin / ProactifSystem, SIREN 510 749 682).
- **Single-file constraint** — keep all code in `index.html` unless there is a compelling reason to split. New assets (icons, etc.) go alongside the file.
