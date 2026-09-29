# BITS Digital CodeForge V1.0 — Bug / Issue Audit

**Application audited:** `BITS_Digital_CodeForge_Challenge.html.html` (single file, 565 lines: HTML + CSS + JS, SheetJS loaded from CDN)
**Audit date:** 2026-09-28
**Status:** Audit only. **No application code was modified.**

---

## 0. Method, evidence levels and limitations (read first)

### 0.1 Functional specification used
The official BITS challenge brief was **not present in the project folder**, so this audit could not be checked against it directly. The specification used here is:

1. The in-app guidance text (lines 226–244): three columns `BITS ID`, `Course`, `Total Marks`; whole-number marks; NC students excluded.
2. The rules the code itself enforces or states: "Instructor name (mandatory)", "Min < Max", "continuous with no gaps or overlaps", 8 grades A…E with defaults.
3. The author's own markers: `// CodeForge challenge bug:` comments (lines 319, 336) and `LOCKED` section markers.
4. Reasonable expectations for a grading tool: every student in the file receives exactly one grade, or is explicitly reported as not graded.

Findings whose classification depends on the brief are marked **[Spec-dependent]**. Please re-check them against the official brief.

> **Note on "LOCKED" markers:** lines 9, 165, 295 and 323 label CSS/JS sections as "LOCKED (VERBATIM)". Yet an author-flagged bug (line 336) sits *inside* the locked JS block. The brief must clarify which sections may be edited before Stage 1 fixes begin.

### 0.2 Evidence levels
Every finding states how it was established:

| Label | Meaning |
|---|---|
| **EXECUTED** | The app's `<script>` block was extracted **verbatim** from the HTML and run in Node.js v24 with a mocked DOM, using a scratch harness outside the project folder (test IDs `T01`–`T38`). The app's own logic really ran; the output quoted is real. |
| **CODE INSPECTION** | Inferred from reading the source only. Not reproduced. |
| **NEEDS BROWSER** | Depends on real browser/rendering/OS behaviour (layout, file dialogs, Excel parsing, screen readers, fonts). **Not tested.** |

**Limitations of the EXECUTED tests:**
- **SheetJS was mocked.** Rows were injected directly as if `XLSX.utils.sheet_to_json` had returned them. **No real `.xls`/`.xlsx` file was parsed**, so the behaviour of real Excel files (header whitespace, cell types, dates, merged cells, corrupt files) is not verified.
- The canvas was mocked. Drawing calls (`fillRect`, `lineTo`) were recorded and checked numerically, but **nothing was visually rendered**.
- `alert`/`confirm` were mocked (confirm returns "OK").
- Timing figures in T37 come from a mock DOM and understate real browser costs.

---

## 1. Summary table

| # | Title | Category | Severity | Stage |
|---|---|---|---|---|
| 1 | Course dropdown adds one option per student row (duplicates) | Confirmed Bug | High | 1 |
| 2 | Stale course options survive a second upload *(author-flagged)* | Confirmed Bug | High | 1 |
| 3 | New upload does not refresh the displayed course (stale stats vs. export) | Confirmed Bug | High | 1 |
| 4 | Students matching no grade range are silently dropped from summary and CSV | Confirmed Bug | **Critical** | 1 |
| 5 | Min and Max statistics labels are swapped | Confirmed Bug | High | 1 |
| 6 | Validation does not require A max = 100 and E min = 0 | Confirmed Bug | High | 1 |
| 7 | Decimal marks fall between grade ranges and are dropped | Confirmed Bug | High | 1 |
| 8 | CSV fields are not escaped (commas/quotes break the file) | Confirmed Bug | High | 1 |
| 9 | Histogram bars are not scaled; normal class sizes overflow the canvas | Confirmed Bug | High | 1 |
| 10 | Missing/misnamed columns fail silently (no validation of headers) | Robustness | High | 1 |
| 11 | Blank or non-numeric marks give `undefined`/`NaN` stats and are silently dropped | Robustness | High | 1 |
| 12 | Text-formatted numeric marks are concatenated as strings (avg "4045.00") | Robustness | High | 1 |
| 13 | Numeric course codes never match (101 vs "101"), so the course appears empty | Robustness | High | 1 |
| 14 | Negative and >100 marks are accepted, shown in the histogram, then dropped | Robustness | High | 1 |
| 15 | Selecting the placeholder course shows undefined/NaN and still allows export | Confirmed Bug | Medium | 1 |
| 16 | "Mandatory" instructor name can be cleared after course selection | Confirmed Bug | Medium | 1 |
| 17 | Reset Range before selecting a course throws TypeError | Robustness | Medium | 1 |
| 18 | Cancelling the file dialog throws TypeError | Robustness | Medium | 1 |
| 19 | Bell curve horizontally misaligned with bars (30 px vs 32 px per 10 marks) | Confirmed Bug | Medium | 1 |
| 20 | Bell curve vertical scale hard-coded (`*3000`) and goes off canvas | Confirmed Bug | Medium | 1 |
| 21 | Attempt counter is global across courses ("second attempt" for a new course) | Confirmed Bug | Medium | 1 |
| 22 | Timer clock freezes after first finalize but later messages report more time | UX/UI | Medium | 1 |
| 23 | Timer measures from page load, not from the start of grading | Potential — Needs Testing [Spec-dependent] | Medium | 1 |
| 24 | Guidance rounding rule contradicts its own example (80.2 → 81) | Confirmed Bug | Medium | 1 |
| 25 | CSV / spreadsheet formula injection | Robustness (security) | Medium | 1 |
| 26 | Switching course silently discards customised ranges | UX/UI | Medium | 1 |
| 27 | Changing a Max does not cascade; asymmetric editing | UX/UI | Medium | 1 |
| 28 | Range error message is generic; the offending grade is not highlighted | UX/UI | Medium | 1 |
| 29 | File picker accepts only `.xls` | UX/UI [Spec-dependent] | Medium | 1 |
| 30 | Corrupt / wrong-type file: uncaught exception, no user message | Potential — Needs Testing | Medium | 1 |
| 31 | Re-selecting the same (edited) file does not reload it | Potential — Needs Testing | Medium | 1 |
| 32 | Duplicate BITS IDs / duplicate rows not detected | Robustness | Medium | 1 |
| 33 | SheetJS loaded from CDN unpinned, no SRI, known CVEs, silent offline failure | Robustness (reliability/security) | Medium | 1 |
| 34 | Not responsive: no viewport meta, fixed 420 px column, fixed canvas | UX/UI | Medium | 1/2 |
| 35 | Accessibility: unlabeled controls, no live regions, no canvas alternative, no `lang` | UX/UI | Medium | 1/2 |
| 36 | Timer display waits for first interval tick *(author-flagged)* | Confirmed Bug | Low | 1 |
| 37 | Ordinal suffix wrong for 21, 22, 23 … ("21th"); words/numerals mixed | Confirmed Bug | Low | 1 |
| 38 | Stray space in CSV header line (`Course, CS101`) | Confirmed Bug | Low | 1 |
| 39 | Lift / pulse feedback animations removed after 30 ms (invisible) | Confirmed Bug | Low | 1 |
| 40 | Bell curve disappears when all marks are equal / single student (std = 0) | Robustness | Low | 1 |
| 41 | Cascade can set a Max to −1, leaving a blank select | Robustness | Low | 1 |
| 42 | Reset Range asks for confirmation twice | UX/UI | Low | 1 |
| 43 | Empty file gives no feedback | Robustness | Low | 1 |
| 44 | Course names differing only by whitespace/case become separate courses | Robustness | Low | 1 |
| 45 | Row with missing Course creates a blank option identical to the placeholder | Robustness | Low | 1 |
| 46 | Histogram bin labels ambiguous; >100 marks lumped into "90-100" | UX/UI | Low | 1 |
| 47 | Histogram fully re-animates on every range change | Performance / UX | Low | 1/2 |
| 48 | Overlapping histogram animations can redraw stale data | Potential — Needs Testing | Low | 2 |
| 49 | Reduced-motion handling contradictory and ignores canvas animation | UX/UI | Low | 1/2 |
| 50 | Course `alert()` fires on every keyboard arrow step when no instructor | Potential — Needs Testing | Low | 1/2 |
| 51 | Guidance collapsed by default; summary has `cursor:default` | UX/UI | Low | 2 |
| 52 | Min = Max (single-mark grade band) is rejected | Potential — Needs Testing [Spec-dependent] | Low | 1 |
| 53 | No UTF-8 BOM in CSV; non-ASCII names may garble in Excel | Potential — Needs Testing | Low | 1/2 |
| 54 | Object URL never revoked; anchor clicked while detached | Potential — Needs Testing | Low | 2 |
| 55 | Relies on implicit global element IDs (`window.min`, `window.file`, …) | Robustness (code quality) | Low | 2 |
| 56 | Grade-assignment logic duplicated and re-reads 16 DOM selects per student | Performance | Low | 2 |
| 57 | Very large files: parse + option creation on main thread | Performance | Low (Medium with #1) | 1/2 |
| 58 | Rebuilds 1,616 `<option>` elements on every course change | Performance | Low | 2 |
| 59 | Only the first worksheet is read; title rows above headers break parsing | Robustness | Low | 2 |
| 60 | Rapid sequential uploads can race (two FileReaders) | Potential — Needs Testing | Low | 2 |
| 61 | Canvas not HiDPI-aware (blurry on high-DPI screens) | UX/UI — Needs Browser | Low | 2 |
| 62 | Timer arc stays full after 60 min with no indication | UX/UI | Low | 2 |
| 63 | File name has a double extension `.html.html` | UX/UI | Low | 1 |
| 64 | `readAsBinaryString` is a legacy API | Stage 2 Enhancement | Low | 2 |
| 65 | Fixed download name `grades.csv` | Stage 2 Enhancement | Low | 2 |
| 66 | No persistence / refresh loses all work without warning | Stage 2 Enhancement | Low | 2 |
| 67 | Histogram lacks axes, counts, grade cut-off markers | Stage 2 Enhancement | Low | 2 |
| 68 | CSV lacks course per row, grade scheme, timestamp, student count | Stage 2 Enhancement | Low | 2 |
| 69 | Native `alert`/`confirm` dialogs; no in-page notices | Stage 2 Enhancement | Low | 2 |
| 70 | No total-students / ungraded / percentage display in summary | Stage 2 Enhancement | Low | 2 |

---

## 2. Detailed findings

> Line numbers refer to the unmodified `BITS_Digital_CodeForge_Challenge.html.html`.

### Data import and course list

#### #1 — Course dropdown adds one option per student row (duplicates)
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T01, T02, T37)
- **Reproduce:** Upload a file where 4 students take CS101 and 1 takes MA102.
- **Expected:** The dropdown lists `CS101`, `MA102` once each.
- **Actual:** `["", "CS101","CS101","CS101","MA102","CS101"]`. A 50,000-row file created **50,000** `<option>` elements (T37).
- **Root cause:** lines 341–343, `data.map(d=>d.Course).forEach(c=>course.add(new Option(c,c)))`. No de-duplication.
- **Why it matters:** The core navigation control is unusable for real class sizes (hundreds of identical entries), and the DOM bloats with large files.
- **Recommended fix:** Build `new Set()` of trimmed course values, sort it, then add one option per unique course.
- **Stage:** Stage 1.

