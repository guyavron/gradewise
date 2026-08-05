# CLAUDE.md — Map of `index.html`

Single-file Hebrew/RTL university GPA calculator, branded **GradeWise**, dark "academic terminal" theme (HTML + Tailwind CDN + vanilla JS, no build step). This file is a navigation index into `index.html` so you can jump straight to the relevant lines instead of re-reading the whole file. **Whenever you edit `index.html` in a way that adds/removes/moves a function or section, update the line numbers below.**

## How the app works (read this first)

- All app state lives in one in-memory object: `state = { years: [...], filter: {...} }` (index.html:100), auto-saved to `localStorage` on every mutation — see Persistence below.
- Hierarchy: `state.years[]` → `year.semesters[]` → `semester.courses[]` → `course.components[]` (exam / quiz / assignment rows — see Component types below).
- Two render paths, both important to understand before touching render code:
  - `render()` (index.html:738) — full rebuild of `#dashboard`, `#recommendation`, `#filter-bar`, `#years-root` via `innerHTML`. Used after **structural** changes (add/delete year/semester/course/component, toggle binary, toggle Magen checkbox, change filter, apply a recommendation). Destroys and recreates every DOM node in those containers.
  - `updateComputedOnly()` (index.html:749) — **targeted** patch of just the grade badge, weight-sum text, per-row Magen "not included" styling, dashboard numbers, and the recommendation banner, using `querySelector` + direct property assignment (no `innerHTML`). Used on every keystroke in a grade/weight/credits/bonus field (via the `input` listener, index.html:879-923) so the focused input never loses focus/cursor position.
  - **Do not call `render()` from a `change`/`blur` handler on a text-editing field.** That was tried and reverted (see comment at index.html:938-942) — it destroys the input the user just clicked into and silently eats their next keystroke. If you need a new computed value to refresh live, add a `data-*` hook and patch it inside `updateComputedOnly()` instead.
- All interactivity is event-delegated on `document` (three listeners: click / input / change, index.html:835 onward) using `data-action` / `data-field` attributes — no per-element `addEventListener` calls, so re-rendered HTML never needs listeners re-attached.

## Persistence (`localStorage`, index.html:105-129)

- **Auto-save only, no accounts.** `persistState()` (index.html:109-116) writes `{ years, filter }` to `localStorage` under the key `"gradewise-state-v1"` on every mutation. It's called from `render()`, from inside `updateComputedOnly()`, and explicitly from the two `input`-listener branches that skip `updateComputedOnly()` (`course-name`, `component-name`, index.html:879-923) — every state-mutating code path ends up persisting.
- `loadPersistedState()` (index.html:117-129) runs once at the bottom of the IIFE, before the first `render()` (index.html:954-959). It's defensive: wrapped in try/catch, requires `parsed.years` to be an array, and silently starts fresh (empty state) on any parse failure or missing data — never throws, never blocks load.
- Storage is **per-browser, per-device, per-origin** — there is no login and no server. Two different people opening the same deployed URL each get their own independent `localStorage`, automatically isolated; nothing is shared or synced between them, and switching browsers/devices loses the trail (by design — see the AskUserQuestion decision in conversation history if reconsidering cloud sync later).
- A one-time toast ("הנתונים שוחזרו מהדפדפן") fires on load only when data was actually restored, so a first-time visit with no saved state stays silent.
- `resetAll()` calls `persistState()` after clearing `state.years`, so "אפס הכל" also wipes the saved copy — otherwise a reload would immediately restore the just-cleared data.
- Extra defensive saves on `visibilitychange` (when the tab goes hidden) and `pagehide` (index.html:890-897) — mobile browsers, especially iOS Safari/WebKit, can suspend or kill a backgrounded tab without a reliable `beforeunload`, so these fire `persistState()` on every signal the page might be about to disappear. Every mutation already persists synchronously and immediately (not debounced), so this is redundant insurance, not the primary save path — if data still doesn't survive a close/reopen on a given device after this, the cause is the OS/browser evicting already-written `localStorage`, not a missed save.

## Manual backup (export/import, index.html:295-330)

- Added because `localStorage` alone isn't fully reliable long-term — iOS in particular is known to evict site storage for non-installed web pages more aggressively than desktop browsers, so a course list entered only on a phone can vanish between sessions with no error surfaced anywhere.
- `exportBackup()` (index.html:295-308) serializes `{ years, filter, exportedAt }` to a downloaded `gradewise-backup-YYYY-MM-DD.json` via a `Blob` + temporary `<a download>` — no server round-trip, works entirely client-side.
- `importBackupFile(file)` (index.html:309-330) reads the file via `FileReader`, validates `parsed.years` is an array before touching state, requires a `confirm()` since it **replaces** `state.years` wholesale (not a merge), then calls `render()` (which persists via the normal path).
- Wired to two header buttons ("ייצוא גיבוי" / "ייבוא גיבוי", index.html:67-72) plus a hidden `#import-file-input` file picker that "ייבוא גיבוי" triggers by proxy (`data-action="import-backup"` → `.click()` on the file input, index.html:912-913) — the file input's own `change` listener (index.html:884-888) is separate from the main delegated `input`/`change` listeners since it needs the raw `FileList`, not a `data-field`.

