---
shaping: true
---

# V7: Video in carousel

## Goal

Add video support to the mixed media carousel. Video slides render as thumbnail + play button overlay; tap plays inline via iframe; swipe away stops playback. Listing card thumbnails remain photo-only.

## Parts

- A7.2 — Video URL transform (`transformVideoUrl()`)
- A7.3 — Mixed carousel rendering (video portion)
- A7.4 — Thumbnail-to-iframe swap + stop-on-swipe

## Changes

### 1. Add `transformVideoUrl(url)` — NEW

Insert after `transformPhotoUrl()` (~line 603).

```javascript
function transformVideoUrl(url) {
  if (!url) return null;
  var match = url.match(/\/file\/d\/([a-zA-Z0-9_-]+)/);
  if (match) {
    return 'https://drive.google.com/file/d/' + match[1] + '/preview';
  }
  return null;
}
```

- Extracts Google Drive file ID using the same regex as `transformPhotoUrl()`
- Returns embed URL (`/preview`) instead of direct image URL
- Returns `null` if URL doesn't match (non-Drive URLs can't be embedded this way)

### 2. Add `getVideoThumbnailUrl(url)` — NEW

Insert after `transformVideoUrl()`.

```javascript
function getVideoThumbnailUrl(url) {
  if (!url) return null;
  var match = url.match(/\/file\/d\/([a-zA-Z0-9_-]+)/);
  if (match) {
    return 'https://drive.google.com/thumbnail?id=' + match[1] + '&sz=w1000';
  }
  return null;
}
```

- Extracts file ID, returns Google Drive thumbnail URL
- Used for the poster image on video slides (before user taps play)

### 3. Add carousel video CSS — NEW

Insert after `.carousel-empty` block (~line 308).

```css
.carousel-slide-video {
  position: relative;
  cursor: pointer;
}

.carousel-slide-video img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.carousel-play-btn {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.85);
  border: none;
  font-size: 24px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.carousel-play-btn:active {
  background: rgba(255, 255, 255, 1);
}

.carousel-slide iframe {
  width: 100%;
  height: 100%;
  border: 0;
  background: #000;
}

.carousel-slide.playing {
  cursor: default;
}
```

- `.carousel-slide-video`: relative positioning for play button overlay
- `.carousel-play-btn`: centered white circle with ▶ symbol
- `.carousel-slide iframe`: fills 4:3 container; Google Drive player handles video aspect ratio internally (letterboxes 16:9 content)
- `.carousel-slide.playing`: marks active video (used by `stopVideoOnSlide()`)

### 4. Modify `renderCarousel(row)` — lines 798–865

**Before (V6):**
```javascript
function renderCarousel(row) {
  const allItems = getMediaItems(row);
  const photos = [];
  for (var i = 0; i < allItems.length; i++) {
    if (allItems[i].type === 'photo') photos.push(allItems[i].url);
  }

  const carousel = document.getElementById('carousel');
  if (photos.length === 0) {
    carousel.parentElement.style.display = 'none';
    return;
  }

  const track = document.getElementById('carousel-track');
  const dotsContainer = document.getElementById('carousel-dots');

  // Build slides
  photos.forEach((url, i) => {
    const slide = document.createElement('div');
    slide.className = 'carousel-slide';
    const img = document.createElement('img');
    img.src = transformPhotoUrl(url);
    img.alt = '房源照片 ' + (i + 1);
    img.loading = i === 0 ? 'eager' : 'lazy';
    slide.appendChild(img);
    track.appendChild(slide);

    const dot = document.createElement('button');
    dot.className = 'carousel-dot' + (i === 0 ? ' active' : '');
    dot.setAttribute('aria-label', '照片 ' + (i + 1));
    dotsContainer.appendChild(dot);
  });

  // Hide nav if only 1 photo
  if (photos.length <= 1) {
    document.getElementById('carousel-prev').style.display = 'none';
    document.getElementById('carousel-next').style.display = 'none';
    dotsContainer.style.display = 'none';
    return;
  }

  let current = 0;
  const dots = dotsContainer.querySelectorAll('.carousel-dot');

  function goTo(index) {
    if (index < 0) index = photos.length - 1;
    if (index >= photos.length) index = 0;
    current = index;
    track.style.transform = 'translateX(-' + (current * 100) + '%)';
    dots.forEach((d, i) => d.classList.toggle('active', i === current));
  }

  document.getElementById('carousel-prev').addEventListener('click', () => goTo(current - 1));
  document.getElementById('carousel-next').addEventListener('click', () => goTo(current + 1));
  dots.forEach((dot, i) => dot.addEventListener('click', () => goTo(i)));

  // Touch swipe support
  let touchStartX = 0;
  let touchEndX = 0;
  carousel.addEventListener('touchstart', (e) => { touchStartX = e.changedTouches[0].screenX; }, { passive: true });
  carousel.addEventListener('touchend', (e) => {
    touchEndX = e.changedTouches[0].screenX;
    const diff = touchStartX - touchEndX;
    if (Math.abs(diff) > 50) {
      goTo(diff > 0 ? current + 1 : current - 1);
    }
  }, { passive: true });
}
```

