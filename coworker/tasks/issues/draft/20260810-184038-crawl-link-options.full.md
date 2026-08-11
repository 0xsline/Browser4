## Deliverables

---

### A. Task Result

The crawl evaluation is **Partially Successful**. AC1 (basic depth-0 crawl) passed. AC4 (seed file bulk fetch) partially passed — URLs were resolved but page content had fetch errors. AC2 and AC3 (link discovery mode) both failed due to a backend protocol-not-found issue with localhost URLs.

---

### B. Execution Trace

| Step | Command | Outcome |
|------|---------|---------|
| 1 | `./b4w.ps1 help` | Loaded full command reference |
| 2 | `crawl ... --depth 0 --refresh` | AC1: 1 page, "Crawl Test Hub", ~140s |
| 3 | `crawl ... -d 2 -ol "a.product" -olp "/product/"` | AC2: 1 page only, no links followed |
| 4 | `crawl ... --depth 2 --out-link-selector "a.product" ...` | AC2 retry: same result, 1 page |
| 5 | `crawl ... --depth 3 --refresh` | AC3: Shown "Link discovery disabled (no --out-link-selector)" |
| 6 | `crawl --seed-file ... --depth 0 --refresh` | AC4: 2 URLs found, 1 fetch error |
| 7 | `doctor log pulsar grep "crawl\|Protocol"` | Diagnosed root cause |

**Major workarounds:** Used `crawl cancel`/`crawl clear` to clean up hung tasks. Used backend log inspection to identify root cause. AC3 confirmed documentation is correct (no link discovery without `-ol`), but the AC spec itself is wrong.

---

### C & D. Issues and Assessment