## Component types (exam / quiz / assignment)

Three component types exist on `course.components[]`, each with different Magen eligibility — this was a deliberate change from an earlier version that only had "exam" and "assignment":

| Type | Hebrew | Magen-eligible? | Notes |
|---|---|---|---|
| `"exam"` | מבחן | **No, never** | The anchor grade Magen components protect against. Every new course seeds exactly one (`"מבחן סופי"`, weight 100) in `addCourse` (index.html:183-199); there is no "add exam" button — only the seeded one exists unless the user renames/repurposes a row. |
| `"quiz"` | בוחן | Yes | Added via the "+ הוספת בוחן" button (index.html:653, `data-type="quiz"`), rendered together with `"exam"` rows in the same "מבחנים ובחנים" section (`renderCourseBody`, index.html:630-681 — the `exams` array there is actually `type==="exam" || type==="quiz"`, named for legacy/brevity). |
| `"assignment"` | עבודה | Yes | Added via "+ הוספת עבודה", rendered in its own "עבודות" section. |

`renderComponentRow` (index.html:683-707) shows the "מגן" checkbox and the "לא נכלל" dropped-label only for `type === "quiz" || type === "assignment"` — never for `"exam"`.

## Magen logic (`computeCourseGrade`, index.html:269-326)

- **Anchor** = the `"exam"`-type component with the highest weight (index.html:274-278). Since exam-type is never Magen-eligible, there's no need to filter out `isMagen` here anymore.
- For every `quiz`/`assignment` component with `isMagen === true`: if its grade is higher than the anchor's grade, it counts normally; otherwise it's added to `droppedIds` and its weight is redistributed *proportionally* across the remaining active components (index.html:280-315).
- If a course has no `"exam"`-type component at all (user deleted the seeded one), `anchor` is `null` and Magen components are never dropped — same edge case as before, just re-scoped to the new type split.

## Binary recommendation ("what if I mark this course pass/fail")

- `recommendBinarySwap()` (index.html:362-382) simulates, for every non-binary numeric course with a valid graded result, what the **cumulative** GPA would become if that one course were switched to binary (excluded from the GPA sum, credits still counted) — using `computeStats`'s now-exposed `weightedSum` field (index.html:328-353) to avoid recomputing from scratch. It returns the single swap with the largest positive delta (`> 0.005`), or `null` if none would help (e.g. 0-1 graded courses, or every course is already at-or-above the cumulative average).
- `renderRecommendation()` (index.html:470-495) renders a teal-accented banner into `#recommendation` (empty string if no recommendation) with a "סמן כעובר בינארי" button (`data-action="apply-recommendation"`).
- `applyRecommendation(courseId)` (index.html:248-254) sets `course.isBinary = true` and re-renders.
- This is always scoped to the **whole-degree cumulative** GPA regardless of the active filter tab — it does not change per semester/year view.
- Wired into both render paths: full `render()` and the live `updateComputedOnly()`, so the banner's numbers (and whether it shows at all) update on every grade keystroke, not just on structural changes.

## Brand & theme reference

- Name "GradeWise" is a deliberate English wordmark inside an otherwise all-Hebrew UI — it appears only in `<title>` (index.html:6) and the header (index.html:54-63). Don't translate or Hebraize it; don't add more English strings elsewhere.
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
- Grade-band badge colors (`gradeColorClasses`, index.html:396-403) are tuned for the dark background: ≥90 teal (`accent`/`accentBg`), ≥80 blue-950/blue-300, ≥70 amber-950/amber-300, ≥60 orange-950/orange-300, else red-950/red-300. Binary courses get a distinct purple badge and weight-mismatch gets its own amber badge (`courseGradeBadgeHTML`, index.html:405-419) — kept visually separate from the grade bands since it's an actionable warning, not a grade.
- Numeric values are set in `font-mono` (JetBrains Mono, loaded alongside Heebo at index.html:9) everywhere a number appears: grade badges, dashboard stat numbers, weight/grade/credits/bonus inputs, weight-sum text, recommendation banner numbers. Labels, buttons, and Hebrew copy stay in Heebo (the default `font-sans`).
- Native checkboxes use `accent-color: #4ee3c2` (index.html:36). Number inputs have their native up/down spinner arrows removed (`-webkit-appearance: none` / `appearance: textfield`, index.html:43-44) — users type values directly, no steppers.

## Section map (index.html)

| Lines | What's there |
|---|---|
| 1–47 | `<head>`: meta, title, Google Fonts (Heebo + JetBrains Mono), Tailwind CDN + `tailwind.config` (color tokens + font families), custom `<style>` (checkbox accent-color, no-spinner number inputs, scrollbar, `.fade-in`, `.chip-active`) |
| 48–96 | Static HTML shell: `#toast-root`, header (wordmark + export/import/reset buttons + hidden `#import-file-input`), empty `#dashboard` / `#recommendation` / `#filter-bar` / `#years-root` containers, "הוספת שנת לימודים" button |
| 100–1018 | `<script>` — everything below is inside one IIFE `(function(){ "use strict"; ... })()` |

