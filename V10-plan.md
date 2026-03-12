---
shaping: true
---

# V10 Implementation Plan: Property Description with Multiline CSV Support

## Overview

Two changes in one slice:
1. **A1.3** — Fix `parseCSV()` to handle quoted fields with embedded newlines
2. **A8** — Add `description_zh`/`description_en` columns + render on detail page

## Steps

### Step 1: Fix `parseCSV()` for multiline quoted fields

**File:** `index.html` (line ~694)

**Current code:**
```js
function parseCSV(csvText) {
  const lines = csvText.trim().split('\n');
  // ...iterates lines, calls parseCSVLine() per line
}
```

**Problem:** `split('\n')` breaks multiline cells into separate rows.

**Change:** Replace the naive `split('\n')` with a quote-aware row splitter:
- Walk through `csvText` character by character
- Track `inQuotes` state (toggled by `"`)
- Only treat `\n` as a row boundary when `inQuotes === false`
- Handle `""` (escaped quotes) inside quoted fields
- Feed each complete row to `parseCSVLine()` as before

`parseCSVLine()` remains unchanged — it already handles quotes within a line.

### Step 2: Add description HTML section

**File:** `index.html` — HTML structure (after equipment section, line ~610, before closing `</div>` of `#property`)

Insert a new card for the description, placed **after** the detail card (address/size/equipment) and **before** the closing `</div>` of `.property-content`:

```html
<!-- Description -->
<div class="card" id="description-card" style="display:none;">
  <div class="description" id="description"></div>
</div>
```

Hidden by default. Shown only when description text exists.

### Step 3: Add description CSS

**File:** `index.html` — CSS section

```css
.description {
  padding: 16px;
  font-size: 14px;
  line-height: 1.7;
  color: #444;
}

.description p {
  margin: 0 0 12px 0;
}

.description p:last-child {
  margin-bottom: 0;
}

.description ul {
  margin: 0 0 12px 0;
  padding-left: 20px;
  list-style: disc;
}

.description li {
  margin-bottom: 4px;
}

.description strong {
  display: block;
  margin-top: 12px;
  margin-bottom: 4px;
  color: #222;
  font-size: 15px;
}
```

### Step 4: Add `renderDescription(row)` function

**File:** `index.html` — JS section, before `renderProperty()`

**Logic:** Convert plain text → HTML using simple rules:

```
Input text (from Google Sheet cell):
─────────────────────────────────
Chic Shared Living near Nangang Station
Looking for more than just a room?

Why You'll Love It:
* Massive sun-drenched living room
* Fully equipped kitchen
* 2 bathrooms

Monthly Rent:
Room C: NT$ 9,500
```

**Conversion rules:**
1. Split text on `\n`
2. Group consecutive lines into blocks separated by blank lines
3. For each line:
   - **Blank line** → start new `<p>`
   - **Line starting with `*` or `·`** → `<li>` (collect consecutive bullet lines into `<ul>`)
   - **Line ending with `:`** → `<strong>line</strong>`
   - **Otherwise** → append to current `<p>` with `<br>`

**Function:**
```js
function renderDescription(row) {
  const descKey = 'description_' + currentLang;
  const text = (row[descKey] || row.description_zh || '').trim();
  const card = document.getElementById('description-card');

  if (!text) {
    card.style.display = 'none';
    return;
  }

  const lines = text.split('\n');
  let html = '';
  let inList = false;

  for (const line of lines) {
    const trimmed = line.trim();

    if (!trimmed) {
      // Blank line: close any open list, start new paragraph
      if (inList) { html += '</ul>'; inList = false; }
      html += '</p><p>';
      continue;
    }

    if (trimmed.startsWith('*') || trimmed.startsWith('·')) {
      // Bullet item
      if (!inList) { html += '<ul>'; inList = true; }
      html += '<li>' + escapeHTML(trimmed.replace(/^[*·]\s*/, '')) + '</li>';
      continue;
    }

    // Close list if we were in one
    if (inList) { html += '</ul>'; inList = false; }

    if (trimmed.endsWith(':')) {
      // Heading line
      html += '<strong>' + escapeHTML(trimmed) + '</strong>';
    } else {
      // Normal text line
      html += escapeHTML(trimmed) + '<br>';
    }
  }

  if (inList) html += '</ul>';

  // Wrap in <p>, clean up empty <p> tags
  html = '<p>' + html + '</p>';
  html = html.replace(/<p><\/p>/g, '');

  document.getElementById('description').innerHTML = html;
  card.style.display = '';
}
```

Need an `escapeHTML()` helper (if not already present) to prevent XSS from sheet content:
```js
function escapeHTML(str) {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}
```

### Step 5: Wire `renderDescription()` into `renderProperty()`

**File:** `index.html` — `renderProperty()` function (line ~1194)

After `renderEquipment(row);`, add:
```js
renderDescription(row);
```

This ensures description re-renders on language toggle (since `switchLanguage()` calls `renderProperty()`).

### Step 6: Update test data

**File:** `test-data.csv`

Add `description_zh` and `description_en` columns to the header and populate at least one row with multiline test data (using actual newlines inside quoted fields).

Test cases in CSV:
- **Row 1 (daan-3F):** Full description with headings, bullets, paragraphs in both ZH and EN
- **Row 2 (xinyi-5F):** EN only (ZH empty — tests fallback)
- **Row 3 (zhongshan-2F):** Both empty (tests hidden state)
- **Row 4 (expired-1F):** Both empty

---

## Testing Plan

### Test 1: CSV parser handles multiline fields

**Method:** Modify `test-data.csv` to have multiline description fields. Open `index.html` with `?id=daan-3F` using local file or simple HTTP server.

**Verify:**
- Page loads without errors (no broken row parsing)
- All 4 properties still parse correctly (check listing page shows 3 active cards)
- Description text appears on detail page with correct content

**Edge cases:**
- Cell with double quotes inside: `She said ""hello""` → renders as `She said "hello"`
- Cell with commas inside quoted field: `"Room A, Room B"` → single field, not split
- Empty description cells → no breakage

### Test 2: Description renders with correct formatting

**Method:** Open `?id=daan-3F` and inspect the description section.

**Verify:**
- Heading lines (ending with `:`) render as `<strong>` (bold)
- Bullet lines (starting with `*`) render as `<ul><li>` list
- Blank lines create paragraph breaks
- Normal text lines separated by `<br>`
- No raw HTML visible (escaping works)

### Test 3: Bilingual toggle works

**Method:** On detail page, click language toggle.

**Verify:**
- Description switches between ZH and EN content
- All other fields (rent, address, equipment) still switch correctly
- URL updates with `&lang=en` / `&lang=zh`

### Test 4: Empty description hidden

**Method:** Open `?id=zhongshan-2F` (no description in test data).

**Verify:**
- No description card visible
- No empty card/whitespace
- Rest of page renders normally

### Test 5: Fallback when one language missing

**Method:** Open `?id=xinyi-5F&lang=zh` (has EN description but no ZH).

**Verify:**
- Falls back to ZH description if available, or shows nothing
- No JS errors in console

### Test 6: Listing page unaffected

**Method:** Open listing page (no `?id=`).

**Verify:**
- All cards render correctly
- Multiline descriptions in CSV don't break other fields
- Card count matches expected active properties

### Test 7: XSS prevention

**Method:** Put `<script>alert('xss')</script>` in a description cell in test data.

**Verify:**
- Text renders as literal `<script>alert('xss')</script>`
- No alert fires
- `escapeHTML()` works correctly
