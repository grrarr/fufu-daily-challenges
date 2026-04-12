# Video Question Grabber -- Design Spec

## Problem

Each PSL daily challenge quiz has a video that shows a math question on a blue background near the end. We extract solution text + images via the Thinkific API, but the actual **question content** is only visible as an image in the video frame. Currently the `question` field in our app data is either empty or manually typed. We need an automated pipeline to capture these question frames and transcribe them.

---

## Architecture Overview

```
+------------------+     +------------------+     +-------------------+
|  Phase 1: RECON  |     |  Phase 2: CAPTURE|     |  Phase 3: OCR     |
|  (API metadata)  |---->|  (video frames)  |---->|  (transcription)  |
+------------------+     +------------------+     +-------------------+
        |                         |                         |
        v                         v                         v
  Quiz URL list            PNG files in               Transcribed text
  + content IDs            images/ folder             + answer choices
  + video durations                                         |
                          +------------------+              v
                          |  Phase 4: VERIFY |     +-------------------+
                          |  (blue-check)    |     |  Phase 5: MERGE   |
                          +------------------+     |  (into app data)  |
                                  |                +-------------------+
                                  v
                          Re-attempt queue
                          for failed frames
```

### Data Flow

```
daily.poshenloh.com
        |
        |  Thinkific API: /api/course_player/v2/courses/{slug}
        v
  Content ID list (per quiz)
        |
        |  Thinkific API: /api/course_player/v2/contents/{id}
        v
  Wistia video embed URL + video hash
        |
        |  Load Wistia embed directly (skip full page load)
        v
  video.currentTime = duration - N
        |
        |  canvas.drawImage(video, ...)
        v
  PNG frame capture --> blue-check --> save or retry
        |
        |  Claude vision (Read tool on PNG)
        v
  Transcribed question text + answer choices
        |
        |  Merge into fufu-daily-challenges data
        v
  Updated challenges with question + questionImageUrl
```

---

## Phase 1: Build Quiz URL List (No Extra API Calls)

### Goal
Build a list of quiz page URLs to visit. NO batch API calls -- reuse data we already have.

### Approach

We already have all quiz URLs from the sidebar DOM scraping done during extraction. These are stored in the extracted data files and can be reconstructed from the course structure API calls we already made (one call per course, already cached).

**Option A: Reconstruct from existing data**
The app data file (`fufu-daily-challenges-data.json`) has all challenge entries with day/quiz metadata. The quiz page URLs follow the pattern we already know:
```
/courses/take/{course-slug}/quizzes/{contentable-id}-{slug-name}
```

**Option B: One course structure call per module (already done)**
We already call `/api/course_player/v2/courses/{slug}` once per module during extraction. This returns ALL content IDs. No additional API calls needed.

### Key principle: NO batch API requests for recon
The video duration and Wistia hash are discovered **during capture** when we navigate to each quiz page (the iframe `src` contains the Wistia hash, and `video.duration` is available after the video loads). This eliminates the separate recon phase entirely.

### Output
A simple list of quiz page URLs to visit, built from existing data. No manifest file needed -- metadata is collected during capture.

---

## Phase 2: Poster Image Capture (PROVEN APPROACH)

### Goal
For each quiz, grab the Wistia poster/thumbnail image URL from the DOM and download it directly. No video playback, no seeking, no foreground tab needed.

### Discovery (tested 2026-04-12)
The Wistia player embed stores a poster image as an `<img>` element inside the iframe. This image:
- Shows the **full question slide** on blue background -- clean, no play button overlay
- Includes hints and answer choice markers
- Is a **static CDN URL** that can be downloaded directly
- Is accessible via simple DOM query even in a **background tab**

### Example
```
Poster URL found: https://embed-ssl.wistia.com/deliveries/85ed730d3a0ba834e0e51f057cd267572f8f6530.jpg
```
This returned a clean 960x540 image of "What is 2.5 x 0.25 x 1.3-bar x 1.2?" with hint text.

### Capture Sequence

```js
// === Step 2a: Get poster image URL from Wistia iframe ===
const iframe = document.querySelector('iframe[src*="play"]');
const posterImg = iframe.contentDocument.querySelector('img');
const posterUrl = posterImg ? posterImg.src : null;
// Filter out blank.gif placeholder
const isReal = posterUrl && !posterUrl.includes('blank.gif') && posterUrl.includes('wistia');

// === Step 2b: If poster URL found, we're done ===
// The URL is a direct CDN link -- download it to disk
// No video loading, no seeking, no canvas, no CORS issues
```

