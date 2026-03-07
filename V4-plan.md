---
shaping: true
---

# V4: Listing Page — Implementation Plan

## Goal

Client opens `hungmi.github.io/rental-listing/` (no `?id=`), sees a branded page "Jerry in the house" with rich property cards. Tapping a card navigates to the detail page. Properties with past vacancy dates are hidden. Empty state shows a message with CTA buttons.

## Deliverables

1. Updated `index.html` — listing view added alongside existing detail view
2. Updated `test-data.csv` — add `district_zh`, `district_en` columns, include one expired property for filter testing
3. Updated i18n — new strings for listing page

---

## Steps

### Step 1: Add `district_zh` / `district_en` columns to test data

Add two new columns to `test-data.csv`. Also add a 4th property with a past `vacancy_end` to test auto-filtering.

### Step 2: Add listing HTML structure

Add a new `#listing` container inside the HTML body (hidden by default, alongside existing `#property`):

```
+---------------------------+
| Jerry in the house    [EN]|  <- brand header + lang toggle
+---------------------------+
| [photo 4:3]               |
| NT$ 18,000 / 月           |
| 台北市大安區  ·  12坪      |
| 空房 2026/4/1 ~ 5/1       |
+---------------------------+
| [photo 4:3]               |
| NT$ 15,000 / 月           |
| 台北市信義區  ·  8坪       |
| 空房 2026/3/15 ~ 4/15     |
+---------------------------+
```

Key structural changes:
- Move language toggle outside `#property` — shared between listing and detail
- `#listing` is a sibling of `#property`, toggled by presence of `?id=`
- Listing does NOT show sticky footer CTA (only detail does)

### Step 3: Add listing CSS

- `.listing-header` — brand title + lang toggle row
- `.listing-card` — card with 4:3 thumbnail, text info below
- `.listing-card-thumb` — 4:3 aspect ratio, `object-fit: cover`
- `.listing-card-body` — padding, text layout
- `.listing-card-rent` — red, bold, same as detail
- `.listing-card-meta` — district + size on one line, grey
- `.listing-card-vacancy` — vacancy period
- `.listing-empty` — empty state container

### Step 4: Add i18n strings

```js
// zh
listing_title: 'Jerry in the house',
listing_empty: '目前沒有空房，請保持跟我聯絡！',
listing_vacancy: '空房',

// en
listing_title: 'Jerry in the house',
listing_empty: 'No vacancies at the moment. Stay in touch!',
listing_vacancy: 'Vacancy',
```

Brand name "Jerry in the house" stays the same in both languages.

### Step 5: Implement `filterActiveProperties(data)`

```js
function filterActiveProperties(data) {
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  return data.filter(row => {
    if (!row.vacancy_end) return false;
    const parts = row.vacancy_end.split('/');
    const endDate = new Date(parts[0], parts[1] - 1, parts[2]);
    return endDate >= today;
  });
}
```

Parse `vacancy_end` (format `YYYY/M/D`), compare to today. Keep only future dates.

### Step 6: Implement `renderListing(data)`

1. Call `filterActiveProperties(data)` → get active rows
2. If zero rows → show empty state (message + LINE/WhatsApp buttons)
3. If rows exist → for each row, call `buildCard(row)` and append to listing grid
4. Update page title, brand header, lang toggle text

### Step 7: Implement `buildCard(row)`

Creates a card `<a>` element linking to `?id=xxx&lang=currentLang`:

1. Thumbnail: `photo_1` as `<img>` with 4:3 aspect ratio. If no photo, show grey placeholder.
2. Rent: formatted `NT$ XX,XXX / 月` (reuse existing `formatRent` logic, adapted for bilingual unit)
3. District + Size: `district_zh` or `district_en` + ` · ` + `size_ping 坪`
4. Vacancy: `空房 2026/4/1 ~ 5/1`

### Step 8: Update `main()` routing

```js
// Current: no id → showError('no_id_title', 'no_id_msg')
// New: no id → renderListing(data)

if (!id) {
  const data = await fetchSheetCSV(config.SHEET_CSV_URL);
  document.getElementById('loading').classList.add('hidden');
  renderListing(data);
  return;
}
```

### Step 9: Move language toggle

