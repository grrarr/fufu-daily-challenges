# PSL Daily Challenges -- Claude Code Context

## What this is

A collection app for daily challenge questions from Christopher's Po-Shen Loh math curriculum. Companion to the PSL progress tracker (fufu-poshen-loh). Single HTML file hosted on GitHub Pages.

**App URL**: https://grrarr.github.io/fufu-daily-challenges/
**Repo**: https://github.com/grrarr/fufu-daily-challenges
**Source**: `index.html` (single file -- React 18 + Babel standalone, localStorage for persistence)

---

## What this app does (and doesn't)

- **Does**: Collect, browse, filter, and display daily challenge questions + answers + solution steps from the PSL course
- **Does**: Flag challenges for promotion to fufu-spaced-rep
- **Does NOT**: Spaced rep scheduling (that's fufu-spaced-rep's job)
- **Does NOT**: Exam triage workflow (that's fufu-poshen-loh's job)
- **Does NOT**: Progress tracking or percentages

---

## Data structure per challenge

```js
{
  id: timestamp,
  module: 0,              // 0-5
  week: 1,                // 1-4
  day: 3,                 // lesson day within week
  topic: "Fractions",     // short topic label
  question: "What is 1/2 + 1/3?",
  questionImageUrl: "",   // image URL for the question
  answer: "5/6",
  answerImageUrl: "",     // image URL for the answer
  solutionSteps: "Step 1: Find LCD...",
  solutionImageUrls: [],  // array of image URLs for multi-step solutions
  dateAdded: "2026-04-12",
  promoted: false         // true = flagged for promotion to fufu-spaced-rep
}
```

---

## Naming convention

`M{module}-D{day}-Q{n}` -- e.g., M1-D1-Q3 = Module 1, Day 1, Question 3

Within a day, questions are numbered sequentially across all quiz items:
- Q1 = Something to Think About
- Q2 = Challenge Explanation (1 of N)
- Q3 = Challenge Explanation (2 of N)
- ...continues through all Challenge Explanations...
- QN = Your Turn
- QN+1 = Your Turn Explanation (1 of M)
- ...etc.

Example: Day 1 (Fraction Gymnastics) has 7 items = Q1-Q7. Day 2 (Continued Fraction) has 8 items = Q1-Q8.

Image naming: `M1-D1-Q3-question.png`, `M1-D1-Q3-solution-1.png`, `M1-D1-Q3-solution-2.png`

---

## Extraction workflow (via Claude in Chrome)

Daily challenges are on daily.poshenloh.com inside each module's lesson pages.

### Structure per day
Each day (e.g., "Day 1 Challenge: Fraction Gymnastics") contains 7-8 quiz items:
1. **Something to Think About** (QUIZ - 1 QUESTION - PREREQUISITE)
2. **Challenge Explanation (1 of N)** through **(N of N)** (QUIZ - 1 QUESTION) -- typically 3-4
3. **Your Turn** (QUIZ - 1 QUESTION - PREREQUISITE)
4. **Your Turn Explanation (1 of M)** through **(M of M)** (QUIZ - 1 QUESTION) -- typically 2

Every single item has a quiz with 1 question.

### Two cases when landing on a quiz

**Case 1: Already completed** (checkmark in sidebar)
- Shows "You completed [quiz name] / Your score / 100%" page
- Click **VIEW AGAIN** to retake and see the question
- Or click **CONTINUE** to skip to next item

**Case 2: New/untaken** (no checkmark)
- Shows the video + quiz directly

### Programmatic extraction (via Claude in Chrome JS)

Each step is a separate JS call with waits between (page navigation resets window objects).

**Step 1: Navigate to quiz URL**
```js
// Navigate via Chrome MCP navigate tool
```

**Step 2: Detect state + record Fufu score (wait 5s after nav)**
```js
const t = (document.querySelector('main') || document.body).innerText;
const wasCompleted = t.includes('You completed');
const fufuScore = t.includes('100%') && wasCompleted ? '100%' : (wasCompleted ? '0%' : 'untaken');
// fufuScore goes in Excel tracker
```

**Step 3: Click View Again if completed (wait 4s after)**
```js
for (const b of document.querySelectorAll('button, a')) {
  if (b.textContent.trim().includes('View Again')) { b.click(); break; }
}
```

