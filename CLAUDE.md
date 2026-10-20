# CLAUDE.md — Map of `index.html`

Single-file Hebrew/RTL university GPA calculator, branded **GradeWise**, dark "academic terminal" theme (HTML + Tailwind CDN + vanilla JS, no build step). This file is a navigation index into `index.html` so you can jump straight to the relevant lines instead of re-reading the whole file. **Whenever you edit `index.html` in a way that adds/removes/moves a function or section, update the line numbers below.**

## How the app works (read this first)

- All app state lives in one in-memory object: `state = { years: [...], filter: {...} }` (index.html:114), auto-saved to `localStorage` on every mutation — see Persistence below.
- Hierarchy: `state.years[]` → `year.semesters[]` → `semester.courses[]` → `course.components[]` (exam / quiz / assignment rows — see Component types below).
- Two render paths, both important to understand before touching render code:
  - `render()` (index.html:1405) — full rebuild of `#dashboard`, `#recommendation`, `#filter-bar`, `#years-root` via `innerHTML`. Used after **structural** changes (add/delete year/semester/course/component, toggle binary, toggle Magen checkbox, change filter, apply a recommendation). Destroys and recreates every DOM node in those containers.
  - `updateComputedOnly()` (index.html:1416) — **targeted** patch of just the grade badge, weight-sum text, per-row Magen "not included" styling, dashboard numbers, and the recommendation banner, using `querySelector` + direct property assignment (no `innerHTML`). Used on every keystroke in a grade/weight/credits/bonus field (via the `input` listener, index.html:1572-1632) so the focused input never loses focus/cursor position.
  - **Do not call `render()` from a `change`/`blur` handler on a text-editing field.** That was tried and reverted (see comment at index.html:1653-1657) — it destroys the input the user just clicked into and silently eats their next keystroke. If you need a new computed value to refresh live, add a `data-*` hook and patch it inside `updateComputedOnly()` instead.
- All interactivity is event-delegated on `document` (three listeners: click / input / change, index.html:1528 onward) using `data-action` / `data-field` attributes — no per-element `addEventListener` calls, so re-rendered HTML never needs listeners re-attached.

## Persistence (`localStorage`, index.html:119-143)

- **Auto-save only, no accounts.** `persistState()` (index.html:123-130) writes `{ years, filter }` to `localStorage` under the key `"gradewise-state-v1"` on every mutation. It's called from `render()`, from inside `updateComputedOnly()`, and explicitly from the two `input`-listener branches that skip `updateComputedOnly()` (`course-name`, `component-name`, index.html:1572-1632) — every state-mutating code path ends up persisting.
- `loadPersistedState()` (index.html:131-143) runs once at the bottom of the IIFE, before the first `render()` (index.html:1692-1694). It's defensive: wrapped in try/catch, requires `parsed.years` to be an array, and silently starts fresh (empty state) on any parse failure or missing data — never throws, never blocks load.
- Storage is **per-browser, per-device, per-origin** — there is no login and no server. Two different people opening the same deployed URL each get their own independent `localStorage`, automatically isolated; nothing is shared or synced between them, and switching browsers/devices loses the trail (by design — see the AskUserQuestion decision in conversation history if reconsidering cloud sync later).
- **In-app browser warning + persistent storage request** (init block at the bottom of the IIFE): `checkInAppBrowser()` shows the static `#inapp-warning` banner when the user-agent matches a known in-app browser (Instagram, Facebook/Messenger, WhatsApp, LinkedIn, Snapchat, TikTok, LINE) — those embedded browsers often wipe storage when closed, which is why friends had to re-type courses on every visit. Dismissed via `data-action="dismiss-inapp"` (not persisted). `navigator.storage.persist()` is also requested on load (Chrome/Edge/Firefox honor it; Safari may ignore it).
- A one-time toast ("הנתונים שוחזרו מהדפדפן") fires on load only when data was actually restored, so a first-time visit with no saved state stays silent.
- `resetAll()` calls `persistState()` after clearing `state.years`, so "אפס הכל" also wipes the saved copy — otherwise a reload would immediately restore the just-cleared data.
- Extra defensive saves on `visibilitychange` (when the tab goes hidden) and `pagehide` (index.html:1519-1526) — mobile browsers, especially iOS Safari/WebKit, can suspend or kill a backgrounded tab without a reliable `beforeunload`, so these fire `persistState()` on every signal the page might be about to disappear. Every mutation already persists synchronously and immediately (not debounced), so this is redundant insurance, not the primary save path — if data still doesn't survive a close/reopen on a given device after this, the cause is the OS/browser evicting already-written `localStorage`, not a missed save.