#### #2 — Stale course options survive a second upload *(author-flagged, line 336)*
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T02, T03)
- **Reproduce:** Upload file A (CS101, MA102), then upload file B (PH103).
- **Expected:** The dropdown shows only PH103.
- **Actual:** `["", CS101 ×4, MA102, PH103]`. Uploading the same file twice doubles every entry (T02).
- **Root cause:** `file.onchange` never clears `course.options` before adding (lines 335–346).
- **Why it matters:** The instructor can select a course that no longer exists in the loaded data, which gives empty stats and a header-only CSV (see #15).
- **Recommended fix:** On each successful load, reset the select to only the placeholder (`course.length = 1`), reset `course.value = ""`, clear the grade UI/stats/canvas/summary/welcome/thank-you text, then repopulate.
- **Stage:** Stage 1.

#### #3 — New upload does not refresh the currently displayed course
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T30)
- **Reproduce:** Upload file, select CS101 (stats Min 15 / Max 85, 4 students). Upload a new file where CS101 has one student with 99. Do not reselect. Click Finalize.
- **Expected:** The display resets or refreshes to the new data.
- **Actual:** Stats, histogram and grade summary still show the **old** data (`A:1 A-:1 C:1 E:1`), but the CSV contains the **new** data (`N1,99,A`). What the instructor reviews differs from what is exported.
- **Root cause:** `file.onchange` replaces `data` but does not call `updateAll()` or reset the course selection.
- **Why it matters:** The instructor approves grades based on a screen that does not match the exported file.
- **Recommended fix:** As in #2, reset the course selection and all derived UI after every upload.
- **Stage:** Stage 1.

#### #4 — Students matching no grade range are silently dropped from the grade summary and the CSV
- **Category:** Confirmed Bug | **Severity:** **Critical** | **Evidence:** EXECUTED (T08, T15, T16)
- **Reproduce:** Any student whose mark is decimal (79.5), negative, >100, blank, non-numeric, or outside a narrowed A-max/E-min range (see #6, #7, #11, #14).
- **Expected:** Every student in the course appears in the export with a grade, or the export is blocked with a clear list of students who cannot be graded.
- **Actual:** In T08, 7 students went in and only **2** rows came out of the CSV, with no warning. The grade-summary counts add up to 2, while the histogram and stats included the others.
- **Root cause:** Lines 506–511 and 537–544: the `for…of GRADES` loop simply finds no match and `break` is never reached. There is no "else / unmatched" branch, and nothing counts or reports unmatched rows.
- **Why it matters:** This is the application's primary output. A missing student in an official grade sheet is the most serious possible failure, and it happens silently.
- **Recommended fix:** Track unmatched rows. Show "N students could not be graded" with their IDs, and disable Finalize until it is resolved. Show total students vs. sum of grade counts.
- **Stage:** Stage 1.

#### #5 — Min and Max statistics labels are swapped
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T01)
- **Reproduce:** Upload CS101 marks 85, 72, 45, 15 and select CS101.
- **Expected:** Min = 15, Max = 85.
- **Actual:** The box labelled **Min shows 85** and the box labelled **Max shows 15**.
- **Root cause:** Lines 255–256: `Min<br><b id="max">` and `Max<br><b id="min">`. `computeStats` (lines 496–497) correctly writes the smallest value to `#min`, but `#min` sits under the "Max" label.
- **Why it matters:** The instructor reads the wrong statistic when deciding cut-offs.
- **Recommended fix:** Swap the IDs in the HTML so the label and ID agree (`Min → id="min"`).
- **Stage:** Stage 1.

#### #6 — Validation does not require the grade scale to cover 0–100 (A max = 100, E min = 0)
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T15, T16)
- **Reproduce:** Change A Max to 90 (student with 95), or E Min to 5 (student with 2).
- **Expected:** A validation error because the scale no longer covers all marks.
- **Actual:** No error. Finalize stays enabled and the student with 95 (or 2) is **omitted from the CSV**.
- **Root cause:** `validateRanges` (lines 419–431) only checks `min<max` and adjacent continuity. It never checks the ends of the scale.
- **Why it matters:** This passes validation yet silently loses top or bottom students (feeds #4).
- **Recommended fix:** Add `Amax === 100` and `Emin === 0` checks, or better, validate that every actual mark is covered. Consider making A-max/E-min read-only.
- **Stage:** Stage 1.

#### #7 — Decimal marks fall between grade ranges and are dropped
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T08: `79.5` absent from CSV)
- **Reproduce:** A student has Total Marks 79.5 (A- = 70–79, A = 80–100).
- **Expected:** Per the guidance, the value is rounded (or rejected with a message), then graded.
- **Actual:** No range matches, so the student is silently excluded from the summary and CSV. The mark still counts in avg/median and the histogram.
- **Root cause:** Integer-only ranges with inclusive `>=min && <=max` (lines 509, 540). The guidance *asks* the user to round, but the app neither rounds nor validates.
- **Why it matters:** Decimal totals are very common in real mark sheets.
- **Recommended fix:** Detect non-integers on upload and either reject with a list or apply the documented rounding rule (after #24 is clarified). Alternatively define ranges as `mark >= min && mark < nextMin`.
- **Stage:** Stage 1.

#### #8 — CSV fields are not escaped
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T26)
- **Reproduce:** Instructor `Rao, "K"`, course `CS, Intro`.
- **Expected:** Values containing commas/quotes/newlines are quoted per RFC 4180 (`"Rao, ""K"""`).
- **Actual:** `Instructor,Rao, "K" =HYPERLINK("x")` and `Course, CS, Intro`. Extra columns are created and the header block is corrupted.
- **Root cause:** Raw template-string concatenation (lines 535, 541).
- **Why it matters:** Names with commas ("Sharma, R.") are realistic, and they corrupt the official file.
- **Recommended fix:** A `csvCell()` helper that wraps in quotes and doubles internal quotes, applied to every field.
- **Stage:** Stage 1.

