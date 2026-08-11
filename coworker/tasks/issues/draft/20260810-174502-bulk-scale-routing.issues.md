# Issues: bulk-scale-routing

> **Source:** `20260810-174502-bulk-scale-routing.full.md` | **Date:** 20260810-174502 | **Mode:** dev

## Scenario Background

### Task

The task was **partially successful**. Five of six acceptance criteria were completed successfully, while one (AC3 — crawl link discovery) revealed a fundamental backend bug that prevented full completion.

### Successful:
- **AC1** — `htmlsnapshot query` with `DOM_LOAD_AND_SELECT` extracted 6 product rows (title + price + link) from the MockSite listing page
- **AC2** — `crawl --seed-file --depth 0 --sql` extracted 3 product detail rows with URL, title, and price
- **AC4** — `swarm create` + `swarm query --seed-file` extracted 3 product rows in parallel (after backend restart)
- **AC5** — `loop -- eval` monitored a product price across 3 iterations at 5-second intervals
- **AC6** — Bash `for` loop with `goto`, `htmlsnapshot`, and `htmlsnapshot get text` extracted titles and prices for 3 URLs

### Failed/Workaround:
- **AC3** — `crawl` with link discovery (`--depth >= 1`) is broken: the crawl engine's CSS selector matching returns zero elements for all selectors tested (`a.product`, `a[href]`, `a[href*='product/']`, `a`). A workaround using manual link extraction via `htmlsnapshot get all attr` + `crawl --seed-file --depth 0` partially succeeded (1 of 3 rows extracted with correct data).

---

### Execution Context

**Key Commands:**

1. `./b4w.ps1 help` — learned available commands
2. Read `skills/browser4-cli/SKILL.md` and all reference docs
3. `./b4w.ps1 goto "http://localhost:18080/ec/b?node=1292115012"` — AC1 navigation
4. `./b4w.ps1 htmlsnapshot` — capture listings page
5. `./b4w.ps1 htmlsnapshot query "..." --sql @ac1-query.sql` — AC1 X-SQL extraction
6. `./b4w.ps1 crawl --seed-file ... --depth 0 --sql @... --format table --refresh` — AC2 bulk fetch
7. `./b4w.ps1 crawl <url> -d 1 -ol "a.product" -olp "product" --refresh` — AC3 link discovery (repeated with multiple selectors and configurations, all failed)
8. `./b4w.ps1 kill-all` — restart backend to clear stuck state
9. `./b4w.ps1 swarm create --display-mode HEADLESS --max-browser-contexts 2 --max-open-tabs 4 --clear-stale` — AC4 session
10. `./b4w.ps1 swarm query --sql @... --seed-file ... --refresh --wait` — AC4 extraction
11. `./b4w.ps1 swarm result <id>` — AC4 result fetch
12. `./b4w.ps1 -s price-watch goto "..."` — AC5 named session
13. `./b4w.ps1 loop --name mock-price-watch-3 --count 3 -i 5 -- eval "..."` — AC5 monitoring
14. Bash `for` loop with `goto`, `htmlsnapshot`, `htmlsnapshot get text` — AC6

**Workarounds Applied During Task:**

- Backend restart (kill-all) for AC4 to work
- Default session for loop (AC5) due to -s flag not passing through
- Manual link extraction for AC3 crawl discovery
- Explicit htmlsnapshot recapture before each `get text` in AC6

---

