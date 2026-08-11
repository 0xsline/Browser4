# Issues: swarm-parallel-scraping

> **Source:** `20260810-195428-swarm-parallel-scraping.full.md` | **Date:** 20260810-195428 | **Mode:** dev

## Scenario Background

### Task

All 10 product pages were successfully extracted:

| # | Product ID | Title | Price | Image |
|---|-----------|-------|-------|-------|
| 1 | B0E000001 | 4K OLED TV 55 | $899.99 | picsum.photos/seed/1250857624 |
| 2 | B0E000002 | Wireless Noise-Cancelling Headphones | $199.99 | picsum.photos/seed/1250857625 |
| 3 | B0E000003 | Portable Bluetooth Speaker | $49.99 | picsum.photos/seed/1250857626 |
| 4 | B0E000004 | Smartphone 128GB | $599.00 | picsum.photos/seed/1250857627 |
| 5 | B0E000005 | USB-C Hub 7-in-1 | $29.95 | picsum.photos/seed/1250857628 |
| 6 | B0E000006 | Wireless Mouse | $24.99 | picsum.photos/seed/1250857629 |
| 7 | B0H000001 | Vacuum Cleaner Smart | $159.99 | picsum.photos/seed/-381598629 |
| 8 | B0H000002 | Air Purifier | $129.50 | picsum.photos/seed/-381598628 |
| 9 | B0H000003 | Essential Oil Diffuser | $22.99 | picsum.photos/seed/-381598627 |
| 10 | B0H000004 | LED Desk Lamp | $35.99 | picsum.photos/seed/-381598626 |

The full results are saved to `.test-sessions/extracted-products.json`.

**Workaround used:** The primary swarm path failed due to a `ProtoNotFound(1600)` error in the `FetchComponent`. Data was ultimately extracted via interactive browser navigation (`goto` + `eval`) which uses CDP directly.

---

### Execution Context

**Key Commands:**

1. `./b4w.ps1 help` — verified CLI is functional
2. `./b4w.ps1 swarm create --display-mode HEADLESS --max-browser-contexts 2 --max-open-tabs 4` — created swarm (×2, first session recreated after timeout)
3. `./b4w.ps1 swarm query --sql @.test-sessions/extract-products.sql --seed-file .test-sessions/seed-urls.txt --refresh` — submitted 10 X-SQL jobs
4. `./b4w.ps1 swarm submit "http://localhost:18080/ec/dp/B0E000001" --refresh` — submitted plain scrape job
5. `./b4w.ps1 swarm list` — polled task status (×6, all stuck "queued")
6. `./b4w.ps1 swarm status <id>` — polled individual task (×3)
7. `./b4w.ps1 swarm result <id>` — retrieved task results (empty due to fetch failure)
8. `./b4w.ps1 --json swarm status <id>` — used raw JSON to discover discrepancy (backend `done:true` vs CLI `isDone:false`)
9. `./b4w.ps1 swarm close` — closed session (×2)
10. `./b4w.ps1 swarm list --clear` — cleared stale tasks
11. `./b4w.ps1 crawl --seed-file .test-sessions/seed-urls.txt --depth 0 --sql @.test-sessions/extract-products.sql` — crawl workaround (partial success: 4/10 extraction)
12. `./b4w.ps1 crawl result <id>` — retrieved crawl results
13. `./b4w.ps1 goto <url>` — navigated to each product (×10)
14. `./b4w.ps1 --json eval 'JSON.stringify({...})'` — extracted via live DOM (×10)
15. `./b4w.ps1 crawl list` / `swarm list` — listed task history
16. `./b4w.ps1 swarm close` — final cleanup

**Key decisions:**
- Restarted the swarm session after 120s timeout (per doc recovery steps) — did not fix the issue
- Switched from swarm to `crawl` when swarm was non-functional — crawl partially worked
- Used `eval` (CDP-based) for complete extraction when both swarm and `htmlsnapshot query` degraded
- Discovered the CLI/backend status reporting mismatch by inspecting `--json` raw output

---

---

## Issues Found (10 issues)

### Issue 1: Swarm FetchComponent returns ProtoNotFound(1600) for localhost URLs

**Severity:** Critical
**Category:** Reliability

#### Reproduction

1. Start MockSite on localhost:18080
2. Create swarm session: ./b4w.ps1 swarm create --display-mode HEADLESS
3. Submit query: ./b4w.ps1 swarm query "http://localhost:18080/ec/dp/B0E000001" --sql @query.sql --refresh
4. Check logs: ./b4w.ps1 doctor log pulsar grep "ProtoNotFound"