## Manual backup (export/import, index.html:359-394)

- Added because `localStorage` alone isn't fully reliable long-term — iOS in particular is known to evict site storage for non-installed web pages more aggressively than desktop browsers, so a course list entered only on a phone can vanish between sessions with no error surfaced anywhere.
- `exportBackup()` (index.html:359-372) serializes `{ years, filter, exportedAt }` to a downloaded `gradewise-backup-YYYY-MM-DD.json` via a `Blob` + temporary `<a download>` — no server round-trip, works entirely client-side.
- `importBackupFile(file)` (index.html:373-394) reads the file via `FileReader`, validates `parsed.years` is an array before touching state, requires a `confirm()` since it **replaces** `state.years` wholesale (not a merge), then calls `render()` (which persists via the normal path).
- Wired to two header buttons ("ייצוא גיבוי" / "ייבוא גיבוי", index.html:73-78) plus a hidden `#import-file-input` file picker that "ייבוא גיבוי" triggers by proxy (`data-action="import-backup"` → `.click()` on the file input, index.html:1555-1556) — the file input's own `change` listener (index.html:1502-1506) is separate from the main delegated `input`/`change` listeners since it needs the raw `FileList`, not a `data-field`.

## Component types (exam / quiz / assignment)

Three component types exist on `course.components[]`, each with different Magen eligibility — this was a deliberate change from an earlier version that only had "exam" and "assignment":

| Type | Hebrew | Magen-eligible? | Notes |
|---|---|---|---|
| `"exam"` | מבחן | **No, never** | The anchor grade Magen components protect against. Every new course seeds exactly one (`"מבחן סופי"`, weight 100) in `addCourse` (index.html:197-213); there is no "add exam" button — only the seeded one exists unless the user renames/repurposes a row. |
| `"quiz"` | בוחן | Yes | Added via the "+ הוספת בוחן" button (index.html:1335, `data-type="quiz"`), rendered together with `"exam"` rows in the same "מבחנים ובחנים" section (`renderCourseBody`, index.html:1316-1368 — the `exams` array there is actually `type==="exam" || type==="quiz"`, named for legacy/brevity). |
| `"assignment"` | עבודה | Yes | Added via "+ הוספת עבודה", rendered in its own "עבודות" section. |

`renderComponentRow` (index.html:1369-1398) shows the "מגן" checkbox and the "לא נכלל" dropped-label only for `type === "quiz" || type === "assignment"` — never for `"exam"`.

## Magen logic (`computeCourseGrade`, index.html:283-390)

- **Anchor** = the `"exam"`-type component with the highest weight (index.html:288-292). Since exam-type is never Magen-eligible, there's no need to filter out `isMagen` here anymore.
- For every `quiz`/`assignment` component with `isMagen === true`: if its grade is higher than the anchor's grade, it counts normally; otherwise it's added to `droppedIds` and its weight is redistributed *proportionally* across the remaining active components (index.html:294-379).
- If a course has no `"exam"`-type component at all (user deleted the seeded one), `anchor` is `null` and Magen components are never dropped — same edge case as before, just re-scoped to the new type split.

## Binary recommendation ("what if I mark this course pass/fail")