```json
{
  "issues": [
    {
      "title": "Crawl link discovery (depth >= 1) CSS selector matching is fundamentally broken",
      "severity": "Critical",
      "category": "Product",
      "reproduction": "./b4w.ps1 crawl http://localhost:18080/generated/crawl/index.html -d 1 -ol \"a\" --refresh\n\nAny CSS selector passed via --out-link-selector (including the bare 'a' tag selector) matches zero elements during crawl link discovery, even though the page has 12 anchors and 8044B of HTML. The diagnostic confirms: 'The page has 12 anchors and 8044B of HTML, but the CSS selector 'a' matched zero elements.' Meanwhile, goto + htmlsnapshot on the same page correctly matches elements with the same selectors.",
      "expected": "CSS selectors in crawl link discovery should match elements the same way htmlsnapshot does. 'a' should match all anchor elements.",
      "actual": "All CSS selectors (a.product, a[href], a[href*='product/'], a) match zero elements in the crawl engine, producing 0 pages found. The diagnostic message suggests the page is being fetched but DOM/CSS parsing is failing.",
      "rootCause": "The crawl engine uses a different DOM loading/parsing path than the interactive browser session (goto + htmlsnapshot). The Jsoup-based or headless parsing appears to not apply CSS classes or attributes correctly in certain page contexts. Logs show 'Protocol not found' errors (ProtoNotFound(1600)) on many fetch operations, suggesting the fetch component also has protocol registration issues. The link extraction code in the crawl pipeline likely receives a parsed DOM that differs from what the interactive browser session renders.",
      "codePointer": "browser4-rest/.../CrawlService.kt — link extraction pipeline; browser4-core/.../FetchComponent.kt or equivalent — protocol registration",
      "suggestion": "- Investigate why crawl engine DOM parsing differs from interactive browser DOM (likely Jsoup vs CDP rendering difference)\n- The crawl engine should use the same CDP-based rendering path as goto + htmlsnapshot for accurate DOM access\n- Add integration tests that verify crawl link discovery matches goto + htmlsnapshot results\n- Fix the ProtoNotFound(1600) protocol registration errors that appear across all fetch operations\n- Add a diagnostic mode that dumps the parsed DOM to help debug selector mismatches"
    },
    {
      "title": "Backend degrades into stuck state requiring restart (ProtoNotFound errors)",
      "severity": "Critical",
      "category": "Reliability",
      "reproduction": "1. Run several crawl and swarm operations over 15-30 minutes\n2. Observe that subsequent operations hang indefinitely\n3. Check doctor logs: 'ProtoNotFound(1600)' errors on all fetch operations\n4. Crawl depth 0 tasks complete but X-SQL extraction returns empty strings\n5. Fix requires kill-all + restart",
      "expected": "Backend should remain stable across multiple operations without protocol registration failures or degredation.",
      "actual": "After 2-3 crawl operations, the backend enters a state where all fetch operations fail with ProtoNotFound(1600). Crawl tasks say 'waiting for first page' indefinitely. Swarm tasks stay in 'queued' state forever. Crawl results show pages were fetched (contentLength > 0) but X-SQL extraction returns empty strings for all fields. Only kill-all + restart recovers the system.",
      "rootCause": "The FetchComponent's protocol registry appears to get corrupted or lose entries after a few operations. The error 'ProtoNotFound(1600)' suggests an internal protocol ID lookup failure. This could be a resource leak (e.g., protocol handlers not being re-registered after use), a race condition in protocol initialization, or a state corruption in the fetch pipeline. The empty X-SQL extraction results may be caused by the same root issue — pages are cached/downloaded but not properly parsed.",
      "codePointer": "browser4-core/.../FetchComponent.kt — protocol lookup; browser4-core/.../ProtocolRegistry or equivalent",
      "suggestion": "- Add protocol registration health checks that detect and auto-recover from ProtoNotFound state\n- Investigate resource leaks in the protocol/fetch pipeline that cause degredation over time\n- Add a circuit breaker that detects repeated ProtoNotFound errors and triggers automatic recovery\n- Improve error messages: 'ProtoNotFound(1600)' is opaque and doesn't help users diagnose the issue\n- Consider adding a 'doctor --fix' repair that resets the protocol registry without full restart"
    },
    {
      "title": "Crawl X-SQL extraction returns empty strings despite pages being fetched",
      "severity": "High",
      "category": "Reliability",
      "reproduction": "Run crawl --seed-file with --depth 0 --sql @query.sql after the backend has been running for a while. First run succeeds with data. Second run (after other operations) returns pages with content but all extracted fields are empty strings.\n\nExample: 3 pages crawled, 3 rows extracted but title='' and price=''.",
      "expected": "X-SQL extraction should consistently return actual text content when pages are successfully fetched and elements exist.",
      "actual": "After backend degredation, X-SQL extraction returns empty strings for all fields even though pages are fetched (contentLength > 0). The first run against the same URLs with the same query works correctly. This is correlated with the ProtoNotFound state but may be a separate issue.",
      "rootCause": "Likely related to the same protocol/fetch degredation issue. When the fetch pipeline is in a bad state, pages may be loaded from cache or partially loaded without proper DOM construction, causing X-SQL functions to return empty strings. Alternatively, the DOM parser (Jsoup) may not be processing the HTML correctly in certain backend states.",
      "codePointer": "browser4-rest/.../CrawlService.kt — X-SQL result aggregation; browser4-core/.../DomFunctions or X-SQL execution engine",
      "suggestion": "- Verify X-SQL extraction works on a freshly parsed DOM, not on stale/cached page state\n- Add validation: if a page has contentLength > 0 but all X-SQL fields are empty, log a warning\n- Consider re-fetching pages when X-SQL extraction returns all-empty results\n- Add a health check for the DOM parsing pipeline similar to the protocol check"
    },
    {
      "title": "Loop subcommand mode does not pass -s (session) flag to nested browser4-cli process",
      "severity": "High",
      "category": "Product",
      "reproduction": "./b4w.ps1 loop --name test --count 2 -i 10 -- -s price-watch eval \"document.querySelector('#product-price').textContent.trim()\"\n\nOutput: [ERROR] Iteration 1: Error: Unknown command: 'price-watch'",
      "expected": "The -s price-watch flag should be passed through to the nested browser4-cli process, allowing the loop to target a named session.",
      "actual": "The -s flag is lost or mis-parsed, and 'price-watch' is treated as a command name rather than the value of -s. The loop runs successfully on the default session instead.",
      "rootCause": "The argument parsing after -- appears to consume -s as a loop-level flag or mishandles the argument parsing when passing to the nested process. The loop documentation warns about 'known loop flags' after -- but doesn't specifically address -s handling. The nested process may receive the arguments in a different order or with different tokenization than expected.",
      "codePointer": "cli/browser4-cli/src/commands/loop.rs — argument parsing for subcommand mode",
      "suggestion": "- Fix argument passthrough to ensure -s <name> is correctly forwarded to the nested browser4-cli process\n- Add a dedicated test: loop with -s targeting a named session should run eval on that session\n- Document the -s limitation or provide a workaround (e.g., --session flag for loop itself)\n- Consider adding loop-level --session flag that sets the session for all subcommand iterations"
    },
    {
      "title": "Crawl operations are extremely slow (2+ minutes for 3 simple localhost pages)",
      "severity": "High",
      "category": "UX",
      "reproduction": "./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql --format table --refresh\n\n3 URLs on localhost:18080 take 96-110 seconds to complete.",
      "expected": "3 simple static HTML pages on localhost should complete in under 10 seconds.",
      "actual": "Crawl operations show 'waiting for first page' for 86-96 seconds before processing begins, even for localhost pages. Total time for 3 URLs is ~100 seconds. The CLI polls every 10 seconds but the backend takes much longer to begin processing.",
      "rootCause": "The crawl pipeline has significant startup latency — likely JVM warmup, browser context initialization, and page loading infrastructure setup. For localhost pages that should load instantly, the overhead dominates. The 'waiting for first page' phase suggests the backend's task queue or worker pool has high latency before processing begins. The task TTL of 60 minutes is excessive for simple crawls.",
      "codePointer": "browser4-rest/.../CrawlService.kt — task submission and worker pool initialization",
      "suggestion": "- Reduce worker pool startup latency for simple/local crawls\n- Add a fast path for localhost/same-machine URLs that skips unnecessary initialization\n- Show more granular progress during the 'waiting for first page' phase so users understand what's happening\n- Consider a 'warm' mode that pre-initializes browser contexts for faster first-page load\n- Reduce the polling interval from 10s to 2s or make it configurable"
    },
    {
      "title": "Swarm session state conflicts between restarts (stale tasks + SWARM session warning)",
      "severity": "Medium",
      "category": "Reliability",
      "reproduction": "1. Create swarm session, submit tasks, then kill-all the backend\n2. Restart backend and create new swarm session\n3. Old swarm tasks from prior session may still be tracked\n4. Backend logs: 'SWARM session does not exist, falling back to default'",
      "expected": "After kill-all + restart, all swarm state should be clean. New swarm sessions should work without warnings or conflicts.",
      "actual": "Stale swarm tasks from prior sessions persist in the task store. The swarm create command needs --clear-stale to work around this. Backend logs show warnings about SWARM session not existing when tasks from prior sessions try to run. In the first attempt, swarm tasks stayed in 'queued' state indefinitely.",
      "rootCause": "The SWARM session uses a fixed session ID ('SWARM'). After kill-all, the backend creates a new session but old task records still reference the previous SWARM session. The task store (in-memory or persistent) doesn't fully clear on restart. The --clear-stale flag exists as a workaround but the UX is poor (users must know about it).",
      "codePointer": "browser4-rest/.../SwarmController.kt — session lifecycle; browser4-rest/.../SwarmService or task store",
      "suggestion": "- Auto-clear stale swarm tasks on backend startup\n- Use unique session IDs instead of fixed 'SWARM' to avoid conflicts\n- Improve error message: instead of 'SWARM session does not exist, falling back to default', fail with a clear message about stale state\n- Make --clear-stale the default behavior or auto-detect stale state"
    },
    {
      "title": "htmlsnapshot requires explicit recapture after each goto — discoverability gap",
      "severity": "Medium",
      "category": "Discoverability",
      "reproduction": "for url in url1 url2; do\n  ./b4w.ps1 goto \"$url\"\n  ./b4w.ps1 htmlsnapshot get text \"#title\"  # fails with 'No elements matched'\ndone",
      "expected": "After goto navigates to a new page, htmlsnapshot get should either auto-capture or give a clear error telling the user to run htmlsnapshot first.",
      "actual": "htmlsnapshot get returns 'No elements matched' with a note about stale snapshots and JavaScript-modified pages. The message is helpful once you read it, but the auto-snapshot from goto only captures the AXTree (for refs), not the HTML snapshot (for CSS extraction). This creates a 'two-step dance' that is easy to miss.",
      "rootCause": "goto triggers an AXTree snapshot (for interaction refs) but NOT an HTML snapshot (for content extraction). The user must explicitly run htmlsnapshot between goto and htmlsnapshot get. The distinction between AXTree snapshot and HTML snapshot is documented in SKILL.md §4a but easy to overlook in practice.",
      "codePointer": "cli/browser4-cli/src/commands/goto.rs — could optionally auto-capture htmlsnapshot",
      "suggestion": "- Consider adding an auto-capture option: goto --htmlsnapshot to capture both AXTree and HTML in one step\n- Improve the 'No elements matched' error to explicitly say 'Run htmlsnapshot first to capture the new page'\n- Add a tip after goto: '💡 Tip: Run htmlsnapshot to capture page content for CSS extraction'\n- Document the goto → htmlsnapshot → htmlsnapshot get workflow more prominently in quick-start examples"
    },
    {
      "title": "Confusing diagnostic: crawl says CSS selector matched zero elements when htmlsnapshot shows matches",
      "severity": "Medium",
      "category": "UX",
      "reproduction": "1. ./b4w.ps1 goto http://localhost:18080/generated/crawl/index.html\n2. ./b4w.ps1 htmlsnapshot get all attr \"a.product\" href  → returns [\"product/1.html\", \"product/2.html\", \"product/3.html\"]\n3. ./b4w.ps1 crawl <same-url> -d 1 -ol \"a.product\" → 'CSS selector a.product matched zero elements'",
      "expected": "The same CSS selector should produce consistent results whether used in htmlsnapshot or crawl link discovery. When they differ, the diagnostic should explain why and suggest a workaround.",
      "actual": "htmlsnapshot matches 3 elements with a.product while crawl matches 0. The diagnostic message says to 'try a broader selector' but even 'a' matches 0. This creates confusion because the selector clearly works in one context but not another.",
      "rootCause": "The crawl engine and interactive browser session use different DOM representations or parsing paths. The diagnostic assumes the selector is wrong rather than the DOM being different. This is misleading because the real issue is the DOM parsing, not the selector.",
      "codePointer": "browser4-rest/.../CrawlService.kt — diagnostic generation",
      "suggestion": "- When CSS selectors match zero elements but the page has many matching tags (e.g., 12 anchors), suggest the DOM parsing may differ rather than blaming the selector\n- Add a diagnostic that dumps the first few elements found to help debug DOM differences\n- Consider unifying the DOM path between crawl and interactive sessions\n- The diagnostic should mention: 'Try htmlsnapshot inspect on the same page to verify selectors work in the interactive session'"
    },
    {
      "title": "No progress feedback during crawl 'waiting for first page' phase",
      "severity": "Low",
      "category": "UX",
      "reproduction": "Run any crawl command. The output shows 'Crawling... waiting for first page (Xs elapsed, N URLs queued)' with no indication of what's happening for up to 90+ seconds.",
      "expected": "Progress should show what the backend is doing during the long wait: initializing browser context, loading page, parsing DOM, etc.",
      "actual": "The CLI shows a monotonically increasing timer with no insight into backend activity. Users have no way to know if the crawl is working or stuck.",
      "rootCause": "The crawl backend doesn't expose fine-grained progress events during the initialization and page-loading phases. The CLI only knows about 'pages found' which requires a page to be fully processed.",
      "codePointer": "browser4-rest/.../CrawlService.kt — progress reporting; cli/browser4-cli/src/commands/crawl.rs — status display",
      "suggestion": "- Add backend progress stages: initializing, loading_seed, fetching_page_N, parsing, extracting\n- Expose these stages in crawl status output\n- If 'waiting for first page' exceeds 30s, show a hint about what might be happening\n- Consider a --verbose flag for more detailed progress"
    }
  ],
  "assessment": {
    "completionStatus": "Partially Successful — 5 of 6 acceptance criteria completed. AC3 (crawl link discovery) failed due to a fundamental backend bug where CSS selectors match zero elements in the crawl engine. A workaround using manual link discovery + depth-0 crawl partially succeeded. AC4 required a backend restart to recover from a stuck state (ProtoNotFound errors). All other ACs completed successfully.",
    "successRate": "83% — 5 of 6 ACs completed, but with significant reliability issues requiring backend restarts and workarounds",
    "issuesFound": 9,
    "majorBlockers": "Crawl link discovery (depth >= 1) CSS selectors match zero elements — this is a product bug that prevents basic crawl functionality. Backend ProtoNotFound degredation requires kill-all + restart between operations.",
    "mostConfusingAspects": "1. The crawl link discovery diagnostic says 'CSS selector matched zero elements' when the same selector works perfectly in htmlsnapshot — the root cause (different DOM parsing) is not explained. 2. The distinction between snapshot (AXTree for interaction) and htmlsnapshot (HTML for extraction) is documented but easy to miss — several commands fail silently when htmlsnapshot wasn't captured first. 3. The loop subcommand's -s flag handling is unintuitive — it looks like correct syntax but silently fails. 4. Crawl operations are extremely slow even for localhost pages, creating uncertainty about whether the command is working or stuck.",
    "mostValuableImprovements": "1. Fix crawl link discovery CSS selector matching (critical product bug). 2. Fix backend ProtoNotFound degredation (critical reliability). 3. Add backend health auto-recovery to avoid manual kill-all. 4. Fix loop subcommand -s flag passthrough. 5. Reduce crawl startup latency for simple/localhost use cases. 6. Unify CSS selector behavior between crawl engine and interactive browser session.",
    "usabilityRating": 4
  }
}
```

