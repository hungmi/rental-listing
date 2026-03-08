---
shaping: true
---

# A7 Spike: Mixed Media Carousel (v2)

## Context

Agent wants 2–3 videos per property, interleaved with photos in a custom order within the same carousel. This changes A7 from "separate video section" to "mixed media carousel." Two key unknowns:

1. **A7.1 (Data model)**: Three alternatives proposed — which one works best for the agent?
2. **A7.3 (Carousel rendering)**: Can iframe video slides coexist with img photo slides in the current carousel? What breaks?

### Prior findings (v1, still valid)

- Video embed URL: `https://drive.google.com/file/d/{ID}/preview` (iframe)
- Same regex extracts file ID from Drive sharing URLs
- Separate `transformVideoUrl()` needed (different output from photo transform)
- Google Drive `/preview` works on mobile browsers

## Goal

Resolve the A7.1 data model and A7.3 mixed carousel mechanism so both ⚠️ flags can be cleared.

## Questions

| # | Question |
|---|----------|
| **A7-Q1** | Which data model (A/B/C) best balances agent simplicity with ordering control? |
| **A7-Q2** | How does the carousel code distinguish photo slides from video slides? |
| **A7-Q3** | Do iframe slides break touch/swipe in the current carousel? |
| **A7-Q4** | What aspect ratio should the mixed carousel use? (Photos are 4:3, videos are 16:9) |
| **A7-Q5** | How does the listing page thumbnail work with unified media columns? |
| **A7-Q6** | What happens when a video slide is swiped away — does the video keep playing? |

## Acceptance

Spike is complete when all questions are answered, one data model is recommended, and we can describe the mixed carousel mechanism concretely.

---

## Findings

### A7-Q1: Which data model best balances agent simplicity with ordering control?

Evaluating the three alternatives:

**A7.1-A: Unified columns + type columns**

```
media_1      = https://drive.google.com/file/d/abc/view?usp=sharing
media_1_type = photo
media_2      = https://drive.google.com/file/d/xyz/view?usp=sharing
media_2_type = video
...up to media_8 / media_8_type
```

- Order = column order (media_1 is first, media_2 is second...) — intuitive
- Type = dropdown validation in Google Sheets (`photo` / `video`) — foolproof
- 16 columns total (8 URL + 8 type)
- Replaces current `photo_1`–`photo_5` (5 columns)
- Net change: +11 columns

**A7.1-B: Separate photo/video columns + order string**

```
photo_1 = https://drive.google.com/...
photo_2 = https://drive.google.com/...
video_1 = https://drive.google.com/...
video_2 = https://drive.google.com/...
media_order = p1,v1,p2,v2
```

- Agent pastes URLs exactly as before (familiar)
- One new concept: the `media_order` string
- 8 URL columns (5 photo + 3 video) + 1 order column = 9 columns
- Net change: +4 columns (3 video + 1 order)
- Risk: agent forgets to update `media_order` when adding/removing media, causing mismatches
- Risk: typos in order string (`p1,v2,p2` — is `v2` valid if only `video_1` is filled?)

**A7.1-C: Unified columns + video: prefix**

```
media_1 = https://drive.google.com/file/d/abc/view?usp=sharing
media_2 = video:https://drive.google.com/file/d/xyz/view?usp=sharing
```

- Fewest columns (8)
- Order = column order
- Risk: agent forgets `video:` prefix → video rendered as broken image
- Breaks the "just paste a URL" simplicity — violates R4's spirit

**Recommendation: A7.1-A.** Despite more columns, it has the strongest R4 fit:

- Order is implicit in column order — no string to parse, no sync issues
- Type dropdown prevents errors — Google Sheets data validation makes it a simple selection
- Agent workflow: paste URL → pick type from dropdown → done
- 16 columns is manageable in Google Sheets (can freeze/group columns)
- Replacing `photo_1`–`photo_5` with `media_1`–`media_8` is a clean migration

### A7-Q2: How does the carousel code distinguish photo slides from video slides?

With A7.1-A, the code reads `media_N` and `media_N_type` pairs:

```javascript
function getMediaItems(row) {
  var items = [];
  for (var i = 1; i <= 8; i++) {
    var url = row['media_' + i];
    var type = (row['media_' + i + '_type'] || 'photo').toLowerCase();
    if (url) {
      items.push({ url: url, type: type });
    }
  }
  return items;
}
```

Each item carries its type. The carousel render loop checks `item.type` to create either an `<img>` or a video slide element.

### A7-Q3: Do iframe slides break touch/swipe in the current carousel?

**Yes.** The current swipe implementation (lines 831–841) listens for `touchstart`/`touchend` on the carousel container. When the active slide is an iframe, the iframe captures touch events — the carousel container never receives them, so swiping fails on video slides.

**Solution: thumbnail-with-play-button approach.** Instead of embedding the iframe directly in the carousel slide:

1. Video slides render as a **thumbnail image with a play button overlay**
2. Thumbnail URL: `https://drive.google.com/thumbnail?id={ID}&sz=w1000` — Google Drive provides thumbnails for video files
3. On **tap/click**, the thumbnail is replaced with the live iframe (`/preview` URL)
4. On **swipe away**, the iframe is removed and reverted to thumbnail (stops playback, see Q6)

This solves three problems at once:
- **Swipe works** — all slides are initially images, touch events flow normally
- **Fast loading** — no iframes until user explicitly plays a video
- **Consistent appearance** — all carousel slides look uniform (image + optional play overlay)

