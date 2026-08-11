# Issues: comprehensive-ecommerce-workflow

> **Source:** `20260810-175058-comprehensive-ecommerce-workflow.full.md` | **Date:** 20260810-175058 | **Mode:** dev

## Scenario Background

### Task

The task completed successfully. All 18 steps were executed:

- **Product Detail Page (4K OLED TV 55):** Title, price ($899.99), image URL, description, features (55 inch, HDR10+, OLED panel, 3840x2160 resolution), stock status, and category were extracted via multiple methods (htmlsnapshot get, X-SQL query, eval, extract).
- **Electronics Listing Page:** 6 products extracted with titles and prices.
- **Browser state** saved to `.test-sessions/browser-state-2026-08-10.json`.
- **Summary report** written to `.test-sessions/product-summary-report.md`.

### Execution Context

| Step | Command | Outcome |
|------|---------|---------|
| 0 | Verified pwd at repo root | OK |
| 0 | Read `./b4w.ps1 help` + `SKILL.md` | OK |
| 1 | `goto "http://localhost:18080/ec/"` | OK — session created, page loaded |
| 2 | `snapshot -v 0 --stdout` | OK — full viewport AX tree |
| 3 | `snapshot -i --stdout` | OK — interactive elements (verbose) |
| 4 | `click e134` (first product link) | OK — navigate to product detail |
| 5 | `htmlsnapshot` | OK — captured 15KB snapshot |
| 6 | `htmlsnapshot inspect --max 3 --depth 2` | Partial — auto-discovered recommendation cards, not main product |
| 7 | `htmlsnapshot get all text "#details"` + `eval --base64` | OK via eval workaround |
| 8 | X-SQL `@file` query | OK — extracted title/price/image |
| 9 | `htmlsnapshot grep 'price.*\$'` | OK — f...

(truncated — see full.md for complete trace)

---

## Issues Found (8 issues)

### Issue 1: screenshot positional argument misinterpreted as CSS selector instead of filename

**Severity:** Medium
**Category:** UX

#### Reproduction

./b4w.ps1 screenshot ".test-sessions/my-file.png"

#### Expected Behavior

Screenshot saved to .test-sessions/my-file.png, or an error message that the path is not a valid CSS selector.

#### Actual Behavior

Screenshot saved to default .browser4-cli/snapshot/screenshot-<timestamp>.png with no error or warning about the ignored argument. The 3-byte file suggests a broken capture.

#### Root Cause Analysis

The positional argument to `screenshot` is defined as `[ref]` (CSS selector or snapshot ref), not a filename. The `--filename`/`-o` flag must be used for custom paths. The CLI silently treats an invalid path as an unresolved CSS selector and falls back to a full-page screenshot saved at the default location, without warning the user.

#### Code Pointer

`cli/browser4-cli/src/commands/screenshot.rs: argument parsing and resolution logic`

#### AI Suggested Improvement

- When the positional argument contains a path separator (/ or \) and does not look like a CSS selector, warn the user that they probably meant --filename
- Consider a heuristic: if the arg ends in .png/.jpg/.jpeg and doesn't start with #/./[a-zA-Z], suggest --filename
- Add a tip to screenshot help: "To specify a filename, use --filename or -o; the positional argument is a CSS selector."

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: htmlsnapshot inspect auto-discovers wrong container pattern when no selector given

**Severity:** Medium
**Category:** UX

#### Reproduction

On a product detail page, run `./b4w.ps1 htmlsnapshot inspect --max 3 --depth 2` without specifying a selector.

#### Expected Behavior