---

## Issues Found (9 issues)

### Issue 1: Crawl link discovery (depth >= 1) CSS selector matching is fundamentally broken

**Severity:** Critical
**Category:** Product

#### Reproduction

./b4w.ps1 crawl http://localhost:18080/generated/crawl/index.html -d 1 -ol "a" --refresh

Any CSS selector passed via --out-link-selector (including the bare 'a' tag selector) matches zero elements during crawl link discovery, even though the page has 12 anchors and 8044B of HTML. The diagnostic confirms: 'The page has 12 anchors and 8044B of HTML, but the CSS selector 'a' matched zero elements.' Meanwhile, goto + htmlsnapshot on the same page correctly matches elements with the same selectors.

#### Expected Behavior

CSS selectors in crawl link discovery should match elements the same way htmlsnapshot does. 'a' should match all anchor elements.

#### Actual Behavior

All CSS selectors (a.product, a[href], a[href*='product/'], a) match zero elements in the crawl engine, producing 0 pages found. The diagnostic message suggests the page is being fetched but DOM/CSS parsing is failing.

#### Root Cause Analysis

The crawl engine uses a different DOM loading/parsing path than the interactive browser session (goto + htmlsnapshot). The Jsoup-based or headless parsing appears to not apply CSS classes or attributes correctly in certain page contexts. Logs show 'Protocol not found' errors (ProtoNotFound(1600)) on many fetch operations, suggesting the fetch component also has protocol registration issues. The link extraction code in the crawl pipeline likely receives a parsed DOM that differs from what the interactive browser session renders.

