# Issues: webminer-structuring-routing

> **Source:** `20260810-191909-webminer-structuring-routing.full.md` | **Date:** 20260810-191909 | **Mode:** dev

---

## Issues Found (8 issues)

### Issue 1: Swarm worker pool permanently stalled — all tasks stuck in 'queued'

**Severity:** Critical
**Category:** Reliability

#### Reproduction

1. ./b4w.ps1 swarm create --display-mode HEADLESS
2. ./b4w.ps1 swarm query --sql @query.sql --seed-file urls.txt --refresh --wait
3. Observe all tasks remain in 'queued' with statusCode=201 forever
4. Follow documented recovery (swarm list --clear, swarm close, swarm create, retry) — issue persists
5. Tried swarm submit (no X-SQL, single URL) — same result

#### Expected Behavior

Swarm workers should pick up queued tasks within seconds and complete page fetch/X-SQL extraction. With --wait, jobs should complete well within the 300s timeout.

#### Actual Behavior

All tasks permanently stuck in 'queued' (statusCode 201, lifecycleState 'queued'). Three attempts across two swarm sessions: 0/6, 0/3, and 0/1 jobs completed. --wait always exhausts full ~300s timeout. Documented recovery procedure does not resolve the issue.

#### Root Cause Analysis

Worker pool appears to never start or never dequeue tasks. The backend is a 4.13.2-SNAPSHOT development build with version mismatch vs installed runtime (4.12.3). Possible causes: (a) worker thread pool not initialized in headless mode, (b) browser context creation failing silently, (c) task queue consumer not registered. The doctor output shows no swarm-specific diagnostics. Investigation needed: check backend logs for swarm worker initialization, verify browser contexts can launch in headless mode, check if the SNAPSHOT build has a regression in swarm task dispatching.

#### Code Pointer

`browser4-rest/src/main/kotlin — likely in swarm session management or task dispatch layer; also check PulsarWebDriver browser context creation in headless mode`

#### AI Suggested Improvement

- Add swarm worker pool health to `doctor` output (worker count, active jobs, queue depth)
- Add a startup self-check that verifies at least one worker can process a test job
- Surface worker-pool errors in `swarm status` output instead of silently staying 'queued'
- Add a --timeout flag to swarm submit/query that controls the --wait polling timeout separately from the HTTP timeout
- Implement early-exit for --wait: if no job transitions out of 'queued' within 30s, fail fast with a diagnostic message instead of burning the full timeout

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: CLI and runtime version mismatch — 4.13.2 CLI vs 4.12.3 installed runtime

**Severity:** High
**Category:** Reliability

#### Reproduction

Run `./b4w.ps1 doctor --verbose` and observe the version warnings:
- CLI version: 4.13.2
- Installed runtime: v4.12.3
- Backend version: 4.13.2-SNAPSHOT
- Warning: 'Backend is a development snapshot — it may be unstable'

#### Expected Behavior

CLI, installed runtime, and backend should all be the same version. The local dev build should use the locally-built JAR that matches the current source tree.

#### Actual Behavior

Three different versions in play: CLI built from source (4.13.2), previously installed runtime bundle (4.12.3), and dev-mode backend (4.13.2-SNAPSHOT). The SNAPSHOT backend may differ from the installed runtime's expectations, potentially causing the swarm worker pool failure.

#### Root Cause Analysis

The dev-mode auto-start launches the locally-built SNAPSHOT JAR, but a previously-installed 4.12.3 runtime bundle also exists on disk. The CLI uses the dev backend but the runtime bundle's configuration or dependencies may conflict. The `browser4-cli install` command would overwrite the installed runtime, but this wasn't run after pulling the latest source.

#### Code Pointer

`cli/browser4-cli/src/ — version detection and runtime path resolution logic`

#### AI Suggested Improvement

- Add a prominent warning at CLI startup when versions mismatch, not just in `doctor`
- In dev mode (running from source), skip or quarantine the installed runtime to prevent conflicts
- Add a `--check-versions` flag that exits non-zero on mismatch for CI use
- Document the expected workflow: after `git pull`, run `browser4-cli install` to sync runtime, or use the dev build exclusively

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: Swarm crawl/submit extremely slow — 126s for 6 localhost pages at depth 0

**Severity:** Medium
**Category:** Reliability

#### Reproduction

Create seed file with 6 localhost URLs. Run: `./b4w.ps1 crawl --seed-file seed-urls.txt --depth 0 --refresh`. Observe ~126 seconds elapsed with 'waiting for first page' messages for the first ~120 seconds, then all 6 pages complete rapidly.

