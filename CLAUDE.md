# CLAUDE.md — Map of `index.html`

Single-file Hebrew/RTL university GPA calculator, branded **GradeWise**, dark "academic terminal" theme (HTML + Tailwind CDN + vanilla JS, no build step). This file is a navigation index into `index.html` so you can jump straight to the relevant lines instead of re-reading the whole file. **Whenever you edit `index.html` in a way that adds/removes/moves a function or section, update the line numbers below.**

## How the app works (read this first)

- All app state lives in one in-memory object: `state = { years: [...], filter: {...}, maxBinary, maxBinaryCredits }` (index.html:127), auto-saved to `localStorage` on every mutation — see Persistence below. (`maxBinary` / `maxBinaryCredits` hold the university's pass/fail limits — see Binary recommendation.)
- Hierarchy: `state.years[]` → `year.semesters[]` → `semester.courses[]` → `course.components[]` (exam / quiz / assignment rows — see Component types below).
- Two render paths, both important to understand before touching render code:
  - `render()` (index.html:1496) — full rebuild of `#dashboard`, `#recommendation`, `#filter-bar`, `#years-root` via `innerHTML`. Used after **structural** changes (add/delete year/semester/course/component, toggle binary, toggle Magen checkbox, change filter, apply a recommendation). Destroys and recreates every DOM node in those containers.
  - `updateComputedOnly()` (index.html:1508) — **targeted** patch of just the grade badge, weight-sum text, per-row Magen "not included" styling, dashboard numbers, and the recommendation banner, using `querySelector` + direct property assignment (no `innerHTML`). Used on every keystroke in a grade/weight/credits/bonus field (via the `input` listener, index.html:1664-1714) so the focused input never loses focus/cursor position.
  - **Do not call `render()` from a `change`/`blur` handler on a text-editing field.** That was tried and reverted (see comment at index.html:1754-1759) — it destroys the input the user just clicked into and silently eats their next keystroke. If you need a new computed value to refresh live, add a `data-*` hook and patch it inside `updateComputedOnly()` instead.
- All interactivity is event-delegated on `document` (three listeners: click / input / change, index.html:1620 onward) using `data-action` / `data-field` attributes — no per-element `addEventListener` calls, so re-rendered HTML never needs listeners re-attached.

## Persistence (`localStorage`, index.html:136-170)

- **Auto-save only, no accounts.** `persistState()` (index.html:140-146) writes `{ years, filter, maxBinary, maxBinaryCredits }` to `localStorage` under the key `"gradewise-state-v1"` on every mutation. It's called from `render()`, from inside `updateComputedOnly()`, and explicitly from the two `input`-listener branches that skip `updateComputedOnly()` (`course-name`, `component-name`, index.html:1664-1714) — every state-mutating code path ends up persisting.
- `loadPersistedState()` (index.html:148-162) runs once at the bottom of the IIFE, before the first `render()` (index.html:1793-1796). `validMaxBinary()` / `validMaxBinaryCredits()` (index.html:164-170) turn missing/invalid limits (older saves and backups) into the defaults: 1 course, no credit limit (`""`). It's defensive: wrapped in try/catch, requires `parsed.years` to be an array, and silently starts fresh (empty state) on any parse failure or missing data — never throws, never blocks load.
- Storage is **per-browser, per-device, per-origin** — there is no login and no server. Two different people opening the same deployed URL each get their own independent `localStorage`, automatically isolated; nothing is shared or synced between them, and switching browsers/devices loses the trail (by design — see the AskUserQuestion decision in conversation history if reconsidering cloud sync later).
- **In-app browser warning + persistent storage request** (init block at the bottom of the IIFE): `checkInAppBrowser()` shows the static `#inapp-warning` banner when the user-agent matches a known in-app browser (Instagram, Facebook/Messenger, WhatsApp, LinkedIn, Snapchat, TikTok, LINE) — those embedded browsers often wipe storage when closed, which is why friends had to re-type courses on every visit. Dismissed via `data-action="dismiss-inapp"` (not persisted). `navigator.storage.persist()` is also requested on load (Chrome/Edge/Firefox honor it; Safari may ignore it).
- A one-time toast ("הנתונים שוחזרו מהדפדפן") fires on load only when data was actually restored, so a first-time visit with no saved state stays silent.
- `resetAll()` calls `persistState()` after clearing `state.years`, so "אפס הכל" also wipes the saved copy — otherwise a reload would immediately restore the just-cleared data.
- Extra defensive saves on `visibilitychange` (when the tab goes hidden) and `pagehide` (index.html:1611-1618) — mobile browsers, especially iOS Safari/WebKit, can suspend or kill a backgrounded tab without a reliable `beforeunload`, so these fire `persistState()` on every signal the page might be about to disappear. Every mutation already persists synchronously and immediately (not debounced), so this is redundant insurance, not the primary save path — if data still doesn't survive a close/reopen on a given device after this, the cause is the OS/browser evicting already-written `localStorage`, not a missed save.

## Manual backup (export/import, index.html:378-417)

- Added because `localStorage` alone isn't fully reliable long-term — iOS in particular is known to evict site storage for non-installed web pages more aggressively than desktop browsers, so a course list entered only on a phone can vanish between sessions with no error surfaced anywhere.
- `exportBackup()` (index.html:381-393) serializes `{ years, filter, maxBinary, maxBinaryCredits, exportedAt }` to a downloaded `gradewise-backup-YYYY-MM-DD.json` via a `Blob` + temporary `<a download>` — no server round-trip, works entirely client-side.
- `importBackupFile(file)` (index.html:395-417) reads the file via `FileReader`, validates `parsed.years` is an array before touching state, requires a `confirm()` since it **replaces** `state.years` wholesale (not a merge), then calls `render()` (which persists via the normal path).
- Wired to two header buttons ("ייצוא גיבוי" / "ייבוא גיבוי", index.html:73-78) plus a hidden `#import-file-input` file picker that "ייבוא גיבוי" triggers by proxy (`data-action="import-backup"` → `.click()` on the file input, index.html:1648) — the file input's own `change` listener (index.html:1594-1598) is separate from the main delegated `input`/`change` listeners since it needs the raw `FileList`, not a `data-field`.

## Component types (exam / quiz / assignment)

Three component types exist on `course.components[]`, each with different Magen eligibility — this was a deliberate change from an earlier version that only had "exam" and "assignment":

| Type | Hebrew | Magen-eligible? | Notes |
|---|---|---|---|
| `"exam"` | מבחן | **No, never** | The anchor grade Magen components protect against. Every new course seeds exactly one (`"מבחן סופי"`, weight 100) in `addCourse` (index.html:250-266); there is no "add exam" button — only the seeded one exists unless the user renames/repurposes a row. |
| `"quiz"` | בוחן | Yes | Added via the "+ הוספת בוחן" button (index.html:1426, `data-type="quiz"`), rendered together with `"exam"` rows in the same "מבחנים ובחנים" section (`renderCourseBody`, index.html:1407-1458 — the `exams` array there is actually `type==="exam" || type==="quiz"`, named for legacy/brevity). |
| `"assignment"` | עבודה | Yes | Added via "+ הוספת עבודה", rendered in its own "עבודות" section. |

`renderComponentRow` (index.html:1460-1491) shows the "מגן" checkbox and the "לא נכלל" dropped-label only for `type === "quiz" || type === "assignment"` — never for `"exam"`.

## Magen logic (`computeCourseGrade`, index.html:966-1023)

- **Anchor** = the `"exam"`-type component with the highest weight (index.html:986). Since exam-type is never Magen-eligible, there's no need to filter out `isMagen` here anymore.
- For every `quiz`/`assignment` component with `isMagen === true`: if its grade is higher than the anchor's grade, it counts normally; otherwise it's added to `droppedIds` and its weight is redistributed *proportionally* across the remaining active components (index.html:990-1020).
- If a course has no `"exam"`-type component at all (user deleted the seeded one), `anchor` is `null` and Magen components are never dropped — same edge case as before, just re-scoped to the new type split.

## Binary recommendation ("what if I mark this course pass/fail")

- Universities limit pass/fail by **both** a number of courses and a credit total (e.g. "3 courses, up to 18 credits"). Two state fields hold this: `state.maxBinary` (1–5, default 1) and `state.maxBinaryCredits` (number, or `""` = no credit limit, the default). Both are saved in `localStorage` and the JSON backup and reset by "אפס הכל". A course is all-or-nothing — it can't be split to fill the credit limit. The user enters what they have **left**; courses already marked binary are not subtracted automatically (some courses are binary by nature and don't use the quota).
- `renderBinaryLimits()` (index.html:1202-1227) draws the "מגבלת עובר בינארי: עד [N] קורסים · עד [X] נק״ז" row into `#binary-limits`, shown whenever any course exists. It is called **only from `render()`**: the credit input updates state on `input` and calls `updateComputedOnly()`, which re-renders only `#recommendation-tip` — so the input is never destroyed while typing (same focus rule as everywhere else). The select's `change` → full `render()`.
- `recommendBinarySwap()` (index.html:1055-1113) collects the eligible courses (not binary, valid numeric grade, credits > 0), then checks **every combination of 1 to `maxBinary` of them whose credits fit `maxBinaryCredits`** ("up to N", not "exactly N" — forcing exactly N could add a course that lowers the result). For each set it computes the new **cumulative** GPA from `computeStats`'s `weightedSum` (credits still counted, grades removed from the sum) and returns the best set (delta `> 0.005`) as `{ courses, credits, delta, newGpa, currentGpa }`, courses in transcript order, or `null`. Exhaustive on purpose: picking the best single course repeatedly can miss the best pair (counter-example: A 95/1, B 70/4, C 70/3.5, D 60/1 — greedy picks D then B → 75.56; best is B+C → 77.50; with a 5-credit limit it's B+D → 75.56). With `maxBinary = 1` and no credit limit the result is identical to the old single-course version.
  - **Performance:** it runs on every grade keystroke. 40 courses choose up to 5 ≈ 760,000 sets, so it reuses one `picked` array (push/pop), carries points/credits along (O(1) per set), and sorts candidates by credits so the credit limit can `break` early. Measured on desktop with 40 graded courses: N≤3 ≈ 3 ms, N=4 ≈ 10 ms, N=5 ≈ 45 ms per keystroke (a phone is roughly 3× slower). Don't raise the max above 5 without a smarter algorithm (e.g. a knapsack table over credits).