#### Code Pointer

`browser4-rest/.../CrawlService.kt — link extraction pipeline; browser4-core/.../FetchComponent.kt or equivalent — protocol registration`

#### AI Suggested Improvement

- Investigate why crawl engine DOM parsing differs from interactive browser DOM (likely Jsoup vs CDP rendering difference)
- The crawl engine should use the same CDP-based rendering path as goto + htmlsnapshot for accurate DOM access
- Add integration tests that verify crawl link discovery matches goto + htmlsnapshot results
- Fix the ProtoNotFound(1600) protocol registration errors that appear across all fetch operations
- Add a diagnostic mode that dumps the parsed DOM to help debug selector mismatches

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: Backend degrades into stuck state requiring restart (ProtoNotFound errors)

**Severity:** Critical
**Category:** Reliability

#### Reproduction

1. Run several crawl and swarm operations over 15-30 minutes
2. Observe that subsequent operations hang indefinitely
3. Check doctor logs: 'ProtoNotFound(1600)' errors on all fetch operations
4. Crawl depth 0 tasks complete but X-SQL extraction returns empty strings
5. Fix requires kill-all + restart

#### Expected Behavior

Backend should remain stable across multiple operations without protocol registration failures or degredation.

#### Actual Behavior

After 2-3 crawl operations, the backend enters a state where all fetch operations fail with ProtoNotFound(1600). Crawl tasks say 'waiting for first page' indefinitely. Swarm tasks stay in 'queued' state forever. Crawl results show pages were fetched (contentLength > 0) but X-SQL extraction returns empty strings for all fields. Only kill-all + restart recovers the system.