#### #9 — Histogram bars are not scaled; normal class sizes overflow the canvas
- **Category:** Confirmed Bug | **Severity:** High | **Evidence:** EXECUTED (T13) numeric; visual rendering NEEDS BROWSER
- **Reproduce:** 200 students in the 60–70 range.
- **Expected:** The tallest bar fits in the chart area.
- **Actual:** Bar height **2400 px**, top at y = −2190 on a 240 px canvas. Any bin with more than 17 students (210 / 12) is clipped, so all tall bins look identical.
- **Root cause:** Line 461, `const h=v*12*p;`, a fixed 12 px per student.
- **Why it matters:** The histogram is meaningless for typical class sizes (50–300).
- **Recommended fix:** Scale by the maximum bin: `h = v / maxBin * 190 * p`. Show bin counts.
- **Stage:** Stage 1.

#### #10 — Missing or misnamed columns fail silently
- **Category:** Robustness | **Severity:** High | **Evidence:** EXECUTED (T11), real-file header behaviour NEEDS BROWSER
- **Reproduce:** Header `Total marks` (lower-case m), `Marks`, `BITS Id`, or trailing spaces (`"Total Marks "`). A title row above the headers has the same effect.
- **Expected:** "Required column 'Total Marks' not found. Found: …"
- **Actual:** The upload succeeds. Stats show `undefined / undefined / NaN / undefined`, all grade counts are 0, and the CSV is empty. No message is shown.
- **Root cause:** Hard-coded exact keys `d.Course`, `d["Total Marks"]`, `d["BITS ID"]`, with no header check after `sheet_to_json` (line 340).
- **Why it matters:** Header typos are the most likely user error, and nothing tells the user what went wrong.
- **Recommended fix:** After parsing, verify the three required headers (trim + case-insensitive match). Report missing ones and abort.
- **Stage:** Stage 1.