#### Expected Behavior

Fetching 6 pages from localhost (essentially zero network latency) should complete in a few seconds. Each page is small (~15KB HTML).

#### Actual Behavior

Total elapsed: ~126 seconds. The first ~120 seconds showed 'waiting for first page (Xs elapsed, 6 URLs queued)' before any page was fetched, suggesting a long initialization or warm-up period before the crawl engine begins processing URLs.

#### Root Cause Analysis

Crawl has a significant cold-start latency — possibly browser context initialization, JVM warm-up, or a fixed delay before the first URL is dequeued. Once pages start processing, they complete quickly (6 pages in ~6 seconds after the first page appeared). The '4 pages found so far' message appeared at 126s, then '6 pages found' immediately after, confirming a bottleneck in crawl startup rather than per-page fetch time.

#### Code Pointer

`browser4-rest/ — crawl task initialization and first-page dispatch logic`

#### AI Suggested Improvement

- Investigate and reduce crawl cold-start latency (browser context pre-warming, parallel initialization)
- Show a more informative progress message during initialization (e.g., 'Initializing browser context...' vs 'waiting for first page')
- Consider pre-initializing browser contexts when the crawl command is received rather than lazily on first page load
- Add a progress indicator showing pages queued / pages dispatched / pages completed

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: webminer.ps1 not found at repo root — SKILL.md path is incorrect for source-tree users

**Severity:** Medium
**Category:** Documentation

#### Reproduction

From the repo root (/home/vincent/workspace/Browser4-4.13), run `./webminer.ps1 install` as documented in skills/browser4-cli/SKILL.md and skills/scent-miner/SKILL.md. The file does not exist at the repo root.

#### Expected Behavior

The documented command `./webminer.ps1 install` should work from the repository root, or the documentation should specify the correct relative path.

#### Actual Behavior

webminer.ps1 is located at `./skills/scent-miner/scripts/webminer.ps1`, not at the repo root. Users following the SKILL.md instructions from the repo root get 'No such file or directory'. The JAR-based fallback (`java -jar scent-miner.jar`) also fails from repo root because the JAR is at `~/.scent/webminer/lib/scent-miner.jar`.

#### Root Cause Analysis

SKILL.md documentation is written assuming either (a) the user is in the skills/scent-miner/scripts/ directory, or (b) the webminer.ps1 has been installed to PATH or symlinked to the repo root. Neither assumption holds for a first-time user in the repo root. The `browser4-cli install` command does not install webminer.ps1.

#### Code Pointer

`skills/browser4-cli/SKILL.md:301 and skills/scent-miner/SKILL.md:13 — documentation paths`

#### AI Suggested Improvement

- Update SKILL.md to use the full relative path: `./skills/scent-miner/scripts/webminer.ps1 install`
- Or add a symlink/copy step: document that users should run `ln -s skills/scent-miner/scripts/webminer.ps1 .` from repo root
- Add the absolute path to the installed JAR in the fallback command: `java -jar ~/.scent/webminer/lib/scent-miner.jar all <dir>`
- Consider adding webminer.ps1 to the repo root as a convenience symlink checked into git

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 5: Swarm --wait timeout burns full duration with no early-exit on stalled jobs

**Severity:** Medium
**Category:** UX

#### Reproduction

Submit a swarm query with --wait when the worker pool is stalled (or any scenario where no jobs make progress). The --wait flag will poll for the full ~300 seconds even though 0 jobs have ever transitioned out of 'queued'.

#### Expected Behavior

--wait should detect that no progress is being made and fail fast (e.g., if no job leaves 'queued' within 60s, abort with a meaningful error).

#### Actual Behavior

--wait prints '0/N job(s) completed' every 30 seconds for the full ~300 second duration, then exits with 'Timeout after 301s. 0 of N job(s) completed.' The user waits 5 minutes to learn something they could have known after 30-60 seconds.

#### Root Cause Analysis

The --wait polling loop simply counts completed jobs vs total and checks elapsed time against a fixed 300s timeout. It has no stall detection: if completed count stays at 0 for an extended period, it doesn't recognize this as a stall condition.

#### Code Pointer

`cli/browser4-cli/src/ — swarm --wait polling loop`

#### AI Suggested Improvement

- Implement stall detection: if 0 jobs have completed after 60s, exit early with 'No jobs have started processing. The worker pool may be stalled. Check `swarm list` and `doctor` for diagnostics.'
- Add a --wait-timeout flag to let users control the polling duration
- Show per-job status in --wait output (queued/processing/completed counts) not just completed/total
- Consider making the polling interval adaptive: faster (5s) initially, slower (15s) after the first minute

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 6: No swarm diagnostics in doctor output — can't troubleshoot stalled workers