### Batch extraction (fire-and-forget in browser)

```js
// Navigate via Ember sidebar links (no page reload)
// For each quiz:
//   1. Click sidebar link (~1s Ember transition)
//   2. Click View Again if completed page
//   3. Wait for iframe to load (~2s)
//   4. Read iframe.contentDocument.querySelector('img').src
//   5. Store URL in results array
//   6. Move to next quiz

// The poster URLs are static CDN links that don't expire
// Download all images AFTER collecting URLs (batch download, no website interaction)
```

### Key advantages over video seek approach
- **No video playback needed** -- poster is a static image
- **Works in background tab** -- no Chrome throttling issues
- **No CORS issues** -- we read a URL, not canvas pixels
- **No play button overlay** -- the poster image is the raw frame
- **Much faster** -- DOM query is instant, no waiting for video load/seek
- **Clean image** -- full resolution, no compression artifacts from canvas

### What if poster doesn't show the question?
Some videos (explanation videos) may have a poster showing the instructor rather than a question slide. Detection:
- Check if URL contains question-related content (blue-check via downloading the image)
- Cross-reference with quiz title -- "Challenge Explanation" videos often don't have their own question
- Mark these as `NO_QUESTION_SLIDE` and skip

### Handling Wistia Player API

If available on the page, the Wistia JS API provides additional control:

```js
// After Wistia embed loads, the API is available:
const wistiaEmbed = document.querySelector('.wistia_embed');
const api = window._wq || [];
api.push({
  id: '{hash}',
  onReady: function(video) {
    video.time(video.duration() - 5);  // seek
    video.pause();
    // Wistia renders the frame at the seeked position
  }
});
```

### Chrome Tab Throttling

Chrome throttles background tabs, which may prevent video frame rendering. Solutions:

1. **Keep the tab in foreground** during capture (recommended)
2. Use `chrome.debugger` or DevTools protocol to disable throttling (complex)
3. Accept that the operator (Claude in Chrome) already has the tab focused

### Timing Per Quiz
- Page load (Strategy A): ~1.5s
- Seek + render: ~1.5s
- Canvas capture: ~0.05s
- Total: ~3s per quiz (Strategy A) or ~7s per quiz (Strategy B)

---

## Phase 3: Blue-Check Verification

### Goal
Automatically verify that the captured frame is actually a question slide (blue background with white text) and not a black frame, instructor face, or end card.

### Color Histogram Approach

```js
function isQuestionSlide(canvas) {
  const ctx = canvas.getContext('2d');
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const pixels = imageData.data; // RGBA flat array

  let bluePixels = 0;
  let darkPixels = 0;
  let totalPixels = canvas.width * canvas.height;

  for (let i = 0; i < pixels.length; i += 4) {
    const r = pixels[i];
    const g = pixels[i + 1];
    const b = pixels[i + 2];

    // Blue range for PSL question slides: #2B3B8E to #5B6BBE
    // R: 40-100, G: 50-110, B: 130-200
    if (r >= 30 && r <= 110 && g >= 40 && g <= 120 && b >= 120 && b <= 210) {
      bluePixels++;
    }

    // Dark/black check (video not rendered)
    if (r < 20 && g < 20 && b < 20) {
      darkPixels++;
    }
  }

  const blueRatio = bluePixels / totalPixels;
  const darkRatio = darkPixels / totalPixels;

  return {
    isBlue: blueRatio > 0.25,        // >25% blue = likely question slide
    isBlack: darkRatio > 0.90,       // >90% black = failed render
    blueRatio: blueRatio,
    darkRatio: darkRatio,
    verdict: blueRatio > 0.25 ? 'QUESTION_SLIDE'
           : darkRatio > 0.90 ? 'BLACK_FRAME'
           : 'UNKNOWN_CONTENT'       // instructor, end card, etc.
  };
}
```

### Retry Strategy for Failed Captures

If the blue-check fails, try earlier timestamps:

```
Attempt 1: duration - 5
Attempt 2: duration - 10
Attempt 3: duration - 15
Attempt 4: duration - 20
Attempt 5: duration - 30
```