#### Root Cause Analysis

The FetchComponent's protocol registry appears to get corrupted or lose entries after a few operations. The error 'ProtoNotFound(1600)' suggests an internal protocol ID lookup failure. This could be a resource leak (e.g., protocol handlers not being re-registered after use), a race condition in protocol initialization, or a state corruption in the fetch pipeline. The empty X-SQL extraction results may be caused by the same root issue — pages are cached/downloaded but not properly parsed.

#### Code Pointer

`browser4-core/.../FetchComponent.kt — protocol lookup; browser4-core/.../ProtocolRegistry or equivalent`

#### AI Suggested Improvement

- Add protocol registration health checks that detect and auto-recover from ProtoNotFound state
- Investigate resource leaks in the protocol/fetch pipeline that cause degredation over time
- Add a circuit breaker that detects repeated ProtoNotFound errors and triggers automatic recovery
- Improve error messages: 'ProtoNotFound(1600)' is opaque and doesn't help users diagnose the issue
- Consider adding a 'doctor --fix' repair that resets the protocol registry without full restart

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: Crawl X-SQL extraction returns empty strings despite pages being fetched

**Severity:** High
**Category:** Reliability

#### Reproduction

Run crawl --seed-file with --depth 0 --sql @query.sql after the backend has been running for a while. First run succeeds with data. Second run (after other operations) returns pages with content but all extracted fields are empty strings.

Example: 3 pages crawled, 3 rows extracted but title='' and price=''.

#### Expected Behavior

X-SQL extraction should consistently return actual text content when pages are successfully fetched and elements exist.

#### Actual Behavior