### State & lookups
| Lines | What's there |
|---|---|
| 107 | `HEBREW_ORDINALS` |
| 109–112 | `state` object shape |
| 114–138 | `STORAGE_KEY`, `persistState()`, `loadPersistedState()` — see Persistence above |
| 140–142 | `uid()` |
| 144–147 | `ordinalYearName(index)` |
| 149–154 | `nextSemesterName(count)` |
| 159–184 | `findYear`, `locateSemester`, `locateCourse` |
| 181–184 | `allCourses()` |

### Mutations (all call `render()` at the end)
| Lines | What's there |
|---|---|
| 190–234 | `addYear`, `deleteYear` (confirm), `addSemester`, `deleteSemester` (confirm), `addCourse` — seeds **one `"exam"` component** (`"מבחן סופי"`, weight 100) |
| 235–270 | `deleteCourse` (confirm), `toggleCourseExpanded`, `addComponent(courseId, type)` — `type` is `"quiz"` or `"assignment"` (never `"exam"` from the UI), `deleteComponent` |
| 271–283 | `resetFilter()`, `resetAll()` (confirm + `persistState()` + toast) |
| 284–293 | `applyRecommendation(courseId)` — sets `isBinary = true`, re-renders, toasts |
| **295–330** | **`exportBackup()`, `importBackupFile(file)`** — see Manual backup above |

### Calculations — **the core grading logic**
| Lines | What's there |
|---|---|
| 335–338 | `num(v)` |
| 340–342 | `weightSum(components)` |
| **345–402** | **`computeCourseGrade(course)`** — see Magen logic above |
| 404–427 | `computeStats(courseList)` — returns `{ gpa, weightedSum, totalCredits, gpaCredits, excludedBinaryCredits, invalidCount, courseCount }`; `weightedSum` is exposed specifically so `recommendBinarySwap` can simulate without recomputing |
| 430–432 | `semesterStats`, `yearStats`, `cumulativeStats` |
| **438–458** | **`recommendBinarySwap()`** — see Binary recommendation above |

### Render helpers
| Lines | What's there |
|---|---|
| 462–465 | `esc(str)` |
| 467–470 | `fmt(n, digits)` |
| 472–479 | `gradeColorClasses(grade)` |
| 481–495 | `courseGradeBadgeHTML(course)` |

### Dashboard, recommendation, filter bar
| Lines | What's there |
|---|---|
| 496–520 | `renderDashboard()` |
| 521–534 | `statCardHTML(label, sub, stats)` |
| 535–545 | `creditsCardHTML(cumulative)` |
| **546–571** | **`renderRecommendation()`** — see Binary recommendation above |
| 572–605 | `renderFilterBar()` |
| 606–621 | `visibleYears()`, `visibleSemesters(year)` |

### Course tree (`#years-root`)
| Lines | What's there |
|---|---|
| 622–639 | `renderYearsRoot()` |
| 640–660 | `renderYearBlock(year)` — `data-year-summary="{id}"` |
| 661–680 | `renderSemesterBlock(semester)` — `data-semester-summary="{id}"` |
| 681–705 | `renderCourseCard(course)` — `data-grade-badge="{id}"` |
| 706–758 | `renderCourseBody(course, result)` — exam+quiz section, assignment section, `data-weight-summary`, bonus input, `data-dropped-count="{id}"` |
| 759–783 | `renderComponentRow(course, comp, result)` — `data-component-row="{comp.id}"`; Magen checkbox/label only for `quiz`/`assignment` |

### Render orchestration
| Lines | What's there |
|---|---|
| 787–796 | `render()` — persists, then rebuilds dashboard/recommendation/filter bar/years root |
| 798–837 | `updateComputedOnly()` — persists at the top, then patches computed values in place |
| 839–859 | `renderFilterBarSummaries()` |
| 863–871 | `showToast(msg)` |

### Event delegation
| Lines | What's there |
|---|---|
| 878–882 | `focus` listener (capture) — auto-select number-input contents |
| 884–888 | `#import-file-input` `change` listener — reads the picked file and calls `importBackupFile()`; separate from the main delegated listeners since it needs the raw `FileList` |
| 890–897 | `visibilitychange` + `pagehide` listeners — defensive `persistState()` calls, see Persistence above |
| 899–940 | `click` listener — `switch` over `data-action`, including `"apply-recommendation"`, `"export-backup"`, `"import-backup"` |
| 943–987 | `input` listener — per-keystroke field updates; `course-name`/`component-name` call `persistState()` directly since they skip `updateComputedOnly()` |
| 989–1009 | `change` listener — only `course-binary` and `component-magen` call full `render()` |
| 1011–1018 | `clampNumber(raw, min, max)` |
| 1022–1025 | `loadPersistedState()` + initial `render()` + restore toast — initial call on load |

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
- The app is dark-only, no theme toggle — a deliberate design decision from earlier in the project, not derivable from the code alone.