If all 5 attempts fail with `UNKNOWN_CONTENT`, the video likely has no question slide (it is an explanation-only video). Mark it as `NO_QUESTION_SLIDE` in the manifest.

If attempts fail with `BLACK_FRAME`, the video frame did not render. Solutions:
1. Play the video briefly (`video.play()`, wait 500ms, `video.pause()`) then re-capture
2. Try with the tab in foreground
3. Flag for manual review

---

## Phase 4: OCR Transcription

### Goal
Extract the math question text and answer choices from the captured PNG.

### Approach: Claude Vision via Read Tool

Claude Code's `Read` tool is multimodal and can read PNG files directly. This is the best approach for math notation because Claude understands mathematical expressions natively.

**Workflow**:
1. Save PNG to `images/M1-D1-Q1-question.png`
2. Use Claude Code's `Read` tool to view the image
3. Prompt Claude to transcribe in a structured format

**Expected output format**:
```json
{
  "questionText": "What is the value of 1/(2 + 1/(3 + 1/4))?",
  "hasImage": true,
  "answerChoices": [
    {"letter": "A", "text": "13/30"},
    {"letter": "B", "text": "7/15"},
    {"letter": "C", "text": "11/30"},
    {"letter": "D", "text": "3/10"}
  ],
  "mathNotation": "fraction with nested continued fraction",
  "confidence": "high"
}
```

### Alternative: Tesseract.js (not recommended)

Tesseract.js could run in-browser but struggles with:
- Math notation (fractions, exponents, radicals)
- White text on blue background (needs inversion)
- Mixed text + image content
- Roman numeral answer labels

Claude vision is dramatically better for this use case.

### Parallelization

OCR is independent per image. Multiple Claude Code agent threads can transcribe different images simultaneously. Batch size: 10-20 images per parallel agent.

---

## Phase 5: Data Merge

### Goal
Update the app's challenge data with transcribed questions and image URLs.

### Merge Logic

```js
// For each transcribed question:
// 1. Find the matching challenge by module/day/quiz number
// 2. Update fields:
challenge.question = transcription.questionText;
challenge.questionImageUrl = `images/M1-D1-Q1-question.png`;
// Or store as GitHub-hosted URL after commit:
challenge.questionImageUrl =
  `https://raw.githubusercontent.com/grrarr/fufu-daily-challenges/master/images/M1-D1-Q1-question.png`;
```

### Storage

Question PNGs go in `images/` directory in the repo, committed alongside data updates. GitHub Pages serves them directly. Naming convention follows existing pattern: `M{module}-D{day}-Q{quiz}-question.png`.

---

## Phase 6: Batch Orchestration Script

### The Full Pipeline (browser console script)

This runs in the browser console on daily.poshenloh.com. It handles Phase 1 + Phase 2 + Phase 3 in one pass per quiz.

```js
// === Configuration ===
const MODULE_SLUG = 'algebra-basics';
const SEEK_OFFSETS = [5, 10, 15, 20, 30]; // seconds from end
const INTER_QUIZ_DELAY_MS = [3000, 8000]; // random range
const BLUE_THRESHOLD = 0.25;

// === State ===
window.__VQG = {
  manifest: [],      // quiz metadata
  captures: {},      // { 'M1-D1-Q1': dataUrl }
  failures: [],      // quizzes that failed all attempts
  log: []
};

// === Helper: random delay ===
function randomDelay(min, max) {
  return new Promise(r =>
    setTimeout(r, min + Math.random() * (max - min))
  );
}

// === Helper: blue-check on canvas ===
function blueCheck(canvas) {
  const ctx = canvas.getContext('2d');
  const d = ctx.getImageData(0, 0, canvas.width, canvas.height).data;
  let blue = 0, dark = 0, total = canvas.width * canvas.height;
  for (let i = 0; i < d.length; i += 4) {
    const r = d[i], g = d[i+1], b = d[i+2];
    if (r >= 30 && r <= 110 && g >= 40 && g <= 120 && b >= 120 && b <= 210) blue++;
    if (r < 20 && g < 20 && b < 20) dark++;
  }
  return {
    isBlue: blue / total > BLUE_THRESHOLD,
    isBlack: dark / total > 0.90,
    blueRatio: (blue / total).toFixed(3),
    darkRatio: (dark / total).toFixed(3)
  };
}