- `renderRecommendation()` (index.html:1229-1257) renders a teal-accented banner into `#recommendation-tip` (empty if no recommendation) listing every course in the set (name + semester · year), the set's total credits, current → new average and delta, and a "סמן כעובר בינארי" button (`data-action="apply-recommendation"`, `data-course-ids` = comma-separated ids).
- `applyRecommendation(courseIds)` (index.html:318-326) marks every course in the set `isBinary = true`, re-renders once, one toast.
- This is always scoped to the **whole-degree cumulative** GPA regardless of the active filter tab — it does not change per semester/year view.
- Wired into both render paths: full `render()` and the live `updateComputedOnly()`, so the banner's numbers (and whether it shows at all) update on every grade keystroke, not just on structural changes.

## Demo data ("טען נתוני דמו", index.html:328-376)

- Added so a first-time visitor (or a recruiter) can see every feature without typing grades. **All course names and grades are fictional** — never put real user data here.
- `buildDemoYears()` (index.html:331-367) returns a fresh `years[]` each call (new `uid()`s), built with small local factory helpers (`exam`, `quiz`, `task`, `course`, `semester`, `year`). The data is chosen to exercise every feature: a Magen quiz that counts (חדו"א 1), a Magen quiz that is dropped and redistributed (חדו"א 2, seeded `expanded: true` so the "לא נכלל" label is visible), a binary course, a low-grade 4-credit course that triggers the pass/fail recommendation (פיזיקה 1), and a current semester with ungraded finals.
- `loadDemoData()` (index.html:369-376) is a no-op if `state.years` is non-empty — the button only exists in the empty state of `renderYearsRoot()`, so it can never overwrite real data and needs no `confirm()`. It persists through the normal `render()` path; "אפס הכל" clears it.
- Wired via `data-action="load-demo"` in the click listener.