#### Expected Behavior

The swarm should fetch localhost URLs successfully, extracting product data via X-SQL.

#### Actual Behavior

All swarm tasks remain stuck in queued state. Backend logs show: WARN FetchComponent - Protocol not found | http://localhost:18080/... with status code 1600. The task's raw JSON shows done:true but pageStatusCode:1600 and empty/null resultSet. No data is extracted.

#### Root Cause Analysis

The swarm's FetchComponent (a.p.p.s.w.c.FetchComponent) does not have an HTTP protocol handler registered for localhost URLs. This is different from the CDP-based browser path (goto/snapshot/eval) which works correctly. The htmlsnapshot query path also showed this error intermittently, suggesting the protocol handler can degrade over time or is initialized asynchronously. The error code 1600 maps to ProtoNotFound in Pulsar's internal status codes.

#### Code Pointer

`browser4-core/.../FetchComponent.kt — needs HTTP protocol handler registration for localhost. Also check ai.platon.pulsar.scraper.web.crawl.FetchComponent for protocol resolution logic.`

#### AI Suggested Improvement

- Register an HTTP protocol handler for localhost in the FetchComponent used by swarm tasks
- Add a startup health check that verifies the HTTP protocol handler is initialized before accepting swarm tasks
- Provide a clear, actionable error message when Protocol not found occurs (instead of silent queued-state hang)
- Document that swarm requires publicly-accessible URLs or specific protocol configuration for localhost

#### Human Review

- [x] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: CLI reports isDone:false when backend JSON has done:true

**Severity:** High
**Category:** UX

#### Reproduction

1. Submit a swarm query that hits the ProtoNotFound error
2. Run: ./b4w.ps1 swarm status <task-id>
3. Run: ./b4w.ps1 --json swarm status <task-id>
4. Compare the human-readable output (isDone:false, lifecycleState:queued) with the raw JSON (done:true, event:onLoaded)

#### Expected Behavior

The CLI's isDone should match the backend's done field. If the backend reports the task is finished (with error), the CLI should show the task as completed/failed, not queued.

#### Actual Behavior

CLI output shows isDone:false and lifecycleState:queued, while the raw backend JSON shows done:true with finishTime set. This mismatch prevents users from discovering that tasks have actually completed (with errors) and that results are available via swarm result.

#### Root Cause Analysis

The CLI likely derives isDone from statusCode (checking for 200) rather than using the backend's done field. When a task completes with a non-200 statusCode (like 201 for failed fetches), the CLI incorrectly reports it as still queued. The lifecycleState mapping also does not account for the done+error state.

#### Code Pointer

`cli/browser4-cli/src/ (swarm status command output formatting) — the isDone field mapping from backend statusCode/done fields`

#### AI Suggested Improvement

- Use the backend's done field as the primary indicator for isDone, not statusCode
- Add an error/failed lifecycleState for tasks with done:true but non-200 statusCode
- Show a warning when a task is done but statusCode indicates an error
- Consider surfacing pageStatusCode in the human-readable output when non-200

#### Human Review

- [x] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: swarm list shows all tasks as queued even after backend completion

**Severity:** High
**Category:** UX

#### Reproduction

1. Submit swarm tasks that complete (with or without errors)
2. Run: ./b4w.ps1 swarm list
3. Observe all tasks show STATUS=queued indefinitely
4. Run: ./b4w.ps1 --json swarm status <id> on any task to see actual state

#### Expected Behavior

swarm list should show accurate, live task statuses reflecting the backend state (completed, failed, processing).

#### Actual Behavior

swarm list shows all tasks as queued even 2+ minutes after submission, well past the 120s timeout. No task ever transitions to processing or completed in the list view, even though the backend has finished processing them (done:true in raw JSON).

#### Root Cause Analysis

Same root cause as Issue 2 — the CLI's status determination logic for the list view relies on a statusCode-based mapping that doesn't handle the error-completion state. Additionally, the list may cache status rather than polling the backend on each invocation as documented.

#### Code Pointer

`cli/browser4-cli/src/ (swarm list command — task status resolution logic)`

#### AI Suggested Improvement

- Fix the status mapping to use the backend's done field and lifecycleState
- Ensure swarm list polls the backend for each tracked task's current status on every invocation
- Add a status column for error conditions like pageStatusCode
- Document the expected status progression and timeout behaviors

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: htmlsnapshot query fetch reliability degrades over time