Slide structure:

```html
<!-- Photo slide -->
<div class="carousel-slide">
  <img src="https://lh3.googleusercontent.com/d/{ID}=w1000" />
</div>

<!-- Video slide (not yet playing) -->
<div class="carousel-slide carousel-slide-video" data-video-src="https://drive.google.com/file/d/{ID}/preview">
  <img src="https://drive.google.com/thumbnail?id={ID}&sz=w1000" />
  <button class="carousel-play-btn">▶</button>
</div>

<!-- Video slide (playing, after tap) -->
<div class="carousel-slide carousel-slide-video playing">
  <iframe src="https://drive.google.com/file/d/{ID}/preview"
    allow="autoplay; encrypted-media" allowfullscreen></iframe>
</div>
```

### A7-Q4: What aspect ratio should the mixed carousel use?

**Keep 4:3.** Current carousel is 4:3 (`aspect-ratio: 4 / 3` on line 228). Photos are composed for this ratio. Videos in 16:9 will display with letterbox bars (black/gray bars top and bottom) inside the 4:3 container via `object-fit: contain` on the iframe or thumbnail.

Changing to 16:9 would crop all existing photos. Keeping 4:3 preserves photo quality and accommodates video acceptably. The Google Drive player handles its own aspect ratio inside the iframe.

CSS for video slides within the 4:3 container:

```css
.carousel-slide iframe {
  width: 100%;
  height: 100%;
  border: 0;
  background: #000;
}
```

The iframe fills the 4:3 slide; Google's player renders the video at native ratio with black bars if needed.

### A7-Q5: How does the listing page thumbnail work with unified media columns?

Current `buildCard()` uses `photo_1` for the thumbnail. With A7.1-A, it should use the **first `photo`-type media item**:

```javascript
function getFirstPhoto(row) {
  for (var i = 1; i <= 8; i++) {
    var url = row['media_' + i];
    var type = (row['media_' + i + '_type'] || 'photo').toLowerCase();
    if (url && type === 'photo') return url;
  }
  return null;
}
```

If `media_1` is a video, skip it and use the first photo for the card thumbnail. Listing cards should always show a static photo, never a video embed.

### A7-Q6: What happens when a video slide is swiped away?

**Revert to thumbnail.** When `goTo(index)` is called and the current slide has a playing video:

1. Remove the iframe from the current slide
2. Restore the thumbnail + play button
3. This stops playback immediately (removing iframe from DOM stops the video)

```javascript
function stopVideoOnSlide(slideEl) {
  if (slideEl.classList.contains('playing')) {
    var src = slideEl.querySelector('iframe').src;
    slideEl.innerHTML = '<img src="..." /><button class="carousel-play-btn">▶</button>';
    slideEl.classList.remove('playing');
    slideEl.dataset.videoSrc = src;
  }
}
```

This is called inside `goTo()` before transitioning. No background audio leak.

---

## Summary

### Data model decision: A7.1-A (unified columns with type)

| Column | Example | Notes |
|--------|---------|-------|
| `media_1` | `https://drive.google.com/file/d/abc/view?usp=sharing` | First carousel item |
| `media_1_type` | `photo` | Dropdown: `photo` or `video` |
| `media_2` | `https://drive.google.com/file/d/xyz/view?usp=sharing` | Second carousel item |
| `media_2_type` | `video` | |
| ... | ... | Up to `media_8` / `media_8_type` |

Replaces `photo_1`–`photo_5`. Agent controls order by column position.

### Mixed carousel mechanism

| Aspect | Mechanism |
|--------|-----------|
| Slide type | `getMediaItems(row)` reads `media_N` + `media_N_type` pairs |
| Photo slide | `<img>` with `transformPhotoUrl()` — same as today |
| Video slide (idle) | Thumbnail image (`drive.google.com/thumbnail?id={ID}&sz=w1000`) + play button overlay |
| Video slide (playing) | On tap: replace thumbnail with `<iframe src=".../preview">` |
| Swipe on video | Works — slides are thumbnails until explicitly played |
| Swipe away playing video | Revert to thumbnail, iframe removed, playback stops |
| Aspect ratio | Keep 4:3 — video letterboxes inside container |
| Listing card thumbnail | `getFirstPhoto(row)` — first `photo`-type media item |

### Steps to implement

| Step | What | Where |
|------|------|-------|
| 1 | Replace `photo_1`–`photo_5` with `media_1`–`media_8` + `media_1_type`–`media_8_type` in Google Sheet | Google Sheet |
| 2 | Add `transformVideoUrl(url)` function | `index.html` after `transformPhotoUrl()` |
| 3 | Add `getMediaItems(row)` to read unified media columns | `index.html` |
| 4 | Modify `renderCarousel()` to handle both photo and video slides | `index.html` |
| 5 | Add play button CSS + thumbnail-to-iframe swap logic | `index.html` |
| 6 | Add stop-on-swipe logic in `goTo()` | `index.html` |
| 7 | Modify `buildCard()` to use `getFirstPhoto(row)` for listing thumbnail | `index.html` |
| 8 | Add `.carousel-play-btn` and `.carousel-slide iframe` CSS | `index.html` |
| 9 | Update `test-data.csv` with new column format | `test-data.csv` |

All questions answered. No remaining unknowns.