## Screenshot import — OCR ("ייבוא מצילום מסך", index.html:419-953)

Reads a screenshot of the **BGU app's grades screen** (title "ציונים 2026 סמסטר א", one white card per course, a blue grade box on top of an orange credits box) and turns it into courses. **Runs entirely in the browser** — Tesseract.js is lazy-loaded from jsDelivr on first use (~7 MB: core wasm + Hebrew + English models, cached by the browser afterwards); the image is never uploaded anywhere. Chosen over a server-side AI model: free, no backend, grades stay on the device (see the portfolio repo's DECISIONS.md).

Pipeline:
1. `fileToCanvas()` — returns `{ full, small, f }`: `full` is the screenshot at its **original resolution** (all text is read from it), `small` is scaled to `ANALYSIS_WIDTH = 940`px and used **only** to find the layout, so pixel thresholds are device-independent. `f` converts small → full coordinates.
2. `analyzeLayout()` (index.html:551) — **no OCR, color only.** A "box row" has many strongly-colored pixels (max−min channel > 90) on the left half and almost none on the right half — that excludes the full-width orange status bar and section headers. Consecutive rows group into bands, classified blue (grade) or orange (credits). Grade band directly above a credits band = one course; a credits band alone = a course with no grade yet (current semester). The card's height is measured in the white margin **left** of the boxes (a probe on the right side hits long, wrapped course names); the name crop is everything right of the boxes.
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
- Grade-band badge colors (`gradeColorClasses`, index.html:1128-1135) are tuned for the dark background: ≥90 teal (`accent`/`accentBg`), ≥80 blue-950/blue-300, ≥70 amber-950/amber-300, ≥60 orange-950/orange-300, else red-950/red-300. Binary courses get a distinct purple badge and weight-mismatch gets its own amber badge (`courseGradeBadgeHTML`, index.html:1137-1150) — kept visually separate from the grade bands since it's an actionable warning, not a grade.
- Numeric values are set in `font-mono` (JetBrains Mono, loaded alongside Heebo at index.html:9) everywhere a number appears: grade badges, dashboard stat numbers, weight/grade/credits/bonus inputs, weight-sum text, recommendation banner numbers. Labels, buttons, and Hebrew copy stay in Heebo (the default `font-sans`).
- Native checkboxes use `accent-color: #4ee3c2` (index.html:36). Number inputs have their native up/down spinner arrows removed (`-webkit-appearance: none` / `appearance: textfield`, index.html:43-44) — users type values directly, no steppers.

