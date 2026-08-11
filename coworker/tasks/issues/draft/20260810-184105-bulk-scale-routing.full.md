---

# Evaluation Report: browser4-cli Bulk/Scale Approaches

## A. Task Result

The task — exercising all six branches of SKILL.md §4b "Choosing Bulk/Scale Approach" — was **partially successful**. Four of six acceptance criteria were fully verifiable; two (AC3 link discovery, AC4 swarm X-SQL) failed due to backend issues with the X-SQL pipeline.

**Working approaches:**
- **AC2 — crawl depth 0 (seed file):** Works but is slow (~140s for 3 URLs) and flaky (inconsistent X-SQL results across runs)
- **AC5 — loop monitoring:** Works correctly with `eval --file` for repeated page checks
- **AC6 — shell loop:** Simple, reliable, fast — the best option for small URL sets
- **AC1 — htmlsnapshot query:** Broken (417 errors), but `eval` workaround works

**Broken approaches:**
- **AC3 — crawl link discovery:** Completely broken — `--out-link-selector` finds zero links, crawls timeout after 600s
- **AC4 — swarm X-SQL:** Tasks timeout (408) or stay queued indefinitely; same X-SQL pipeline failure as AC1

## B. Execution Trace

### Commands Used (major)

| Step | Command | Outcome |
|------|---------|---------|
| Setup | `./b4w.ps1 help` | Full help displayed |
| Setup | Read SKILL.md + crawl/swarm/loop/x-sql references | Documentation reviewed |
| AC1 | `goto`, `htmlsnapshot`, `htmlsnapshot inspect` | Page structure discovered |
| AC1 | `htmlsnapshot query ... --sql @query.sql` (×5) | All failed with 417 |
| AC1 | `eval --file extract.js` | **Workaround**: 6 products extracted |
| AC2 | `crawl --seed-file ... --depth 0 --sql @q.sql --refresh` (×3) | Flaky: 1/3, 2/3, then 3/3 success |
| AC3 | `crawl ... -d 2 -ol "a.product" -olp "/product/"` | 0 links discovered |
| AC3 | `crawl ... -d 1 -ol "a.product"` (no filter) | Timed out after 600s |
| AC3 | `eval` → manual link extraction → seed file → `crawl --depth 0` | **Workaround**: 3 pages crawled |
| AC4 | `swarm create --display-mode HEADLESS ...` | Session created |
| AC4 | `swarm query --sql @q.sql --seed-file ... --wait` (×2) | Jobs stuck queued / timed out (408) |
| AC4 | Recovery: `swarm list --clear`, `swarm close`, `swarm create` | Recovery procedure works |
| AC5 | `loop --name mock-price-watch --count 2 -i 5 -- eval --file ...` | 2 iterations, both returned `$899.99` |
| AC5 | `loop --list`, `loop --history` | Working correctly |
| AC6 | Shell `for` loop over 3 URLs with `goto`, `htmlsnapshot`, `get text` | All 3 products extracted perfectly |

### Key Decisions

1. Used `eval --file` instead of inline JS expressions to avoid shell quoting issues — following the SKILL.md warning about Windows quoting
2. Used `--file` approach for X-SQL queries (`@file.sql`) throughout
3. When X-SQL failed for AC1, used `eval` + JS DOM API as workaround
4. When crawl link discovery failed for AC3, manually extracted links via `eval` and used seed file approach
5. Skipped `-s` session flag in loop subcommands after discovering it's not passed through `--`

### Workarounds Required