#### #11 — Blank or non-numeric marks give `undefined`/`NaN` stats and are dropped
- **Category:** Robustness | **Severity:** High | **Evidence:** EXECUTED (T08)
- **Reproduce:** Marks cell blank, or containing `AB`, `NC`, `-`.
- **Expected:** Row-level validation error listing the offending BITS IDs.
- **Actual:** In T08 the Min label showed `undefined` (the blank mark sorts last), Avg showed `NaN`, and the rows were omitted from the CSV. The histogram adds the value to a hidden `bins["NaN"]` property.
- **Root cause:** No type validation. `sort` pushes `undefined` to the end (line 495). `reduce` produces `NaN` (line 498). `bins[Math.min(9, NaN)]` (line 452).
- **Why it matters:** Stats become unreadable and students are lost (#4). Guidance says NC students must be excluded, but a user who leaves them in gets no warning.
- **Recommended fix:** Validate each row on upload with `Number.isFinite(Number(x))`. List invalid rows and block grading until fixed.
- **Stage:** Stage 1.

#### #12 — Text-formatted numeric marks are concatenated as strings
- **Category:** Robustness | **Severity:** High | **Evidence:** EXECUTED (T09) with mocked parser. Whether SheetJS returns strings for a given real file NEEDS BROWSER/real file.
- **Reproduce:** Marks stored as text in Excel (common when data is pasted or imported), e.g. `"80"`, `"90"`.
- **Expected:** Avg 85.00, Median 85.
- **Actual:** **Avg `4045.00`, Median `4045`** (`"80"+"90"` = `"8090"`, then /2). Grading itself happened to work because `>=`/`<=` coerce to numbers.
- **Root cause:** `reduce((a,b)=>a+b,0)` (line 498) and `m[a]+m[b]` (line 500) on strings. There is no `Number()` conversion on import.
- **Why it matters:** Wildly wrong statistics mislead cut-off decisions.
- **Recommended fix:** Normalise `Total Marks` to `Number` once at import (combined with #11 validation).
- **Stage:** Stage 1.

#### #13 — Numeric course codes never match, so the course appears empty
- **Category:** Robustness | **Severity:** High | **Evidence:** EXECUTED (T10)
- **Reproduce:** Course column contains a number (e.g. `101`, or a numeric code stored as number).
- **Expected:** Selecting "101" shows its 2 students.
- **Actual:** Stats `undefined/NaN`, all counts 0, CSV empty. (The option also appears twice, #1.)
- **Root cause:** `<option>` values are strings (`"101"`), but `d.Course===course.value` uses strict equality against the number `101` (lines 448, 494, 505, 536).
- **Why it matters:** An entire course is silently ungradeable.
- **Recommended fix:** Normalise `Course` with `String(d.Course).trim()` at import.
- **Stage:** Stage 1.

#### #14 — Negative and >100 marks are accepted, partly displayed, then dropped
- **Category:** Robustness | **Severity:** High | **Evidence:** EXECUTED (T08)
- **Reproduce:** Marks `-5` and `105`.
- **Expected:** Rejected with a message (valid range 0–100).
- **Actual:** `-5` becomes the displayed "Max" (it is the true minimum, shown under the swapped label, #5). It lands in `bins[-1]`, so it is invisible in the histogram. `105` is clamped into the "90-100" bar. **Both are excluded from the CSV.**
- **Root cause:** No range validation. `Math.min(9, …)` clamps only the top (line 452).
- **Why it matters:** Data-entry errors are silently absorbed, and students go missing (#4).
- **Recommended fix:** Validate `0 ≤ mark ≤ 100` at import and report violations.
- **Stage:** Stage 1.

### Course selection and instructor

#### #15 — Selecting the placeholder course shows undefined/NaN and still allows export
- **Category:** Confirmed Bug | **Severity:** Medium | **Evidence:** EXECUTED (T05)
- **Reproduce:** Enter a name, select CS101, then re-select "Select a course to begin".
- **Expected:** The workspace resets to an empty state and Finalize is disabled.
- **Actual:** Stats show `undefined, undefined, NaN, NaN`. Finalize stays **enabled** and downloads a CSV with `Course, ` and no rows. The same happens for any course with zero matching students (#13, #2).
- **Root cause:** `course.onchange` (lines 348–357) doesn't handle `value===""`. `updateAll` only gates download on range validity. `computeStats` has no empty guard.
- **Why it matters:** The user can "finalize" an empty grade sheet, and the stats panel shows programmer-facing text.
- **Recommended fix:** Guard for empty course/empty data: clear the panels, show an empty-state message, disable Finalize.
- **Stage:** Stage 1.

#### #16 — "Mandatory" instructor name can be cleared after course selection
- **Category:** Confirmed Bug | **Severity:** Medium | **Evidence:** EXECUTED (T31)
- **Reproduce:** Enter name, select course, clear the name field, click Finalize.
- **Expected:** Finalize is blocked until a name is entered.
- **Actual:** The download proceeds with `Instructor,` (blank). The welcome banner still shows the old name. Editing the name also never updates the banner.
- **Root cause:** The name is checked only in `course.onchange` (line 349). There is no `input` listener, and `download.onclick` has no check.
- **Why it matters:** The official export can be produced without the mandatory attribution.
- **Recommended fix:** Re-validate the name in `updateAll()`/`download.onclick`, and listen to `instructor.oninput`.
- **Stage:** Stage 1.

#### #26 — Switching course silently discards customised ranges
- **Category:** UX/UI | **Severity:** Medium | **Evidence:** EXECUTED (T25)
- **Reproduce:** Set A min to 90 on CS101, switch to MA102, switch back.
- **Expected:** Either the per-course ranges are kept, or the user is warned before losing edits.
- **Actual:** A min is back to 80 with no warning.
- **Root cause:** `buildGradeUI()` is rebuilt from `DEFAULTS` on every course change (line 355).
- **Why it matters:** Lost work, and the risk of finalising with unintended defaults.
- **Recommended fix:** Stage 1: warn before discarding unsaved edits. Stage 2: remember ranges per course.
- **Stage:** Stage 1 (warning) / Stage 2 (memory).

#### #50 — Course `alert()` fires on each keyboard step when the instructor name is empty
- **Category:** Potential — Needs Testing | **Severity:** Low | **Evidence:** CODE INSPECTION + NEEDS BROWSER
- **Reproduce:** Without a name, focus the course select and press ↓.
- **Expected:** One clear inline message.
- **Actual (inferred):** In browsers that fire `change` on arrow keys for a closed select (e.g. Chrome/Windows), an `alert` appears and the value resets on every keystroke.
- **Root cause:** Blocking `alert` in `course.onchange` (line 350).
- **Recommended fix:** Inline message next to the name field; disable the course select until a name is present.
- **Stage:** Stage 1/2.

### Grade-range configuration and validation

#### #27 — Changing a Max does not cascade; asymmetric editing
- **Category:** UX/UI | **Severity:** Medium | **Evidence:** EXECUTED (T19, T20, T21)
- **Reproduce:** Change A- Max from 79 to 75 (gap) or 85 (overlap).
- **Expected:** Behaviour consistent with Min edits (Min changes auto-update the next grade's Max, T21), or an explanation.
- **Actual:** A Min stays 80 and an error appears. The user must manually find and fix the neighbour. Min edits also never adjust the *same* grade's Max when they create `min ≥ max` (T18: B min 75 → error).
- **Root cause:** Only `minSel.onchange` calls `cascadeMaxFrom` (lines 383–394).
- **Why it matters:** It's confusing: half the controls auto-adjust, half don't. Every Max select is effectively redundant.
- **Recommended fix:** Either cascade Max changes upward (`GRADES[i-1].min = max+1`) or derive Max values automatically and make them read-only.
- **Stage:** Stage 1.

#### #28 — Range error message is generic; the offending grade is not highlighted
- **Category:** UX/UI | **Severity:** Medium | **Evidence:** EXECUTED (T17–T20 messages) + CODE INSPECTION
- **Actual:** Only "Each grade must have Min < Max." or "…no gaps or overlaps." There is no grade name and no visual marker, and only the first error is reported.
- **Root cause:** `validateRanges` returns a fixed string on the first failure (lines 424, 427).
- **Recommended fix:** Include the grade(s), e.g. "B: Min (75) must be less than Max (69)". Add an error class on the card.
- **Stage:** Stage 1.

#### #17 — Reset Range before selecting a course throws TypeError
- **Category:** Robustness | **Severity:** Medium | **Evidence:** EXECUTED (T23)
- **Reproduce:** Open the page and click Reset Range, then confirm twice.
- **Expected:** Nothing to reset. The button is disabled or shows a message.
- **Actual:** `TypeError: Cannot set properties of null (setting 'value')`. The dialogs appear, then nothing happens (console error only).
- **Root cause:** Lines 412–415 access `#Amin` etc., which don't exist until `buildGradeUI()` runs.
- **Recommended fix:** Disable Reset until the grade UI exists, or guard for null.
- **Stage:** Stage 1.

#### #41 — Cascade can set a Max to −1, leaving a blank select
- **Category:** Robustness | **Severity:** Low | **Evidence:** EXECUTED (T22)
- **Reproduce:** Set D Min to 0.
- **Actual:** E Max is set to −1. No such option exists, so the select shows **blank** (value `""`), which `+""` then treats as 0. An error is shown ("Min < Max") but the E card displays an empty box.
- **Root cause:** `cascadeMaxFrom` (line 362) writes `prevMin-1` without clamping.
- **Recommended fix:** Clamp, and explain that lower grades have no remaining room.
- **Stage:** Stage 1.

#### #42 — Reset Range asks for confirmation twice
- **Category:** UX/UI | **Severity:** Low | **Evidence:** EXECUTED (T24: 2 confirm dialogs)
- **Root cause:** Two consecutive `confirm()` calls (lines 410–411).
- **Recommended fix:** One confirmation (or an undo).
- **Stage:** Stage 1.

#### #52 — Min = Max (single-mark grade band) is rejected
- **Category:** Potential — Needs Testing **[Spec-dependent]** | **Severity:** Low | **Evidence:** EXECUTED (T17)
- **Actual:** E 19–19 gives "Each grade must have Min < Max."
- **Why it matters:** Mathematically, a one-mark band is a valid, continuous range. Whether it is allowed depends on BITS policy / the brief.
- **Recommended fix:** If allowed, change to `minV > maxV`.
- **Stage:** Stage 1 if the brief allows it.

### Statistics and visualisation

#### #19 — Bell curve horizontally misaligned with bars
- **Category:** Confirmed Bug | **Severity:** Medium | **Evidence:** EXECUTED (T14) numeric
- **Reproduce:** Marks centred at 95.
- **Expected:** The curve peak sits over the 90–100 bar.
- **Actual:** The curve peak is at x = **315**, while the bar centre is at x = **330**. The drift grows across the axis.
- **Root cause:** Bars use 32 px per 10 marks (`30+i*32`, line 463). The curve uses 30 px per 10 marks (`30+(x/10)*30`, line 486).
- **Recommended fix:** Share one x-scale function between bars and curve.
- **Stage:** Stage 1.

#### #20 — Bell curve vertical scale hard-coded and goes off canvas
- **Category:** Confirmed Bug | **Severity:** Medium | **Evidence:** EXECUTED (T14)
- **Actual:** In T14 (tight distribution, std ≈ 1) the curve peak was at **y = −987** (far above the canvas top), while the bar top was at y = 90. The curve height depends on std only, not on student count, so it never corresponds to the bars.
- **Root cause:** `py=210-y*3000` (line 487), a pdf multiplied by a magic constant.
- **Recommended fix:** Scale the pdf to expected counts: `n × binWidth(10) × pdf`, using the same y-scale as the bars (#9).
- **Stage:** Stage 1.

#### #40 — Bell curve disappears when all marks are equal / single student
- **Category:** Robustness | **Severity:** Low | **Evidence:** EXECUTED (T12, T34: 0 finite curve points)
- **Root cause:** `std = 0` gives a division by zero, so every point is `NaN` and `lineTo(NaN)` is ignored (lines 478–488). No crash, but silently no curve.
- **Recommended fix:** Skip the curve (with a note) when `std === 0` or n is very small.
- **Stage:** Stage 1.

#### #46 — Histogram bin labels ambiguous; >100 lumped into "90-100"
- **Category:** UX/UI | **Severity:** Low | **Evidence:** CODE INSPECTION (line 466), EXECUTED for clamping (T08)
- **Actual:** Labels "0-10", "10-20", … share endpoints (is 10 in the first or second bin?). The last bin also silently absorbs >100.
- **Recommended fix:** Labels "0–9", "10–19", …, "90–100", and reject >100 (#14).
- **Stage:** Stage 1.

#### #47 — Histogram fully re-animates on every range change
- **Category:** Performance / UX | **Severity:** Low | **Evidence:** EXECUTED (T38: 5 animation frames per dropdown change)
- **Root cause:** `updateAll()` always calls `drawHistogram()`, although the histogram does not depend on the ranges.
- **Recommended fix:** Redraw only on course/data change (unless cut-off lines are added, #67).
- **Stage:** Stage 1/2.

#### #48 — Overlapping histogram animations can redraw stale data
- **Category:** Potential — Needs Testing | **Severity:** Low | **Evidence:** CODE INSPECTION
- **Scenario:** Select a course, then within 400 ms switch to an empty course (or change ranges quickly). The old `requestAnimationFrame` loop is never cancelled and keeps redrawing the previous course's bars after the new call cleared the canvas and returned early (line 449).
- **Recommended fix:** Keep the rAF id and `cancelAnimationFrame` before starting a new one.
- **Stage:** Stage 2.

#### #61 — Canvas not HiDPI-aware
- **Category:** UX/UI | **Severity:** Low | **Evidence:** NEEDS BROWSER
- **Root cause:** Fixed `width=380 height=240` with no `devicePixelRatio` scaling, so it is expected to look blurry on high-DPI displays.
- **Stage:** Stage 2.

### Timer

#### #36 — Timer display waits for first interval tick *(author-flagged, line 319)*
- **Category:** Confirmed Bug | **Severity:** Low | **Evidence:** EXECUTED (T32)
- **Actual:** Before the first tick, the text shows the static HTML placeholder `00:00`, and the arc uses the SVG attribute. The timer code only takes over after 1 s. **Honest note:** because the placeholder is `00:00`, the visible effect is small. The author's comment confirms that the intended fix is an immediate first render.
- **Root cause:** `setInterval` only. There is no initial call of the update function (lines 304–315).
- **Recommended fix:** Extract the tick body into a function, call it once immediately, then schedule it.
- **Stage:** Stage 1.

#### #22 — Timer clock freezes after first finalize, but later messages report more time
- **Category:** UX/UI | **Severity:** Medium | **Evidence:** EXECUTED (T27)
- **Actual:** After the first finalize the clock stays at `11:00`. The second finalize message says "a total of **15 min 0 sec**" while the clock still reads 11:00.
- **Root cause:** `clearInterval` on every finalize (line 529), but elapsed time is always computed from page load (line 531).
- **Recommended fix:** Decide the semantics (per course? per session?) and make the clock and message agree.
- **Stage:** Stage 1.

#### #23 — Timer measures from page load, not from the start of grading
- **Category:** Potential — Needs Testing **[Spec-dependent]** | **Severity:** Medium | **Evidence:** EXECUTED (T27)
- **Actual:** Page opened at 0:00, course selected at 10:00, finalised at 11:00. The message says "You completed grading in **11 min**" (actual grading took 1 min).
- **Root cause:** `const gradingStartTime = Date.now()` at script load (line 296).
- **Recommended fix:** If the brief defines grading as starting at course selection or upload, start the timer then.
- **Stage:** Stage 1 if the brief defines it.

#### #62 — Timer arc stays full after 60 minutes with no indication
- **Category:** UX/UI | **Severity:** Low | **Evidence:** EXECUTED (T33: text `180:00`, arc offset 0)
- **Stage:** Stage 2.

### Finalize and CSV export

#### #21 — Attempt counter is global across courses
- **Category:** Confirmed Bug | **Severity:** Medium | **Evidence:** EXECUTED (T27)
- **Actual:** After finalising CS101, finalising **MA102** for the first time says "…in your **second** attempt".
- **Root cause:** A single `finalizeCount` for the whole page (lines 333, 528).
- **Recommended fix:** Track attempts per course (or reset on course change). Clarify semantics with the brief.
- **Stage:** Stage 1.

#### #37 — Ordinal suffix wrong for 21, 22, 23 …
- **Category:** Confirmed Bug | **Severity:** Low | **Evidence:** EXECUTED (T28: `21th`, `22th`; also `4th` after words "second"/"third")
- **Root cause:** Lines 556–558 special-case 2 and 3 only.
- **Recommended fix:** A proper ordinal function, with a consistent style (all numerals or all words).
- **Stage:** Stage 1.

#### #38 — Stray space in CSV header line
- **Category:** Confirmed Bug | **Severity:** Low | **Evidence:** EXECUTED (T01: `Course, CS101`)
- **Root cause:** `Course, ${course.value}` (line 535). The instructor line has no space, so the two lines are inconsistent.
- **Stage:** Stage 1.

#### #25 — CSV / spreadsheet formula injection
- **Category:** Robustness (security) | **Severity:** Medium | **Evidence:** EXECUTED (T26: `=1+1` and `=HYPERLINK(...)` written unmodified). Excel's actual execution NEEDS BROWSER/Excel.
- **Why it matters:** Cells starting with `= + - @` are evaluated as formulas when opened in Excel. The exported CSV is an official document that will be opened in spreadsheets.
- **Recommended fix:** Prefix such cells with `'` (in addition to quoting, #8). Note: negative marks would also start with `-`, so validate marks first (#14).
- **Stage:** Stage 1.

#### #53 — No UTF-8 BOM in the CSV
- **Category:** Potential — Needs Testing | **Severity:** Low | **Evidence:** CODE INSPECTION (line 548)
- **Why:** Excel on Windows commonly misreads BOM-less UTF-8, which would garble non-ASCII instructor names (e.g. Devanagari or accented characters).
- **Recommended fix:** Prepend `﻿`.
- **Stage:** Stage 1/2.

#### #54 — Object URL never revoked; anchor clicked while detached
- **Category:** Potential — Needs Testing | **Severity:** Low | **Evidence:** CODE INSPECTION (lines 547–550)
- **Detail:** Each download leaks a Blob URL. Clicking an anchor that is not attached to the DOM works in current Chromium/Firefox but historically failed in older Firefox.
- **Recommended fix:** Append, click, remove, then `URL.revokeObjectURL`.
- **Stage:** Stage 2.

### File handling and reliability

#### #18 — Cancelling the file dialog throws TypeError
- **Category:** Robustness | **Severity:** Medium | **Evidence:** EXECUTED (T29, with an empty `files` list simulated). Whether a given browser fires `change` on cancel NEEDS BROWSER (Chromium generally clears the selection and fires `change` when a previously chosen file is cancelled).
- **Actual:** `readAsBinaryString(undefined)` gives `TypeError … parameter 1 is not of type 'Blob'`, an uncaught error.
- **Root cause:** No `if(!e.target.files[0]) return;` (line 345).
- **Stage:** Stage 1.

#### #29 — File picker accepts only `.xls`
- **Category:** UX/UI **[Spec-dependent]** | **Severity:** Medium | **Evidence:** CODE INSPECTION (line 222)
- **Why:** `.xlsx` has been Excel's default format since 2007. The guidance says only "Excel file". Users with `.xlsx` must change the dialog filter to "All files", which is confusing. SheetJS itself could parse `.xlsx`/`.csv`.
- **Recommended fix:** `accept=".xls,.xlsx"` (+ `.csv` if the brief allows).
- **Stage:** Stage 1 if the brief permits `.xlsx`.

#### #30 — Corrupt / wrong-type file: uncaught exception, no user message
- **Category:** Potential — Needs Testing | **Severity:** Medium | **Evidence:** CODE INSPECTION. Needs real files.
- **Detail:** There is no `try/catch` around `XLSX.read` (line 339) and no `reader.onerror`. Depending on the file, SheetJS may throw (uncaught, silent to the user) or "successfully" parse garbage (then #10 applies).
- **Recommended fix:** `try/catch` with a user-visible error, plus `onerror`.
- **Stage:** Stage 1.

#### #31 — Re-selecting the same (edited) file does not reload it
- **Category:** Potential — Needs Testing | **Severity:** Medium | **Evidence:** CODE INSPECTION + NEEDS BROWSER
- **Detail:** Browsers do not fire `change` when the same path is chosen again. An instructor who fixes marks in Excel and re-uploads the same file gets no reload, and the app keeps old data.
- **Recommended fix:** Reset `file.value = ""` after reading.
- **Stage:** Stage 1.

#### #32 — Duplicate BITS IDs / duplicate rows not detected
- **Category:** Robustness | **Severity:** Medium | **Evidence:** EXECUTED (T35: `S1` exported three times with two different grades, no warning)
- **Recommended fix:** Detect duplicates per course on import and warn/block.
- **Stage:** Stage 1.

#### #33 — SheetJS loaded from CDN: unpinned, no SRI, known CVEs, silent offline failure
- **Category:** Robustness (reliability/security) | **Severity:** Medium | **Evidence:** CODE INSPECTION (line 6). The resolved version was **not** verified at runtime.
- **Detail:**
  1. `cdn.jsdelivr.net/npm/xlsx/...` has no version, so it resolves to whatever npm "latest" is. SheetJS stopped publishing to npm at 0.18.5, which has public advisories (prototype pollution CVE-2023-30533, ReDoS CVE-2024-22363). This is based on public advisories and was not verified here.
  2. There is no `integrity` (SRI) attribute.
  3. Offline or CDN blocked: `XLSX` is undefined and the upload fails with an uncaught `ReferenceError` and no message.
  4. It is a render-blocking script in `<head>`.
- **Recommended fix:** Pin a patched version (SheetJS CDN ≥ 0.20.2 or bundle locally), add SRI, add `defer`, and check `typeof XLSX` with a user message.
- **Stage:** Stage 1 (pin + error message) / Stage 2 (bundling).

#### #43 — Empty file gives no feedback
- **Category:** Robustness | **Severity:** Low | **Evidence:** EXECUTED (T06: no options, no message)
- **Stage:** Stage 1.

#### #44 — Course names differing only by whitespace/case become separate courses
- **Category:** Robustness | **Severity:** Low | **Evidence:** EXECUTED (T36: `"CS101"` and `"CS101 "`)
- **Recommended fix:** Trim (and optionally normalise case) at import.
- **Stage:** Stage 1.

#### #45 — Row with missing Course creates a blank option identical to the placeholder
- **Category:** Robustness | **Severity:** Low | **Evidence:** EXECUTED (T11: an extra option with value `""`)
- **Stage:** Stage 1 (covered by header/row validation #10/#11).

#### #59 — Only the first worksheet is read; title rows above headers break parsing
- **Category:** Robustness | **Severity:** Low | **Evidence:** CODE INSPECTION (line 340)
- **Stage:** Stage 2 (at minimum, mention it in the guidance).

#### #60 — Rapid sequential uploads can race
- **Category:** Potential — Needs Testing | **Severity:** Low | **Evidence:** CODE INSPECTION
- **Detail:** Each change creates a new `FileReader`. If a large first file finishes after a small second one, the first overwrites `data`.
- **Stage:** Stage 2.

### Content, CSS, animation, accessibility, responsiveness

#### #24 — Guidance rounding rule contradicts its own example
- **Category:** Confirmed Bug | **Severity:** Medium | **Evidence:** CODE INSPECTION (line 234)
- **Actual:** "rounded to the **nearest** integer (for example, **80.2 → 81**)". Nearest would give 80, and 81 is rounding **up** (ceiling).
- **Why it matters:** Instructors following the example versus the rule produce different grades at boundaries (e.g. 79.2 → 79 = A- vs 80 = A).
- **Recommended fix:** Confirm the official BITS rule, then fix either the word or the example.
- **Stage:** Stage 1.

#### #39 — Lift / pulse animations removed after 30 ms
- **Category:** Confirmed Bug | **Severity:** Low | **Evidence:** CODE INSPECTION (lines 385, 392, 520 vs. CSS transitions 0.25 s, lines 130, 143). Visual effect NEEDS BROWSER.
- **Actual:** The class is removed after 30 ms of a 250 ms transition, so the card moves roughly 0.2 px. The feedback is essentially invisible.
- **Recommended fix:** Remove the class after ≥ 250 ms (or use `animationend`).
- **Stage:** Stage 1.

#### #49 — Reduced-motion handling contradictory and ignores canvas animation
- **Category:** UX/UI | **Severity:** Low | **Evidence:** CODE INSPECTION (lines 203–208)
- **Detail:** The first block disables transitions. A duplicate second block re-applies `.grade.lift{transform}`. The histogram/curve JS animation (rAF, 400 ms) ignores `prefers-reduced-motion` entirely.
- **Recommended fix:** Merge the blocks, and check `matchMedia('(prefers-reduced-motion: reduce)')` in `drawHistogram` to draw instantly.
- **Stage:** Stage 1/2.

#### #34 — Not responsive
- **Category:** UX/UI | **Severity:** Medium | **Evidence:** CODE INSPECTION. Layout NEEDS BROWSER.
- **Detail:** There is no `<meta name="viewport">`, so phones render a zoomed-out desktop page. `.workspace` uses a fixed `420px` column and `.controls` a fixed 3-column grid, with no media queries. The canvas is fixed at 380 px. At widths below about 800 px, horizontal overflow and cramped controls are expected.
- **Recommended fix:** Add the viewport meta, a single-column layout under ~800 px, and a responsive canvas.
- **Stage:** Stage 1 (viewport + stacking) / Stage 2 (polish).

#### #35 — Accessibility gaps
- **Category:** UX/UI | **Severity:** Medium | **Evidence:** CODE INSPECTION. Screen-reader behaviour NEEDS BROWSER.
- **Detail:**
  - The instructor input has a placeholder but no `<label>`. The file input and course select have no labels.
  - The 16 grade selects have no accessible names (the "Min"/"Max" text is in a `div`, not a `<label>`), so a screen reader only hears "80, combo box".
  - `#rangeError`, `#welcome` and `#thankyou` have no `aria-live`, so errors are not announced.
  - The canvas has no text alternative (a data table or `aria-label`).
  - `<html>` has no `lang`.
  - Keyboard operation is technically possible (native controls), but see #50.
- **Stage:** Stage 1 (labels, live region) / Stage 2 (canvas alternative).

#### #51 — Guidance collapsed by default; summary has `cursor:default`
- **Category:** UX/UI | **Severity:** Low | **Evidence:** CODE INSPECTION (lines 63–64, 226)
- **Detail:** The critical format rules are hidden, and the arrow cursor suggests they are not clickable.
- **Stage:** Stage 2.

#### #55 — Relies on implicit global element IDs
- **Category:** Robustness (code quality) | **Severity:** Low | **Evidence:** CODE INSPECTION
- **Detail:** The code uses `file`, `course`, `instructor`, `min`, `max`, `avg`, `med`, `download`, `grades`, `hist`, etc. as bare variables through the browser's named-element access. This is fragile: any future global with the same name shadows it, and it's a common source of confusion (it contributed to the misleading `min`/`max` swap, #5).
- **Recommended fix:** Use explicit `document.getElementById` constants.
- **Stage:** Stage 2.

#### #63 — File name has a double extension `.html.html`
- **Category:** UX/UI | **Severity:** Low | **Evidence:** Observed on disk.
- **Stage:** Stage 1 (rename only if the brief doesn't require the exact filename).

### Performance

#### #56 — Grade-assignment logic duplicated and re-reads DOM per student
- **Category:** Performance | **Severity:** Low | **Evidence:** EXECUTED (T37: ~280–330 ms per range change/download for a 10k-student course in mock DOM; real DOM likely slower)
- **Root cause:** The same matching loop appears in `computeGradeSummary` (lines 505–513) and `download.onclick` (lines 536–545). Each performs up to 16 `getElementById` + parses **per student**. `data.filter` runs four times per `updateAll`.
- **Recommended fix:** Read ranges once into an array, filter the course once, and share one `gradeFor(mark)` function.
- **Stage:** Stage 2 (Stage 1 if the fix for #4 touches it anyway).

#### #57 — Very large files: parse and option creation on the main thread
- **Category:** Performance | **Severity:** Low (Medium combined with #1) | **Evidence:** EXECUTED for option count (T37: 50,000 options). UI freeze NEEDS BROWSER.
- **Stage:** Stage 1 (fixing #1 resolves most of it) / Stage 2 (worker).

#### #58 — Rebuilds 1,616 `<option>` elements on every course change
- **Category:** Performance | **Severity:** Low | **Evidence:** CODE INSPECTION (lines 375–378: 8 grades × 2 selects × 101 options)
- **Stage:** Stage 2.

### Stage 2 enhancements (not bugs)

| # | Enhancement |
|---|---|
| 64 | Replace legacy `readAsBinaryString` with `readAsArrayBuffer` + `{type:"array"}`. |
| 65 | Descriptive download name, e.g. `CS101_grades_2026-09-28.csv`. |
| 66 | Persist work (e.g. `localStorage`) and warn on refresh/close with unsaved edits. |
| 67 | Histogram: y-axis, bin counts, axis titles, vertical lines at grade cut-offs. |
| 68 | CSV: course per row, grade-range table, timestamp, total students, instructor, and an optional `.xlsx` export. |
| 69 | Replace `alert`/`confirm` with in-page, accessible notices and dialogs. |
| 70 | Summary: total students, percentage per grade, and an "Ungraded" count (Stage 1 if used to fix #4). |

---

## A. CONFIRMED STAGE 1 BUGS
Recommended for Stage 1, highest severity first:

1. **#4** Students silently dropped from summary/CSV (**Critical**)
2. **#1** Duplicate course options (High)
3. **#2** Stale course options after re-upload (High, author-flagged)
4. **#3** Display not refreshed on new upload; screen ≠ export (High)
5. **#5** Min/Max labels swapped (High)
6. **#6** Scale ends (A max 100 / E min 0) not validated (High)
7. **#7** Decimal marks dropped (High)
8. **#8** CSV not escaped (High)
9. **#9** Histogram bars not scaled (High)
10. **#10–#14** Import validation: headers, blank/non-numeric, text numbers, numeric course codes, out-of-range marks (High, Robustness)
11. **#15** Placeholder/empty course: NaN stats and empty export allowed (Medium)
12. **#16** Instructor name can be cleared before export (Medium)
13. **#17** Reset before course selection crashes (Medium)
14. **#18** File-dialog cancel crashes (Medium)
15. **#19, #20** Bell-curve alignment and scaling (Medium)
16. **#21** Attempt count global across courses (Medium)
17. **#22** Clock vs. message mismatch (Medium)
18. **#24** Contradictory rounding guidance (Medium)
19. **#25** CSV formula injection (Medium)
20. **#26–#28** Range UX: lost edits on course switch, non-cascading Max, vague errors (Medium)
21. **#32** Duplicate IDs undetected (Medium)
22. **#33** Unpinned CDN / no load-failure message (Medium)
23. **#36** Timer first-tick (Low, author-flagged)
24. **#37, #38, #39, #40, #41, #42, #43, #44, #45, #46** Low-severity confirmed/robustness items
25. **#34, #35** Responsive and accessibility basics (Medium; minimal Stage 1 portion)

## B. POTENTIAL / NEEDS TESTING
These need a real browser, real Excel files, or the official brief:

- **#23** Timer start point (brief)
- **#29** `.xls`-only picker (brief)
- **#30** Corrupt/wrong-type files (real files)
- **#31** Same-file re-upload (browser)
- **#48** Overlapping rAF animations (browser timing)
- **#50** Alert on keyboard arrow (browser/OS)
- **#52** Min = Max bands (brief/policy)
- **#53** CSV encoding in Excel (Excel)
- **#54** Detached anchor download / URL leak (browser)
- **#60** Upload race (browser timing)
- **#61** HiDPI blur (display)
- Also: every **real-Excel parsing** behaviour behind #10–#13 (header whitespace, cell types) needs confirmation with actual `.xls`/`.xlsx` files, because SheetJS was mocked.
- Also: the **visual** results of #9, #19, #20, #34, #39 were verified numerically or by code only, not rendered.

## C. STAGE 2 PRODUCT IMPROVEMENT OPPORTUNITIES
- **#64–#70** (table above)
- **#26** (per-course range memory)
- **#47** (don't re-animate on range change)
- **#51** (guidance visibility)
- **#55** (explicit DOM references)
- **#56–#58** (performance refactors)
- **#59** (sheet selection)
- **#62** (timer >60 min indicator)
- **#35** (full accessibility incl. data-table alternative)
- **#34** (full responsive polish)

Further ideas:
- Sorted course list with student counts
- Drag-handles on the histogram to set cut-offs
- Undo
- Preview table of graded students before export
- Explicit NC handling
- Print-friendly summary
- Dark mode

---

## Testing Matrix

**Legend**
- **Passed / Failed (EXECUTED)**: the app's real script ran in the Node harness with mocked DOM/SheetJS.
- **Failed/Passed (INFERRED)**: code inspection only; not run.
- **Not tested**: requires a real Excel file and/or real browser interaction.

| Scenario | Test ID | Result | Related # |
|---|---|---|---|
| Normal flow: upload → name → course → grades → CSV | T01 | **Passed** for grade assignment; **Failed** for course list & Min/Max labels (EXECUTED) | 1, 5, 38 |
| No file uploaded, then select course | — | Passed (INFERRED): only the placeholder exists, so nothing to select | — |
| No instructor name, then select course | T04 | **Passed** (EXECUTED): alert and course reset | 50 |
| Instructor name cleared after selection, then finalize | T31 | **Failed** (EXECUTED) | 16 |
| Wrong file type (PDF/image/renamed) | — | **Not tested**: needs real file + real SheetJS | 30 |
| `.xlsx` file via picker | — | **Not tested**: needs browser (accept filter inferred) | 29 |
| Empty Excel file | T06 | **Failed** (EXECUTED): no feedback | 43 |
| Missing / incorrect column names | T11 | **Failed** (EXECUTED, mocked parser) | 10 |
| Extra columns | — | Passed (INFERRED): extra keys ignored | — |
| Blank cells in marks | T08 | **Failed** (EXECUTED) | 11, 4 |
| Non-numeric marks | T08 | **Failed** (EXECUTED) | 11, 4 |
| Numeric marks stored as text | T09 | **Failed** (EXECUTED; real-file cell typing not tested) | 12 |
| Negative marks | T08 | **Failed** (EXECUTED) | 14 |
| Marks above 100 | T08 | **Failed** (EXECUTED) | 14 |
| Decimal marks (79.5) | T08 | **Failed** (EXECUTED) | 7 |
| Marks exactly on boundaries (0,19,20,29,79,80,100) | T07 | **Passed** (EXECUTED) | — |
| Duplicate BITS IDs / duplicate records | T35 | **Failed** (EXECUTED): no warning | 32 |
| Multiple courses | T01 | **Failed** (EXECUTED): duplicate options | 1 |
| Numeric course code | T10 | **Failed** (EXECUTED) | 13 |
| Course names differing by whitespace | T36 | **Failed** (EXECUTED) | 44 |
| Row with missing course | T11 | **Failed** (EXECUTED) | 45 |
| Course with zero students / placeholder selected | T05 | **Failed** (EXECUTED) | 15 |
| Same file uploaded twice | T02 | **Failed** (EXECUTED) | 2 |
| Different files uploaded sequentially | T03 | **Failed** (EXECUTED) | 2 |
| New upload while course selected | T30 | **Failed** (EXECUTED) | 3 |
| Re-choose same file after editing it | — | **Not tested**: browser behaviour | 31 |
| Cancel file dialog | T29 | **Failed** (EXECUTED with simulated empty `files`; real browser event not tested) | 18 |
| Selecting courses repeatedly / switching | T25 | **Failed** (EXECUTED): custom ranges lost | 26 |
| Change a grade Min (valid) — cascade | T21 | **Passed** (EXECUTED) | — |
| Change a grade Max (gap) | T19 | Passed validation / **Failed** UX (EXECUTED) | 27, 28 |
| Change a grade Max (overlap) | T20 | Passed validation / **Failed** UX (EXECUTED) | 27, 28 |
| Min greater than Max | T18 | **Passed** (EXECUTED): error shown | 28 |
| Min equal to Max | T17 | Rejected (EXECUTED); correctness **spec-dependent** | 52 |
| Min cascade to −1 (D min 0) | T22 | **Failed** (EXECUTED): blank select | 41 |
| A Max lowered below 100 | T15 | **Failed** (EXECUTED) | 6, 4 |
| E Min raised above 0 | T16 | **Failed** (EXECUTED) | 6, 4 |
| Reset ranges after edits | T24 | **Passed** functionally; double confirm (EXECUTED) | 42 |
| Reset before course selection | T23 | **Failed** (EXECUTED): TypeError | 17 |
| Instructor/course names with commas, quotes | T26 | **Failed** (EXECUTED) | 8 |
| Formula-like values (`=…`) | T26 | **Failed** (EXECUTED); Excel evaluation not tested | 25 |
| Non-ASCII names in Excel | — | **Not tested**: needs Excel | 53 |
| Very large class (200 in one bin) — histogram | T13 | **Failed** (EXECUTED numerically; not rendered) | 9 |
| Very large dataset (50,000 rows) — performance | T37 | **Failed** for option count; timings are mock-DOM only | 1, 56, 57 |
| All students same mark | T12 | **Failed** (EXECUTED): no bell curve | 40 |
| Single student | T34 | Stats **Passed**; curve **Failed** (EXECUTED) | 40 |
| All students identical grade | T09/T12 | Passed (EXECUTED): counts correct | — |
| Extremely skewed distribution (all ~95) | T14 | **Failed** (EXECUTED): curve misaligned and off-canvas | 19, 20 |
| Repeated finalization / downloads | T27, T28 | **Failed** (EXECUTED): suffix, cross-course count, clock mismatch | 21, 22, 37 |
| Timer initial display | T32 | **Failed** per author's definition (EXECUTED); visual impact minimal | 36 |
| Timer start point vs. grading start | T27 | Behaviour confirmed (EXECUTED); pass/fail **spec-dependent** | 23 |
| Timer beyond 60 min | T33 | **Failed** (EXECUTED): no indication | 62 |
| Histogram re-animation on range change | T38 | **Failed** (EXECUTED) | 47 |
| Refreshing/reloading the page | — | Failed (INFERRED): all state lost, no warning | 66 |
| Browser resizing / mobile widths | — | **Not tested**: needs browser (inferred failure) | 34 |
| Keyboard-only interaction | — | **Not tested**: needs browser | 35, 50 |
| Screen reader | — | **Not tested**: needs browser + AT | 35 |
| Reduced-motion setting | — | **Not tested**: needs browser (inferred issues) | 49 |
| Lift/pulse animation visibility | — | **Not tested**: needs browser (inferred) | 39 |
| CDN unavailable / offline | — | **Not tested**: needs browser (inferred failure) | 33 |
| Console/runtime errors | T23, T29 | **Failed** (EXECUTED): 2 uncaught TypeErrors found | 17, 18 |
| Real `.xls` parsing end-to-end | — | **Not tested**: SheetJS was mocked | 10–13, 30 |

---

*End of audit. No changes were made to `BITS_Digital_CodeForge_Challenge.html.html`. The test harness lives only in the session scratchpad, outside the project folder.*

---

## Stage 1 Fixes Applied (summary — details in BUG_FIX_LOG.md)

| Bug / Issue Identified | How You Reproduced | Root Cause | Fix Implemented | How You Tested Fix |
|---|---|---|---|---|
| #1/#2 Course dropdown duplicates courses and keeps stale courses after re-upload. | Uploaded same/different files. | One option per row; never cleared. | Reset list; unique courses. | Options unique, old ones gone. |
| #3 Old stats shown after new upload. | Uploaded while course selected. | Panels not reset on upload. | Reset course and panels on upload. | Panels cleared; CSV matches screen. |
| #4 Unmatched students silently dropped from CSV. | Marks outside ranges. | No "no grade" handling. | Block Finalize, list ungraded students. | 101/101 exported; uncovered blocks. |
| #5 Min/Max labels swapped. | Viewed stats. | IDs swapped in HTML. | Swapped IDs. | Min 15 / Max 85 correct. |
| #6 Ranges needn't cover 0–100. | A Max 90 / E Min 5. | Ends not validated. | Require A Max 100, E Min 0. | Error shown, Finalize blocked. |
| #7 Decimal marks dropped. | Mark 79.5. | No whole-number check. | Reject with row number. | 79.5 rejected. |
| #8 CSV breaks on commas/quotes. | Name with comma/quote. | No CSV escaping. | RFC 4180 quoting. | Quoted output correct. |
| #9 Histogram bars overflow. | 200 students in one bin. | Fixed 12 px per student. | Scale to largest bin. | Tallest bar fits. |
| #10 Bad headers fail silently. | "Marks" header. | No header check. | Validate headers; show error. | Clear error shown. |
| #11 Blank/non-numeric marks give NaN. | "AB", blank cells. | No type validation. | Reject with row numbers. | Rows reported. |
| #12 Text marks concatenated. | "80","90" → avg 4045. | String addition. | Convert to Number. | Avg 85.00. |
| #13 Numeric course codes empty. | Course 101. | Number vs string compare. | Convert course to string. | Course 101 works. |
| #14 Marks <0 or >100 accepted. | −5, 105. | No range check. | Reject outside 0–100. | Both rejected. |
| #15 Placeholder course shows NaN, allows export. | Re-selected placeholder. | No empty-course guard. | Clear panels; block Finalize. | Blank panels, disabled. |
| #16 Instructor name clearable before export. | Cleared name, finalized. | Checked only on course change. | Require name to finalize. | Finalize blocked until re-entered. |
| #17 Reset before course crashed. | Clicked Reset first. | Selects not built yet. | Ignore until course chosen. | No error. |
| #18 Cancelling file dialog crashed. | Empty file list. | No file check. | Return early. | No error; data kept. |
| #19 Bell curve misaligned with bars. | Mean 95. | 30 px vs 32 px scale. | Shared x-scale. | Peak over its bar. |
| #20 Bell curve height arbitrary, off canvas. | Tight marks. | Fixed ×3000 scale. | Same y-scale as bars. | Curve fits chart. |
| #21 Attempts counted across courses. | Finalized 2 courses. | One global counter. | Count per course. | "first attempt" each. |
| #22 Clock ≠ finalize message time. | Finalized twice. | Clock frozen at first. | Redraw clock on finalize. | Both show 15:00. |
| #24 Rounding example contradicted rule. | Read guidance. | 80.2 → 81 typo. | 80.2 → 80, 80.5 → 81. | Text checked. |
| #25 CSV formula injection. | ID "=1+1". | No neutralising. | Prefix with '. | '=1+1 exported. |
| #29 .xlsx not accepted. | File picker. | accept=".xls". | Accept .xlsx,.xls. | Real .xlsx graded. |
| #30 Corrupt file: silent crash. | Bad file. | No try/catch. | Show error message. | PNG → message. |
| #31 Same file can't be re-uploaded. | Re-chose file. | Input kept value. | Clear on dialog open. | Value cleared. |
| #32 Duplicate BITS IDs exported. | S1 ×3 in a course. | No duplicate check. | Reject with rows. | Error names rows. |
| #33 Unpinned CDN library. | Offline / script tag. | No version/SRI/check. | Pin 0.20.3 + SRI + message. | Hash matches; message shown. |
| #36 Timer waits for first tick. | Page load. | No initial render. | Render immediately. | Drawn on load. |
| #37 "21th" ordinal. | 21 finalizes. | Only 2/3 handled. | Proper ordinal(). | "21st" shown. |
| #39 Feedback animations invisible. | Changed a range. | 30 ms timeout. | 250 ms. | Code checked. |
| #40 Bell curve vanishes (std 0). | Identical marks. | Divide by zero. | Skip curve. | Bars only, no NaN. |
| #42 Reset asks twice. | Clicked Reset. | Two confirms. | One confirm. | 1 dialog. |
| #43 Empty file: no feedback. | Empty upload. | No check. | Message shown. | Message shown. |
| #44 Spaced course names split. | "CS101 ". | No trim. | Trim names. | One option. |
| #45 Blank course → blank option. | Empty Course cell. | No check. | Reject row. | Error shown. |
| #48 Stale chart redraw. | Quick course switch. | Animations not cancelled. | Cancel previous. | Chart stays cleared. |
| #49 Reduced motion ignored. | OS setting. | Duplicate CSS; JS ignored. | One block; instant chart. | Single frame. |
| #53 CSV garbles non-English names. | Excel open. | No BOM. | Add UTF-8 BOM. | BOM present. |