Currently the toggle button is inside `#property`. Move it to a shared location:
- On listing: appears in the listing header row next to "Jerry in the house"
- On detail: appears above the carousel (existing position)

Implementation: keep one toggle button in the listing header. For detail view, add a separate toggle button in the detail header. Both call the same `switchLanguage()`. On language switch, re-render the current view (listing or detail).

### Step 10: Update `switchLanguage()` for listing

Currently `switchLanguage()` only re-renders `renderProperty(currentRow)`. Extend to also handle listing:

```js
function switchLanguage() {
  currentLang = currentLang === 'zh' ? 'en' : 'zh';
  const params = new URLSearchParams(window.location.search);
  params.set('lang', currentLang);
  history.replaceState(null, '', '?' + params.toString());

  if (currentRow) {
    renderProperty(currentRow);
  } else if (currentData) {
    renderListing(currentData);
  }
}
```

---

## Testing Plan

### Test 1: Listing renders with active properties

Serve locally, open `http://localhost:8000/` (no `?id=`). Verify:

- [ ] Brand title "Jerry in the house" is displayed
- [ ] Language toggle button is visible
- [ ] Property cards are rendered (one per active row)
- [ ] Each card shows: photo, rent, district, size, vacancy

### Test 2: Card content is correct

Check each card against test data. Verify:

- [ ] Photo thumbnail uses `photo_1` with 4:3 aspect ratio
- [ ] Rent shows `NT$ 18,000 / 月` (formatted with comma)
- [ ] District shows `台北市大安區` (not full address)
- [ ] Size shows `12 坪`
- [ ] Vacancy shows `2026/4/1 ~ 2026/5/1`

### Test 3: Auto-filter expired properties

Add a 4th row to test-data.csv with `vacancy_end` in the past (e.g. `2025/1/1`). Verify:

- [ ] Expired property does NOT appear in the listing
- [ ] Only 3 active properties are shown (or however many have future dates)

### Test 4: Card click navigates to detail

Click a property card. Verify:

- [ ] URL changes to `?id=xxx&lang=zh`
- [ ] Detail page renders correctly for that property
- [ ] Browser back button returns to listing

### Test 5: Empty state

Set all `vacancy_end` dates to past dates in test data. Verify:

- [ ] No property cards shown
- [ ] Message "目前沒有空房，請保持跟我聯絡！" is displayed
- [ ] LINE and WhatsApp buttons are visible and functional

### Test 6: Language toggle on listing

On listing page, click language toggle. Verify:

- [ ] Toggle text changes from "EN" to "中文"
- [ ] District switches from `台北市大安區` to `Da'an, Taipei`
- [ ] Rent unit switches from `/ 月` to `/ mo`
- [ ] Size unit switches from `坪` to `ping`
- [ ] Vacancy label switches from `空房` to `Vacancy`
- [ ] URL updates to `?lang=en`
- [ ] Brand title stays "Jerry in the house" (same in both languages)

### Test 7: Language persists to detail

On listing with `&lang=en`, click a card. Verify:

- [ ] Detail page opens with `?id=xxx&lang=en`
- [ ] Detail page renders in English

### Test 8: Mobile layout

Open listing in mobile simulation (375px width). Verify:

- [ ] Cards are full-width, no horizontal scroll
- [ ] Photos are clear, text is readable
- [ ] Cards are tappable (large enough touch target)

### Test 9: No photo placeholder

Have one test property with no `photo_1`. Verify:

- [ ] Card shows grey placeholder instead of broken image
- [ ] Card is still fully functional

### Test 10: Detail view unchanged

Open `?id=daan-3F` directly. Verify:

- [ ] Detail page works exactly as before (carousel, equipment, CTA, etc.)
- [ ] Language toggle still works on detail page
- [ ] Sticky footer CTA appears on detail (not on listing)

---

## Files modified

| File | Change |
|------|--------|
| `index.html` | Add listing HTML/CSS/JS, update routing in `main()`, move lang toggle |
| `test-data.csv` | Add `district_zh`, `district_en` columns, add expired test row |

---

## Out of scope for V4

- Search or filter by district/rent range
- Sort controls (client-side)
- Pagination (50 properties is manageable in a single scroll)