- `recommendBinarySwap()` (index.html:959-979) simulates, for every non-binary numeric course with a valid graded result, what the **cumulative** GPA would become if that one course were switched to binary (excluded from the GPA sum, credits still counted) — using `computeStats`'s now-exposed `weightedSum` field (index.html:392-950) to avoid recomputing from scratch. It returns the single swap with the largest positive delta (`> 0.005`), or `null` if none would help (e.g. 0-1 graded courses, or every course is already at-or-above the cumulative average).
- `renderRecommendation()` (index.html:1067-1092) renders a teal-accented banner into `#recommendation` (empty string if no recommendation) with a "סמן כעובר בינארי" button (`data-action="apply-recommendation"`).
- `applyRecommendation(courseId)` (index.html:262-268) sets `course.isBinary = true` and re-renders.
- This is always scoped to the **whole-degree cumulative** GPA regardless of the active filter tab — it does not change per semester/year view.
- Wired into both render paths: full `render()` and the live `updateComputedOnly()`, so the banner's numbers (and whether it shows at all) update on every grade keystroke, not just on structural changes.

## Demo data ("טען נתוני דמו", index.html:306-354)

- Added so a first-time visitor (or a recruiter) can see every feature without typing grades. **All course names and grades are fictional** — never put real user data here.
- `buildDemoYears()` (index.html:306-345) returns a fresh `years[]` each call (new `uid()`s), built with small local factory helpers (`exam`, `quiz`, `task`, `course`, `semester`, `year`). The data is chosen to exercise every feature: a Magen quiz that counts (חדו"א 1), a Magen quiz that is dropped and redistributed (חדו"א 2, seeded `expanded: true` so the "לא נכלל" label is visible), a binary course, a low-grade 4-credit course that triggers the pass/fail recommendation (פיזיקה 1), and a current semester with ungraded finals.
- `loadDemoData()` (index.html:347-354) is a no-op if `state.years` is non-empty — the button only exists in the empty state of `renderYearsRoot()`, so it can never overwrite real data and needs no `confirm()`. It persists through the normal `render()` path; "אפס הכל" clears it.
- Wired via `data-action="load-demo"` in the click listener.

## Screenshot import — OCR ("ייבוא מצילום מסך", index.html:396-927)

Reads a screenshot of the **BGU app's grades screen** (title "ציונים 2026 סמסטר א", one white card per course, a blue grade box on top of an orange credits box) and turns it into courses. **Runs entirely in the browser** — Tesseract.js is lazy-loaded from jsDelivr on first use (~7 MB: core wasm + Hebrew + English models, cached by the browser afterwards); the image is never uploaded anywhere. Chosen over a server-side AI model: free, no backend, grades stay on the device (see the portfolio repo's DECISIONS.md).

