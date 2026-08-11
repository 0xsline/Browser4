# Issues: crawl-advanced-extraction

> **Source:** `20260810-180401-crawl-advanced-extraction.full.md` | **Date:** 20260810-180401 | **Mode:** dev

## Scenario Background

### Task

The task was completed with mixed success:

- **Seed file**: Created with 3 MockSite Electronics product URLs (B0E000001, B0E000002, B0E000003) ✓
- **X-SQL query**: Created extracting `#productTitle` and `#product-price` via `DOM_LOAD_AND_SELECT` ✓
- **Baseline crawl** (`--refresh --parse --expires 1h --priority 1 --page-load-timeout 30s`): Successfully extracted all 3 products with correct titles and prices ✓
- **Background crawl**: Submitted asynchronously and completed, but extraction returned **empty** results for all 3 pages — despite using identical flags to the successful foreground crawl ✗
- **`--ignore-url-query` crawl**: Pages fetched but extraction returned **empty** ✗
- **`--no-norm` crawl**: Pages fetched but extraction returned **empty** ✗
- **`--readonly` crawl**: Only the **first** page extracted correctly; pages 2 and 3 returned empty ✗
- **Crawl list**: All tasks visible and tracked correctly ✓

The core extraction mechanism works under ideal conditions but fails silently under several flag combinations, making the feature unreliable for real-world use.

### Execution Context