**Severity:** Medium
**Category:** Discoverability

#### Reproduction

When swarm tasks are stuck in 'queued', run `./b4w.ps1 doctor --verbose`. The output shows build info, LLM status, and general backend logs, but nothing about swarm worker pool health, active browser contexts, queue depth, or worker thread status.

#### Expected Behavior

`doctor` or `doctor --verbose` should include swarm-specific diagnostics: number of worker threads, active browser contexts, queue depth, last job processing time, any worker errors.

#### Actual Behavior

No swarm-related information appears in doctor output. Users have no built-in way to diagnose why swarm jobs aren't being processed.

#### Root Cause Analysis

The doctor command was designed before swarm became a core feature, or swarm diagnostics were never integrated. The doctor output covers build info, LLM config, and log tails, but has no category for swarm/job-queue health.

#### Code Pointer

`browser4-rest/ — doctor diagnostics endpoint; cli/browser4-cli/src/ — doctor command output formatting`

#### AI Suggested Improvement

- Add a 'Swarm Health' section to doctor output: worker pool size, active workers, queued job count, last job completion time
- Add backend health endpoint for swarm status that the CLI can query
- Include swarm worker errors in the log tail shown by doctor --verbose
- Add `doctor swarm` subcommand for focused swarm diagnostics

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 7: htmlsnapshot export help text does not clearly state prior capture requirement

**Severity:** Low
**Category:** Discoverability

#### Reproduction

Run `./b4w.ps1 htmlsnapshot export --help` or read the main help. The description says 'Export snapshot HTML from Browser4's page storage to a local file' but does not explicitly state that a prior `htmlsnapshot` (capture) is required. A new user might try `htmlsnapshot export --file out.html` immediately after `goto` without first capturing.

#### Expected Behavior

The help text should clearly state the prerequisite: 'Requires a prior htmlsnapshot capture. Run htmlsnapshot first to store the page HTML, then export it.'

#### Actual Behavior

The help text describes what export does but not when it can be used. Users learn about the capture prerequisite from the SKILL.md decision tree table (§4a), not from the command's own help. If they haven't read the full SKILL.md, they'd encounter a confusing error.

#### Root Cause Analysis

The help text generation in the CLI focuses on describing what each command does, not on prerequisites. The SKILL.md §4a table is comprehensive but users may not read it before trying commands.

#### Code Pointer

`cli/browser4-cli/src/ — help text definitions for htmlsnapshot subcommands`

#### AI Suggested Improvement

- Add a 'Prerequisites' line to command help: 'Prerequisite: htmlsnapshot capture'
- When export is called without a prior capture, include 'Run htmlsnapshot first to capture the page' in the error message
- Add a --capture-and-export convenience flag that does capture+export in one step

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 8: Swarm close causes unexpected navigation in DEFAULT browser session

**Severity:** Low
**Category:** UX

#### Reproduction

1. Open a DEFAULT session and navigate to a page
2. Create and use a swarm session
3. Run `swarm close`
4. Observe the DEFAULT session's browser navigates to a different page (the last page from the swarm or a reconnection)

#### Expected Behavior

Closing a swarm session should not affect other browser sessions. The DEFAULT session should remain on its current page.

#### Actual Behavior

After `swarm close`, the output shows a snapshot from the DEFAULT session at a different URL than where it was left. Example: DEFAULT was on the MockSite home page, but after swarm close, it shows 'Category: Electronics'. This is confusing and could cause data loss if the user was in the middle of a workflow on the DEFAULT session.

#### Root Cause Analysis

When the swarm session closes, it may trigger a session reconnection or browser state refresh that affects the DEFAULT session. The snapshot auto-capture on DEFAULT after swarm close suggests the CLI is inadvertently targeting the DEFAULT session after swarm operations.

#### Code Pointer

`cli/browser4-cli/src/ — swarm close command handler and session state management`

#### AI Suggested Improvement

- Isolate swarm session state from other sessions — closing swarm should not trigger any action on DEFAULT
- If the navigation is caused by auto-snapshot, suppress auto-snapshot on non-swarm sessions during swarm close
- Add a test: verify DEFAULT session URL is unchanged after swarm create → swarm close cycle

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

**Completion Status:** Partially Successful — 3 of 5 acceptance criteria completed fully (AC1, AC3, AC4). AC5 (swarm) partially completed — swarm create works, but the worker pool never processes jobs. AC2 is documentation-only. The core single-page acquisition and WebMiner pipeline flows work correctly. Bulk crawl via --seed-file --depth 0 works but has high cold-start latency.