After backend degredation, X-SQL extraction returns empty strings for all fields even though pages are fetched (contentLength > 0). The first run against the same URLs with the same query works correctly. This is correlated with the ProtoNotFound state but may be a separate issue.

#### Root Cause Analysis

Likely related to the same protocol/fetch degredation issue. When the fetch pipeline is in a bad state, pages may be loaded from cache or partially loaded without proper DOM construction, causing X-SQL functions to return empty strings. Alternatively, the DOM parser (Jsoup) may not be processing the HTML correctly in certain backend states.

#### Code Pointer

`browser4-rest/.../CrawlService.kt — X-SQL result aggregation; browser4-core/.../DomFunctions or X-SQL execution engine`

#### AI Suggested Improvement

- Verify X-SQL extraction works on a freshly parsed DOM, not on stale/cached page state
- Add validation: if a page has contentLength > 0 but all X-SQL fields are empty, log a warning
- Consider re-fetching pages when X-SQL extraction returns all-empty results
- Add a health check for the DOM parsing pipeline similar to the protocol check

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: Loop subcommand mode does not pass -s (session) flag to nested browser4-cli process

**Severity:** High
**Category:** Product

#### Reproduction

./b4w.ps1 loop --name test --count 2 -i 10 -- -s price-watch eval "document.querySelector('#product-price').textContent.trim()"

Output: [ERROR] Iteration 1: Error: Unknown command: 'price-watch'

#### Expected Behavior

The -s price-watch flag should be passed through to the nested browser4-cli process, allowing the loop to target a named session.

#### Actual Behavior

The -s flag is lost or mis-parsed, and 'price-watch' is treated as a command name rather than the value of -s. The loop runs successfully on the default session instead.

#### Root Cause Analysis

The argument parsing after -- appears to consume -s as a loop-level flag or mishandles the argument parsing when passing to the nested process. The loop documentation warns about 'known loop flags' after -- but doesn't specifically address -s handling. The nested process may receive the arguments in a different order or with different tokenization than expected.

#### Code Pointer

`cli/browser4-cli/src/commands/loop.rs — argument parsing for subcommand mode`

#### AI Suggested Improvement

- Fix argument passthrough to ensure -s <name> is correctly forwarded to the nested browser4-cli process
- Add a dedicated test: loop with -s targeting a named session should run eval on that session
- Document the -s limitation or provide a workaround (e.g., --session flag for loop itself)
- Consider adding loop-level --session flag that sets the session for all subcommand iterations

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 5: Crawl operations are extremely slow (2+ minutes for 3 simple localhost pages)

**Severity:** High
**Category:** UX

#### Reproduction

./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql --format table --refresh

3 URLs on localhost:18080 take 96-110 seconds to complete.

#### Expected Behavior

3 simple static HTML pages on localhost should complete in under 10 seconds.

#### Actual Behavior

Crawl operations show 'waiting for first page' for 86-96 seconds before processing begins, even for localhost pages. Total time for 3 URLs is ~100 seconds. The CLI polls every 10 seconds but the backend takes much longer to begin processing.

#### Root Cause Analysis

The crawl pipeline has significant startup latency — likely JVM warmup, browser context initialization, and page loading infrastructure setup. For localhost pages that should load instantly, the overhead dominates. The 'waiting for first page' phase suggests the backend's task queue or worker pool has high latency before processing begins. The task TTL of 60 minutes is excessive for simple crawls.

#### Code Pointer

`browser4-rest/.../CrawlService.kt — task submission and worker pool initialization`

#### AI Suggested Improvement

- Reduce worker pool startup latency for simple/local crawls
- Add a fast path for localhost/same-machine URLs that skips unnecessary initialization
- Show more granular progress during the 'waiting for first page' phase so users understand what's happening
- Consider a 'warm' mode that pre-initializes browser contexts for faster first-page load
- Reduce the polling interval from 10s to 2s or make it configurable

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 6: Swarm session state conflicts between restarts (stale tasks + SWARM session warning)

**Severity:** Medium
**Category:** Reliability

#### Reproduction

1. Create swarm session, submit tasks, then kill-all the backend
2. Restart backend and create new swarm session
3. Old swarm tasks from prior session may still be tracked
4. Backend logs: 'SWARM session does not exist, falling back to default'

#### Expected Behavior

After kill-all + restart, all swarm state should be clean. New swarm sessions should work without warnings or conflicts.

#### Actual Behavior

Stale swarm tasks from prior sessions persist in the task store. The swarm create command needs --clear-stale to work around this. Backend logs show warnings about SWARM session not existing when tasks from prior sessions try to run. In the first attempt, swarm tasks stayed in 'queued' state indefinitely.

#### Root Cause Analysis

The SWARM session uses a fixed session ID ('SWARM'). After kill-all, the backend creates a new session but old task records still reference the previous SWARM session. The task store (in-memory or persistent) doesn't fully clear on restart. The --clear-stale flag exists as a workaround but the UX is poor (users must know about it).