```json
{
  "issues": [
    {
      "title": "Crawl link discovery fails with 'Protocol not found' for localhost URLs",
      "severity": "Critical",
      "category": "Product",
      "reproduction": "./b4w.ps1 crawl \"http://localhost:18080/generated/crawl/index.html\" -d 2 -ol \"a.product\" --refresh",
      "expected": "Crawl should discover and follow links matching the CSS selector, returning multiple pages at different depths.",
      "actual": "Only 1 page found (the seed URL). Backend logs show 'Protocol not found | http://localhost:18080/...' warnings for every discovered link. The crawl returns quickly with no linked pages loaded. With broader selectors like 'a[href]', the crawl times out after 300s per seed URL with 'Timed out waiting for 300000 ms'.",
      "rootCause": "Backend FetchComponent cannot handle http://localhost URLs when crawling with out-link-selector. Log entries show 'Protocol not found (1600)' errors. The seed URL loads (from cache) but discovered out-links fail to fetch. The link extraction itself works (logs show product/1.html through product/3.html and category pages were discovered), but the follow-up fetch fails silently.",
      "codePointer": "browser4-rest — FetchComponent or the protocol handler responsible for fetching linked pages during crawl. The issue appears in the 'U for RR' (URL for re-render?) fetch path that returns status 1600 (ProtoNotFound).",
      "suggestion": "- Fix the FetchComponent protocol handler to support http://localhost URLs during crawl link following\n- Add clear error messages to the CLI when linked pages fail to load, rather than silently returning only the seed page\n- Consider using the same page-loading mechanism for both seed URLs and discovered out-links\n- Add a diagnostic message like the one seen for external URLs ('Portal page returned near-empty content') to localhost crawl failures"
    },
    {
      "title": "Crawl page loading is extremely slow (~140s per page)",
      "severity": "High",
      "category": "Reliability",
      "reproduction": "./b4w.ps1 crawl \"http://localhost:18080/generated/crawl/index.html\" --depth 0 --refresh",
      "expected": "A simple static HTML page served from localhost should load in under 5 seconds.",
      "actual": "The crawl took ~140 seconds to load a single page from localhost. Progress messages show 'waiting for first page' at 10s intervals for over 2 minutes. AC4 showed similar slowness (~130s for 2 product pages).",
      "rootCause": "Unknown — the slow loading affects both depth-0 (bulk fetch) and link-discovery modes. The 'waiting for first page' messages suggest the backend's page-loading mechanism for crawl has high latency even for localhost pages. This may be related to browser context startup, portal initialization, or an internal retry/timeout loop.",
      "codePointer": "",
      "suggestion": "- Profile the crawl page-loading pipeline to identify the bottleneck (browser startup? network timeout? retry logic?)\n- For localhost URLs, reduce internal timeouts and retry intervals\n- Consider adding a fast-path for localhost/loopback addresses\n- Show more granular progress (e.g., 'starting browser', 'loading page', 'extracting content') instead of just 'waiting for first page'"
    },
    {
      "title": "Inconsistent crawl behavior with and without --out-link-selector",
      "severity": "High",
      "category": "Reliability",
      "reproduction": "Compare: (a) `crawl <url> --depth 0 --refresh` vs (b) `crawl <url> -d 1 -ol \"a.product\" --refresh`",
      "expected": "Both should load the seed page using the same mechanism; (b) should additionally extract and follow links.",
      "actual": "(a) takes ~140s, shows progress messages, returns page with correct title. (b) completes in ~16s, shows NO progress messages, returns page with depth=1 (not depth=0). The page-loading code path appears entirely different depending on whether -ol is present.",
      "rootCause": "The crawl appears to use different fetch mechanisms depending on whether out-link-selector is provided. Without -ol, it uses a browser/portal-based load (slower but functional). With -ol, it uses FetchComponent (faster but broken for localhost). The depth numbering also differs: depth=1 with -ol vs depth=1 without (should be depth=0 for seed).",
      "codePointer": "browser4-rest CrawlService — the branching logic that selects the fetch mechanism based on the presence of out-link-selector.",
      "suggestion": "- Unify the page-loading code path regardless of whether link discovery is enabled\n- Fix depth numbering to be consistent (seed URL should always be depth 0)\n- Use the same page-loading mechanism for all crawl modes\n- Ensure progress reporting is consistent across all crawl modes"
    },
    {
      "title": "AC3 acceptance criterion contradicts documented crawl behavior",
      "severity": "Medium",
      "category": "Documentation",
      "reproduction": "Run `crawl <url> --depth 3 --refresh` without --out-link-selector as specified in AC3.",
      "expected": "Per AC3 spec: 'traverses 4 levels (0–3) and reaches terminal depth-3 pages'.",
      "actual": "CLI correctly notes 'Link discovery disabled (no --out-link-selector). Processing seed URLs only.' and returns only 1 page. The documentation is RIGHT — the AC spec is WRONG.",
      "rootCause": "The acceptance criteria for this evaluation task were written without accounting for the documented requirement that --out-link-selector is mandatory for link discovery. The crawl documentation clearly states: '--out-link-selector is required for link discovery. Without it, only seed URLs are processed regardless of depth.'",
      "codePointer": "",
      "suggestion": "- Update AC3 to include --out-link-selector (e.g., `crawl <url> --depth 3 --refresh -ol \"a.product\"`)\n- Consider whether a default out-link-selector (e.g., 'a[href]') would improve usability for simple crawl use cases\n- The current behavior is correct per documentation; the acceptance criteria need fixing, not the code"
    },
    {
      "title": "Seed file crawl returns fetch errors for valid localhost URLs",
      "severity": "Medium",
      "category": "Reliability",
      "reproduction": "./b4w.ps1 crawl --seed-file .test-sessions/crawl-seed.txt --depth 0 --refresh",
      "expected": "Both product pages should load successfully with correct titles.",
      "actual": "2 URLs processed, but one page shows empty title and the other shows '⚠ fetch returned 0 bytes (possible protocol handler not ready)'. The warning message 'protocol handler not ready' is vague and doesn't help diagnose the issue.",
      "rootCause": "Related to Issue 1 — the FetchComponent protocol handler for localhost URLs is unreliable. The 'protocol handler not ready' message suggests a race condition where the fetch mechanism isn't initialized before the crawl attempts to use it.",
      "codePointer": "browser4-rest FetchComponent — protocol handler initialization/readiness check.",
      "suggestion": "- Add a readiness check before starting crawl fetches to ensure the protocol handler is initialized\n- Retry failed fetches automatically (at least once) when 'protocol handler not ready' is detected\n- Provide more specific error messages (e.g., which protocol handler, what's needed to fix it)\n- Consider a pre-flight check that validates the fetch pipeline before processing the full URL list"
    },
    {
      "title": "Crawl tasks silently accumulate and cause interference between test runs",
      "severity": "Medium",
      "category": "UX",
      "reproduction": "Run multiple crawl commands in sequence. One hung task blocks subsequent ones.",
      "expected": "Each crawl command should be independent; a timeout or failure in one should not affect the next.",
      "actual": "After a crawl with 'a[href]' timed out, subsequent crawls showed 'waiting for first page' indefinitely. The backend had accumulated 8 tasks with 2 still stuck in 'processing' state. Had to manually cancel and clear tasks before proceeding.",
      "rootCause": "The crawl task store retains tasks across invocations, and the CLI's polling loop doesn't distinguish between its own task and previously-stuck tasks. When a task hangs on the backend, the CLI's progress polling may get confused. Also, the CLI timeout doesn't cancel the backend task — it stays in 'processing' state.",
      "codePointer": "cli/browser4-cli — crawl polling/cleanup logic; browser4-rest CrawlService — task lifecycle management",
      "suggestion": "- Auto-cancel the backend task when the CLI hits its timeout\n- Add a 'crawl clean' or 'crawl reset' command that cancels all running tasks and clears the store in one step\n- Show a warning if there are stuck/processing tasks when starting a new crawl\n- Add a --force flag to crawl that auto-cancels existing stuck tasks for the same session"
    },
    {
      "title": "Crawl progress reporting is sparse and unhelpful for debugging",
      "severity": "Medium",
      "category": "UX",
      "reproduction": "Run any crawl with link discovery that fails silently.",
      "expected": "Progress output should indicate how many links were discovered, how many pages loaded successfully vs failed, and any per-page errors.",
      "actual": "Progress only shows 'waiting for first page' and 'N pages found so far'. When link discovery fails, the output is 'Crawl completed. 1 pages found.' with no indication that 6+ links were discovered but all failed to load. The --verbose flag added no additional detail.",
      "rootCause": "The CLI's crawl polling only reports aggregate page counts and doesn't surface per-link or per-page status during the crawl. The --verbose flag only affects the final result display, not progress reporting.",
      "codePointer": "cli/browser4-cli — crawl polling/progress display; browser4-rest CrawlService — progress reporting API",
      "suggestion": "- Add live per-page status during crawling: 'loaded 1/7, 2 failed, 4 pending'\n- Show discovered link count: 'found 6 out-links, crawling...'\n- Surface per-page errors during progress (not just in final result)\n- Make --verbose add more detail to progress output, not just final results\n- Add a --debug flag that shows full fetch diagnostics per page"
    },
    {
      "title": "Crawl depth numbering is inconsistent (depth=1 for seed URL)",
      "severity": "Low",
      "category": "Product",
      "reproduction": "./b4w.ps1 crawl \"http://localhost:18080/generated/crawl/index.html\" --depth 0 --refresh",
      "expected": "Seed URL should be reported at depth=0.",
      "actual": "Output shows 'depth=1 | ...index.html | Crawl Test Hub' and JSON shows '\"depth\": 1'. The seed URL is consistently reported at depth 1 instead of depth 0.",
      "rootCause": "CrawlService uses 1-based depth counting internally while the documentation and CLI flags use 0-based semantics (depth 0 = no link following). This is a mismatch between internal representation and user-facing semantics.",
      "codePointer": "browser4-rest CrawlService — depth assignment for seed URLs",
      "suggestion": "- Change seed URL depth to 0 (or adjust documentation to clearly state depth is 1-based)\n- Ensure consistency: if --depth 0 means 'no link discovery', then seed URLs should be depth 0\n- Document the depth numbering scheme explicitly in crawl --help"
    },
    {
      "title": "No crawler-level retry for transient fetch failures",
      "severity": "Low",
      "category": "UX",
      "reproduction": "Run seed file crawl — one URL gets 'fetch returned 0 bytes (possible protocol handler not ready)'.",
      "expected": "Transient fetch failures should be automatically retried at least once.",
      "actual": "The crawl reports the error and moves on. The user must re-run the entire crawl to retry failed URLs.",
      "rootCause": "The crawl's seed processing doesn't include retry logic for individual URL fetch failures. The FetchComponent reports the error, and CrawlService treats it as terminal for that URL.",
      "codePointer": "browser4-rest CrawlService — seed URL processing loop",
      "suggestion": "- Add per-URL retry with configurable count (--retry N, default 1)\n- Implement exponential backoff between retries\n- Report retry attempts in progress output\n- Consider automatic retry for 'protocol handler not ready' errors specifically"
    }
  ],
  "assessment": {
    "completionStatus": "Partially Successful — AC1 passed fully; AC4 passed partially (URLs resolved, content had errors); AC2 and AC3 failed due to backend protocol handler issues with localhost URLs. The crawl command's depth-0 bulk fetch mode works but has reliability issues; link discovery mode is non-functional for localhost URLs in this environment.",
    "successRate": "35% — 1 of 4 acceptance criteria passed cleanly, 1 partially passed, 2 failed. AC1: 100%, AC2: 0%, AC3: 0%, AC4: 40%.",
    "issuesFound": 9,
    "majorBlockers": "The 'Protocol not found' error in FetchComponent makes link discovery mode (--out-link-selector) non-functional for localhost URLs. This blocks all crawl scenarios that require following links (AC2, AC3). The ~140s page-load time for localhost pages also makes iterative testing prohibitively slow.",
    "mostConfusingAspects": "1. The crawl completed 'successfully' (status: OK) with 1 page found when link discovery silently failed — no error indicated that 6+ links were discovered but all failed to load. 2. The drastically different behavior (timing, depth numbering, progress output) between crawl with and without --out-link-selector made it hard to understand what was happening. 3. The 'Link discovery disabled (no --out-link-selector)' note only appeared when running without -ol, but never appeared as a warning when -ol was provided and links still weren't followed — making it seem like -ol should have worked.",
    "mostValuableImprovements": "1. Fix the FetchComponent protocol handler for localhost URLs (Critical — unblocks all link discovery). 2. Add transparent error reporting when link discovery finds links but fails to load them (Critical — current silent failure is misleading). 3. Reduce page-load latency for localhost URLs (High — 140s/page makes testing impractical). 4. Add per-URL progress and error reporting during crawl (Medium — essential for debugging). 5. Auto-cancel backend tasks on CLI timeout (Medium — prevents task accumulation).",
    "usabilityRating": 3
  }
}
```

---

### Summary

The `crawl` command's **bulk fetch mode** (depth 0, seed files) is functional but slow and occasionally unreliable for localhost URLs. The **link discovery mode** (`--out-link-selector`) is completely broken for localhost URLs due to a `Protocol not found` error in the backend's `FetchComponent`. 

The documentation is clear and well-structured — the crawl reference (`skills/browser4-cli/references/crawl.md`) accurately describes the flags, constraints, and behavior. The `--help` output is comprehensive with good examples. However, the **silent failure mode** (crawl reports "completed, 1 pages found" when links were discovered but all failed to load) is a serious UX problem that would mislead users into thinking their selector was wrong rather than revealing a backend issue.

The most impactful fix would be addressing the `Protocol not found` error in FetchComponent, followed by improving crawl progress transparency so users can diagnose issues without inspecting backend logs.