## Section map (index.html)

| Lines | What's there |
|---|---|
| 1–47 | `<head>`: meta, title, Google Fonts (Heebo + JetBrains Mono), Tailwind CDN + `tailwind.config` (color tokens + font families), custom `<style>` (checkbox accent-color, no-spinner number inputs, scrollbar, `.fade-in`, `.chip-active`) |
| 48–114 | Static HTML shell: `#import-modal` (OCR review overlay), `#toast-root`, header (wordmark + "ייבוא מצילום מסך" / export / import / reset buttons + hidden `#import-file-input` and `#screenshot-input`), empty `#dashboard` / `#recommendation` (holding `#binary-limits` + `#recommendation-tip`) / `#filter-bar` / `#years-root` containers, "הוספת שנת לימודים" button |
| 118–1797 | `<script>` — everything below is inside one IIFE `(function(){ "use strict"; ... })()` |

### State & lookups
| Lines | What's there |
|---|---|
| 125 | `HEBREW_ORDINALS` |
| 127–134 | `state` object shape (`years`, `filter`, `maxBinary`, `maxBinaryCredits`) |
| 136–170 | `STORAGE_KEY`, `persistState()`, `loadPersistedState()`, `validMaxBinary()`, `validMaxBinaryCredits()` — see Persistence above |
| 172–174 | `uid()` |
| 176–179 | `ordinalYearName(index)` |
| 181–186 | `nextSemesterName(count)` |
| 191–211 | `findYear`, `locateSemester`, `locateCourse` |
| 213–216 | `allCourses()` |