#### Code Pointer

`browser4-rest/.../SwarmController.kt — session lifecycle; browser4-rest/.../SwarmService or task store`

#### AI Suggested Improvement

- Auto-clear stale swarm tasks on backend startup
- Use unique session IDs instead of fixed 'SWARM' to avoid conflicts
- Improve error message: instead of 'SWARM session does not exist, falling back to default', fail with a clear message about stale state
- Make --clear-stale the default behavior or auto-detect stale state

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 7: htmlsnapshot requires explicit recapture after each goto — discoverability gap

**Severity:** Medium
**Category:** Discoverability

#### Reproduction

for url in url1 url2; do
  ./b4w.ps1 goto "$url"
  ./b4w.ps1 htmlsnapshot get text "#title"  # fails with 'No elements matched'
done

#### Expected Behavior

After goto navigates to a new page, htmlsnapshot get should either auto-capture or give a clear error telling the user to run htmlsnapshot first.

#### Actual Behavior

htmlsnapshot get returns 'No elements matched' with a note about stale snapshots and JavaScript-modified pages. The message is helpful once you read it, but the auto-snapshot from goto only captures the AXTree (for refs), not the HTML snapshot (for CSS extraction). This creates a 'two-step dance' that is easy to miss.

#### Root Cause Analysis

goto triggers an AXTree snapshot (for interaction refs) but NOT an HTML snapshot (for content extraction). The user must explicitly run htmlsnapshot between goto and htmlsnapshot get. The distinction between AXTree snapshot and HTML snapshot is documented in SKILL.md §4a but easy to overlook in practice.

#### Code Pointer

`cli/browser4-cli/src/commands/goto.rs — could optionally auto-capture htmlsnapshot`

#### AI Suggested Improvement

- Consider adding an auto-capture option: goto --htmlsnapshot to capture both AXTree and HTML in one step
- Improve the 'No elements matched' error to explicitly say 'Run htmlsnapshot first to capture the new page'
- Add a tip after goto: '💡 Tip: Run htmlsnapshot to capture page content for CSS extraction'
- Document the goto → htmlsnapshot → htmlsnapshot get workflow more prominently in quick-start examples

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 8: Confusing diagnostic: crawl says CSS selector matched zero elements when htmlsnapshot shows matches

**Severity:** Medium
**Category:** UX

#### Reproduction

1. ./b4w.ps1 goto http://localhost:18080/generated/crawl/index.html
2. ./b4w.ps1 htmlsnapshot get all attr "a.product" href  → returns ["product/1.html", "product/2.html", "product/3.html"]
3. ./b4w.ps1 crawl <same-url> -d 1 -ol "a.product" → 'CSS selector a.product matched zero elements'

#### Expected Behavior

The same CSS selector should produce consistent results whether used in htmlsnapshot or crawl link discovery. When they differ, the diagnostic should explain why and suggest a workaround.

#### Actual Behavior

htmlsnapshot matches 3 elements with a.product while crawl matches 0. The diagnostic message says to 'try a broader selector' but even 'a' matches 0. This creates confusion because the selector clearly works in one context but not another.

#### Root Cause Analysis

The crawl engine and interactive browser session use different DOM representations or parsing paths. The diagnostic assumes the selector is wrong rather than the DOM being different. This is misleading because the real issue is the DOM parsing, not the selector.

#### Code Pointer

`browser4-rest/.../CrawlService.kt — diagnostic generation`

#### AI Suggested Improvement

- When CSS selectors match zero elements but the page has many matching tags (e.g., 12 anchors), suggest the DOM parsing may differ rather than blaming the selector
- Add a diagnostic that dumps the first few elements found to help debug DOM differences
- Consider unifying the DOM path between crawl and interactive sessions
- The diagnostic should mention: 'Try htmlsnapshot inspect on the same page to verify selectors work in the interactive session'

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 9: No progress feedback during crawl 'waiting for first page' phase

**Severity:** Low
**Category:** UX

#### Reproduction

Run any crawl command. The output shows 'Crawling... waiting for first page (Xs elapsed, N URLs queued)' with no indication of what's happening for up to 90+ seconds.

#### Expected Behavior

Progress should show what the backend is doing during the long wait: initializing browser context, loading page, parsing DOM, etc.

#### Actual Behavior

The CLI shows a monotonically increasing timer with no insight into backend activity. Users have no way to know if the crawl is working or stuck.

#### Root Cause Analysis

The crawl backend doesn't expose fine-grained progress events during the initialization and page-loading phases. The CLI only knows about 'pages found' which requires a page to be fully processed.

#### Code Pointer

`browser4-rest/.../CrawlService.kt — progress reporting; cli/browser4-cli/src/commands/crawl.rs — status display`

#### AI Suggested Improvement

- Add backend progress stages: initializing, loading_seed, fetching_page_N, parsing, extracting
- Expose these stages in crawl status output
- If 'waiting for first page' exceeds 30s, show a hint about what might be happening
- Consider a --verbose flag for more detailed progress

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