// === Helper: capture video frame ===
async function captureFrame(video, seekOffset) {
  video.currentTime = video.duration - seekOffset;
  await new Promise(resolve => {
    video.addEventListener('seeked', resolve, { once: true });
  });
  // Extra wait for Wistia to render the frame
  await new Promise(r => setTimeout(r, 600));

  const canvas = document.createElement('canvas');
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  canvas.getContext('2d').drawImage(video, 0, 0);
  return canvas;
}

// === Main: process one quiz ===
async function processQuiz(wistiaHash, quizId) {
  // Navigate to direct Wistia embed
  // (This function would be called by the Chrome MCP navigate tool,
  //  not directly in console -- see orchestration notes below)

  const video = document.querySelector('video');
  if (!video) return { quizId, status: 'NO_VIDEO' };

  // Wait for video metadata to load
  if (video.readyState < 1) {
    await new Promise(resolve => {
      video.addEventListener('loadedmetadata', resolve, { once: true });
    });
  }

  // Try each seek offset until we get a blue frame
  for (const offset of SEEK_OFFSETS) {
    const canvas = await captureFrame(video, offset);
    const check = blueCheck(canvas);

    if (check.isBlue) {
      const dataUrl = canvas.toDataURL('image/png');
      return { quizId, status: 'SUCCESS', dataUrl, offset, check };
    }

    if (check.isBlack) {
      // Frame not rendered -- try playing briefly
      video.play();
      await new Promise(r => setTimeout(r, 800));
      video.pause();
      const retryCanvas = await captureFrame(video, offset);
      const retryCheck = blueCheck(retryCanvas);
      if (retryCheck.isBlue) {
        const dataUrl = retryCanvas.toDataURL('image/png');
        return { quizId, status: 'SUCCESS', dataUrl, offset, check: retryCheck };
      }
    }
  }

  return { quizId, status: 'NO_QUESTION_SLIDE' };
}
```

### Orchestration via Claude in Chrome

The batch pipeline cannot run as a single console script because page navigation resets all JS state. Instead, Claude in Chrome orchestrates it:

1. **Claude navigates** to each quiz page via `navigate` tool (5-12s random delay between pages)
2. **Claude executes** the capture JS via `javascript_tool` (returns base64 PNG data URL)
3. **Claude reads** the result and does blue-check inline
4. **Claude saves** the PNG to disk IMMEDIATELY via Write tool -- `images/M{mod}-D{day}-Q{n}-question.png`
5. **Claude updates** a progress tracker file on disk (`capture-progress.json`) with the quiz ID + status

### Crash resilience

**Key principle: save each capture to disk immediately, never batch in memory.**

- Each PNG is written to `images/` folder on the local Dropbox drive the instant it's captured
- Dropbox syncs to cloud automatically -- survives browser crash, tab close, reboot
- A `capture-progress.json` file tracks which quizzes have been captured successfully
- On restart, read `capture-progress.json` to skip already-captured quizzes
- **Maximum data loss from any crash: 1 screenshot** (the one being captured at crash time)

```json
// capture-progress.json format
{
  "lastUpdated": "2026-04-12T10:30:00",
  "captured": [
    {"id": "M1-D1-Q1", "status": "OK", "file": "images/M1-D1-Q1-question.png"},
    {"id": "M1-D1-Q2", "status": "NO_QUESTION_SLIDE"},
    {"id": "M1-D1-Q3", "status": "OK", "file": "images/M1-D1-Q3-question.png"}
  ],
  "failed": [
    {"id": "M1-D1-Q4", "status": "RETRY_EXHAUSTED", "reason": "all 5 timestamps showed instructor"}
  ]
}
```

This is inherently serial (one tab, one video at a time) but each iteration is fast (~3-7s per quiz).

---

## Test Results (2026-04-12)

### Test 3: Ember sidebar navigation (WORKS)
- Clicking sidebar quiz links triggers Ember router transition -- **no full page reload**
- `window` JS state persists across quiz navigations
- URL updates correctly via pushState
- **Saves ~5s per quiz** vs full `navigate` calls
- Helpers/data stored in `window` survive across quizzes

### Test 4: Chrome MCP screenshot (PARTIAL)
- Can capture the video poster/thumbnail frame without video playback
- **Blocker**: Chrome throttles background tab video loading -- `video.duration` stays `null`, can't jump to end
- The poster frame sometimes shows the question (D1-Q1 did), but this isn't reliable
- Play button overlay is visible on poster frames
- **Requires Chrome tab to be in foreground** for video to load and seek

### Key finding: Background tab video blocker
Chrome won't load video metadata (duration, frames) for tabs not in the foreground. This blocks the `currentTime = duration - 3` seek approach. Solutions:
1. **User keeps Chrome focused** during capture runs
2. **Wistia thumbnail API** -- may return frame at specific timestamp without playback
3. **Poster frame capture** -- skip seeking, just grab whatever frame is showing (works for some quizzes)
4. **Batch when convenient** -- run in foreground bursts

### Navigation strategy: Ember sidebar + View Again
The proven approach combines:
1. Click sidebar link (Ember transition, ~1s, no page reload)
2. Click View Again if completed quiz
3. Wait for video to load (requires foreground tab)
4. Seek to end, screenshot

---

## Estimated Timings

| Phase | Per Quiz | ~500 Quizzes | Notes |
|-------|----------|--------------|-------|
| 1. URL list | 0s | 0 min | Built from existing data, no API calls |
| 2a. Poster URL collection | ~2s | ~17 min | Ember sidebar nav + DOM query, background tab OK |
| 2b. Image download | ~0.1s | ~1 min | Batch download CDN URLs after collection |
| 3. Blue-check | 0.05s | included | Runs on downloaded images |
| 4. OCR transcription | 5s | 42 min | Parallelizable (10x = 4 min) |
| 5. Data merge | 0.1s | 1 min | Batch update |
| **Total (poster approach)** | | **~23 min** | With 10x parallel OCR |
| **Total (Strategy A)** | | **~45 min** | With 10x parallel OCR |
| **Total (Strategy B)** | | **~75 min** | With 10x parallel OCR |

### Re-attempt overhead
Expect ~10-15% of quizzes to need retries (no question slide, explanation-only videos, black frames). Budget an additional ~5-10 minutes for retry passes.

---

## Known Risks and Mitigations

### Risk 1: CORS blocks canvas.drawImage on cross-origin video
- **Likelihood**: High for Strategy B (iframe), Low for Strategy A (direct embed)
- **Impact**: Cannot capture frame via canvas
- **Mitigation**: Use Strategy A (direct Wistia embed) to make video same-origin. Fallback to Chrome MCP screenshot + crop.

### Risk 2: Wistia embed requires authentication or referrer
- **Likelihood**: Medium. Wistia embeds are often public, but PSL may restrict.
- **Impact**: Direct embed URL returns 403 or empty player
- **Mitigation**: Test with one video first. If blocked, add referrer header or fall back to Strategy B. Could also try Wistia's oEmbed API: `https://fast.wistia.com/oembed?url=https://fast.wistia.net/medias/{hash}`

