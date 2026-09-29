# Grading Console

A single-page web app for BITS Pilani Digital instructors. Give it a marks sheet, pick a course, set the grade cut-offs, and it shows what every cut-off does to the class before you export the final grades as a CSV.

No build step, no backend, no install. It is one file, `index.html`. Marks are processed in the browser and never leave the instructor's machine.

## What it does

1. **Instructor** enters their name (required to finalize).
2. **Marks** are loaded from an Excel file (`.xlsx` / `.xls`), **or typed / pasted straight into the app** (see [Enter marks without leaving the app](#enter-marks-without-leaving-the-app)).
3. **Course**: choose one course from the file.
4. **Grade ranges**: adjust the cut-offs. The grade scale, histogram and per-grade counts update live.
5. **Export**: finalize and download `grades.csv`.

**Input:** three columns, `BITS ID`, `Course`, `Total Marks`. Marks are whole numbers from 0 to 100. Students who are to receive an `NC` grade are left out of the file.

**Default ranges:** A 80–100, A- 70–79, B 60–69, B- 50–59, C 40–49, C- 30–39, D 20–29, E 0–19.

**Output:** `grades.csv` with the instructor and course on top, then `BITS ID, Total Marks, Grade` for every student.

## Top 3 enhancements

These are the three changes that matter most to an instructor using the console.

### 1. Enter marks without leaving the app
An in-card spreadsheet editor with paste from Excel, live validation, and one-click **Fix in editor** for rejected uploads. It runs through the same validation as a file upload. [Details below.](#enter-marks-without-leaving-the-app)

### 2. Live grade scale, histogram and range table
Changing any cut-off immediately redraws:
- a **grade scale** with one dot per student, shaded grade bands, hatching over any marks no grade covers, and the class average marked;
- a **histogram** of the marks with a normal curve on the same scale;
- the **grade table** with a student count and share bar for each grade.

Instructors see the effect of every decision before they commit, instead of grading blind and checking a CSV afterwards.

### 3. Precise, all-or-nothing validation, with a guarded Finalize
The whole file is checked before anything is loaded, and every problem is reported with its row number (for example `Row 4: Total Marks 79.5 must be a whole number`). Finalize stays disabled, with the reason stated, until the ranges cover 0–100 without gaps or overlaps, every student falls in a grade, and the instructor name is filled in. Wrong data cannot silently reach the exported CSV.

## Enter marks without leaving the app

**The problem.** A marks sheet is rarely clean the first time. A stray `8o` for `80`, a blank cell, a mark like `72.5`, a duplicated ID or a misnamed column is enough for the console to reject the whole file, because a half-loaded class would mean wrong grades. Before, the only way forward was to leave the console, open the file in Excel, find the row, save, come back, and upload again. That repeated for every mistake. Some instructors also don't have a file at all: the marks are in an email, an LMS page or a spreadsheet they only want a few rows from.

Leaving the console matters here because the **session timer keeps running**: the console reports how long grading took, so every trip to Excel adds to the clock.

**The solution.** The marks card has an **Upload file | Enter manually** switch. Enter manually turns the same card into a spreadsheet-style grid (BITS ID, Course, Total Marks) with:

- **Paste from Excel or Sheets** with `Ctrl+V`: tab-separated data fills as many rows as needed, and a header line is skipped automatically.
- **Live validation** using exactly the same rules as file upload: blank cells, non-numeric marks, marks outside 0–100, decimals, and duplicate IDs within a course. Bad cells are marked as you type, and the footer names the first problem.
- **Fix in editor**: when an upload is rejected, one click opens the file's own rows in the grid with the bad cells already highlighted. Fix them and import. If a file has rows but a required column is misnamed, the grid maps it for you (for example a lone `Marks` column becomes `Total Marks`) and asks you to check.
- **Spreadsheet keyboard model**: Tab, Enter, and arrow keys move between cells, a new row copies the course from the row above, and `Alt+Delete` removes the current row.
- **Expand** for large classes: the same grid opens in a large dialog and returns to the card when closed.
- **Nothing is lost**: switching between Upload and Manual keeps your draft in memory.
- **Scales up**: handles 1,000+ rows per import; only the edited row is re-checked as you type.

Because typed rows go through the same `parseRows` step as an uploaded file, everything downstream (course list, analytics, grade ranges, CSV export) works identically. There is one set of rules, so a row cannot pass in one path and fail in the other.

The editor is desktop-only (1024px and wider). On smaller screens the card is upload-only.

## UI / UX

- **Welcome screen** with an animated grade scale that also waits for the Excel reader to load, so uploads never hit a half-ready page.
- **Light and dark themes**: follows the system setting, remembers the instructor's choice, and every colour comes from shared design tokens.
- **Guided steps** in the top bar (Instructor, Marks file, Course, Grade ranges, Export) that tick off as each is completed.
- **Upload zone with clear states**: idle, drag-over, reading, success and error. Errors list each problem with its row number.
- **Format guidance** on the page, next to the upload.
- **"Chalkboard" grade scale** as the one bold visual, with charts and tables in a calm, consistent style.
- **Sticky export bar** that always shows whether the class is ready to export, and if not, why.
- **In-page dialogs and toasts** instead of browser `alert` / `confirm` popups.
- **Session timer** that stops on download and reports the time taken.
- **Responsive layout** from phone to wide desktop.
- **Accessibility**: labelled controls, visible focus rings, screen-reader announcements for status and errors, and `prefers-reduced-motion` respected throughout.

## Bugs fixed

40 defects were found in the original code and fixed. The tables below give each one's cause and fix. The **full log**, with how each bug was reproduced and how each fix was tested, is in [`BUG_FIX_LOG.md`](BUG_FIX_LOG.md). The complete list of findings is in [`BUG_AUDIT.md`](BUG_AUDIT.md). `#` numbers match those documents.

### Course list and stale data

| # | Bug and cause | Fix |
|---|---|---|
| 1 | The course dropdown listed each course once per student row (50,000 rows made 50,000 options). Every row added an option with no de-duplication. | Courses are de-duplicated, in order of first appearance. |
| 2 | Old course options stayed after uploading another file, and uploading the same file twice doubled the list. Options were never cleared. | The dropdown resets before each upload is loaded. |
| 3 | After a new upload the screen showed old stats while the CSV used the new data. Only the data was replaced, not what was derived from it. | `clearGrading()` resets the course, grade cards, stats, summary and Finalize on every upload. |
| 15 | Going back to "Select a course" showed `undefined` / `NaN` and still allowed Finalize with an empty CSV. No handling for an empty course. | The placeholder clears the grading view, blanks the stats and blocks Finalize. |
| 44 | `CS101`, `CS101 ` and ` CS101` became separate courses. Course names weren't trimmed. | Course names are trimmed on import. |
| 13 | A numeric course code such as `101` never matched, so the course looked empty. Option values were strings and rows kept numbers. | Course is converted to a trimmed string on import. |

### Data validation on import

| # | Bug and cause | Fix |
|---|---|---|
| 10 | Missing or misnamed columns failed silently and gave `undefined` / `NaN`. Hard-coded exact header names, no check. | Headers match ignoring case and spaces. A missing column or an empty file shows a clear message. |
| 43 | An empty file gave no feedback. No empty check. | "File not loaded: it contains no student rows." |
| 11 | Blank or non-numeric marks (`AB`, empty) gave `NaN` stats and dropped students. No type validation. | Rejected on import with the Excel row number (first 10 shown, then "and N more"). |
| 45 | A row with no course created a blank option. No blank check. | A blank course is rejected with its row number. |
| 14 | Marks below 0 or above 100 were partly shown, then dropped. No range check. | Rejected on import with row numbers. |
| 7 | Decimal marks (79.5) fell between grade ranges and vanished from the CSV. Integer ranges, no whole-number check. | Non-whole marks are rejected with the row number. |
| 12 | Marks stored as text were joined instead of added (average of 80 and 90 showed `4045.00`). `+` on strings concatenates. | Marks are converted to numbers once, on import. |
| 32 | The same BITS ID twice in a course was exported twice, sometimes with two grades. No duplicate check. | Repeats within a course are rejected (case-insensitive) and both rows are named. The same ID in different courses is allowed. |
| 30 | A corrupt or non-Excel file caused an uncaught error and no message. Parsing had no error handling. | Parsing and file reads are guarded and show "File not loaded: ...". |
| 18 | Cancelling the file dialog crashed. The code read `files[0]` without checking. | Returns early and keeps the loaded data. |
| 31 | Choosing the same (edited) file again did not reload it. The input kept its old value. | The input is cleared each time the dialog opens. |
| 29 | The file picker accepted only `.xls`, hiding `.xlsx`. Wrong `accept` filter. | Now accepts `.xlsx` and `.xls`. |
| 33 | The Excel library loaded unpinned (a version with known vulnerabilities), unchecked, and failed silently offline. Unversioned CDN URL, no integrity check. | Pinned to SheetJS 0.20.3 with a SHA-384 integrity hash. A clear message is shown if it can't load. |

### Grading correctness

| # | Bug and cause | Fix |
|---|---|---|
| 4 | Students whose marks matched no grade were silently left out of the summary and CSV. There was no "no match" case. | One shared `gradeOf()` for summary and export. Finalize is blocked and lists any unmatched students. |
| 6 | Grade ranges didn't have to cover 0–100, so students could fall outside. Only Min < Max and continuity were checked. | A's Max must be 100 and E's Min must be 0. |
| 5 | The Min and Max statistic labels were swapped. Their HTML ids were the wrong way round. | Swapped back. |
| 16 | The mandatory instructor name could be cleared after choosing a course, and the CSV still exported. Checked only when the course changed. | Finalize requires a non-blank name at all times. The welcome text updates as you type. |
| 17 | Reset Range before choosing a course crashed. It wrote to controls that didn't exist yet. | Does nothing until a course is selected. |
| 42 | Reset Range asked for confirmation twice. Two `confirm()` calls in a row. | A single confirmation. |

### Charts

| # | Bug and cause | Fix |
|---|---|---|
| 9 | Histogram bars ran off the chart above 17 students per bin. Fixed 12 px per student. | Bars scale to the tallest bin. |
| 19 | The bell curve was shifted sideways from the bars. Bars and curve used different widths. | One shared geometry maps each bin centre to its bar centre. |
| 20 | Bell curve height was an arbitrary constant and could leave the chart. Density multiplied by a fixed number. | Drawn as expected students per bin on the bars' own scale, so both always fit. |
| 40 | The curve silently disappeared when all marks were equal. Division by zero when the spread is 0. | The curve is skipped in that case and the bars still draw. |
| 48 | An old chart animation could redraw after the chart was cleared. Animation frames were never cancelled. | Running animations are cancelled on redraw and on clear. |
| 49 | Reduced-motion was contradictory in the CSS and ignored by the chart. Duplicate CSS block, no JS check. | Duplicate removed. The chart draws in one frame when reduced motion is set. |
| 39 | Grade-card highlight and count pulse were invisible. The class was removed after 30 ms of a 250 ms transition. | Timeouts match the transition. |

### Export and messages

| # | Bug and cause | Fix |
|---|---|---|
| 8 | Commas and quotes in values broke the CSV. Values were joined without quoting. | Fields are quoted and escaped per RFC 4180. |
| 25 | CSV values starting with `=`, `+`, `-` or `@` could run as spreadsheet formulas. No neutralising. | Such values are prefixed with `'`. |
| 53 | The CSV had no UTF-8 marker, so Excel could garble non-English names. No BOM. | A UTF-8 BOM and charset are added. |
| 21 | The attempt counter counted all courses together. One page-wide counter. | Attempts are counted per course. |
| 22 | After the first Finalize the clock froze, but later messages reported a longer time. The message used the current time, the clock the frozen one. | Each Finalize redraws the clock with the same elapsed time used in its message. |
| 37 | Ordinals were wrong after the third attempt ("21th", "22th"). Only 2 and 3 were special-cased. | Correct words and suffixes (`21st`, `22nd`, `111th`). |
| 36 | The timer display waited for the first tick before appearing. Only `setInterval`, no initial render. | Rendered once immediately, then every second. |
| 24 | The file guidance said "nearest integer" but gave `80.2 → 81`. Contradictory example. | Example corrected to `80.2 → 80` and `80.5 → 81`. |

**How the fixes were tested:** the app's script was run in Node.js with a mocked DOM (88 checks, including real `.xls` / `.xlsx` and corrupt-file tests with the real SheetJS library). The manual-entry editor was tested end to end in headless Chrome. Visual review of the original fixes in a real browser was not part of that run.

## Run it

Open `index.html` in a browser. The Excel reader loads from the SheetJS CDN, so an internet connection is needed the first time.

## Deploy

Deployed as a static site (for example on Vercel):

- Framework preset: **Other**
- Build command: none
- Output directory: none (the repository root)

`index.html` is served at `/`. `.vercelignore` keeps the audit documents out of the deployed site.

## Repository

| File | Purpose |
|---|---|
| `index.html` | The whole app: HTML, CSS and JavaScript |
| `BUG_FIX_LOG.md` | Full fix log: reproduction, root cause, fix and test for each bug |
| `BUG_AUDIT.md` | Complete audit of the original code, including findings not fixed and later enhancement ideas |
| `.vercelignore` | Excludes the documents above from the Vercel deployment |