**Completion Status:** Partially Successful — 5 of 6 acceptance criteria completed. AC3 (crawl link discovery) failed due to a fundamental backend bug where CSS selectors match zero elements in the crawl engine. A workaround using manual link discovery + depth-0 crawl partially succeeded. AC4 required a backend restart to recover from a stuck state (ProtoNotFound errors). All other ACs completed successfully.

**Success Rate:** 83% — 5 of 6 ACs completed, but with significant reliability issues requiring backend restarts and workarounds

**Issues Found:** 9

**Major Blockers:** Crawl link discovery (depth >= 1) CSS selectors match zero elements — this is a product bug that prevents basic crawl functionality. Backend ProtoNotFound degredation requires kill-all + restart between operations.

**Most Confusing Aspects:** 1. The crawl link discovery diagnostic says 'CSS selector matched zero elements' when the same selector works perfectly in htmlsnapshot — the root cause (different DOM parsing) is not explained. 2. The distinction between snapshot (AXTree for interaction) and htmlsnapshot (HTML for extraction) is documented but easy to miss — several commands fail silently when htmlsnapshot wasn't captured first. 3. The loop subcommand's -s flag handling is unintuitive — it looks like correct syntax but silently fails. 4. Crawl operations are extremely slow even for localhost pages, creating uncertainty about whether the command is working or stuck.

**Most Valuable Improvements:** 1. Fix crawl link discovery CSS selector matching (critical product bug). 2. Fix backend ProtoNotFound degredation (critical reliability). 3. Add backend health auto-recovery to avoid manual kill-all. 4. Fix loop subcommand -s flag passthrough. 5. Reduce crawl startup latency for simple/localhost use cases. 6. Unify CSS selector behavior between crawl engine and interactive browser session.

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

#### Issue 1: Crawl link discovery (depth >= 1) CSS selector matching is fundamentally broken

./b4w.ps1 crawl http://localhost:18080/generated/crawl/index.html -d 1 -ol "a" --refresh

Any CSS selector passed via --out-link-selector (including the bare 'a' tag selector) matches zero elements during crawl link discovery, even though the page has 12 anchors and 8044B of HTML. The diagnostic confirms: 'The page has 12 anchors and 8044B of HTML, but the CSS selector 'a' matched zero elements.' Meanwhile, goto + htmlsnapshot on the same page correctly matches elements with the same selectors.

#### Issue 2: Backend degrades into stuck state requiring restart (ProtoNotFound errors)

1. Run several crawl and swarm operations over 15-30 minutes
2. Observe that subsequent operations hang indefinitely
3. Check doctor logs: 'ProtoNotFound(1600)' errors on all fetch operations
4. Crawl depth 0 tasks complete but X-SQL extraction returns empty strings
5. Fix requires kill-all + restart

#### Issue 3: Crawl X-SQL extraction returns empty strings despite pages being fetched

Run crawl --seed-file with --depth 0 --sql @query.sql after the backend has been running for a while. First run succeeds with data. Second run (after other operations) returns pages with content but all extracted fields are empty strings.

Example: 3 pages crawled, 3 rows extracted but title='' and price=''.

#### Issue 4: Loop subcommand mode does not pass -s (session) flag to nested browser4-cli process

./b4w.ps1 loop --name test --count 2 -i 10 -- -s price-watch eval "document.querySelector('#product-price').textContent.trim()"

Output: [ERROR] Iteration 1: Error: Unknown command: 'price-watch'

#### Issue 5: Crawl operations are extremely slow (2+ minutes for 3 simple localhost pages)

./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql --format table --refresh

3 URLs on localhost:18080 take 96-110 seconds to complete.

#### Issue 6: Swarm session state conflicts between restarts (stale tasks + SWARM session warning)

1. Create swarm session, submit tasks, then kill-all the backend
2. Restart backend and create new swarm session
3. Old swarm tasks from prior session may still be tracked
4. Backend logs: 'SWARM session does not exist, falling back to default'

#### Issue 7: htmlsnapshot requires explicit recapture after each goto — discoverability gap

for url in url1 url2; do
  ./b4w.ps1 goto "$url"
  ./b4w.ps1 htmlsnapshot get text "#title"  # fails with 'No elements matched'
done

#### Issue 8: Confusing diagnostic: crawl says CSS selector matched zero elements when htmlsnapshot shows matches

1. ./b4w.ps1 goto http://localhost:18080/generated/crawl/index.html
2. ./b4w.ps1 htmlsnapshot get all attr "a.product" href  → returns ["product/1.html", "product/2.html", "product/3.html"]
3. ./b4w.ps1 crawl <same-url> -d 1 -ol "a.product" → 'CSS selector a.product matched zero elements'

#### Issue 9: No progress feedback during crawl 'waiting for first page' phase

Run any crawl command. The output shows 'Crawling... waiting for first page (Xs elapsed, N URLs queued)' with no indication of what's happening for up to 90+ seconds.

