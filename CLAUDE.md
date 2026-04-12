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

## Key extraction workflow

Daily challenges are on daily.poshenloh.com inside each lesson. The flow to capture them:
1. Navigate to a lesson page on the course
2. The challenge question appears (often with an image/diagram)
3. Student answers -> "You got it right!" page -> click Continue
4. Detailed solution page with step-by-step explanation (often multiple images)
5. Extract: question text, question image URL, answer, solution steps, solution image URLs
6. Paste into the app via the Add form

---

## PSL curriculum structure

- **Module 0**: Introduction
- **Module 1**: Algebra Basics
- **Module 2**: Geometry Tools
- **Module 3**: Algebra Tools
- **Module 4**: Combinatorics Tools
- **Module 5**: Number Theory Tools
- Each module has 4 weeks, each week has multiple lesson days with daily challenges

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