**After (V7):**
```javascript
function renderCarousel(row) {
  const allItems = getMediaItems(row);

  // Filter to renderable items (photos always; videos only if they have a valid Drive ID)
  const mediaSlides = [];
  for (var i = 0; i < allItems.length; i++) {
    var item = allItems[i];
    if (item.type === 'photo') {
      mediaSlides.push(item);
    } else if (item.type === 'video' && transformVideoUrl(item.url)) {
      mediaSlides.push(item);
    }
  }

  const carousel = document.getElementById('carousel');
  if (mediaSlides.length === 0) {
    carousel.parentElement.style.display = 'none';
    return;
  }

  const track = document.getElementById('carousel-track');
  const dotsContainer = document.getElementById('carousel-dots');

  // Build slides
  mediaSlides.forEach(function(item, i) {
    var slide = document.createElement('div');

    if (item.type === 'photo') {
      slide.className = 'carousel-slide';
      var img = document.createElement('img');
      img.src = transformPhotoUrl(item.url);
      img.alt = '房源照片 ' + (i + 1);
      img.loading = i === 0 ? 'eager' : 'lazy';
      slide.appendChild(img);
    } else {
      // Video slide: thumbnail + play button
      var embedUrl = transformVideoUrl(item.url);
      var thumbUrl = getVideoThumbnailUrl(item.url);
      slide.className = 'carousel-slide carousel-slide-video';
      slide.setAttribute('data-video-src', embedUrl);

      var img = document.createElement('img');
      img.src = thumbUrl;
      img.alt = '房源影片 ' + (i + 1);
      img.loading = i === 0 ? 'eager' : 'lazy';
      slide.appendChild(img);

      var playBtn = document.createElement('button');
      playBtn.className = 'carousel-play-btn';
      playBtn.textContent = '▶';
      playBtn.setAttribute('aria-label', '播放影片');
      slide.appendChild(playBtn);
    }

    track.appendChild(slide);

    var dot = document.createElement('button');
    dot.className = 'carousel-dot' + (i === 0 ? ' active' : '');
    dot.setAttribute('aria-label', (item.type === 'photo' ? '照片' : '影片') + ' ' + (i + 1));
    dotsContainer.appendChild(dot);
  });

  // Hide nav if only 1 item
  if (mediaSlides.length <= 1) {
    document.getElementById('carousel-prev').style.display = 'none';
    document.getElementById('carousel-next').style.display = 'none';
    dotsContainer.style.display = 'none';
    if (mediaSlides.length === 1 && mediaSlides[0].type === 'video') {
      // Single video — still need play handler
      setupPlayHandlers(track);
    }
    return;
  }

  var current = 0;
  var slides = track.querySelectorAll('.carousel-slide');
  var dots = dotsContainer.querySelectorAll('.carousel-dot');

  function stopVideoOnSlide(slideEl) {
    if (slideEl.classList.contains('playing')) {
      var videoSrc = slideEl.getAttribute('data-video-src');
      var thumbUrl = slideEl.querySelector('iframe') ? null : null;
      // Restore thumbnail + play button
      slideEl.innerHTML = '';
      slideEl.classList.remove('playing');

      var img = document.createElement('img');
      // Re-derive thumbnail from data-video-src
      var match = videoSrc.match(/\/file\/d\/([a-zA-Z0-9_-]+)/);
      if (match) {
        img.src = 'https://drive.google.com/thumbnail?id=' + match[1] + '&sz=w1000';
      }
      img.alt = '房源影片';
      slideEl.appendChild(img);

      var playBtn = document.createElement('button');
      playBtn.className = 'carousel-play-btn';
      playBtn.textContent = '▶';
      playBtn.setAttribute('aria-label', '播放影片');
      slideEl.appendChild(playBtn);
    }
  }

  function goTo(index) {
    if (index < 0) index = mediaSlides.length - 1;
    if (index >= mediaSlides.length) index = 0;
    // Stop video on current slide before transitioning
    stopVideoOnSlide(slides[current]);
    current = index;
    track.style.transform = 'translateX(-' + (current * 100) + '%)';
    dots.forEach(function(d, i) { d.classList.toggle('active', i === current); });
  }

  document.getElementById('carousel-prev').addEventListener('click', function() { goTo(current - 1); });
  document.getElementById('carousel-next').addEventListener('click', function() { goTo(current + 1); });
  dots.forEach(function(dot, i) { dot.addEventListener('click', function() { goTo(i); }); });

  // Play button handler (delegated)
  setupPlayHandlers(track);

  // Touch swipe support
  var touchStartX = 0;
  var touchEndX = 0;
  carousel.addEventListener('touchstart', function(e) { touchStartX = e.changedTouches[0].screenX; }, { passive: true });
  carousel.addEventListener('touchend', function(e) {
    touchEndX = e.changedTouches[0].screenX;
    var diff = touchStartX - touchEndX;
    if (Math.abs(diff) > 50) {
      goTo(diff > 0 ? current + 1 : current - 1);
    }
  }, { passive: true });
}

function setupPlayHandlers(track) {
  track.addEventListener('click', function(e) {
    var playBtn = e.target.closest('.carousel-play-btn');
    if (!playBtn) return;
    var slide = playBtn.closest('.carousel-slide-video');
    if (!slide || slide.classList.contains('playing')) return;

    var videoSrc = slide.getAttribute('data-video-src');
    if (!videoSrc) return;

    // Replace thumbnail + play button with iframe
    slide.innerHTML = '';
    var iframe = document.createElement('iframe');
    iframe.src = videoSrc;
    iframe.setAttribute('allow', 'autoplay; encrypted-media');
    iframe.setAttribute('allowfullscreen', '');
    slide.appendChild(iframe);
    slide.classList.add('playing');
  });
}
```