The tool should inspect the main content area (#product-page) or prompt the user to specify a starting selector when multiple repeating patterns exist.

#### Actual Behavior

The tool auto-discovered `.recommendation-card` elements (the 'Customers also viewed' section at the page bottom) instead of the product detail content. The product's title, price, description, and image selectors were not surfaced.

#### Root Cause Analysis

Inspect's auto-discovery walks from `:root` looking for sibling-group repeating patterns. The recommendation cards (4 siblings) are more structurally regular than the single product detail container, so they're picked first. The user has no way to direct discovery toward the main content without already knowing the right container selector.

#### Code Pointer

`browser4-core/... (htmlsnapshot inspect logic — auto-discovery heuristic from :root)`

#### AI Suggested Improvement

- When multiple sibling-group patterns exist, list them all with a summary line so the user can pick one with `inspect <selector>`
- Prioritize patterns with larger bounding boxes (main content area) over smaller ones (sidebars, recommendations)
- Add a hint after auto-discovery: "To inspect the main product area, try: htmlsnapshot inspect '#product-page'"
- Document in SKILL.md that `htmlsnapshot inspect` with no selector auto-discovers from :root and may pick sidebars

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: htmlsnapshot get all text with class-based selectors fails silently on product description

**Severity:** Medium
**Category:** Reliability

#### Reproduction

./b4w.ps1 htmlsnapshot get all text "#product-page [class*='about']" on the product detail page after htmlsnapshot capture.

#### Expected Behavior

Returns text content from elements matching the attribute selector.

#### Actual Behavior

Returns empty array with message: "No elements matched. The snapshot may be stale..." even though the selector was based on observed class names in the page. Multiple attempts with different class patterns ([class*='about'], .about-section, [class*='section']) all returned empty.

#### Root Cause Analysis

Likely a mismatch between the CSS class names rendered in the stored HTML snapshot and the class patterns being tried. The htmlsnapshot captures initial server HTML, but the class names may not match expected patterns. Alternatively, the attribute selector syntax may have limited support in the htmlsnapshot CSS engine.

#### Code Pointer

`browser4-core/... htmlsnapshot CSS selector engine (class attribute matching)`

#### AI Suggested Improvement

- When no elements match, suggest using `htmlsnapshot inspect` with a broader selector to discover actual class names
- Consider adding a `--suggest` flag that prints the top-N class names found in the snapshot when a selector returns empty
- Document common class name patterns for MockSite product pages in the scenario description

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: Interactive-only snapshot (-i) output is excessively verbose due to parent text accumulation

**Severity:** Low
**Category:** UX

#### Reproduction

./b4w.ps1 snapshot -i --stdout on a content-rich e-commerce page.

#### Expected Behavior

Interactive-only snapshot should show clearly delineated interactive elements (buttons, links, inputs) with concise labels.

#### Actual Behavior

Parent elements accumulate all descendant text into their accessible names. For example, a product card `article` element shows its entire text content concatenated (title + price + rating + description + badge), making it hard to distinguish interactive children from noise.

#### Root Cause Analysis

The AX tree's accessible name computation for generic container elements includes all descendant text. In `-i` mode, parent containers that have interactive descendants are still shown with their accumulated text names, defeating the purpose of a 'clean' interactive view.

#### Code Pointer

`cli/browser4-cli/src/snapshot rendering — interactive-mode filtering logic`

#### AI Suggested Improvement

- In `-i` mode, truncate or suppress the accessible name of generic container elements that only serve as wrappers for interactive children
- Add an `--interactive-flat` mode that shows interactive elements as a flat list without parent hierarchy
- Consider a `--terse` flag that strips redundant parent text when it's a superset of child text

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 5: extract command defaults to file output requiring extra read step

**Severity:** Low
**Category:** UX

#### Reproduction

Run `./b4w.ps1 extract "product name and price"` without --stdout.

#### Expected Behavior

Extracted data displayed inline, or a clear message that --stdout is available for inline output.

#### Actual Behavior

Output is a file path reference. User must `cat` the file separately to see the content. The tip about --stdout only appears in --help, not in the default output.

#### Root Cause Analysis

Extract is designed for potentially large outputs, so the default is file-based. However, for quick interactive use, this adds friction. The output message mentions the file path but doesn't hint about --stdout.

#### Code Pointer

`cli/browser4-cli/src/commands/extract.rs`

#### AI Suggested Improvement

- Add a tip to the extract output: "Use --stdout to print content directly instead of saving to a file."
- Consider whether the default should be --stdout for short extractions (under a size threshold) and file-based for larger ones
- Mention the --stdout flag in the SKILL.md extract examples more prominently

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 6: htmlsnapshot get all prints misleading warning for unique-ID selectors

**Severity:** Low
**Category:** UX

#### Reproduction

Run `./b4w.ps1 htmlsnapshot get all text "#product-page"` on a page where #product-page is a unique container.

#### Expected Behavior

The output clearly indicates one match was found (expected for a unique ID).

#### Actual Behavior

Output says "Only 1 result(s) found for '#product-page'. The page structure may have changed since the snapshot was captured. Try htmlsnapshot inspect..." — this reads as a warning/error when 1 result is correct for a unique ID selector.

#### Root Cause Analysis

The warning logic applies the same message for both genuinely low match counts (stale snapshot) and naturally unique selectors. The threshold is '≤1' which triggers for all ID selectors.

#### Code Pointer

`cli/browser4-cli/src/ — htmlsnapshot get all output formatting logic`

#### AI Suggested Improvement

- Distinguish between 'no matches' (stale snapshot warning) and '1 match' (normal for IDs, neutral message)
- Suppress the stale-snapshot warning when the selector contains an #id pattern
- Change the message for 1-match ID selectors to: "1 result found (expected for unique ID selector)"

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 7: Invocation confusion: ./b4w.ps1 vs ./b4w.sh on Linux

**Severity:** Low
**Category:** Documentation

#### Reproduction

Read CLAUDE.md, SKILL.md, and the task instructions side by side. CLAUDE.md says use ./b4w.sh on Linux; SKILL.md says ./b4w.sh for Linux/macOS; task instructions mandate ./b4w.ps1.

#### Expected Behavior

A single, consistent invocation command for all platforms, or clear platform-specific guidance that doesn't conflict.

#### Actual Behavior

Three different sources give different guidance. ./b4w.ps1 worked on Linux because pwsh was installed, but this is an implicit dependency that may not be documented. A first-time Linux user seeing ./b4w.ps1 would be confused.

#### Root Cause Analysis

The task template mandates ./b4w.ps1 for consistency, but the project's own documentation (CLAUDE.md, SKILL.md) recommends ./b4w.sh for Linux. The .ps1 wrapper delegates to PowerShell which may not be installed on all Linux systems.

#### AI Suggested Improvement

- Update the task template to use platform-appropriate commands (./b4w.sh on Linux/macOS, ./b4w.ps1 on Windows)
- Add a note in CLAUDE.md that ./b4w.ps1 requires PowerShell (pwsh) on Linux
- Consider a single cross-platform wrapper script (e.g., ./b4w) that auto-detects the shell

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 8: state-save captures no cookies or localStorage despite browsing session

**Severity:** Low
**Category:** Product

#### Reproduction

Navigate multiple pages, then run `./b4w.ps1 state-save <file>`. Inspect the JSON.

#### Expected Behavior

If the site sets cookies or localStorage, they should be captured. If not, this should be documented.

#### Actual Behavior

The saved file contains empty cookies array and empty localStorage per origin. It's unclear whether MockSite doesn't set any storage, or whether state-save has a limitation.

#### Root Cause Analysis

MockSite likely doesn't set cookies or localStorage (it's a static mock). However, the output doesn't distinguish between 'no storage exists' and 'storage capture failed'. The file is only 118 bytes for a session spanning 2+ pages.