**Step 4: Scroll down to reveal answers (wait 1.5s after)**
```js
window.scrollTo(0, 9999);
```

**Step 5: Click answer A (wait 1s after)**
```js
document.querySelectorAll('div[role="radio"]')[0].click();
```

**Step 6: Click Confirm -- case insensitive (wait 3s after)**
```js
for (const b of document.querySelectorAll('button')) {
  if (b.textContent.trim().toLowerCase() === 'confirm') { b.click(); break; }
}
```

**Step 7: Extract solution data**
```js
window.scrollTo(0, 0);
const t = (document.querySelector('main') || document.body).innerText;
const correctMatch = t.match(/correct answer is '([A-Z])'/i);
const correct = correctMatch ? correctMatch[1] : (t.includes('answer is correct') ? 'A' : 'UNKNOWN');
const resultMatch = t.match(/This answer is (?:in)?correct\.?\s*(?:The correct answer is '[A-Z]'\s*\.?\s*)?([\s\S]*?)(?:\u00a9|Copyright|Click here|Next\b)/i);
const solutionText = resultMatch ? resultMatch[1].trim() : '';
const imgs = (document.querySelector('main') || document.body).querySelectorAll('img');
const imageUrls = Array.from(imgs).map(i => i.src).filter(u => (u.includes('thinkific') || u.includes('cdn')) && !u.includes('logo'));
```

**Step 8: Advance to next quiz**
```js
for (const b of document.querySelectorAll('button')) {
  const t = b.textContent.trim().toLowerCase();
  if (t === 'next' || t === 'continue') { b.click(); break; }
}
```

### Preserving Christopher's completion state
No need to reset quiz state. Just record fufuOriginalScore (100% or 0%/untaken) in the Excel tracker before extracting.
- After extraction/enrichment, check the Ember store for `completedAt: null` entries to detect broken completions
- Fix by re-answering those specific quizzes

### Auto-solve
Claude should solve the math problems and auto-select answers without asking the user. The questions are typically:
- Multiple choice with roman numeral options (i through v or vi)
- Math involving fractions, algebra, geometry, combinatorics, number theory
- Hint is often provided on the question slide

### Image extraction -- prefer actual URLs over screenshots
Solution pages (and sometimes question pages) contain actual `<img>` elements hosted on `files.cdn.thinkific.com`. **Always prefer grabbing the image src URL** over screenshotting:
- Use JS to query `<img>` elements on the page and grab their `src` URLs
- Store these CDN URLs directly in `questionImageUrl`, `answerImageUrl`, or `solutionImageUrls`
- Only fall back to screenshots when the content is rendered in video frames (blue background question slides) or when no `<img>` element exists

### Solution pages have mixed text + images
The solution page often contains a mix of:
- **Text labels**: "Solution 1: Convert to decimals:", "Solution 2: Find the least common denominator:"
- **Images**: Worked-out math steps as `<img>` elements with CDN URLs
- Sometimes the text is trivial (just labels), sometimes it's meaningful
- Capture both: text goes in `solutionSteps`, image URLs go in `solutionImageUrls`
- A single solution page may have 2+ images (e.g., Solution 1 image + Solution 2 image)

### Key observations
- DOM selectors: Answer options are `div[role="radio"]` with class `course-player__interactive-checkbox`. Click by index (0=A, 1=B, etc.)
- **Multi-select quizzes**: Some quizzes use `div[role="checkbox"]` instead of `div[role="radio"]` -- extraction code must handle both
- Button text is mixed case ("Confirm" not "CONFIRM", "Next" not "NEXT") -- always use case-insensitive matching
- Video jump to end: `const iframe = document.querySelector('iframe[src*="play"]'); const vid = iframe.contentDocument.querySelector('video'); vid.currentTime = vid.duration - 3; vid.pause();`
- **0% quiz blocker**: Quizzes where Fufu scored 0% sometimes have PREREQUISITE videos that must be watched before answer choices render. Video jump via `currentTime` doesn't trigger the "watched" event. Workaround: user alt-tabs to Chrome to let video play in foreground.
- **Prerequisite chain breakage from extraction**: During the quiz extraction and enrichment passes, some quizzes were left in an "incomplete" state (viewed but not completed). This broke the prerequisite chain:
  - The platform requires ALL quizzes in a day/section to be "completed" (any score) before the next section unlocks
  - Our "View Again" + answer approach sometimes didn't fully complete the quiz (the completion didn't persist)
  - The Ember Data Store (`course-player-v2` app, accessible via `app.__container__.lookup('service:store').peekAll('user-content')`) shows which quizzes have `completedAt: null`
  - Only 6 quizzes were actually broken in M1 (all in D1 and D2), not the 118 the sidebar suggested
  - Fix: manually complete the 6 broken quizzes -- must click through FULL flow: Answer → Confirm → Next (solution page) → score page loads → Continue. Completion only registers after reaching the score page.
  - Prevention: in future extraction, always click through the full flow. Skipping the final Continue/Next leaves the quiz in "viewed but not completed" limbo state.