1. **AC1:** `eval` with DOM API instead of `htmlsnapshot query`
2. **AC3:** Manual link extraction via `eval` → seed file → `crawl --depth 0`
3. **AC4:** None effective — swarm X-SQL is fundamentally blocked by the same backend issue as AC1
4. **AC5:** Used default session instead of named session (loop `--` doesn't pass `-s` through)

---

```json
{
  "issues": [
    {
      "title": "X-SQL DOM_LOAD_AND_SELECT consistently fails with 417 \"object already closed\"",
      "severity": "Critical",
      "category": "Reliability",
      "reproduction": "./b4w.ps1 htmlsnapshot query \"http://localhost:18080/ec/b?node=1292115012\" --sql @query.sql --format table\n\nRun any X-SQL query via htmlsnapshot query. 100% reproducible across 5 attempts on listing pages and product detail pages.",
      "expected": "Structured result set with correlated title, price, rating, and URL for each product card.",
      "actual": "417 Expectation Failed with message: \"The object is already closed; SQL statement: SELECT ... FROM DOM_LOAD_AND_SELECT(...) [90007-197]\". The page is fetched (pageContentBytes > 0, pageStatusCode 200) but the H2 SQL engine reports the session/object is already closed before the query can execute.",
      "rootCause": "Backend race condition in the X-SQL pipeline. The scrape session (H2 database connection or result set) is being closed before the SQL query executes. The error code 90007-197 is an H2 database error indicating a closed object. The page fetch succeeds (200 status, ~10KB content), but the SQL execution layer receives a closed handle. This may be a thread-safety issue in the scrape session lifecycle where the session is marked as done/closed between page load completion and query execution start.",
      "codePointer": "browser4-rest — the scrape session management code that creates/closes H2 database sessions for X-SQL query execution. Likely in the X-SQL executor or scrape session lifecycle handler.",
      "suggestion": "- Add retry logic in the X-SQL query path: if the session is closed, reopen and retry once before failing\n- Investigate thread-safety of session close vs query execution — ensure the session is not closed until the query completes\n- Consider using a session pool or keeping sessions alive for the duration of query submission + execution\n- Add a synchronization barrier between page load completion callback and SQL execution to prevent premature session cleanup"
    },
    {
      "title": "Crawl --out-link-selector link discovery finds zero links",
      "severity": "Critical",
      "category": "Reliability",
      "reproduction": "./b4w.ps1 crawl \"http://localhost:18080/generated/crawl/index.html\" -d 1 -ol \"a.product\"\n\nThe page has 3 <a class=\"product\"> links with href=\"product/1.html\" etc., verified via both htmlsnapshot and eval.",
      "expected": "Crawl discovers 3 product links from the index page and crawls them.",
      "actual": "Crawl loads the seed page but never extracts any links. Progress shows \"1 pages found so far\" for 600 seconds, then times out. With -olp filter, completes immediately with 0 links found.",
      "rootCause": "Likely shares the same root cause as the X-SQL 417 error. Crawl's link discovery uses DOM_LOAD_AND_SELECT (or similar X-SQL pipeline) internally to extract links from crawled pages. Since that pipeline is broken, no links are discovered. The crawl then sits in a polling loop waiting for link extraction results that never arrive.",
      "codePointer": "browser4-rest crawl link extraction logic — specifically the code that processes out-link-selector and extracts hrefs from crawled pages. Likely uses the same X-SQL/scrape session infrastructure as htmlsnapshot query.",
      "suggestion": "- Fix the X-SQL pipeline (see Issue 1) — this will likely fix crawl link discovery as well\n- Add a timeout for individual page link extraction (not just overall crawl timeout)\n- Emit a warning when link extraction returns 0 results but the selector matches elements on the page\n- Add diagnostic output showing how many links the selector matched vs how many were filtered by pattern"
    },
    {
      "title": "Crawl is extremely slow — ~140s for 3 local pages",
      "severity": "High",
      "category": "Product",
      "reproduction": "./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql --format table --refresh\n\nSeed file contains 3 localhost URLs. Run and observe the wall clock time.",
      "expected": "3 local pages should be fetched and queried in < 30 seconds total.",
      "actual": "Crawl takes 116-156 seconds for 3 URLs, with \"waiting for first page\" messages persisting for 100+ seconds even for depth-0 bulk fetch (no link discovery). Each page takes 40-50 seconds to process.",
      "rootCause": "The per-page latency appears to be dominated by page-load wait time and scrape session setup/teardown overhead. The crawl framework may be using conservative defaults for page load timeouts or polling intervals. The 'waiting for first page' phase may include backend cold-start or queue processing delays.",
      "codePointer": "browser4-rest crawl infrastructure — page load timeout defaults, polling intervals, and scrape session initialization.",
      "suggestion": "- Reduce default page load timeouts for local/known-fast sites\n- Start processing the first page immediately rather than waiting for all queued pages to be ready\n- Add a --fast flag that uses shorter timeouts for trusted/known sites\n- Report per-page timing breakdown in output so users can identify bottlenecks"
    },
    {
      "title": "Crawl X-SQL extraction produces inconsistent/flaky results",
      "severity": "High",
      "category": "Reliability",
      "reproduction": "./b4w.ps1 crawl --seed-file urls.txt --depth 0 --sql @query.sql --format table --refresh\n\nRun 3 times with the same 3 URLs. The X-SQL query uses #productTitle and #product-price selectors.",
      "expected": "All 3 URLs consistently return title and price data.",
      "actual": "Run 1: 1/3 rows have data. Run 2: 2/3 rows have data. Run 3: 3/3 rows have data. Failed rows show empty strings for title and price but the URL column is populated, confirming the page was crawled.",
      "rootCause": "Same X-SQL pipeline race condition as Issue 1, but manifesting intermittently rather than consistently. Some pages' SQL queries succeed while others silently fail (empty result instead of 417 error). The timing of session close vs query execution varies per page, leading to non-deterministic failures.",
      "codePointer": "Same as Issue 1 — X-SQL scrape session lifecycle in browser4-rest.",
      "suggestion": "- Same root cause fix as Issue 1\n- Additionally: when X-SQL returns empty results but page fetch succeeded (pageContentBytes > 0), log a warning and automatically retry the query\n- Surface per-page errors in crawl output rather than silently producing empty rows\n- Add a --retry-failed flag that re-queries pages that returned empty result sets"
    },
    {
      "title": "Swarm X-SQL jobs timeout (408) or stay queued indefinitely",
      "severity": "High",
      "category": "Reliability",
      "reproduction": "swarm create --display-mode HEADLESS --max-browser-contexts 2 --max-open-tabs 4\nswarm query --sql @query.sql --seed-file urls.txt --refresh --wait",
      "expected": "3 swarm jobs execute in parallel across browser contexts, each returning structured data.",
      "actual": "2 jobs fail with 408 Request Timeout, 1 stays queued (201 Created) indefinitely. Pages are fetched (pageContentBytes ~15KB) but resultSets are empty.",
      "rootCause": "Same X-SQL pipeline issue (Issue 1). Swarm uses DOM_LOAD_AND_SELECT for query execution. The swarm worker pool picks up jobs, fetches pages successfully, but the X-SQL session closes before query execution. The 408 timeout suggests the worker waits for a result that never arrives. The queued task suggests the worker pool may have a concurrency issue where a stuck worker prevents other tasks from being processed.",
      "codePointer": "browser4-rest swarm worker pool and X-SQL execution path.",
      "suggestion": "- Fix the X-SQL pipeline (Issue 1) — this is the root cause\n- Add a per-job timeout in swarm workers with automatic retry\n- Improve worker pool health monitoring — if a worker is stuck on a 408/timeout, release it to process other jobs\n- Add swarm create --clear-stale as the default behavior when stale tasks are detected"
    },
    {
      "title": "Stale swarm tasks from prior sessions block worker pool",
      "severity": "Medium",
      "category": "UX",
      "reproduction": "Run swarm create after a previous swarm session was closed without clearing tasks. Submit new jobs — they stay queued while old completed tasks occupy tracking slots.",
      "expected": "Swarm create should either auto-clear stale tasks or warn prominently. Workers should not be blocked by tasks from closed sessions.",
      "actual": "swarm create shows a warning about stale tasks but still creates the session. New jobs stay queued (>300s) because the worker pool is blocked by stale tasks. User must manually run swarm list --clear, swarm close, swarm create to recover.",
      "rootCause": "Swarm task tracking persists across sessions but worker allocation is tied to tracked tasks. Stale completed/failed tasks from prior sessions consume worker pool slots. The --clear-stale flag exists but is opt-in rather than default.",
      "codePointer": "browser4-cli swarm create command and backend swarm session manager.",
      "suggestion": "- Make --clear-stale the default behavior when stale tasks are detected from prior sessions\n- Add a prompt in interactive mode: \"3 stale tasks from prior sessions detected. Clear them? [Y/n]\"\n- Decouple worker pool allocation from task tracking — completed tasks from closed sessions should not block new workers\n- Add a swarm reset command that does clear + close + create in one step"
    },
    {
      "title": "Loop subcommand mode does not pass -s (session) flag through -- delimiter",
      "severity": "Low",
      "category": "UX",
      "reproduction": "./b4w.ps1 loop --name test --count 1 -i 5 -- -s mysession page-info\n\nOr: ./b4w.ps1 loop --name test --count 1 -i 5 -- -s mysession eval 'document.title'",
      "expected": "The nested browser4-cli invocation should receive -s mysession as its first argument and target the named session.",
      "actual": "The -s flag is consumed by the loop CLI parser before --. The nested CLI receives 'mysession' as a positional argument (interpreted as a command name), resulting in 'Unknown command: mysession'.",
      "rootCause": "The loop command's argument parser consumes all -s flags before the -- delimiter, treating them as loop-level session flags rather than passing them through to the nested CLI invocation.",
      "codePointer": "cli/browser4-cli/src/loop.rs — argument parsing logic that handles -- delimiter and builds the nested CLI command.",
      "suggestion": "- After --, stop parsing any flags — pass all arguments verbatim to the nested CLI\n- Or: add explicit documentation that -s must be set on the outer loop command and applies to all iterations\n- Or: support a --session flag on loop itself that passes through to nested invocations"
    },
    {
      "title": "First-run latency: every crawl takes 100s+ before processing first page",
      "severity": "Medium",
      "category": "UX",
      "reproduction": "Run any crawl command. Observe the 'waiting for first page' phase.",
      "expected": "First page should begin processing within 10-20 seconds of crawl submission.",
      "actual": "All crawl commands spend 100-116 seconds in 'waiting for first page' phase before any page processing begins, regardless of --refresh, URL count, or depth.",
      "rootCause": "The crawl backend appears to have a fixed initialization/warmup period before it begins processing queued URLs. This could be JVM warmup, browser context initialization, or queue polling interval. The delay is consistent across all crawl invocations (~100-116s), suggesting a fixed timeout or polling cycle rather than variable startup work.",
      "codePointer": "browser4-rest crawl task executor — initialization sequence, browser context creation, and queue polling loop.",
      "suggestion": "- Initialize browser contexts eagerly when crawl starts rather than lazily on first page access\n- Reduce the queue polling interval during the initial phase\n- Show more granular progress: 'Initializing browser context...', 'Loading first page...'\n- Consider pre-warming a browser context during CLI startup for faster first-crawl latency"
    },
    {
      "title": "Documentation: crawl reference shows -olp '/product/' regex that doesn't match relative hrefs",
      "severity": "Medium",
      "category": "Documentation",
      "reproduction": "Follow the SKILL.md AC3 example: crawl <url> -ol \"a.product\" -olp \"/product/\" against the MockSite crawl fixture where links are relative (product/1.html).",
      "expected": "Documentation should clarify whether -olp matches against raw href or resolved absolute URL, and provide examples for both relative and absolute link patterns.",
      "actual": "The pattern /product/ doesn't match relative hrefs like product/1.html (no leading slash). The crawl reference doesn't specify whether matching is against raw href values or resolved absolute URLs. Users must guess and experiment.",
      "rootCause": "Documentation gap — the crawl.md reference describes --out-link-pattern as a regex filter but doesn't specify the matching target (raw href attribute vs resolved URL). The SKILL.md task instructions use /product/ which assumes absolute-path matching, but MockSite uses relative paths.",
      "codePointer": "skills/browser4-cli/references/crawl.md — out-link-pattern documentation section.",
      "suggestion": "- Document whether -olp matches raw href or resolved absolute URL\n- Add examples for both relative paths (product/) and absolute paths (/product/)\n- Add a tip: 'Use htmlsnapshot get all attr to inspect actual href values before writing -olp patterns'\n- Consider supporting both matching modes via a flag: --olp-match raw|resolved"
    }
  ],
  "assessment": {
    "completionStatus": "Partially Successful — 4 of 6 ACs verifiable (AC2, AC5, AC6 fully; AC1 via workaround). AC3 (crawl link discovery) and AC4 (swarm X-SQL) failed due to a shared X-SQL pipeline backend issue.",
    "successRate": "67% — 4 of 6 acceptance criteria produced valid results (2 required workarounds for broken X-SQL). AC2 required 3 attempts for complete results due to flaky extraction.",
    "issuesFound": 9,
    "majorBlockers": "The X-SQL DOM_LOAD_AND_SELECT pipeline is fundamentally broken, causing failures across htmlsnapshot query (417 errors), crawl link discovery (0 links found, 600s timeout), crawl X-SQL extraction (flaky results), and swarm X-SQL (408 timeouts). This single root cause blocks 3 of the 6 bulk/scale approaches documented in SKILL.md §4b.",
    "mostConfusingAspects": "1. The X-SQL error message says 'known backend race condition' but offers no fix timeline or workaround that actually works. 2. Crawl 'waiting for first page' takes 100s+ with no explanation of what's happening. 3. The -olp regex matching behavior is undocumented — users can't tell if it matches raw href or resolved URL. 4. Stale swarm tasks silently block new jobs with only a warning that's easy to miss. 5. The loop -- delimiter consumes -s flags, making named-session loop monitoring impossible.",
    "mostValuableImprovements": "1. Fix the X-SQL pipeline race condition — this unblocks 4 commands (htmlsnapshot query, crawl --sql, crawl --out-link-selector, swarm query). 2. Add retry logic for failed X-SQL queries and surface per-page errors. 3. Reduce crawl initialization latency from 100s to <20s. 4. Make swarm auto-clear stale tasks. 5. Add --session flag to loop command for named-session targeting.",
    "usabilityRating": 4
  }
}
```
