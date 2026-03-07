---
shaping: true
---

# V1: Core Property Page + CTA — Implementation Plan

## Goal

Client opens a link like `?id=daan-3F` on their phone, sees formatted property info (rent, vacancy, discount notice, address, size), and taps LINE/WhatsApp to contact the agent.

## Deliverables

1. `index.html` — single-page app, mobile-first, hosted on GitHub Pages
2. A sample Google Sheet — published to web, with 2-3 test properties
3. Working end-to-end: URL → fetch CSV → render → CTA buttons

---

## Steps

### Step 1: Create sample Google Sheet structure

Create a Google Sheet with these columns and 2-3 rows of test data:

| id | address_zh | address_en | size_ping | rent | vacancy_start | vacancy_end |
|----|-----------|-----------|-----------|------|--------------|------------|
| daan-3F | 台北市大安區復興南路一段100號3樓 | 3F, No.100, Sec.1, Fuxing S. Rd, Da'an, Taipei | 12 | 18000 | 2026/4/1 | 2026/5/1 |
| xinyi-5F | 台北市信義區信義路五段50號5樓 | 5F, No.50, Sec.5, Xinyi Rd, Xinyi, Taipei | 8 | 15000 | 2026/3/15 | 2026/4/15 |

Publish the sheet to web (File > Share > Publish to web > CSV format). Save the published CSV URL.

**This step is manual — I cannot do it programmatically. I will use a mock CSV for development and testing.**

### Step 2: Create `index.html` with mobile-first layout

Single HTML file containing all CSS and JS inline (no build tools, no dependencies beyond what's needed).

**Structure (top to bottom):**

```
+---------------------------+
|   Property title/address  |
+---------------------------+
|   Rent: NT$ 18,000 / 月   |
|   空房: 2026/4/1 ~ 5/1    |
|   折扣: 簽約一年以上...     |
+---------------------------+
|   Address (full)          |
|   Size: 12 坪             |
+---------------------------+
|  [LINE]     [WhatsApp]    |  <- sticky footer
+---------------------------+
```

**CSS approach:**
- Mobile-first: `max-width: 480px` centered card, full-width on small screens
- System font stack (no external fonts to load)
- Sticky footer for CTA buttons (`position: fixed; bottom: 0`)
- Large tap targets (min 48px height for buttons)

### Step 3: Implement `parseQueryParams()`

```js
function parseQueryParams() {
  const params = new URLSearchParams(window.location.search);
  return {
    id: params.get('id'),
    lang: params.get('lang') || 'zh'  // default zh, used in V3
  };
}
```

Read `id` from URL. Store as `propertyId`. For V1, `lang` is read but only `zh` is rendered (bilingual rendering is V3).

### Step 4: Implement `fetchSheetCSV()`

```js
async function fetchSheetCSV(sheetURL) {
  const response = await fetch(sheetURL);
  const csvText = await response.text();
  return parseCSV(csvText);
}
```

Parse CSV into array of objects (keyed by header row). Use a simple CSV parser — no library needed for this structure (no commas in data fields expected, but handle quoted fields for addresses).

**CSV parser:** Split by newline, split first row as headers, map remaining rows to objects. Handle quoted fields (addresses may contain commas).

### Step 5: Implement `renderProperty(id)`

1. Find row where `id` column matches `propertyId`
2. If not found → call `handleNotFound()` → show error message
3. If found → populate DOM:
   - `address_zh` → address display
   - `rent` → formatted as `NT$ XX,XXX / 月` (with thousands separator)
   - `vacancy_start` + `vacancy_end` → `2026/4/1 ~ 2026/5/1`
   - Discount notice → hardcoded bilingual text (zh for V1)
   - `size_ping` → `XX 坪`

### Step 6: Implement CTA buttons

Two buttons in sticky footer:

- **LINE:** `<a href="https://line.me/ti/p/~LINE_ID">` — opens LINE chat
- **WhatsApp:** `<a href="https://wa.me/PHONE_NUMBER">` — opens WhatsApp chat

Both use `<a>` tags (not JS) for maximum mobile compatibility. Agent's LINE ID and phone number are configured as constants at the top of the script.

**Placeholder values for development:** `LINE_ID` and `PHONE_NUMBER` will be placeholder strings that the agent replaces with real values.

### Step 7: Implement `handleNotFound()`

Hide the property card, show a simple error message: "找不到此房源 / Property not found". Include a note to contact the agent if they believe this is an error.

---

## Testing Plan

I will test each part locally without needing a real Google Sheet by using a mock CSV.

### Test 1: Mock CSV fetch + parse

Create a local `test-data.csv` file with 2-3 rows of test data. Serve it via a local HTTP server (`python3 -m http.server`). Verify:

- [ ] CSV is fetched and parsed into correct objects
- [ ] Headers map correctly to object keys
- [ ] Addresses with commas in quoted fields parse correctly

### Test 2: Property rendering — happy path

Open `index.html?id=daan-3F` in browser. Verify:

- [ ] Address displays correctly (台北市大安區...)
- [ ] Rent shows `NT$ 18,000 / 月` (with comma formatting)
- [ ] Vacancy shows `2026/4/1 ~ 2026/5/1`
- [ ] Discount notice shows `簽約一年以上享有折扣，幅度面議`
- [ ] Size shows `12 坪`

### Test 3: Property not found

Open `index.html?id=nonexistent`. Verify:

- [ ] Property card is hidden
- [ ] Error message "找不到此房源" is displayed

### Test 4: No ID parameter

Open `index.html` with no query params. Verify:

- [ ] Error/prompt is shown (same as not found, or a "please use a valid link" message)

### Test 5: CTA buttons

On the happy path page, verify:

- [ ] LINE button is visible in sticky footer
- [ ] WhatsApp button is visible in sticky footer
- [ ] LINE button href is `https://line.me/ti/p/~LINE_ID_PLACEHOLDER`
- [ ] WhatsApp button href is `https://wa.me/PHONE_PLACEHOLDER`
- [ ] Buttons are large enough to tap (>= 48px)

### Test 6: Mobile layout

Open in browser with mobile device simulation (Chrome DevTools, 375px width). Verify:

- [ ] Card is full-width, no horizontal scroll
- [ ] Text is readable without zooming
- [ ] Sticky footer stays at bottom
- [ ] All content is accessible by scrolling (footer doesn't overlap content)

### Test 7: Second property

Open `index.html?id=xinyi-5F`. Verify:

- [ ] Different data renders (rent 15,000, different address, etc.)
- [ ] Confirms the system is data-driven, not hardcoded

---

## Files to create

| File | Purpose |
|------|---------|
| `index.html` | The single-page app (HTML + inline CSS + inline JS) |
| `test-data.csv` | Mock data for local testing (2-3 properties) |

---

## Configuration (agent fills in later)

```js
const CONFIG = {
  SHEET_CSV_URL: 'https://docs.google.com/spreadsheets/d/e/SHEET_ID/pub?output=csv',
  LINE_ID: 'your_line_id',
  PHONE_NUMBER: '886912345678',
};
```

Agent replaces these three values after creating the Google Sheet and before deploying to GitHub Pages.

---

## Out of scope for V1

- Photos / carousel (V2)
- Equipment checklist (V2)
- Language toggle / bilingual rendering (V3)
- `&lang=en` URL param rendering (V3) — param is parsed but not used
