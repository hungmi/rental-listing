---
shaping: true
---

# A6 Spike: Listing Page

## Context

Shape A currently shows a single property when `?id=xxx` is provided. When no `id` is given, it shows an error ("缺少房源編號"). R9 requires a public listing page where clients can browse all available properties with rich preview cards.

## Goal

Understand how to add a listing page to the existing single-page architecture — what changes, what's reused, and what design decisions need answering.

## Questions

### Architecture

| # | Question | Answer |
|---|----------|--------|
| **a6-Q1** | Should the listing be a separate HTML file or the same `index.html`? | Same `index.html`. The `main()` function already checks for `?id=`. Currently no-id shows an error. Change: no-id shows the listing instead. This keeps deployment simple (single file) and reuses the CSV fetch, i18n, and config. |
| **a6-Q2** | Can we reuse the existing `fetchSheetCSV()` and CSV data? | Yes. The listing page fetches the same CSV. Each row becomes a card. No new data source needed. |
| **a6-Q3** | Does the listing page need the sticky footer CTA? | No. The CTA ("contact agent") makes sense on a single property page where the client has decided to inquire. On the listing page, the action is "browse and pick a property." CTA appears only on the detail page. |

### Card Design

| # | Question | Answer |
|---|----------|--------|
| **a6-Q4** | What info goes on each listing card? | From the Sheet row: `photo_1` as thumbnail, `rent` (formatted), `address_zh`/`address_en` (bilingual), `size_ping`, `vacancy_start ~ vacancy_end`. This gives clients enough to decide which property to click into. |
| **a6-Q5** | How does the card link to the detail page? | Each card is an `<a>` linking to `?id=xxx&lang=currentLang`. Clicking navigates to the detail view within the same page (could be a full reload via `<a href>`, or client-side routing). |
| **a6-Q6** | What if a property has no photo? | Show a placeholder background (grey with a house icon), same as the current carousel-empty pattern. |

### Layout

| # | Question | Answer |
|---|----------|--------|
| **a6-Q7** | Mobile layout for listing? | Single-column stack of cards (same max-width 480px container). Each card is a horizontal layout: thumbnail on left (square, ~100px), text info on right. This fits naturally on phone screens and is scannable. |
| **a6-Q8** | Desktop layout? | Same single-column, max-width 480px. This site is mobile-first and the narrow layout works fine on desktop. No need to add a grid. |

### Routing

| # | Question | Answer |
|---|----------|--------|
| **a6-Q9** | How does the routing change in `main()`? | Currently: no `id` → error. New: no `id` → render listing. With `id` → render detail (existing behavior). The branch point is in `main()`. |
| **a6-Q10** | Does the listing need a back button from the detail view? | Since cards use `<a href="?id=xxx">`, the browser back button naturally returns to the listing. No custom back button needed. |
| **a6-Q11** | Does the listing page need the language toggle? | Yes. The listing shows addresses and labels that are bilingual. The toggle should work the same way — and `&lang=` param should persist when clicking into a detail card. |

### Ordering & Filtering

| # | Question | Answer |
|---|----------|--------|
| **a6-Q12** | How should properties be sorted? | Sheet row order (agent controls this by reordering rows in Google Sheets). No client-side sort for now. |
| **a6-Q13** | Should we filter out properties with no vacancy? | Undecided — depends on the agent's preference. Could add an `active` column in the Sheet, or filter by `vacancy_end` being in the future. Worth asking the user. |

### i18n

| # | Question | Answer |
|---|----------|--------|
| **a6-Q14** | What new i18n strings are needed? | `listing_title` (e.g., "可租房源" / "Available Properties"), `size_suffix` for cards (already have `size_unit`), `vacancy_label` for cards (already exists). Minimal additions. |

## Impact on Existing Code

| Area | Change needed |
|------|---------------|
| `main()` | Branch: no `id` → `renderListing(data)` instead of `showError()` |
| HTML | Add a `#listing` container (hidden by default, shown when no id) |
| CSS | Add listing card styles (`.listing-card`, `.listing-thumb`, etc.) |
| JS | Add `renderListing(data)` function |
| i18n | Add `listing_title` to `I18N.zh` and `I18N.en` |
| Language toggle | Move toggle outside `#property` so it's visible on both listing and detail views |
| Sticky footer | Only show on detail view (already the case — `renderProperty()` shows it) |

## Acceptance

Spike is complete when we can describe the concrete changes to `index.html` to add the listing page, including routing logic, card layout, and i18n additions.

**Status: Complete.**

## Open Question for User

**a6-Q13**: Should properties with past vacancy dates (already rented) still appear in the listing? Options:
- **A)** Show everything in the Sheet — agent removes rows manually when rented
- **B)** Add an `active` column (`TRUE`/`FALSE`) — agent marks properties as active/inactive
- **C)** Auto-filter by `vacancy_end` — only show properties where vacancy hasn't passed yet