Key changes:
- **`mediaSlides` replaces `photos`**: now includes both photo and video items in order
- **Photo slides**: unchanged — `<img>` with `transformPhotoUrl()`
- **Video slides**: `<img>` thumbnail + `<button class="carousel-play-btn">▶</button>`, with `data-video-src` attribute storing the embed URL
- **`stopVideoOnSlide()`**: nested inside `renderCarousel()` closure so it has access to `slides`. Called by `goTo()` before transitioning. Removes iframe, restores thumbnail + play button.
- **`setupPlayHandlers()`**: extracted as a top-level function since it's also needed for single-video case. Uses event delegation on the track.
- **`goTo()`**: calls `stopVideoOnSlide(slides[current])` before changing slide — prevents background audio/video playback
- **Dots aria-label**: distinguishes 照片/影片

### 5. Update `test-data.csv`

Add a video entry to the daan-3F row to test mixed photo+video carousel. Insert a video between photos 1 and 2.

**Before:**
```csv
id,...,media_1,media_1_type,media_2,media_2_type,media_3,media_3_type,...
daan-3F,...,https://drive.google.com/file/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-/view?usp=drive_link,photo,https://placehold.co/800x600/d8e8d8/666?text=Room+2,photo,https://placehold.co/800x600/d8d8e8/666?text=Room+3,photo,...
```

**After:**
Add `media_4,media_4_type` columns. Shift Room 3 placeholder to media_4 and insert a video at media_3.

```csv
id,...,media_1,media_1_type,media_2,media_2_type,media_3,media_3_type,media_4,media_4_type,...
daan-3F,...,https://drive.google.com/file/d/1GRBFR-yB2Ish3a6JODmDhuekfyfOLPK-/view?usp=drive_link,photo,https://placehold.co/800x600/d8e8d8/666?text=Room+2,photo,https://drive.google.com/file/d/1qdYdil3QyGKolPTcjbBxsONNHfENXkif/view?usp=sharing,video,https://placehold.co/800x600/d8d8e8/666?text=Room+3,photo,...
```

Row mappings:

| Row | media_1 | type | media_2 | type | media_3 | type | media_4 | type |
|-----|---------|------|---------|------|---------|------|---------|------|
| daan-3F | Drive photo | photo | placeholder Room 2 | photo | Drive video | video | placeholder Room 3 | photo |
| xinyi-5F | placeholder Room A | photo | placeholder Room B | photo | | | | |
| zhongshan-2F | | | | | | | | |
| expired-1F | placeholder Expired | photo | | | | | | |

This tests:
- Mixed photo+video ordering (photo, photo, video, photo)
- Video between photos in the carousel
- `getFirstPhoto()` still returns the Drive photo (not the video) for card thumbnail

---

## Test plan

Serve locally and verify each scenario:

```bash
cd /home/hungmingtsai/Workspace/rental-listing && python3 -m http.server 8080
```

