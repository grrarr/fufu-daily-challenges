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

### Per-quiz extraction flow

1. **Video page loads** -- skip video to the end (scrub progress bar to right)
2. **Question appears** as an image on blue background at end of video -- screenshot this (question image)
3. **Scroll down** below the video to see multiple choice answers (A/B/C/D/E with roman numerals i/ii/iii/iv/v)
4. **Select the correct answer** and click **CONFIRM** (Claude auto-solves, no need to ask user)
5. **Solution page appears** -- "This answer is correct." followed by detailed worked solution
6. **Extract solution content** -- grab BOTH text and images (see below)
7. Add entry to app data

### Preserving Christopher's completion state
Each quiz shows either a checkmark (100%) or no checkmark (0%/untaken) in the sidebar. This must be preserved:
1. **Note the original state** before touching the quiz
2. Answer correctly to extract the solution content
3. **If the original was 0%/untaken**: retake the quiz and intentionally answer wrong to reset it back to 0%
4. **If the original was 100%**: no need to reset, it's already correct

Also track this in the challenge data -- the `promoted` field is for spaced rep flagging, but the original pass/fail state is useful context.

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
- Questions are displayed as **images on blue background** at the end of videos -- typically need screenshot
- Answer choices below video are **text** (roman numerals mapping to values shown in question image)
- Solutions after answering correctly contain **actual `<img>` elements** with CDN URLs -- grab these directly
- Solution pages often have multiple solution approaches (Solution 1, Solution 2) with separate images

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
- Each module/workout has 4 weeks, each week has ~5 lesson days with daily challenges

### Extraction scope
Currently extracting from completed/near-completed sections only:
- Modules 1, 2, 3
- Workouts 1A, 1B

### Extraction tracker
`extraction-tracker.xlsx` -- spreadsheet with one row per day across all in-scope modules/workouts.
Columns: Module/Workout, Week, Day, Day Topic, Quizzes, Extracted, Fufu Score, Notes.
Filterable by Module/Workout column. Update this as extraction progresses.

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
- Browse tab: filter by module, week, topic, promoted status; text search
- Card view: collapsed question -> click to expand answer + solution + images
- Add tab: form for all fields
- Inline editing on expanded cards
- Export/Import JSON buttons in header

---

## Other files

- `extraction-tracker.xlsx` -- Progress tracker for extraction across modules/workouts
- `fufu-daily-challenges-data.json` -- GitHub-synced data file (auto-created on first sync)
