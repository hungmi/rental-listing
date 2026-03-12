---
shaping: true
---

# V9: Media aspect ratio 16:9 + contain

## Goal

Change media containers from 4:3 to 16:9 to match the majority of source material. Switch `object-fit` from `cover` to `contain` so images are always displayed in full without cropping — non-16:9 content will letterbox on the existing grey background.

## Parts

- A1 — `.carousel` aspect ratio 4/3 → 16/9
- A2 — `.listing-card-thumb` aspect ratio 4/3 → 16/9
- A3 — Three `object-fit: cover` → `contain` (carousel photo, video thumbnail, listing card)

## Changes

### 1. `.carousel` aspect ratio (L228)

```css
/* Before */
aspect-ratio: 4 / 3;

/* After */
aspect-ratio: 16 / 9;
```

### 2. `.carousel-slide img` object-fit (L247)

```css
/* Before */
object-fit: cover;

/* After */
object-fit: contain;
```

### 3. `.carousel-slide-video img` object-fit (L318)

```css
/* Before */
object-fit: cover;

/* After */
object-fit: contain;
```

### 4. `.listing-card-thumb` aspect ratio (L445)

```css
/* Before */
aspect-ratio: 4 / 3;

/* After */
aspect-ratio: 16 / 9;
```

### 5. `.listing-card-thumb img` object-fit (L453)

```css
/* Before */
object-fit: cover;

/* After */
object-fit: contain;
```

Total: 5 lines changed, all CSS, no JS.

---

## Test plan

Serve locally and verify:

```bash
cd /home/hungmingtsai/Workspace/rental-listing && python3 -m http.server 8080
```

### Test 1: Structural verification — grep CSS values

```bash
# Verify no 4/3 remains and cover is gone from media elements
curl -s http://localhost:8080/ | grep -c 'aspect-ratio: 16 / 9'   # expect 2
curl -s http://localhost:8080/ | grep -c 'aspect-ratio: 4 / 3'    # expect 0
curl -s http://localhost:8080/ | grep -c 'object-fit: contain'     # expect 3
```

Count `object-fit: cover` should be 0 among the carousel/listing-card rules. (Other unrelated rules may still use `cover`.)

### Test 2: Detail page carousel — 16:9 photo

**URL:** `http://localhost:8080/?id=daan-3F`

**Check:**
- [ ] Carousel container is visually wider/shorter than before (16:9 vs 4:3)
- [ ] 16:9 photos fill the container edge-to-edge with no letterbox
- [ ] Prev/next buttons and dots still work
- [ ] Swipe navigation still works

### Test 3: Detail page carousel — non-16:9 photo

**Check:**
- [ ] Non-16:9 photos display completely (no cropping)
- [ ] Grey (#e0e0e0) bars appear on sides (landscape) or top/bottom (portrait) where content doesn't fill
- [ ] Photo is centered within the container

### Test 4: Detail page carousel — video thumbnail

**URL:** `http://localhost:8080/?id=daan-3F` → navigate to video slide

**Check:**
- [ ] Video thumbnail displays with `contain` (no cropping)
- [ ] Play button overlay still centered
- [ ] Tapping play still swaps to iframe
- [ ] iframe fills the 16:9 container

### Test 5: Listing page card thumbnails

**URL:** `http://localhost:8080/`

**Check:**
- [ ] Card thumbnails are 16:9 aspect ratio (wider/shorter than before)
- [ ] Photos display completely, no cropping
- [ ] Cards with placeholder emoji still render correctly
- [ ] Overall card grid layout is not broken

### Test 6: No regressions on existing features

- [ ] Rent, vacancy, discount, address, area display correctly
- [ ] Equipment checklist renders
- [ ] LINE/WhatsApp CTA buttons work
- [ ] Language toggle (ZH/EN) works
- [ ] Video play/stop-on-swipe still works