| Step | Command | Outcome |
|------|---------|---------|
| Verify working dir | `pwd` | Correct |
| Create temp dir | `mkdir -p .test-sessions` | Done |
| Read help | `./b4w.ps1 help` | Comprehensive help displayed |
| Read SKILL.md | `Read skills/browser4-cli/SKILL.md` | Full documentation consumed |
| Read crawl.md | `Read skills/browser4-cli/references/crawl.md` | Understood crawl options |
| Check MockSite | `curl localhost:18080/ec/dp/B0E000001` | HTTP 200, running |
| Verify selectors | `goto` + `htmlsnapshot get text` | `#productTitle` and `#product-price` confirmed |
| Create seed file | `cat > .test-sessions/seed-urls.txt` | 3 URLs |
| Create X-SQL file | `cat > .test-sessions/extract.sql` | 3-column query |
| Baseline crawl | `crawl --seed-file ... --refresh --parse --expires 1h...

(truncated — see full.md for complete trace)

---

## Issues Found (8 issues)

### Issue 1: --ignore-url-query causes silent X-SQL extraction failure

**Severity:** High
**Category:** Product

#### Reproduction

./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract.sql --refresh --ignore-url-query --format table

#### Expected Behavior

All 3 product pages should have their title and price extracted via X-SQL, same as the baseline crawl without --ignore-url-query.

#### Actual Behavior

All 3 pages are fetched (contentLength > 0, status: fetched), but the X-SQL extraction returns empty strings for title and price on every page. The table output shows empty cells with no error or warning.

#### Root Cause Analysis

The --ignore-url-query flag likely affects how LoadOptions are passed to the X-SQL processor's DOM_LOAD_AND_SELECT re-fetch. At depth 0, --ignore-url-query should have no effect (there are no out-links to strip query params from), yet it appears to interfere with the X-SQL extraction pipeline. The pages are fetched by the crawl engine but the X-SQL layer may be looking up content under a different (non-query-stripped) URL key, or the flag may alter the internal URL representation used by the X-SQL scraper.

#### AI Suggested Improvement

- Investigate how --ignore-url-query interacts with the X-SQL DOM_LOAD_AND_SELECT re-fetch path
- Ensure LoadOptions flags that are irrelevant to X-SQL extraction (at depth 0) are not passed through to the X-SQL processor
- Add a warning when combining --ignore-url-query with --sql at depth 0 (the flag has no semantic meaning in this mode)
- Add integration tests for X-SQL extraction with each LoadOptions flag combination

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: --no-norm causes silent X-SQL extraction failure

**Severity:** High
**Category:** Product

#### Reproduction

./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract.sql --no-norm --format table

#### Expected Behavior

All 3 product pages should have their title and price extracted. --no-norm should only affect URL normalization for dedup/link resolution, not X-SQL content extraction.

#### Actual Behavior

All 3 pages are fetched but X-SQL extraction returns empty strings for every row. The same behavior occurs with or without --refresh.

#### Root Cause Analysis

Disabling URL normalization likely changes the internal URL key used to store/lookup fetched page content. When the X-SQL DOM_LOAD_AND_SELECT re-fetches @url, it may use a normalized URL key while the crawl engine stored content under a non-normalized key, or vice versa. This URL key mismatch causes the X-SQL processor to find no content for extraction.

#### AI Suggested Improvement

- Ensure URL normalization behavior is consistent between the crawl engine's page storage and the X-SQL processor's DOM_LOAD_AND_SELECT
- Add integration tests for X-SQL extraction with --no-norm
- Document the interaction between --no-norm and X-SQL extraction in the crawl reference

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: Background crawl X-SQL extraction fails intermittently

**Severity:** High
**Category:** Reliability

#### Reproduction

1. Run a foreground crawl with --refresh --parse --expires 1h --priority 1 --page-load-timeout 30s --sql @query.sql (succeeds). 2. Run the same crawl with --background. 3. Check results via `crawl result <id>`.

#### Expected Behavior

Background crawl should produce identical extraction results to the foreground crawl with the same flags.

#### Actual Behavior

The background crawl completed with status OK and all pages fetched, but X-SQL extraction returned empty strings for all fields on all pages.

#### Root Cause Analysis

The background crawl ran concurrently with a foreground crawl (--ignore-url-query, started 12s later, finished at the exact same timestamp). Concurrent crawls hitting the same URLs may cause a race condition in the web database or cache layer. The X-SQL DOM_LOAD_AND_SELECT re-fetch may read a partially-written or locked cache entry, returning empty content. Additionally, the first foreground crawl may have primed the cache in a way that subsequent concurrent accesses corrupt.

#### AI Suggested Improvement

- Add locking or isolation between concurrent crawl tasks accessing the same URLs
- Ensure DOM_LOAD_AND_SELECT re-fetches are atomic and not affected by concurrent writes
- Add integration tests for concurrent crawl tasks with X-SQL extraction
- Consider queuing crawls that target overlapping URL sets

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: X-SQL extraction failures are silent — no error or warning

**Severity:** High
**Category:** UX

#### Reproduction

Run any crawl with --sql that returns empty extraction (e.g., with --no-norm, --ignore-url-query, --readonly, or concurrent background crawl).

#### Expected Behavior

When X-SQL extraction returns no data, the CLI should warn the user that extraction may have failed, or at minimum distinguish between 'no matching elements found' and 'extraction error'.

#### Actual Behavior

The table output shows rows with empty cells. There is no indication whether the page had no matching content, the selectors were wrong, or an internal error occurred. The crawl reports '3 pages crawled, 3 rows extracted' which implies success but masks the empty data.

#### Root Cause Analysis

The crawl result formatting treats empty extraction as valid output rather than a potential error condition. The extraction pipeline does not distinguish between 'selector matched nothing' and 'extraction pipeline failure'. The aggregated result message ('3 rows extracted') counts rows regardless of content.

#### AI Suggested Improvement

- Add a warning when all extracted fields are empty/null across all pages (indicates systemic failure)
- Distinguish between 'no elements matched selector' and 'extraction error' in the output
- Include per-page extraction status in the crawl result (extraction_ok, extraction_empty, extraction_error)
- Surface extraction warnings in `crawl list` output so users can detect issues without fetching full results

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 5: --readonly causes partial X-SQL extraction failure

**Severity:** Medium
**Category:** Product

#### Reproduction

./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract.sql --refresh --readonly --format table

#### Expected Behavior

All 3 product pages should be extracted. --readonly should prevent destructive operations but not affect read-only content extraction.

#### Actual Behavior

Only the first page (B0E000001) had correct title and price extracted. Pages 2 and 3 (B0E000002, B0E000003) returned empty strings despite being successfully fetched.

#### Root Cause Analysis

--readonly likely prevents writing parsed content to the web database or intermediate cache. The first page may have succeeded because it was already cached from a prior non-readonly crawl. Pages 2 and 3, needing fresh parse-and-store, failed because --readonly blocked the write. The X-SQL processor then found no stored content for those pages.

#### AI Suggested Improvement

- Clarify in documentation that --readonly prevents storage of parsed content, which makes X-SQL extraction unreliable unless pages are already cached
- Consider making X-SQL extraction work directly from in-memory fetched content rather than requiring a database write
- Add a warning when --readonly is combined with --sql and --refresh

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 6: Crawl performance is very slow for localhost pages

**Severity:** Medium
**Category:** Product

#### Reproduction

Run `crawl --seed-file urls.txt --depth 0 --refresh` against 3 localhost MockSite pages.

#### Expected Behavior

3 simple localhost pages should be fetched and processed in under 10 seconds total.

#### Actual Behavior

Each crawl takes ~100-120 seconds. The 'waiting for first page' phase alone takes ~95 seconds before any progress is reported, even though all 3 URLs are queued from the start.

#### Root Cause Analysis

The crawl engine appears to have a fixed warmup/delay period before beginning page loads. The 'waiting for first page' message persists for ~95 seconds regardless of page complexity or network latency. This suggests an internal poll interval, queue processing delay, or browser context initialization that is unnecessarily slow. Additionally, --refresh with --parse may trigger full re-fetch and re-parse pipelines that have high per-page overhead even for trivial pages.

#### AI Suggested Improvement

- Investigate the ~95s delay before first page load begins — this is the primary bottleneck
- Reduce the poll interval or add a 'fast path' for localhost/simple pages
- Consider parallel page loading within a single crawl task for depth 0 bulk fetches
- Add a progress indicator showing which URL is currently being loaded

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 7: Documentation does not explain LoadOptions flags interaction with X-SQL

**Severity:** Medium
**Category:** Documentation

#### Reproduction

Read the crawl.md reference — the LoadOptions flags table lists --ignore-url-query, --no-norm, and --readonly but does not describe how they affect X-SQL extraction.

#### Expected Behavior

The documentation should note any known interactions or limitations when combining LoadOptions flags with --sql extraction.

#### Actual Behavior

The LoadOptions flags table describes what each flag does for page fetching/URL handling but never mentions X-SQL. Users learn about extraction failures only through trial and error.

#### Root Cause Analysis

The documentation treats LoadOptions flags and X-SQL extraction as orthogonal features, but they interact through the shared web database/cache layer.

#### Code Pointer

`skills/browser4-cli/references/crawl.md:116-130 (LoadOptions flags table)`

#### AI Suggested Improvement

- Add a note to each LoadOptions flag that affects X-SQL extraction about known interactions
- Add a dedicated section 'LoadOptions and X-SQL interactions' to the crawl reference
- Add warnings in the --help output when incompatible flag combinations are detected

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 8: crawl list does not show extraction status or row counts

**Severity:** Low
**Category:** UX

#### Reproduction

Run `crawl list` after running crawls with --sql.

#### Expected Behavior

The task list should show whether extraction succeeded and how many rows were extracted, so users can quickly assess results without running `crawl result` on each task.

#### Actual Behavior

The task list shows '3 URLs, depth 0, X-SQL' in the description column but does not indicate extraction outcome. The STATUS column only shows 'completed' — you must run `crawl result <id>` to discover that extraction was empty.

#### Root Cause Analysis

The crawl task metadata stored by the backend tracks crawl completion status but does not surface extraction-specific metrics (rows extracted, extraction errors) in the list view.

#### AI Suggested Improvement

- Add extraction summary to the crawl list output (e.g., '3 URLs, 3 rows extracted' or '3 URLs, X-SQL: 0 rows')
- Include a warning icon or marker for tasks where extraction returned zero rows
- Add a --verbose flag to `crawl list` that shows per-task extraction statistics

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

**Completion Status:** Partially Successful — The baseline crawl with X-SQL extraction worked correctly. However, 4 out of 6 crawl variations (background, --ignore-url-query, --no-norm, --readonly) produced silent extraction failures. The core mechanism functions under ideal conditions but is fragile when combined with LoadOptions flags or concurrent execution.

**Success Rate:** 33% — only 2 of 6 crawl invocations produced correct extraction results (the baseline foreground crawl and the --readonly crawl which got 1/3 pages). The other 4 crawls all completed with status OK but returned empty data.

**Issues Found:** 8

**Major Blockers:** The most critical blocker is the silent X-SQL extraction failure—when flags like --no-norm, --ignore-url-query, or --readonly are combined with --sql, pages are fetched but extraction returns empty with no error indication. Users would trust the empty output and make incorrect decisions. The background crawl also failed silently when run concurrently with another crawl.

**Most Confusing Aspects:** 1) The distinction between 'pages crawled' and 'rows extracted'—both report '3' in the output even when extracted data is empty. 2) Why LoadOptions flags that affect URL handling cause X-SQL content extraction to fail. 3) The ~95-second delay before any progress appears, even for localhost pages.

**Most Valuable Improvements:** 1) Fix the silent extraction failures with --no-norm, --ignore-url-query, and --readonly flags. 2) Add warnings when X-SQL extraction returns entirely empty results. 3) Improve crawl performance—3 localhost pages should not take 100 seconds. 4) Add extraction status to `crawl list` output.

**Usability Rating:** 5/10

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

#### Issue 1: --ignore-url-query causes silent X-SQL extraction failure

./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract.sql --refresh --ignore-url-query --format table

#### Issue 2: --no-norm causes silent X-SQL extraction failure

./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract.sql --no-norm --format table

#### Issue 3: Background crawl X-SQL extraction fails intermittently

1. Run a foreground crawl with --refresh --parse --expires 1h --priority 1 --page-load-timeout 30s --sql @query.sql (succeeds). 2. Run the same crawl with --background. 3. Check results via `crawl result <id>`.

#### Issue 4: X-SQL extraction failures are silent — no error or warning

Run any crawl with --sql that returns empty extraction (e.g., with --no-norm, --ignore-url-query, --readonly, or concurrent background crawl).

#### Issue 5: --readonly causes partial X-SQL extraction failure

./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract.sql --refresh --readonly --format table

#### Issue 6: Crawl performance is very slow for localhost pages

Run `crawl --seed-file urls.txt --depth 0 --refresh` against 3 localhost MockSite pages.

#### Issue 7: Documentation does not explain LoadOptions flags interaction with X-SQL

Read the crawl.md reference — the LoadOptions flags table lists --ignore-url-query, --no-norm, and --readonly but does not describe how they affect X-SQL extraction.

#### Issue 8: crawl list does not show extraction status or row counts

Run `crawl list` after running crawls with --sql.

