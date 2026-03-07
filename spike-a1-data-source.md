---
shaping: true
---

# A1 Spike: Data Source for Non-Technical Property Updates

## Context

Shape A deploys rental property pages on GitHub Pages (static). The agent manages ~50 properties and cannot edit HTML. A1 is the mechanism that lets the agent update property data without coding. Three candidate approaches have been identified:

- **A1-a**: JSON/YAML files in the repo + build step
- **A1-b**: Google Sheet as data source + build step generates static pages
- **A1-c**: Single HTML page + client-side JS reads Google Sheet at runtime

## Goal

Understand the concrete mechanics, trade-offs, and constraints of each A1 option so we can pick one.

## Questions

### A1-a: JSON/YAML + Build

| # | Question | Answer |
|---|----------|--------|
| **a-Q1** | What does the editing workflow look like? | Agent edits a JSON or YAML file (one file per property, or one big file). Each entry contains: rent, vacancy dates, discount rule, address, size, photos, equipment. Agent commits & pushes to GitHub. GitHub Actions runs a build script (e.g., a simple Node/Python script or static site generator) that produces one HTML page per property. |
| **a-Q2** | What tools does the agent need to learn? | A text editor + basic Git (commit, push). Alternatively, can edit the file directly on GitHub.com's web UI — no local tools needed. |
| **a-Q3** | How are photos handled? | Photos are committed to the repo (e.g., `/images/property-id/`) or hosted externally (e.g., Imgur, Google Drive public links). JSON/YAML references the image paths/URLs. |
| **a-Q4** | What are the failure modes? | Malformed JSON/YAML breaks the build. Agent may not understand the error. No validation UI. |
| **a-Q5** | How hard is initial setup? | Moderate. Need: repo structure, build script or SSG config, GitHub Actions workflow, page template. One-time setup by a developer. |

### A1-b: Google Sheet + Build Step

| # | Question | Answer |
|---|----------|--------|
| **b-Q1** | What does the editing workflow look like? | Agent opens a Google Sheet, fills in columns (address, rent, vacancy start/end, discount %, equipment, photo URLs). A build step (GitHub Actions on schedule or manual trigger) reads the sheet via Google Sheets API, generates one HTML page per row, deploys to GitHub Pages. |
| **b-Q2** | What tools does the agent need to learn? | Google Sheets only. This is the lowest technical barrier — most people already know spreadsheets. |
| **b-Q3** | How are photos handled? | Agent uploads photos somewhere accessible (Google Drive public link, Imgur, etc.) and pastes the URL into the sheet. Build script embeds those URLs in `<img>` tags. |
| **b-Q4** | What are the failure modes? | Column order change breaks the build. Typos in URLs lead to broken images but not build failures. API quota limits (unlikely at 50 properties). Google Sheets API requires a service account key stored as GitHub secret. |
| **b-Q5** | How hard is initial setup? | Moderate-high. Need: Google Cloud project + service account, Sheet template, build script, GitHub Actions workflow, page template. One-time setup by a developer. |
| **b-Q6** | How does the agent trigger a rebuild? | Option 1: Scheduled (e.g., every hour). Option 2: Agent clicks "Run workflow" on GitHub Actions (requires GitHub account). Option 3: A webhook/bookmarklet that triggers the action. |

### A1-c: Client-Side JS Reads Google Sheet

| # | Question | Answer |
|---|----------|--------|
| **c-Q1** | What does the editing workflow look like? | Same as A1-b — agent edits Google Sheet. But no build step. A single HTML page (or a set of static pages) uses JavaScript to fetch data from the published Google Sheet (CSV export URL or Google Sheets API) and renders it client-side. |
| **c-Q2** | What tools does the agent need to learn? | Google Sheets only. Same low barrier as A1-b. |
| **c-Q3** | How does the URL-per-property work? | Use URL query parameters or hash routing (e.g., `site.github.io/?id=property-3` or `site.github.io/#property-3`). A single `index.html` reads the parameter, fetches the matching row from the sheet, and renders it. |
| **c-Q4** | What are the failure modes? | Google Sheets public CSV endpoint can be slow or rate-limited. Client sees a loading spinner. If Google changes the CSV export format, the parser breaks. No SEO (but irrelevant — links are sent privately, not indexed). |
| **c-Q5** | How hard is initial setup? | Low. Need: one HTML file with JS, a published Google Sheet. No build step, no GitHub Actions, no service account. Simplest infrastructure. |
| **c-Q6** | Does the data update immediately? | Yes. When the agent edits the sheet and a client loads the page, they see the latest data. No build/deploy lag. |
| **c-Q7** | Can this work on GitHub Pages? | Yes. GitHub Pages serves the static HTML/JS. The JS fetches data from Google Sheets at runtime. No server needed. |

## Comparison

| Dimension | A1-a (JSON/YAML) | A1-b (Sheet + Build) | A1-c (Sheet + Client JS) |
|-----------|:-:|:-:|:-:|
| Agent editing skill | Text file + Git | Google Sheets | Google Sheets |
| Setup complexity | Moderate | Moderate-high | Low |
| Build step needed | Yes | Yes | No |
| Data updates immediately | No (needs build) | No (needs build) | Yes |
| Infrastructure | GitHub Actions | GitHub Actions + Google API key | None beyond GitHub Pages |
| Failure risk for agent | JSON/YAML syntax errors | Column changes break build | Minimal |
| Photo handling | Same across all — external URLs | Same | Same |
| URL per property | File-based routing (`/property-3/`) | File-based routing (`/property-3/`) | Query param (`?id=property-3`) |

## Acceptance

Spike is complete when we can describe the editing workflow, setup cost, and failure modes for each A1 option, and can make an informed choice.

**Status: Complete.**

## Recommendation for Discussion

**A1-c (Client-Side JS + Google Sheet)** appears to be the strongest fit:

- Lowest setup complexity (no build pipeline, no API keys)
- Lowest ongoing friction for the agent (edit sheet, done)
- Immediate data updates (no rebuild delay)
- The trade-off (query-param URLs instead of path-based) is acceptable since links are sent privately, not shared publicly

The main risk is dependency on Google Sheets' public CSV endpoint stability, which is well-established and widely used.