**Success Rate:** 75% — AC3 (htmlsnapshot export) and AC1 (WebMiner pipeline) worked flawlessly. AC4 (crawl --seed-file) worked but was slow (126s for 6 localhost pages). AC5 (swarm) is blocked by a worker-pool stall. AC2 is meta-documentation.

**Issues Found:** 8

**Major Blockers:** Swarm worker pool is completely non-functional — all jobs stay in 'queued' permanently. This makes the high-throughput acquisition path (AC5) unusable and is the most critical issue found. The version mismatch between CLI (4.13.2), installed runtime (4.12.3), and SNAPSHOT backend may be the root cause.

**Most Confusing Aspects:** 1. The swarm silently fails — jobs appear to submit successfully but never process. No error message, no diagnostic, just eternal 'queued' status.
2. The webminer.ps1 launcher path — SKILL.md says to run it from repo root but it's buried in skills/scent-miner/scripts/.
3. The htmlsnapshot capture-then-export two-step requirement isn't obvious from command help text alone.
4. Swarm close causes the DEFAULT browser session to navigate unexpectedly — side effect that's not documented.

**Most Valuable Improvements:** 1. Fix the swarm worker pool stall — this is a showstopper for the high-throughput acquisition path.
2. Add swarm diagnostics to `doctor` output so users can troubleshoot stalled workers.
3. Implement stall detection in `--wait` polling — don't make users wait 5 minutes for 0/6 jobs.
4. Fix the webminer.ps1 path in documentation or add a symlink to repo root.
5. Reduce crawl cold-start latency (120s of 'waiting for first page' for localhost content).

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

#### Issue 1: Swarm worker pool permanently stalled — all tasks stuck in 'queued'

1. ./b4w.ps1 swarm create --display-mode HEADLESS
2. ./b4w.ps1 swarm query --sql @query.sql --seed-file urls.txt --refresh --wait
3. Observe all tasks remain in 'queued' with statusCode=201 forever
4. Follow documented recovery (swarm list --clear, swarm close, swarm create, retry) — issue persists
5. Tried swarm submit (no X-SQL, single URL) — same result

#### Issue 2: CLI and runtime version mismatch — 4.13.2 CLI vs 4.12.3 installed runtime

Run `./b4w.ps1 doctor --verbose` and observe the version warnings:
- CLI version: 4.13.2
- Installed runtime: v4.12.3
- Backend version: 4.13.2-SNAPSHOT
- Warning: 'Backend is a development snapshot — it may be unstable'

#### Issue 3: Swarm crawl/submit extremely slow — 126s for 6 localhost pages at depth 0

Create seed file with 6 localhost URLs. Run: `./b4w.ps1 crawl --seed-file seed-urls.txt --depth 0 --refresh`. Observe ~126 seconds elapsed with 'waiting for first page' messages for the first ~120 seconds, then all 6 pages complete rapidly.

#### Issue 4: webminer.ps1 not found at repo root — SKILL.md path is incorrect for source-tree users

From the repo root (/home/vincent/workspace/Browser4-4.13), run `./webminer.ps1 install` as documented in skills/browser4-cli/SKILL.md and skills/scent-miner/SKILL.md. The file does not exist at the repo root.

#### Issue 5: Swarm --wait timeout burns full duration with no early-exit on stalled jobs

Submit a swarm query with --wait when the worker pool is stalled (or any scenario where no jobs make progress). The --wait flag will poll for the full ~300 seconds even though 0 jobs have ever transitioned out of 'queued'.

#### Issue 6: No swarm diagnostics in doctor output — can't troubleshoot stalled workers

When swarm tasks are stuck in 'queued', run `./b4w.ps1 doctor --verbose`. The output shows build info, LLM status, and general backend logs, but nothing about swarm worker pool health, active browser contexts, queue depth, or worker thread status.

#### Issue 7: htmlsnapshot export help text does not clearly state prior capture requirement

Run `./b4w.ps1 htmlsnapshot export --help` or read the main help. The description says 'Export snapshot HTML from Browser4's page storage to a local file' but does not explicitly state that a prior `htmlsnapshot` (capture) is required. A new user might try `htmlsnapshot export --file out.html` immediately after `goto` without first capturing.

#### Issue 8: Swarm close causes unexpected navigation in DEFAULT browser session

1. Open a DEFAULT session and navigate to a page
2. Create and use a swarm session
3. Run `swarm close`
4. Observe the DEFAULT session's browser navigates to a different page (the last page from the swarm or a reconnection)