Pipeline:
1. `fileToCanvas()` — returns `{ full, small, f }`: `full` is the screenshot at its **original resolution** (all text is read from it), `small` is scaled to `ANALYSIS_WIDTH = 940`px and used **only** to find the layout, so pixel thresholds are device-independent. `f` converts small → full coordinates.
2. `analyzeLayout()` (index.html:527) — **no OCR, color only.** A "box row" has many strongly-colored pixels (max−min channel > 90) on the left half and almost none on the right half — that excludes the full-width orange status bar and section headers. Consecutive rows group into bands, classified blue (grade) or orange (credits). Grade band directly above a credits band = one course; a credits band alone = a course with no grade yet (current semester). The card's height is measured in the white margin **left** of the boxes (a probe on the right side hits long, wrapped course names); the name crop is everything right of the boxes.
3. Reading — OCR per field, never on the whole image (whole-image OCR gave silent credit errors like `770.00` for `0.00`).
   - **Two Tesseract workers** (`getOcrWorkers()`): `heb+eng` PSM 6 for names/title/"עובר", and a separate **English-only** worker (whitelist `0123456789.`, PSM 13) for numbers. With the Hebrew model loaded, digit reads were much worse ("85.00" → "0").
   - `boxToCanvas(img, rect, threshold)` — lightness = min(R,G,B) from the **full-resolution** image, resized to 2× the small-box height with **our own bilinear interpolation in JS**, thresholded, padded. **Do not replace this with canvas `drawImage` scaling**: browser smoothing differs between Chrome and Safari, and on a real iPhone "88.00" came out as "38.00". (A blocky nearest-neighbor 3× upscale was also tried earlier: thin strokes, "85.00" → "35.00".)
   - `readNumberVotes()` — every number is read at **5 thresholds** (100/120/150/180/200); a value is accepted only with **≥2 agreeing reads and no tie**, otherwise `null` → the field is flagged amber. Rule: **never guess silently — flag instead.**
   - `readGrade()`; a grade box without an accepted number is re-read as text: "עובר" → binary course.
   - `readCredits()` + `findCreditsCut()` — the box reads "4.50 נק״ז"; the cut point between the Hebrew letters and the number is found by scanning columns for white-text ink and cutting in the middle of the **widest empty run** (fixed 47%/52% cuts failed when the number's width changed, e.g. "0.00"; they remain only as a fallback).
   - Title: OCR sometimes returns the RTL title reversed ("סמסטר א 2026"), so both orders are matched.
4. Review modal (`renderImportModal()`, `handleImportField()`) — every value is editable; unsure fields are amber; 0-credit courses and courses already in the target semester (`markDuplicates()`, name-normalized) are unchecked by default. Year/semester come from the title (`guessYearIndex()`: the earliest calendar year in the batch = first study year, or counted from a year already linked by an earlier import via `year.academicYear`) and can be changed per screenshot.
5. `confirmScreenshotImport()` — `ensureSemester()` creates missing years/semesters in order; each course gets one `"exam"` component "ציון סופי" at weight 100 with the final grade (empty for binary/ungraded).

- **Typing in the review modal never re-renders it** (same focus rule as `updateComputedOnly()`); only checkbox/select `change` events re-render. Modal fields use `data-imp` attributes, so the main `data-field` listeners ignore them.
- An unreadable file (not an image / corrupted) becomes a shot with `unreadable: true` and its own message — it never cancels the other files. Only a failure to load the OCR engine itself closes the modal with the "check your internet" toast.
- Values typed in the review table are clamped with `clampNumber()` on import (grade 0–100, credits 0–999), same as the main form.
- Year guess caveat: with no year linked yet (`academicYear`), a single screenshot is always mapped to שנה א׳ — the user sees "זוהה: 2027 סמסטר ב" next to the year select and can change it. After the first import links a year, later imports count from it.
- `year.academicYear` is a new optional field (set only by imports). Old saved data without it keeps working.
- Verified on 4 real screenshots plus resized/JPEG variants (750–1290px wide, 24 runs, ~300 numbers): **0 wrong values, 0 flags**. The first live test on a real iPhone found the 88→38 bug that led to full-resolution reads, own interpolation, voting and the gap-based cut. Test screenshots are private: they live in `test-shots/` (git-ignored) and are **not** in the repo.

## Brand & theme reference

- Name "GradeWise" is a deliberate English wordmark inside an otherwise all-Hebrew UI — it appears only in `<title>` (index.html:6) and the header (index.html:56-65). Don't translate or Hebraize it; don't add more English strings elsewhere.
- Dark-only theme, no light mode / no toggle. Color tokens are defined once in `tailwind.config` (index.html:12-28) and used everywhere via Tailwind class names — **never hardcode a hex color in a template string; add/extend a token in the config instead** so the palette stays centralized:
  | Token | Hex | Use |
  |---|---|---|
  | `ink` | `#0d1420` | page background |
  | `panel` | `#161f30` | card background |
  | `panel2` | `#111a2c` | recessed/nested surfaces (semester wrapper, inputs, year-header strip) |
  | `line` | `#253150` | all borders (no shadows — elevation is border-only per the approved design) |
  | `ink2` | `#e8ecf5` | primary text |
  | `muted` | `#7c8bab` | secondary/placeholder text |
  | `accent` | `#4ee3c2` | teal — GPA numbers, primary buttons/links, active filter chip, checkbox accent-color, recommendation banner |
  | `accentBg` | `#0f2b26` | dark teal tint — ≥90 grade-badge background, recommendation banner background |
- Grade-band badge colors (`gradeColorClasses`, index.html:993-1000) are tuned for the dark background: ≥90 teal (`accent`/`accentBg`), ≥80 blue-950/blue-300, ≥70 amber-950/amber-300, ≥60 orange-950/orange-300, else red-950/red-300. Binary courses get a distinct purple badge and weight-mismatch gets its own amber badge (`courseGradeBadgeHTML`, index.html:1002-1016) — kept visually separate from the grade bands since it's an actionable warning, not a grade.
- Numeric values are set in `font-mono` (JetBrains Mono, loaded alongside Heebo at index.html:9) everywhere a number appears: grade badges, dashboard stat numbers, weight/grade/credits/bonus inputs, weight-sum text, recommendation banner numbers. Labels, buttons, and Hebrew copy stay in Heebo (the default `font-sans`).
- Native checkboxes use `accent-color: #4ee3c2` (index.html:36). Number inputs have their native up/down spinner arrows removed (`-webkit-appearance: none` / `appearance: textfield`, index.html:43-44) — users type values directly, no steppers.

## Section map (index.html)

| Lines | What's there |
|---|---|
| 1–47 | `<head>`: meta, title, Google Fonts (Heebo + JetBrains Mono), Tailwind CDN + `tailwind.config` (color tokens + font families), custom `<style>` (checkbox accent-color, no-spinner number inputs, scrollbar, `.fade-in`, `.chip-active`) |
| 48–110 | Static HTML shell: `#import-modal` (OCR review overlay), `#toast-root`, header (wordmark + "ייבוא מצילום מסך" / export / import / reset buttons + hidden `#import-file-input` and `#screenshot-input`), empty `#dashboard` / `#recommendation` / `#filter-bar` / `#years-root` containers, "הוספת שנת לימודים" button |
| 114–1696 | `<script>` — everything below is inside one IIFE `(function(){ "use strict"; ... })()` |

### State & lookups
| Lines | What's there |
|---|---|
| 121 | `HEBREW_ORDINALS` |
| 123–126 | `state` object shape |
| 128–152 | `STORAGE_KEY`, `persistState()`, `loadPersistedState()` — see Persistence above |
| 154–156 | `uid()` |
| 158–161 | `ordinalYearName(index)` |
| 163–168 | `nextSemesterName(count)` |
| 173–198 | `findYear`, `locateSemester`, `locateCourse` |
| 195–198 | `allCourses()` |

### Mutations (all call `render()` at the end)
| Lines | What's there |
|---|---|
| 204–248 | `addYear`, `deleteYear` (confirm), `addSemester`, `deleteSemester` (confirm), `addCourse` — seeds **one `"exam"` component** (`"מבחן סופי"`, weight 100) |
| 249–284 | `deleteCourse` (confirm), `toggleCourseExpanded`, `addComponent(courseId, type)` — `type` is `"quiz"` or `"assignment"` (never `"exam"` from the UI), `deleteComponent` |
| 285–297 | `resetFilter()`, `resetAll()` (confirm + `persistState()` + toast) |
| 298–304 | `applyRecommendation(courseId)` — sets `isBinary = true`, re-renders, toasts |
| **306–354** | **`buildDemoYears()`, `loadDemoData()`** — see Demo data above |
| **359–394** | **`exportBackup()`, `importBackupFile(file)`** — see Manual backup above |
| **396–927** | **Screenshot import (OCR)**: `getOcrWorkers`, `ocrText`, `fileToCanvas`, `cropCanvas`, `boxToCanvas`, `cropFull`, `analyzeLayout`, `parseOcrNumber`, `readNumberVotes`, `readGrade`, `findCreditsCut`, `readCredits`, `readScreenshot`, `guessYearIndex`, `markDuplicates`, `startScreenshotImport`, `ensureSemester`, `confirmScreenshotImport`, `closeImportModal`, `renderImportModal`, `handleImportField` — see Screenshot import above |

### Calculations — **the core grading logic**
| Lines | What's there |
|---|---|
| 932–935 | `num(v)` |
| 937–939 | `weightSum(components)` |
| **942–999** | **`computeCourseGrade(course)`** — see Magen logic above |
| 1001–1024 | `computeStats(courseList)` — returns `{ gpa, weightedSum, totalCredits, gpaCredits, excludedBinaryCredits, invalidCount, courseCount }`; `weightedSum` is exposed specifically so `recommendBinarySwap` can simulate without recomputing |
| 1027–1029 | `semesterStats`, `yearStats`, `cumulativeStats` |
| **1035–1055** | **`recommendBinarySwap()`** — see Binary recommendation above |

### Render helpers
| Lines | What's there |
|---|---|
| 1059–1062 | `esc(str)` |
| 1064–1067 | `fmt(n, digits)` |
| 1069–1076 | `gradeColorClasses(grade)` |
| 1078–1092 | `courseGradeBadgeHTML(course)` |

### Dashboard, recommendation, filter bar
| Lines | What's there |
|---|---|
| 1093–1117 | `renderDashboard()` |
| 1118–1131 | `statCardHTML(label, sub, stats)` |
| 1132–1142 | `creditsCardHTML(cumulative)` |
| **1143–1168** | **`renderRecommendation()`** — see Binary recommendation above |
| 1169–1202 | `renderFilterBar()` |
| 1203–1218 | `visibleYears()`, `visibleSemesters(year)` |

### Course tree (`#years-root`)
| Lines | What's there |
|---|---|
| 1219–1245 | `renderYearsRoot()` — empty state includes the "טען נתוני דמו" and "ייבוא מצילום מסך" buttons |
| 1246–1266 | `renderYearBlock(year)` — `data-year-summary="{id}"` |
| 1267–1286 | `renderSemesterBlock(semester)` — `data-semester-summary="{id}"` |
| 1287–1315 | `renderCourseCard(course)` — `data-grade-badge="{id}"`. Two-line layout: line 1 is expand toggle + name input + delete button only; line 2 (`flex-wrap`) holds the binary checkbox, credits input, and grade badge. Split specifically so no combination of controls is forced onto one non-wrapping row — see the mobile-overlap bug this fixed, noted below |
| 1316–1368 | `renderCourseBody(course, result)` — exam+quiz section, assignment section, `data-weight-summary`, bonus input, `data-dropped-count="{id}"` |
| 1369–1398 | `renderComponentRow(course, comp, result)` — `data-component-row="{comp.id}"`. Two-line layout: line 1 is the name input + delete button; line 2 (`flex-wrap`) holds labeled weight/grade fields (small `text-[10px]` "משקל"/"ציון" captions stacked above each input — not just the `title` hover tooltip, which is inaccessible on touch devices) plus the Magen checkbox/label and dropped-indicator, shown only for `quiz`/`assignment` |

### Render orchestration
| Lines | What's there |
|---|---|
| 1405–1411 | `render()` — persists, then rebuilds dashboard/recommendation/filter bar/years root |
| 1416–1455 | `updateComputedOnly()` — persists at the top, then patches computed values in place; the component-row `className` reassignment here must stay in sync with the base classes in `renderComponentRow` any time that markup changes |
| 1457–1476 | `renderFilterBarSummaries()` |
| 1481–1488 | `showToast(msg)` |

### Event delegation
| Lines | What's there |
|---|---|
| 1496–1500 | `focus` listener (capture) — auto-select number-input contents |
| 1502–1506 | `#import-file-input` `change` listener — reads the picked file and calls `importBackupFile()`; separate from the main delegated listeners since it needs the raw `FileList` |
| 1508–1517 | `#screenshot-input` `change` listener → `startScreenshotImport()`; document-level `input`/`change` listeners → `handleImportField()` (only `data-imp` fields) |
| 1519–1526 | `visibilitychange` + `pagehide` listeners — defensive `persistState()` calls, see Persistence above |
| 1528–1570 | `click` listener — `switch` over `data-action`, including `"apply-recommendation"`, `"load-demo"`, `"import-screenshot"`, `"import-confirm"`, `"import-cancel"`, `"export-backup"`, `"import-backup"` |
| 1572–1632 | `input` listener — per-keystroke field updates; `course-name`/`component-name` call `persistState()` directly since they skip `updateComputedOnly()` |
| 1634–1658 | `change` listener — only `course-binary` and `component-magen` call full `render()` |
| 1660–1667 | `clampNumber(raw, min, max)` |
| 1691–1694 | `loadPersistedState()` + initial `render()` + restore toast — initial call on load |

## Data model reference

```js
year     = { id, name, semesters: [] }
semester = { id, name, courses: [] }
course   = { id, name, credits, isBinary, bonusPoints, expanded, components: [] }
component (exam)       = { id, type: "exam", name, weight, grade }              // isMagen always false, never shown in UI
component (quiz)       = { id, type: "quiz", name, weight, grade, isMagen }
component (assignment) = { id, type: "assignment", name, weight, grade, isMagen }
```

`grade`/`weight` are stored as numbers or `""` (empty = not yet entered — distinct from `0`, see the `hasUngraded` check inside `computeCourseGrade`).

## Known intentional behaviors (not bugs)

- Data survives reload (auto-saved to `localStorage`, see Persistence above) but is scoped to one browser on one device — there is no account system, no cross-device sync, and no way for two people to share a data set. This was an explicit choice over building real accounts (which would need a backend GitHub Pages can't host).
- A course's *raw* weight-sum badge always reflects the un-redistributed input weights (must total 100%); Magen weight redistribution only affects the *internal* grade calculation.
- No "add exam" button by design — only quizzes and assignments can be added freely; the single seeded exam per course is the Magen anchor.
- The binary recommendation only ever proposes one course at a time (the single best swap), not a ranked list or multi-course optimization — re-click "סמן כעובר בינארי" and let it recompute for the next-best candidate if the user wants to keep going.
- **Course-row and component-row layout rule: never pack more than 2-3 controls into one non-wrapping `flex` row.** An earlier layout crammed 5-6 fixed-width controls (expand toggle, name, credits, binary checkbox, grade badge, delete) into a single `flex items-center` row with `shrink-0` on most items and no wrap. On narrow viewports the row's required width exceeded its container, and since nothing could shrink, controls rendered outside the card's bounds and visually overlapped neighboring UI — confirmed by measuring `getBoundingClientRect()` on a live 375px viewport. The fix (current `renderCourseCard`/`renderComponentRow`) splits each into two `flex-wrap` lines instead: a primary line (name + delete, which never grows past its content) and a secondary line of secondary controls that wraps freely. If you add a new control to either row, add it to the wrapping secondary line, not the primary one.
- **Number inputs show the stored value after editing**: values are clamped while typing (`clampNumber`), and a separate `change` listener (`NUMBER_LIMITS`) rewrites the input to the clamped value on blur, with a toast — otherwise typing 150 showed "150" while the calculation used 100.
- Year/semester headers use `whitespace-nowrap` on the title and "+ סמסטר"/"+ קורס" buttons, with the summary allowed to wrap — at 320px the titles used to break mid-name ("שנה" / "א׳").
- Weight and grade inputs on component rows have a small always-visible `text-[10px]` caption ("משקל" / "ציון") above each — the `title` attribute alone is not sufficient labeling since hover tooltips don't work on touch devices.
- The app is dark-only, no theme toggle — a deliberate design decision from earlier in the project, not derivable from the code alone.