- Wrong answers still show the solution and state the correct answer letter
- Page navigation resets all JS window objects -- cannot persist helpers across pages
- Fufu's original scores tracked in extraction-tracker.xlsx, not in the app
- **Question poster images**: ALL quizzes have their own unique question -- including "Challenge Explanation" and "Your Turn Explanation" quizzes. Capture poster images for ALL quizzes, not just "Something to Think About" and "Your Turn". The poster for main quizzes shows a blue question slide; the poster for explanation quizzes may show the instructor with handwritten work (still useful as the question context). Every quiz gets a poster image.
- Poster URLs from Wistia CDN: `iframe[src*="play"].contentDocument.querySelector('img').src` gives `https://embed-ssl.wistia.com/deliveries/{hash}.jpg` -- clean image, no play button overlay. No OCR needed -- many questions have diagrams.
- **IMPORTANT: Always download images locally to Dropbox and point to GitHub copy.** NEVER trust external CDN URLs (Wistia URLs can expire/change). Workflow: (1) grab Wistia poster URL, (2) download JPG to `images/` folder on Dropbox, (3) set `questionImageUrl` to `https://raw.githubusercontent.com/grrarr/fufu-daily-challenges/master/images/{filename}`, (4) commit images to git. Naming: `M{mod}-D{day}-{stta|yt}-question.jpg`
- Solutions after answering correctly contain **actual `<img>` elements** with CDN URLs -- grab these directly
- Solution pages often have multiple solution approaches (Solution 1, Solution 2) with separate images

### Programmatic extraction via Thinkific API

An alternative to the step-by-step DOM extraction above. Uses Thinkific's internal API to fetch quiz data directly, which is faster and more reliable for enriching existing challenges with solution text and image URLs.

**Course structure endpoint**:
```
GET /api/course_player/v2/courses/{course-slug}
```
Returns all content IDs (chapters, lessons, quizzes) for a course.

**Quiz data endpoint**:
```
GET /api/course_player/v2/quizzes/{contentable-id}
```
Returns quiz data including:
- `questions[0].text_explanation` -- HTML string with solution text + embedded `<img>` tags
- `choices[].credited` -- base64-encoded correct answer indicator

**Required headers**:
- `credentials: 'include'` (uses session cookies from logged-in browser)
- `x-requested-with: XMLHttpRequest`

**Extracting images from HTML explanation**:
```js
const div = document.createElement('div');
div.innerHTML = question.text_explanation;
const imgs = div.querySelectorAll('img');
const imageUrls = Array.from(imgs).map(i => i.src);
```

**Batch fetching approach**:
- Can run as fire-and-forget background script in browser console
- Store results in `window.__ALL_RESULTS`
- Use random 5-12s delays between requests to avoid detection
- ~57 quizzes fetched in ~8 minutes with random delays
- Results downloaded via `document.createElement('a')` + Blob download (may need to allow downloads in Chrome settings)

**Security filter note**: `atob()` on the `credited` field gets blocked by Claude's security filter, but the solution text and images work fine -- focus on those.

**Best workflow**: Extract question text via DOM (screenshots for video-based questions), then enrich with solution text + images via API in bulk.

---

## PSL curriculum structure

- **Module 0**: Introduction (skip for now -- not extracting)
- **Module 1**: Algebra Basics
- **Module 2**: Geometry Tools
- **Module 3**: Algebra Tools
- **Module 4**: Combinatorics Tools (15% complete -- not extracting yet)
- **Module 5**: Number Theory Tools (0% -- not extracting yet)
- **Workout 1A**: Algebra Basics
- **Workout 1B**: Algebra Basics
- Each module/workout has 4 weeks with daily challenges