**Severity:** High
**Category:** Reliability

#### Reproduction

1. Run htmlsnapshot query for a localhost URL — it may succeed initially
2. Run several more htmlsnapshot query commands in quick succession
3. Observe that eventually all queries return pageStatusCode:1600 even for URLs that previously worked

#### Expected Behavior

htmlsnapshot query should consistently return pageStatusCode:200 for valid, accessible URLs.

#### Actual Behavior

Initial queries succeeded (pageStatusCode:200 with valid resultSet). After ~10 sequential queries, all subsequent queries returned pageStatusCode:1600 with null resultSet, even for the same URLs that previously worked. This degradation persisted for all subsequent queries in the session.

#### Root Cause Analysis

The fetch infrastructure (DOM_LOAD_AND_SELECT's underlying HTTP client) appears to have a resource leak or state corruption that causes protocol handler failures after sustained use. The first few queries warm up or initialize the handler, but contention or resource exhaustion eventually causes all queries to fail.

#### Code Pointer

`browser4-core/ (DOM_LOAD_AND_SELECT implementation — the HTTP fetch layer used by X-SQL queries)`

#### AI Suggested Improvement

- Investigate the HTTP client lifecycle in the X-SQL fetch path for resource leaks
- Add connection pooling limits and timeout handling
- Implement automatic retry with backoff when pageStatusCode:1600 is encountered
- Add health monitoring for the fetch protocol handler

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 5: Crawl X-SQL extraction inconsistency between product categories

**Severity:** Medium
**Category:** Product

#### Reproduction

1. Create seed file with Electronics (B0E*) and Home (B0H*) product URLs
2. Run: ./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql
3. Compare extraction results by category

#### Expected Behavior

All product pages with identical HTML structure should produce extraction results when fetched.

#### Actual Behavior

Crawl extracted data for all 4 Home products correctly but returned empty fields for all 6 Electronics products, even though: (a) contentLength confirms pages were fetched, (b) HTML structure uses identical CSS selectors, and (c) interactive browser extraction works for all pages.

#### Root Cause Analysis

Unclear — likely related to the ProtoNotFound issue. The crawl may use different fetch timing or caching for different URL patterns. Electronics pages are ~15KB while Home pages are ~4-16KB. The contentLength values suggest the pages were fetched, but the X-SQL engine may have received a signal (pageStatusCode?) that caused it to skip extraction for the larger Electronics pages. Further investigation of the crawl worker logs during extraction is needed.

#### AI Suggested Improvement

- Log the pageStatusCode for each crawled page to aid debugging
- Add a diagnostic mode to crawl that reports per-page extraction status and errors
- Consider retrying individual pages that return empty extraction despite successful fetch
- Verify that the X-SQL engine receives consistent pageStatusCode across all fetched pages

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 6: Shell argument passing issues with ./b4w.ps1 in bash scripts

**Severity:** Medium
**Category:** UX

#### Reproduction

In a bash script: ./b4w.ps1 goto "$url" -q
Observe the URL in the browser shows %20-q appended

#### Expected Behavior

The -q flag should be treated as a CLI option (quiet mode), not as part of the URL argument.

#### Actual Behavior

The -q flag was concatenated to the URL, resulting in navigation to http://localhost:18080/ec/dp/B0E000001%20-q (404 error). This happens because the PowerShell wrapper (./b4w.ps1) does not reliably separate flags from positional arguments when invoked from bash.

#### Root Cause Analysis

The ./b4w.ps1 PowerShell script's argument parsing does not properly handle the boundary between positional arguments and flags when invoked from bash. The bash→PowerShell argument passing introduces quoting/escaping issues that can cause flags to be treated as part of positional arguments.

#### Code Pointer

`cli/browser4-cli/b4w.ps1 — argument parsing/forwarding logic`

#### AI Suggested Improvement

- Document that ./b4w.sh should be used for bash scripting (the SKILL.md already notes this but the QUICK START doesn't emphasize it enough)
- Add argument boundary detection in the PowerShell wrapper (-- style separator)
- Consider adding a warning when flags appear to be appended to positional arguments
- The SKILL.md note about b4w.sh needs more prominence in the Quick Start section

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 7: No --wait progress feedback when swarm tasks are stuck

**Severity:** Medium
**Category:** UX

#### Reproduction

Run: ./b4w.ps1 swarm query ... --wait when tasks are stuck in queued state
Observe the command hangs indefinitely with no output.

#### Expected Behavior

--wait should show polling progress and eventually timeout with a clear error message.

#### Actual Behavior

The command with --wait hung for 120+ seconds with no output (required manual TaskStop). No progress indication, no timeout message, no error about tasks being stuck. The user gets no feedback about what's happening.

#### Root Cause Analysis

The --wait implementation polls for isDone on each task, but the task status API reports isDone:false (due to Issue 2) even when the task has actually finished (with error). This creates an infinite wait loop. The 5-minute timeout is too long without intermediate progress updates.

#### Code Pointer

`cli/browser4-cli/src/ (swarm query --wait implementation)`

#### AI Suggested Improvement

- Show periodic progress updates during --wait (e.g., still waiting for 3/10 tasks after 30s)
- Detect stuck tasks (statusCode unchanged for >30s) and warn the user
- Add a --wait-timeout option to customize the wait duration
- Surface task-level errors during --wait instead of silently polling

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 8: Swarm documentation does not mention localhost/private network limitations

**Severity:** Medium
**Category:** Documentation

#### Reproduction

Read skills/browser4-cli/references/swarm.md — no mention of URL requirements or known limitations with localhost.

#### Expected Behavior

Documentation should mention any URL requirements, known limitations with localhost/private network URLs, and how to configure the fetch infrastructure for local development.

#### Actual Behavior

The swarm.md reference makes no mention of URL accessibility requirements. It implies any HTTP URL should work, but localhost URLs consistently fail with the ProtoNotFound error. Users testing locally would hit this with no guidance.

#### Root Cause Analysis

The documentation was written assuming publicly-accessible URLs. The localhost fetch limitation is an implementation detail that is not surfaced in user-facing docs.

#### Code Pointer

`skills/browser4-cli/references/swarm.md`

#### AI Suggested Improvement

- Add a 'Known Limitations' section documenting the ProtoNotFound issue with localhost URLs
- Document that local testing should use the interactive browser (goto/eval) or crawl, not swarm
- If there's a configuration option to enable localhost fetching, document it
- Add a troubleshooting entry for ProtoNotFound(1600) errors

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 9: swarm close output includes unrelated page snapshot information

**Severity:** Low
**Category:** UX

#### Reproduction

Run: ./b4w.ps1 swarm close
Observe the output includes a Page/Snapshot section for the current browser page.

#### Expected Behavior

swarm close should output only session closure status information.

#### Actual Behavior

Output includes: ### Page, Page URL, Page Title, ### Snapshot, and a Tip about htmlsnapshot commands. This is confusing because swarm close is about releasing swarm resources, not about page interaction.

#### Root Cause Analysis

The swarm close command closes the swarm session which triggers a browser session state change. The interactive browser session (DEFAULT) may still be open, and the CLI auto-outputs page/snapshot info after any state-modifying command on the active session.

#### Code Pointer

`cli/browser4-cli/src/ (session state output hook — should not fire for swarm close on a different session)`

#### AI Suggested Improvement

- Suppress page/snapshot output when the closed session is SWARM and the active session is DEFAULT
- Only show post-command page info for commands that directly modify the active browsing session
- Move swarm lifecycle messages to a dedicated output section

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 10: Crawl and swarm task history accumulates without automatic cleanup

**Severity:** Low
**Category:** UX

#### Reproduction

Run: ./b4w.ps1 crawl list — observe 22 completed tasks from prior sessions
Run: ./b4w.ps1 swarm create — observe stale task warning

#### Expected Behavior

Completed/failed tasks should be auto-cleaned after a configurable TTL, or the system should not warn about stale tasks that it itself accumulated.

#### Actual Behavior

22 crawl tasks and 2 swarm tasks accumulated across sessions. swarm create warns about stale tasks and requires manual clearing. The task store grows unbounded without user intervention.

#### Root Cause Analysis

Tasks are persisted for history/reference but no automatic TTL-based cleanup is applied. The stale task warning on swarm create is a workaround rather than a fix for the accumulation problem.

#### Code Pointer

`browser4-rest/ (swarm and crawl task store — TTL/cleanup logic)`

#### AI Suggested Improvement

- Implement automatic TTL-based cleanup for completed/failed tasks (e.g., 1 hour default, configurable)
- Add a config option for task retention period
- The --clear-stale flag is good but should be complemented by automatic cleanup
- Show task count in status/doctor output so users are aware of accumulation

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

**Completion Status:** Partially Successful — All 10 product pages were extracted using cd(eval) as a workaround, but both primary paths (swarm query, htmlsnapshot query) failed due to a ProtoNotFound(1600) error in the fetch infrastructure. The crawl partially worked (4/10 extraction). The core task objective was met, but required significant workarounds.

**Success Rate:** 60% — Swarm: 0% (all tasks stuck queued, no extraction). Crawl: 40% (4/10 products extracted). Interactive browser/eval: 100% (all 10 products extracted). Overall, the task was completable only by abandoning the documented swarm workflow.

**Issues Found:** 10

**Major Blockers:** The swarm FetchComponent returning ProtoNotFound(1600) for all localhost URLs is a critical blocker that prevents the primary documented workflow from functioning. This affected swarm query, swarm submit, and (intermittently) htmlsnapshot query. The CLI status reporting mismatch (isDone:false when backend says done:true) compounded the issue by hiding the true task state from users.

**Most Confusing Aspects:** 1. swarm list showing all tasks as 'queued' indefinitely with no indication of the underlying ProtoNotFound error. 2. The disconnect between CLI-reported status (isDone:false) and raw JSON (done:true) required using --json to discover the actual task state. 3. The htmlsnapshot query degradation — queries that worked moments earlier suddenly returning 1600 errors with no explanation. 4. Having to discover through experimentation that goto+eval works when swarm+query doesn't.

**Most Valuable Improvements:** 1. Fix the FetchComponent to support localhost HTTP URLs or document this as a known limitation with clear workaround guidance. 2. Fix the CLI status reporting to use the backend's done field instead of deriving isDone from statusCode. 3. Add health checks and better error messages for protocol handler failures. 4. Improve --wait with progress feedback and stuck-task detection.

**Usability Rating:** 4/10

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

#### Issue 1: Swarm FetchComponent returns ProtoNotFound(1600) for localhost URLs

1. Start MockSite on localhost:18080
2. Create swarm session: ./b4w.ps1 swarm create --display-mode HEADLESS
3. Submit query: ./b4w.ps1 swarm query "http://localhost:18080/ec/dp/B0E000001" --sql @query.sql --refresh
4. Check logs: ./b4w.ps1 doctor log pulsar grep "ProtoNotFound"

#### Issue 2: CLI reports isDone:false when backend JSON has done:true

1. Submit a swarm query that hits the ProtoNotFound error
2. Run: ./b4w.ps1 swarm status <task-id>
3. Run: ./b4w.ps1 --json swarm status <task-id>
4. Compare the human-readable output (isDone:false, lifecycleState:queued) with the raw JSON (done:true, event:onLoaded)

#### Issue 3: swarm list shows all tasks as queued even after backend completion

1. Submit swarm tasks that complete (with or without errors)
2. Run: ./b4w.ps1 swarm list
3. Observe all tasks show STATUS=queued indefinitely
4. Run: ./b4w.ps1 --json swarm status <id> on any task to see actual state

#### Issue 4: htmlsnapshot query fetch reliability degrades over time

1. Run htmlsnapshot query for a localhost URL — it may succeed initially
2. Run several more htmlsnapshot query commands in quick succession
3. Observe that eventually all queries return pageStatusCode:1600 even for URLs that previously worked

#### Issue 5: Crawl X-SQL extraction inconsistency between product categories

1. Create seed file with Electronics (B0E*) and Home (B0H*) product URLs
2. Run: ./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql
3. Compare extraction results by category

#### Issue 6: Shell argument passing issues with ./b4w.ps1 in bash scripts

In a bash script: ./b4w.ps1 goto "$url" -q
Observe the URL in the browser shows %20-q appended

#### Issue 7: No --wait progress feedback when swarm tasks are stuck

Run: ./b4w.ps1 swarm query ... --wait when tasks are stuck in queued state
Observe the command hangs indefinitely with no output.

#### Issue 8: Swarm documentation does not mention localhost/private network limitations

Read skills/browser4-cli/references/swarm.md — no mention of URL requirements or known limitations with localhost.

#### Issue 9: swarm close output includes unrelated page snapshot information

Run: ./b4w.ps1 swarm close
Observe the output includes a Page/Snapshot section for the current browser page.

#### Issue 10: Crawl and swarm task history accumulates without automatic cleanup

Run: ./b4w.ps1 crawl list — observe 22 completed tasks from prior sessions
Run: ./b4w.ps1 swarm create — observe stale task warning