### Risk 3: Video frame not rendered after seek
- **Likelihood**: Medium. Wistia may lazy-render frames.
- **Impact**: Black frame captured
- **Mitigation**: Play briefly after seek, then pause and capture. The blue-check catches this automatically and triggers retry.

### Risk 4: Some videos have no question slide
- **Likelihood**: High (~15-20%). Explanation videos show handwritten work, not a blue question slide.
- **Impact**: All seek offsets fail blue-check
- **Mitigation**: Expected behavior. Mark as `NO_QUESTION_SLIDE` in manifest. These are explanation quizzes that don't have their own question -- they explain a previous question's answer.

### Risk 5: Question slide timing varies widely
- **Likelihood**: Medium. Most are in last 5-10s, but some may be at 30s+ from end.
- **Impact**: Miss the question slide with default offsets
- **Mitigation**: Extended offset list goes up to 30s. Could add 45s, 60s for a second retry pass. Could also scan backward in 5s increments until blue is found.

### Risk 6: Chrome tab throttling prevents frame render
- **Likelihood**: Low if tab is in foreground (Claude in Chrome keeps it focused)
- **Impact**: Black frames consistently
- **Mitigation**: Ensure tab is active. The Wistia player API (if available) may render independently of tab visibility.

### Risk 7: Math OCR accuracy
- **Likelihood**: Low. Claude vision handles math notation well.
- **Impact**: Garbled fraction or exponent transcription
- **Mitigation**: Include confidence rating in output. Flag low-confidence transcriptions for manual review. The PNG is always saved as ground truth.