### Module 1 day topics (16 days, 4 per week)
- **W1**: D1 Fraction Gymnastics, D2 Continued Fraction, D3 Comparing Fractions, D4 Sports Averages
- **W2**: D5 Natural Proportion, D6 Keeping in Proportion, D7 Changing Percentages, D8 Undoing Percentages
- **W3**: D9 Business Accounting, D10 Estimating Profit, D11 Gear Ratios, D12 Winning Ratios
- **W4**: D13 Working Together, D14 100 Bottles, D15 Harmonic Mean, D16 Changing Speed

### Course slugs (for API and URLs)
- M1: `1-algebra-basics`
- M2: `2-geometry`
- M3: `3-algebra`
- W1A: `1a-algebra`
- W1B: `1b-algebra`

### Extraction scope and quiz counts
| Module/Workout | Quizzes | Solution text + images | Question poster images | Notes |
|----------------|---------|----------------------|----------------------|-------|
| M1 (Algebra Basics) | 123 | 123/123 DONE | 32/123 (STTA+YT only) | Missing 91 explanation quiz posters |
| M2 (Geometry Tools) | 126 | 126/126 DONE | 0/126 | Not started |
| M3 (Algebra Tools) | 157 | 157/157 DONE | 0/157 | Not started |
| W1A (Algebra Basics) | 90 | 90/90 DONE | 33/90 (main Qs only) | Missing explanation quiz posters |
| W1B (Algebra Basics) | 92 | 92/92 DONE | 0/92 | Not started |
| **Total** | **588** | **588/588** | **65/588** | |

### Extraction tracker
`extraction-tracker.xlsx` -- spreadsheet with one row per day across all in-scope modules/workouts.
Columns: Module/Workout, Week, Day, Day Topic, Quizzes, Extracted, Fufu Score, Notes.
Filterable by Module/Workout column. Update this as extraction progresses.

---

## Current status and next steps (as of 2026-04-12)

### What's DONE
1. **App built and deployed** -- `index.html` on GitHub Pages, fully functional browse/filter/expand UI
2. **All 588 challenges extracted** -- M1 (123), M2 (126), M3 (157), W1A (90), W1B (92)
3. **All 588 enriched with solution text + image URLs** -- via Thinkific quiz API batch fetch
4. **65 question poster images captured** -- downloaded to `images/` folder, URLs point to GitHub raw content
   - M1: 32 images (STTA + YT for each of 16 days): `M1-D{n}-stta-question.jpg`, `M1-D{n}-yt-question.jpg`
   - W1A: 33 images (main questions Q1-Q33): `W1A-Q{n}-{topic}.jpg`
5. **GitHub sync working** -- PAT in localStorage, `fufu-daily-challenges-data.json` syncs both directions

### What's LACKING (priority order)
1. **523 question poster images still needed** -- the main remaining task
   - M1: 91 explanation quiz posters (Challenge Explanation + Your Turn Explanation quizzes)
   - M2: all 126
   - M3: all 157
   - W1A: ~57 explanation quiz posters
   - W1B: all 92
2. **W1B challenges lack distinguishing marker** -- stored as `module: 0` same as W1A, no `quizUrl` field set. Need to add quizUrl or a workout field to tell W1A from W1B.
3. **D9 enrichment gap** -- D9 challenges have minimal solution text ("see video" boilerplate from API)

### Poster capture -- what works and what failed

**What WORKS:**
- Direct page navigation to quiz URL → wait for load → read `iframe[src*="play"].contentDocument.querySelector('img').src` → gives Wistia CDN poster URL (`https://embed-ssl.wistia.com/deliveries/{hash}.jpg`)
- On "You completed" pages, the video iframe IS present and shows the poster -- no need to click View Again
- Downloading JPG from Wistia URL and saving to `images/` folder locally

**What FAILED (April 12 session):**
- **Ember sidebar scrape**: Navigated 60 M1 quizzes via sidebar clicks (Ember transitions, no full page reload). Got 0 poster URLs. The completion pages loaded via Ember sidebar navigation don't always render the video iframe -- the poster image element isn't in the DOM.
- **Play endpoint API** (`/api/course_player/v2/contents/{id}/play/{playId}`): Returns 404 for most quizzes. The playId is per-content-specific (not a global enrollment ID). Only worked for 1 quiz where we had the exact ID from a prior iframe src.

