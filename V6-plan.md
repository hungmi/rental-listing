---
shaping: true
---

# V6: Migrate to unified media columns

## Goal

Replace `photo_1`–`photo_5` columns with `media_1`–`media_8` + `media_1_type`–`media_8_type`. Photos keep working identically. Video-type items are read but skipped in rendering (stub for V7).

## Parts

- A7.1 — Unified media columns
- A7.3 — Mixed carousel rendering (photo portion only)
- A7.5 — Listing card thumbnail (`getFirstPhoto()`)

## Changes

### 1. Add `getMediaItems(row)` — NEW

Insert after `transformPhotoUrl()` (~line 603).

```javascript
function getMediaItems(row) {
  var items = [];
  for (var i = 1; i <= 8; i++) {
    var url = row['media_' + i];
    if (url) {
      var type = (row['media_' + i + '_type'] || 'photo').trim().toLowerCase();
      items.push({ url: url, type: type });
    }
  }
  return items;
}
```

- Reads `media_1`–`media_8` and `media_1_type`–`media_8_type`
- Defaults type to `photo` if column is empty (backwards-compatible)
- Skips empty URLs
- Returns ordered array of `{url, type}` objects

### 2. Add `getFirstPhoto(row)` — NEW

Insert after `getMediaItems()`.

```javascript
function getFirstPhoto(row) {
  var items = getMediaItems(row);
  for (var i = 0; i < items.length; i++) {
    if (items[i].type === 'photo') return items[i].url;
  }
  return null;
}
```

- Scans media items, returns first photo-type URL
- Returns `null` if no photos (listing card shows placeholder emoji)

### 3. Modify `renderCarousel(row)` — line 776–780

**Before:**
```javascript
const photos = [];
for (let i = 1; i <= 5; i++) {
  const url = row['photo_' + i];
  if (url) photos.push(url);
}
```

**After:**
```javascript
const allItems = getMediaItems(row);
const photos = [];
for (var i = 0; i < allItems.length; i++) {
  if (allItems[i].type === 'photo') photos.push(allItems[i].url);
}
```

- Uses `getMediaItems()` to read unified columns
- Filters for `type === 'photo'` only (video rendering added in V7)
- Rest of `renderCarousel()` unchanged — same `photos` array, same slide/dot/swipe logic

### 4. Modify `buildCard(row)` — line 896

**Before:**
```javascript
if (row.photo_1) {
  const img = document.createElement('img');
  img.src = transformPhotoUrl(row.photo_1);
```

**After:**
```javascript
var firstPhoto = getFirstPhoto(row);
if (firstPhoto) {
  const img = document.createElement('img');
  img.src = transformPhotoUrl(firstPhoto);
```

- Uses `getFirstPhoto()` instead of hardcoded `row.photo_1`
- If `media_1` is a video, it skips to the first photo-type item
- Placeholder emoji still shown when no photos exist

### 5. Update `test-data.csv`

Replace `photo_1,photo_2,photo_3` header and values with `media_1,media_1_type,media_2,media_2_type,media_3,media_3_type`.

Row mappings (same URLs, new columns):

| Row | media_1 | media_1_type | media_2 | media_2_type | media_3 | media_3_type |
|-----|---------|-------------|---------|-------------|---------|-------------|
| daan-3F | Drive URL | photo | placehold.co Room 2 | photo | placehold.co Room 3 | photo |
| xinyi-5F | placehold.co Room A | photo | placehold.co Room B | photo | | |
| zhongshan-2F | | | | | | |
| expired-1F | placehold.co Expired | photo | | | | |

---

## Test plan

Serve locally and verify each scenario:

```bash
cd /home/hungmingtsai/Workspace/rental-listing && python3 -m http.server 8080
```

### Test 1: Detail page carousel — property with multiple photos

**URL:** `http://localhost:8080/?id=daan-3F`

**Check:**
- [ ] Carousel shows 3 slides (same as before)
- [ ] First slide loads the Google Drive photo (via `lh3.googleusercontent.com` transform)
- [ ] Slides 2 and 3 show placeholder images
- [ ] Prev/next buttons and dots work
- [ ] Touch swipe works (test in browser mobile emulation)

### Test 2: Detail page carousel — property with no photos

**URL:** `http://localhost:8080/?id=zhongshan-2F`

**Check:**
- [ ] Carousel is hidden (no crash, no empty container)

### Test 3: Listing page card thumbnails

**URL:** `http://localhost:8080/`

**Check:**
- [ ] daan-3F card shows Drive photo thumbnail
- [ ] xinyi-5F card shows placeholder Room A thumbnail
- [ ] zhongshan-2F card shows house emoji placeholder (no photos)
- [ ] expired-1F card does NOT appear (vacancy expired, auto-filtered)

### Test 4: Verify `getMediaItems()` handles defaults

Add a temporary `console.log(getMediaItems(row))` in `renderCarousel()` to verify:
- [ ] Items have correct `{url, type}` structure
- [ ] Type defaults to `'photo'` when `media_N_type` column is empty
- [ ] Empty `media_N` slots are skipped

Remove the console.log before committing.

### Test 5: Language toggle

**URL:** `http://localhost:8080/?id=daan-3F&lang=en`

**Check:**
- [ ] Toggle ZH/EN — carousel stays the same (photos are language-neutral)
- [ ] Toggle on listing page — card thumbnails unchanged

### Test 6: Verify no regressions on existing features

- [ ] Rent, vacancy, discount, address, size all display correctly
- [ ] Equipment checklist renders correctly
- [ ] LINE and WhatsApp CTA buttons work
- [ ] Language toggle updates all text fields