### Mutations (all call `render()` at the end)
| Lines | What's there |
|---|---|
| 222–266 | `addYear`, `deleteYear` (confirm), `addSemester`, `deleteSemester` (confirm), `addCourse` — seeds **one `"exam"` component** (`"מבחן סופי"`, weight 100) |
| 267–301 | `deleteCourse` (confirm), `toggleCourseExpanded`, `addComponent(courseId, type)` — `type` is `"quiz"` or `"assignment"` (never `"exam"` from the UI), `deleteComponent` |
| 303–316 | `resetFilter()`, `resetAll()` (confirm + resets both pass/fail limits + `persistState()` + toast) |
| 318–326 | `applyRecommendation(courseIds)` — marks every course in the recommended set binary, re-renders once, one toast |
| **328–376** | **`buildDemoYears()`, `loadDemoData()`** — see Demo data above |
| **378–417** | **`exportBackup()`, `importBackupFile(file)`** — see Manual backup above |
| **419–953** | **Screenshot import (OCR)**: `getOcrWorkers`, `ocrText`, `fileToCanvas`, `cropCanvas`, `boxToCanvas`, `cropFull`, `analyzeLayout`, `parseOcrNumber`, `readNumberVotes`, `readGrade`, `findCreditsCut`, `readCredits`, `readScreenshot`, `guessYearIndex`, `markDuplicates`, `startScreenshotImport`, `ensureSemester`, `confirmScreenshotImport`, `closeImportModal`, `renderImportModal`, `handleImportField` — see Screenshot import above |

### Calculations — **the core grading logic**
| Lines | What's there |
|---|---|
| 956–959 | `num(v)` |
| 961–963 | `weightSum(components)` |
| **965–1023** | **`computeCourseGrade(course)`** — see Magen logic above |
| 1025–1049 | `computeStats(courseList)` — returns `{ gpa, weightedSum, totalCredits, gpaCredits, excludedBinaryCredits, invalidCount, courseCount }`; `weightedSum` is exposed specifically so `recommendBinarySwap` can simulate without recomputing |
| 1051–1053 | `semesterStats`, `yearStats`, `cumulativeStats` |
| **1055–1113** | **`recommendBinarySwap()`** — exhaustive search over sets within both limits (`maxBinary` courses, `maxBinaryCredits` credits), see Binary recommendation above |

### Render helpers
| Lines | What's there |
|---|---|
| 1118–1121 | `esc(str)` |
| 1123–1126 | `fmt(n, digits)` |
| 1128–1135 | `gradeColorClasses(grade)` |
| 1137–1150 | `courseGradeBadgeHTML(course)` |

### Dashboard, recommendation, filter bar
| Lines | What's there |
|---|---|
| 1152–1175 | `renderDashboard()` |
| 1177–1189 | `statCardHTML(label, sub, stats)` |
| 1191–1199 | `creditsCardHTML(cumulative)` |
| **1202–1227** | **`renderBinaryLimits()`** — the "מגבלת עובר בינארי" row (`max-binary` select, `max-binary-credits` input) into `#binary-limits`; called only by `render()` |
| **1229–1257** | **`renderRecommendation()`** — tip banner with the course set, its total credits and the numbers into `#recommendation-tip`, see Binary recommendation above |
| 1260–1292 | `renderFilterBar()` |
| 1294–1308 | `visibleYears()`, `visibleSemesters(year)` |

### Course tree (`#years-root`)
| Lines | What's there |
|---|---|
| 1310–1335 | `renderYearsRoot()` — empty state includes the "טען נתוני דמו" and "ייבוא מצילום מסך" buttons |
| 1337–1356 | `renderYearBlock(year)` — `data-year-summary="{id}"` |
| 1358–1376 | `renderSemesterBlock(semester)` — `data-semester-summary="{id}"` |
| 1378–1405 | `renderCourseCard(course)` — `data-grade-badge="{id}"`. Two-line layout: line 1 is expand toggle + name input + delete button only; line 2 (`flex-wrap`) holds the binary checkbox, credits input, and grade badge. Split specifically so no combination of controls is forced onto one non-wrapping row — see the mobile-overlap bug this fixed, noted below |
| 1407–1458 | `renderCourseBody(course, result)` — exam+quiz section, assignment section, `data-weight-summary`, bonus input, `data-dropped-count="{id}"` |
| 1460–1491 | `renderComponentRow(course, comp, result)` — `data-component-row="{comp.id}"`. Two-line layout: line 1 is the name input + delete button; line 2 (`flex-wrap`) holds labeled weight/grade fields (small `text-[10px]` "משקל"/"ציון" captions stacked above each input — not just the `title` hover tooltip, which is inaccessible on touch devices) plus the Magen checkbox/label and dropped-indicator, shown only for `quiz`/`assignment` |