**Recommended approach for next session:**
- Use **direct URL navigation** (full page reload per quiz) instead of Ember sidebar clicks
- Slower (~8-10s per quiz vs ~3s for sidebar) but reliably renders the video iframe
- For 523 remaining quizzes at ~10s each = ~87 minutes total
- Can run as autonomous JS loop: navigate → wait 5s → read poster from iframe → store result → navigate next
- Alternative: try to get Wistia hashes from the course structure API or page source, then construct poster URLs directly without navigation

**CRITICAL: Never click "View Again" during poster scrape** -- it creates prereq chain holes. Read poster from the completion page directly.

### Prereq chain issues (lessons learned)
- The platform requires ALL quizzes in a day/section to be "completed" before the next section unlocks
- "View Again" starts a new quiz attempt -- if you don't complete the FULL flow (Answer → Confirm → Next → score page → Continue), the quiz gets stuck in "incomplete" state
- This breaks the prereq chain for all subsequent quizzes in that day
- The Ember Data Store shows broken completions: `app.__container__.lookup('service:store').peekAll('user-content').filter(u => !u.completedAt)`
- **After fixing prereqs, a full page reload is needed** to clear the Ember cache -- stale prereq errors persist otherwise
- During the April 12 session, prereqs were broken and fixed manually multiple times in M1 D1-D2

### Data file structure notes
- `fufu-daily-challenges-data.json` -- 588 challenges, each with fields per the data structure above
- Module field is numeric: 0 = Workouts (W1A + W1B mixed together), 1 = M1, 2 = M2, 3 = M3
- W1A challenges can be identified by `quizUrl` containing `1a-algebra` (33 have this set)
- W1B challenges have NO `quizUrl` set -- they're indistinguishable from W1A explanation quizzes without quizUrl
- `questionImageUrl` points to `https://raw.githubusercontent.com/grrarr/fufu-daily-challenges/master/images/{filename}` for the 65 that have images

### Files on disk
- `images/` -- 65 JPG poster images (32 M1 + 33 W1A), committed to git
- `extracted-data-m1.json` -- raw M1 extraction data (123 quizzes with correct answers, Fufu scores)
- `poster-progress.json` -- stale progress file from earlier poster URL collection attempt (can ignore)
- `VIDEO-QUESTION-GRABBER-SPEC.md` -- design spec for poster capture pipeline (partially outdated)

---

## Rules

- **At the start of every session, display the app URL**: https://grrarr.github.io/fufu-daily-challenges/
- **Be autonomous** -- Don't ask for confirmation on routine commits/pushes. Just do them.
- **Always `git push` after committing** -- The app is served from GitHub Pages
- **No emoji or unicode decoration in data** -- Plain ASCII/standard characters only
- **Storage keys (DO NOT CHANGE)**:
  - `fufu-daily-challenges-v1` (primary)
  - `fufu-daily-challenges-v1-backup` (backup)
- **Save wipe guard**: Never overwrite N items with 0
- **GitHub sync data file**: `fufu-daily-challenges-data.json`

---

## GitHub deployment

- **GitHub Pages**: Hosted at https://grrarr.github.io/fufu-daily-challenges/
- **Repository**: `grrarr/fufu-daily-challenges` (branch: `master`)
- **Auto-deploys** on every push (~30 seconds)

---

## App architecture

- Single `index.html` file -- React 18 + Babel standalone via CDN
- localStorage for persistence with backup key
- GitHub sync: `ghFetch` (with download_url fallback for >1MB files) and `ghWrite`
- PAT stored in localStorage under `fufu-dc-gh-token`
- Browse tab: filter by module, week, topic, promoted status, fufuOriginalScore (Got Wrong / Got Right); text search
- Card view: collapsed question -> click to expand answer + solution + images
- Expand All / Collapse All buttons for filtered cards (expandedIds is a Set, not single ID)
- Add tab: form for all fields
- Inline editing on expanded cards
- Export/Import JSON buttons in header
- fufuOriginalScore field on challenges (tracks Fufu's original quiz score before extraction)

---

## Other files

- `extraction-tracker.xlsx` -- Progress tracker for extraction across modules/workouts
- `fufu-daily-challenges-data.json` -- GitHub-synced data file (auto-created on first sync)