### Risk 8: Rate limiting / ban from Thinkific or Wistia
- **Likelihood**: Low-Medium for 500 sequential page loads
- **Impact**: 429 errors, temporary blocks, or account suspension
- **Mitigation**:
  - **No batch API calls** -- recon phase eliminated, all data comes from normal page navigation
  - **Random 5-12s delays** between page navigations (same pace as quiz extraction)
  - **Process in batches of 20-30** with 2-5 min pauses between batches
  - **Traffic pattern matches normal usage** -- navigating to quiz pages and watching videos is expected student behavior
  - **Canvas capture is 100% client-side** -- zero network requests for the actual screenshot
  - **OCR runs locally** via Claude's vision model on saved PNGs -- no external API calls
  - If rate-limited: back off exponentially, resume from checkpoint
  - **Key insight**: The capture phase generates the same network traffic as a student clicking through quizzes. The only difference is we jump the video to the end instead of watching it -- but the video is loaded either way.

---

## MVP: What to Build First

### MVP Scope: 1 day (8 quizzes) end-to-end

**Step 1: Test canvas capture on a quiz page** (10 min)
- Navigate to one PSL quiz page in Chrome (normal page load, no extra API calls)
- Jump video to end via `iframe.contentDocument.querySelector('video').currentTime = duration - 5`
- Run canvas capture JS via Chrome MCP javascript_tool
- Verify: does `canvas.drawImage(video)` work from the iframe? If CORS blocks it, fall back to Chrome MCP screenshot + crop
- If yes: Strategy A confirmed. If no: test Strategy B or screenshot fallback.

**Step 3: Test blue-check** (5 min)
- Run blueCheck() on the captured canvas
- Verify thresholds with a known question slide and a known non-question frame
- Tune the blue color range if needed (take a screenshot, inspect actual pixel values)

**Step 4: Capture 8 frames for M1-D1** (5 min)
- Process all 7-8 quizzes for Day 1 (Fraction Gymnastics)
- Save PNGs to images/ folder
- Record which ones pass blue-check and which are NO_QUESTION_SLIDE

**Step 5: OCR transcribe** (10 min)
- Use Claude Code Read tool on each saved PNG
- Transcribe to structured JSON
- Verify accuracy against the actual quiz content

**Step 6: Merge into app** (5 min)
- Update challenge data with transcribed questions
- Commit + push
- Verify on GitHub Pages

### MVP Total: ~40 minutes

### After MVP: Scale Up

1. Build quiz URL list from existing app data (Phase 1) -- no API calls
2. Automate the capture loop across all quizzes
3. Run parallel OCR agents
4. Batch merge all transcriptions

---

## File Layout

```
fufu-daily-challenges/
  index.html                          # Main app (existing)
  CLAUDE.md                           # Project context (existing)
  VIDEO-QUESTION-GRABBER-SPEC.md      # This spec
  # No manifest file needed -- URLs from existing data
  images/                             # Question frame PNGs
    M1-D1-Q1-question.png
    M1-D1-Q2-question.png
    ...
  extraction-scripts/                 # Reusable JS snippets
    capture-frame.js                  # Canvas capture + blue-check
    # No separate recon script needed
    batch-capture.js                  # Full orchestration helper
```

---

## Key Design Decisions

1. **Direct Wistia embed (Strategy A) over full page load (Strategy B)**: 3x faster, avoids CORS issues with canvas capture. Fall back to Strategy B only if Wistia blocks direct access.

2. **Canvas drawImage over Chrome screenshot**: Exact video dimensions, no cropping needed, no UI chrome in the capture. Screenshot is the fallback.

3. **Claude vision over Tesseract.js for OCR**: Math notation is the primary content. Claude handles fractions, exponents, and mixed notation far better than any traditional OCR engine.

4. **Serial capture, parallel OCR**: Video capture is inherently serial (one browser tab), but OCR can fan out. This minimizes total wall time.

5. **Blue-check as automated gate**: Eliminates manual review for ~85% of captures. Only ambiguous cases need human eyes.

6. **PNGs stored in repo**: Simple, version-controlled, served by GitHub Pages. No external image hosting needed. The images are small (~50-150KB each) and the total for 500 quizzes is ~50MB, well within GitHub repo limits.

7. **Retry with earlier timestamps**: Question slides appear at varying distances from the video end. Trying 5 offsets (5s, 10s, 15s, 20s, 30s) covers the vast majority of cases without being excessive.
