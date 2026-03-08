---
shaping: true
---

# V5: Photo URL Transform — Implementation Plan

## Goal

Agent pastes a standard Google Drive sharing URL (e.g. `https://drive.google.com/file/d/1GRBFR.../view?usp=drive_link`) into the Google Sheet's `photo_*` columns, and the photo displays correctly on both the listing page (card thumbnail) and the detail page (carousel).

## Parts

| Part | What |
|------|------|
| A1.1a | Pattern recognition — regex extracts file ID from Drive URL |
| A1.1b | URL rewrite — constructs `lh3.googleusercontent.com/d/{ID}=w1000` |
| A1.1c | Passthrough — non-Drive URLs returned unchanged |
| A1.1d | Call sites — `renderCarousel()` line 784, `buildCard()` line 886 |

## Breadboard Affordance

| # | Affordance | Wires Out | Returns To |
|---|------------|-----------|------------|
| N13 | `transformPhotoUrl(url)` | — | → N8 (`buildCard`), N10 (`renderCarousel`) |

---

## Steps

### Step 1: Add `transformPhotoUrl()` function

Add the utility function after the `config` block (around line 591), before the CSV parser section.

```js
function transformPhotoUrl(url) {
  if (!url) return url;
  var match = url.match(/\/file\/d\/([a-zA-Z0-9_-]+)/);
  if (match) {
    return 'https://lh3.googleusercontent.com/d/' + match[1] + '=w1000';
  }
  return url;
}
```

**Logic:**
- Guard: empty/falsy URL returns as-is
- Regex `/\/file\/d\/([a-zA-Z0-9_-]+)/` captures the file ID from any `drive.google.com/file/d/{ID}/...` variant
- Match → rewrite to Google image proxy URL with `=w1000`
- No match → passthrough (placehold.co, imgur, direct image URLs)

### Step 2: Update `renderCarousel()` call site

Line 784: `img.src = url;` → `img.src = transformPhotoUrl(url);`

### Step 3: Update `buildCard()` call site

Line 886: `img.src = row.photo_1;` → `img.src = transformPhotoUrl(row.photo_1);`

### Step 4: Update `test-data.csv`

Replace one placeholder URL in the first test property (`daan-3F`) with a real Google Drive sharing URL, to test Drive transform alongside existing placehold.co URLs (passthrough).

```
photo_1 → https://drive.google.com/file/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-/view?usp=drive_link
photo_2 → keep placehold.co URL (tests passthrough)
```

---

## Testing Plan

### Test 1: Unit test — function logic (automated)

Create a temporary `test-transform.js` script and run with Node.js to verify all URL patterns:

```js
function transformPhotoUrl(url) {
  if (!url) return url;
  var match = url.match(/\/file\/d\/([a-zA-Z0-9_-]+)/);
  if (match) {
    return 'https://lh3.googleusercontent.com/d/' + match[1] + '=w1000';
  }
  return url;
}

var tests = [
  // [input, expected, description]
  [
    'https://drive.google.com/file/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-/view?usp=drive_link',
    'https://lh3.googleusercontent.com/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-=w1000',
    'Drive URL with ?usp=drive_link'
  ],
  [
    'https://drive.google.com/file/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-/view?usp=sharing',
    'https://lh3.googleusercontent.com/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-=w1000',
    'Drive URL with ?usp=sharing'
  ],
  [
    'https://drive.google.com/file/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-/view',
    'https://lh3.googleusercontent.com/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-=w1000',
    'Drive URL with no query params'
  ],
  [
    'https://placehold.co/800x600/e8e8e8/999?text=Room+1',
    'https://placehold.co/800x600/e8e8e8/999?text=Room+1',
    'Non-Drive URL passes through'
  ],
  [
    'https://i.imgur.com/abc123.jpg',
    'https://i.imgur.com/abc123.jpg',
    'Direct image URL passes through'
  ],
  [
    '',
    '',
    'Empty string passes through'
  ],
  [
    undefined,
    undefined,
    'Undefined passes through'
  ],
];

var passed = 0;
var failed = 0;
tests.forEach(function(t) {
  var result = transformPhotoUrl(t[0]);
  if (result === t[1]) {
    passed++;
    console.log('PASS: ' + t[2]);
  } else {
    failed++;
    console.log('FAIL: ' + t[2]);
    console.log('  expected: ' + t[1]);
    console.log('  got:      ' + result);
  }
});
console.log('\n' + passed + ' passed, ' + failed + ' failed');
if (failed > 0) process.exit(1);
```

Run: `node test-transform.js` — expect 7/7 pass, exit code 0.

### Test 2: Call site verification (grep)

After editing, verify both call sites use `transformPhotoUrl()`:

```bash
grep -n 'img\.src' index.html
```

Expected: every `img.src` assignment for photos goes through `transformPhotoUrl()`. The carousel `goTo()` function does NOT set `img.src` (it only moves the track), so it should not appear.

### Test 3: Integration — mixed URLs in test-data.csv

After updating test-data.csv with mixed Drive + placehold.co URLs, verify by inspecting the generated `img.src` values. Serve locally and check:

```bash
python3 -m http.server 8000
```

Open in browser:
- `http://localhost:8000/` — listing page, first card thumbnail should have `lh3.googleusercontent.com` src
- `http://localhost:8000/?id=daan-3F` — detail page, carousel first slide should have `lh3.googleusercontent.com` src, second slide should have `placehold.co` src

### Test 4: Regression — existing placehold.co URLs unchanged

Properties `xinyi-5F` still use placehold.co URLs. Verify they render as before — passthrough means no change in behavior for non-Drive URLs.

---

## Files modified

| File | Change |
|------|--------|
| `index.html` | Add `transformPhotoUrl()` function, update 2 `img.src` lines |
| `test-data.csv` | Replace `daan-3F` `photo_1` with Drive sharing URL |

## Files created (temporary, delete after testing)

| File | Purpose |
|------|---------|
| `test-transform.js` | Unit test script for `transformPhotoUrl()` — delete after V5 |

---

## Out of scope for V5

- `drive.google.com/open?id={ID}` format (uncommon, easy to add later)
- Variable width parameter per context (e.g. `=w400` for thumbnails)
- Image error handling / fallback if Drive URL is not publicly shared