#### AI Suggested Improvement

- Add a diagnostic message when state-save captures no data: "No cookies or localStorage found — the site may not set any, or browser storage may be unavailable."
- Add a `--verbose` flag to state-save that lists what was checked per origin
- Document in SKILL.md that state-save captures whatever the browser has at save time, which may be empty for stateless sites

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

## Overall Assessment

**Completion Status:** Successful — all 18 task steps completed. Two workarounds were needed: using eval for product description extraction (htmlsnapshot get all returned empty), and accepting the default screenshot path when --filename flag was missed.

**Success Rate:** 89% — 16 of 18 steps completed on first attempt without workarounds. 2 steps required alternative approaches or had minor issues.

**Issues Found:** 8

**Major Blockers:** None. No step was entirely blocked. The screenshot filename issue and product description selector failures were workaround-able.

**Most Confusing Aspects:** 1) The distinction between `snapshot` (AX tree for interaction) vs `htmlsnapshot` (static HTML for extraction) is conceptually clear in docs but takes practice to internalize — several times I reached for the wrong tool. 2) Knowing which CSS selectors are valid requires trial-and-error; `htmlsnapshot inspect` helps but auto-discovers the wrong container. 3) The positional vs named argument distinction (e.g., screenshot ref vs --filename) is easy to miss.

**Most Valuable Improvements:** 1) Make `screenshot` warn when a path-like positional argument looks like a filename, not a CSS selector. 2) Improve `htmlsnapshot inspect` to show all discovered patterns when multiple exist, not just the first one. 3) Add --stdout hints to `extract` default output. 4) Unified platform wrapper script (./b4w) for cross-platform consistency.

**Usability Rating:** 7/10

---

## How to Reproduce

### Common Setup

1. Clone the repository and `cd` to the repo root.
2. The CLI is invoked via `./b4w.ps1` (PowerShell) or `./b4w.sh` (Bash / Git Bash), which auto-build from source when needed.
3. The backend server starts automatically in dev mode.
4. All commands from repo root:

   - **PowerShell:** `./b4w.ps1 <command>`
   - **Bash / Git Bash:** `./b4w.sh <command>`
   - **Direct:** `browser4-cli <command>` (if installed globally)

   > **Note:** `$(./b4w.ps1)` is command substitution in bash — do NOT use it.

### Per-Issue Reproduction Steps

#### Issue 1: screenshot positional argument misinterpreted as CSS selector instead of filename

./b4w.ps1 screenshot ".test-sessions/my-file.png"

#### Issue 2: htmlsnapshot inspect auto-discovers wrong container pattern when no selector given

On a product detail page, run `./b4w.ps1 htmlsnapshot inspect --max 3 --depth 2` without specifying a selector.

#### Issue 3: htmlsnapshot get all text with class-based selectors fails silently on product description

./b4w.ps1 htmlsnapshot get all text "#product-page [class*='about']" on the product detail page after htmlsnapshot capture.

#### Issue 4: Interactive-only snapshot (-i) output is excessively verbose due to parent text accumulation

./b4w.ps1 snapshot -i --stdout on a content-rich e-commerce page.

#### Issue 5: extract command defaults to file output requiring extra read step

Run `./b4w.ps1 extract "product name and price"` without --stdout.

#### Issue 6: htmlsnapshot get all prints misleading warning for unique-ID selectors

Run `./b4w.ps1 htmlsnapshot get all text "#product-page"` on a page where #product-page is a unique container.

#### Issue 7: Invocation confusion: ./b4w.ps1 vs ./b4w.sh on Linux

Read CLAUDE.md, SKILL.md, and the task instructions side by side. CLAUDE.md says use ./b4w.sh on Linux; SKILL.md says ./b4w.sh for Linux/macOS; task instructions mandate ./b4w.ps1.

#### Issue 8: state-save captures no cookies or localStorage despite browsing session

Navigate multiple pages, then run `./b4w.ps1 state-save <file>`. Inspect the JSON.