### Render orchestration
| Lines | What's there |
|---|---|
| 1496–1503 | `render()` — persists, then rebuilds dashboard/binary limits/recommendation/filter bar/years root |
| 1508–1547 | `updateComputedOnly()` — persists at the top, then patches computed values in place; the component-row `className` reassignment here must stay in sync with the base classes in `renderComponentRow` any time that markup changes |
| 1549–1568 | `renderFilterBarSummaries()` |
| 1573–1580 | `showToast(msg)` |

### Event delegation
| Lines | What's there |
|---|---|
| 1588–1592 | `focus` listener (capture) — auto-select number-input contents |
| 1594–1598 | `#import-file-input` `change` listener — reads the picked file and calls `importBackupFile()`; separate from the main delegated listeners since it needs the raw `FileList` |
| 1600–1609 | `#screenshot-input` `change` listener → `startScreenshotImport()`; document-level `input`/`change` listeners → `handleImportField()` (only `data-imp` fields) |
| 1611–1618 | `visibilitychange` + `pagehide` listeners — defensive `persistState()` calls, see Persistence above |
| 1620–1662 | `click` listener — `switch` over `data-action`, including `"apply-recommendation"` (reads `data-course-ids`), `"load-demo"`, `"import-screenshot"`, `"import-confirm"`, `"import-cancel"`, `"export-backup"`, `"import-backup"` |
| 1664–1714 | `input` listener — per-keystroke field updates (including `max-binary-credits` → `updateComputedOnly()`); `course-name`/`component-name` call `persistState()` directly since they skip `updateComputedOnly()` |
| 1716–1729 | `NUMBER_LIMITS` + `change` listener that rewrites clamped number inputs on blur |
| 1731–1760 | `change` listener — only `course-binary`, `component-magen` and `max-binary` call full `render()` |
| 1762–1769 | `clampNumber(raw, min, max)` |
| 1793–1796 | `loadPersistedState()` + initial `render()` + restore toast — initial call on load |

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
- The binary recommendation shows one best set (within the course and credit limits), not a ranked list. After applying it, the banner recomputes and may suggest more courses — the limits apply per recommendation, not as a running total of courses already marked binary.
- **Course-row and component-row layout rule: never pack more than 2-3 controls into one non-wrapping `flex` row.** An earlier layout crammed 5-6 fixed-width controls (expand toggle, name, credits, binary checkbox, grade badge, delete) into a single `flex items-center` row with `shrink-0` on most items and no wrap. On narrow viewports the row's required width exceeded its container, and since nothing could shrink, controls rendered outside the card's bounds and visually overlapped neighboring UI — confirmed by measuring `getBoundingClientRect()` on a live 375px viewport. The fix (current `renderCourseCard`/`renderComponentRow`) splits each into two `flex-wrap` lines instead: a primary line (name + delete, which never grows past its content) and a secondary line of secondary controls that wraps freely. If you add a new control to either row, add it to the wrapping secondary line, not the primary one.
- **Number inputs show the stored value after editing**: values are clamped while typing (`clampNumber`), and a separate `change` listener (`NUMBER_LIMITS`) rewrites the input to the clamped value on blur, with a toast — otherwise typing 150 showed "150" while the calculation used 100.
- Year/semester headers use `whitespace-nowrap` on the title and "+ סמסטר"/"+ קורס" buttons, with the summary allowed to wrap — at 320px the titles used to break mid-name ("שנה" / "א׳").
- Weight and grade inputs on component rows have a small always-visible `text-[10px]` caption ("משקל" / "ציון") above each — the `title` attribute alone is not sufficient labeling since hover tooltips don't work on touch devices.
- The app is dark-only, no theme toggle — a deliberate design decision from earlier in the project, not derivable from the code alone.