### Test 1: Mixed carousel — photos and video interleaved

**URL:** `http://localhost:8080/?id=daan-3F`

**Check:**
- [ ] Carousel shows 4 slides (was 3 in V6)
- [ ] Slide 1: Drive photo (same as before)
- [ ] Slide 2: placeholder Room 2 photo
- [ ] Slide 3: video thumbnail with ▶ play button overlay
- [ ] Slide 4: placeholder Room 3 photo
- [ ] Dots show 4 indicators
- [ ] Prev/next buttons cycle through all 4 slides

### Test 2: Play button tap → video plays inline

**URL:** `http://localhost:8080/?id=daan-3F` → navigate to slide 3

**Check:**
- [ ] Tapping ▶ button replaces thumbnail with Google Drive iframe player
- [ ] Slide gets `.playing` class
- [ ] Video plays within the 4:3 carousel container (may letterbox)
- [ ] Play button disappears after tap

### Test 3: Swipe away from playing video → video stops

**URL:** `http://localhost:8080/?id=daan-3F` → play video on slide 3 → swipe/click to slide 4

**Check:**
- [ ] Video iframe is removed from DOM (no background audio)
- [ ] Slide 3 reverts to thumbnail + ▶ play button
- [ ] `.playing` class is removed
- [ ] Can tap ▶ again to replay

### Test 4: Property with only photos (no video)

**URL:** `http://localhost:8080/?id=xinyi-5F`

**Check:**
- [ ] Carousel shows 2 photo slides (same as V6)
- [ ] No play buttons visible
- [ ] No regression — same behavior as before

### Test 5: Property with no media

**URL:** `http://localhost:8080/?id=zhongshan-2F`

**Check:**
- [ ] Carousel is hidden (no crash, no empty container)

### Test 6: Listing page card thumbnails unchanged

**URL:** `http://localhost:8080/`

**Check:**
- [ ] daan-3F card shows Drive photo thumbnail (not the video thumbnail)
- [ ] xinyi-5F card shows placeholder Room A
- [ ] zhongshan-2F card shows house emoji placeholder
- [ ] expired-1F card does NOT appear (vacancy expired)

### Test 7: `transformVideoUrl()` and `getVideoThumbnailUrl()` correctness

Use Python to verify URL transforms:

```python
import re

def transform_video_url(url):
    match = re.search(r'/file/d/([a-zA-Z0-9_-]+)', url)
    if match:
        return f'https://drive.google.com/file/d/{match.group(1)}/preview'
    return None

def get_video_thumbnail_url(url):
    match = re.search(r'/file/d/([a-zA-Z0-9_-]+)', url)
    if match:
        return f'https://drive.google.com/thumbnail?id={match.group(1)}&sz=w1000'
    return None

test_url = 'https://drive.google.com/file/d/1qdYdil3QyGKolPTcjbBxsONNHfENXkif/view?usp=sharing'
assert transform_video_url(test_url) == 'https://drive.google.com/file/d/1qdYdil3QyGKolPTcjbBxsONNHfENXkif/preview'
assert get_video_thumbnail_url(test_url) == 'https://drive.google.com/thumbnail?id=1qdYdil3QyGKolPTcjbBxsONNHfENXkif&sz=w1000'
assert transform_video_url('https://example.com/notdrive') is None
print('All transform tests pass')
```

### Test 8: Structural verification via curl + grep

```bash
# Verify video slide HTML structure exists when fetching detail page
curl -s http://localhost:8080/ | grep -c 'carousel-play-btn'  # should find CSS rule
curl -s http://localhost:8080/ | grep -c 'carousel-slide-video'  # should find CSS rule
curl -s http://localhost:8080/ | grep -c 'transformVideoUrl'  # should find function
curl -s http://localhost:8080/ | grep -c 'getVideoThumbnailUrl'  # should find function
curl -s http://localhost:8080/ | grep -c 'stopVideoOnSlide'  # should find function
curl -s http://localhost:8080/ | grep -c 'setupPlayHandlers'  # should find function
```

### Test 9: Language toggle with video

**URL:** `http://localhost:8080/?id=daan-3F&lang=en`

**Check:**
- [ ] Toggle ZH/EN — carousel slides unchanged (media is language-neutral)
- [ ] Video thumbnail and play button unaffected by language switch

### Test 10: Verify no regressions on existing features

- [ ] Rent, vacancy, discount, address, size all display correctly
- [ ] Equipment checklist renders correctly
- [ ] LINE and WhatsApp CTA buttons work
- [ ] Language toggle updates all text fields
