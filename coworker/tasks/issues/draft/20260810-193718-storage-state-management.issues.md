# Issues: storage-state-management

> **Source:** `20260810-193718-storage-state-management.full.md` | **Date:** 20260810-193718 | **Mode:** dev

## Scenario Background

### Task

All 17 steps completed successfully. Every cookie, localStorage, sessionStorage, and state management operation worked as documented. Results:

- **Cookies:** Set session_id (with httpOnly, secure flags) and theme (with sameSite=Lax, 1-week expiry). Verified, filtered by domain, retrieved individually, deleted selectively, and cleared in bulk.
- **localStorage:** Stored JSON value `{"lang":"en","tz":"UTC"}`, verified via list and get, deleted individual key, cleared all.
- **sessionStorage:** Stored, listed, retrieved, deleted, and cleared — full lifecycle.
- **State persistence:** Saved browser state (cookies + localStorage) to `browser_state.json`, cleared everything, restored from file, confirmed cookie was restored, cleaned up.

### Execution Context

| Step | Command | Outcome |
|------|---------|---------|
| Prep | `./b4w.ps1 help`, read SKILL.md + storage-state.md | Learned all commands |
| 1 | `./b4w.ps1 goto "http://localhost:18080/generated/interactive-1.html"` | Already on page (existing session) |
| 2 | `./b4w.ps1 cookie-set session_id abc123 --domain localhost --path / --httpOnly --secure` | Cookie set |
| 3 | `./b4w.ps1 cookie-set theme dark --sameSite Lax --expires <timestamp>` | Cookie set with computed timestamp |
| 4 | `./b4w.ps1 cookie-list` | Both cookies shown in JSON |
| 5 | `./b4w.ps1 cookie-list --domain localhost` | Both cookies filtered correctly |
| 6 | `./b4w.ps1 cookie-get theme` | Returned `"value": "dark"` |
| 7 | `./b4w.ps1 cookie-delete session_id` | Deleted |
| 8 | `./b4w.ps1 cookie-list` | Only theme remai...

(truncated — see full.md for complete trace)

---

## Issues Found (4 issues)

### Issue 1: cookie-get returns JSON object but localstorage-get returns raw value — inconsistent output format

**Severity:** Medium
**Category:** UX

#### Reproduction

1. ./b4w.ps1 cookie-get theme
2. ./b4w.ps1 localstorage-get user_prefs
Compare the output formats.

#### Expected Behavior

Both 'get' commands should return consistently-formatted output, either both as raw values or both as structured objects with metadata.

#### Actual Behavior

cookie-get returns a JSON object with name, value, domain, path, expires, httpOnly, secure, sameSite fields. localstorage-get returns only the raw value string (e.g., '{"lang":"en","tz":"UTC"}'). sessionstorage-get also returns raw value. This means scripts parsing 'get' output need to handle two different formats depending on storage type.

#### Root Cause Analysis

Cookies have structured metadata (domain, path, expires, flags) that naturally maps to a JSON object, whereas localStorage/sessionStorage are simple key-value stores. However, the CLI could still wrap localStorage values in a consistent envelope like {"key": "...", "value": "..."} for uniform machine parsing.

#### AI Suggested Improvement

- Wrap localstorage-get and sessionstorage-get output in a JSON envelope: {"key": "...", "value": "..."} to match the structured pattern of cookie-get
- Or add a --value-only flag to cookie-get for users who only want the value, making both modes available
- Document the output format of each 'get' command in their --help text so users know what to expect before running

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [x] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: cookie-set does not echo back applied properties — no way to verify flags took effect without a separate cookie-get

**Severity:** Low
**Category:** UX

#### Reproduction

./b4w.ps1 cookie-set session_id abc123 --domain localhost --path / --httpOnly --secure

#### Expected Behavior

Output confirms all flags that were applied, e.g., 'Cookie set: session_id (domain=localhost, path=/, httpOnly, secure)'

#### Actual Behavior

Output is just 'Cookie set: session_id' — no indication of which flags were applied. If a flag was silently ignored due to a typo (e.g., --httpOnly misspelled), the user would not notice without running cookie-get.

#### Root Cause Analysis

The cookie-set success message is generated from only the cookie name, without including the resolved cookie attributes. The backend likely sets the cookie successfully but the CLI output doesn't reflect the full cookie object.

#### AI Suggested Improvement

- Echo back key cookie properties in the success message: domain, path, httpOnly, secure, sameSite, expires
- This provides immediate feedback and helps catch typos in flag names (e.g., --httponly vs --httpOnly)
- Same pattern would benefit localstorage-set and sessionstorage-set (echo the key and value truncated if long)

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: Dev wrappers (./b4w.ps1, ./b4w.sh) not mentioned in --help output for in-repo usage

**Severity:** Low
**Category:** Discoverability

#### Reproduction

Run ./b4w.ps1 help and search for 'b4w' or './b4w.ps1'

#### Expected Behavior

Help output mentions the dev wrapper scripts for users working inside the repository.

#### Actual Behavior

Help output only shows 'Dev mode: cargo run --manifest-path cli/browser4-cli/Cargo.toml -- <command>' at the very bottom. It does not mention ./b4w.ps1 or ./b4w.sh. A new contributor would need to read CLAUDE.md or SKILL.md to discover these wrappers.

#### Root Cause Analysis

The help text is generated from the CLI binary's clap definitions, which describe the installed binary (browser4-cli). The dev wrappers are repo-level convenience scripts that live outside the binary's help system.

#### AI Suggested Improvement

- Add a 'Dev wrappers' note in the Quick Start section of --help mentioning ./b4w.ps1 and ./b4w.sh
- Or add a post-build step that injects repo-specific usage notes into the help text when built from source
- The SKILL.md already documents this; linking to it from --help would help

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: cookie-list --domain uses exact match only — no substring or pattern matching

**Severity:** Low
**Category:** Product

#### Reproduction

./b4w.ps1 cookie-list --domain localhost (works with exact match). Try --domain local (partial match) — returns empty array even if localhost cookies exist.

#### Expected Behavior

Documentation clearly states it's an exact match, but a --domain-pattern or wildcard support would be more flexible for users managing cookies across subdomains.

#### Actual Behavior

Exact domain match only. This is documented correctly in --help ('Only include cookies with the exact domain'), so it's not a bug — but it limits usefulness for subdomain scenarios.

#### Root Cause Analysis

The --domain flag performs an exact string match against the cookie's domain field. While correct for the documented behavior, many real-world scenarios involve subdomain patterns (e.g., .example.com matching app.example.com).

#### AI Suggested Improvement

- Consider adding a --domain-suffix flag for subdomain matching (e.g., --domain-suffix example.com matches app.example.com, api.example.com)
- Or support glob patterns in --domain (e.g., --domain '*.example.com')
- At minimum, document the exact-match behavior prominently with an example showing why a partial match returns nothing

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

**Completion Status:** Successful — all 17 task steps completed without errors or workarounds. Every cookie, localStorage, sessionStorage, and state management operation worked as documented.

**Success Rate:** 100% — every command executed correctly on the first attempt

**Issues Found:** 4

**Most Confusing Aspects:** 1. The output format difference between cookie-get (JSON object with metadata) and localstorage-get (raw value string) is the most jarring inconsistency — a user learning one 'get' pattern would be surprised by the other. 2. A first-time user in the repo might try 'browser4-cli' directly (not installed) before discovering ./b4w.ps1 — the SKILL.md covers this but the --help output doesn't.

**Most Valuable Improvements:** 1. Unify 'get' output formats across storage types (cookie-get vs localstorage-get vs sessionstorage-get). 2. Echo back applied properties in cookie-set success messages for immediate verification. 3. Add dev wrapper mention to --help output for in-repo users.

**Usability Rating:** 8/10

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

#### Issue 1: cookie-get returns JSON object but localstorage-get returns raw value — inconsistent output format

1. ./b4w.ps1 cookie-get theme
2. ./b4w.ps1 localstorage-get user_prefs
Compare the output formats.

#### Issue 2: cookie-set does not echo back applied properties — no way to verify flags took effect without a separate cookie-get

./b4w.ps1 cookie-set session_id abc123 --domain localhost --path / --httpOnly --secure

#### Issue 3: Dev wrappers (./b4w.ps1, ./b4w.sh) not mentioned in --help output for in-repo usage

Run ./b4w.ps1 help and search for 'b4w' or './b4w.ps1'

#### Issue 4: cookie-list --domain uses exact match only — no substring or pattern matching

./b4w.ps1 cookie-list --domain localhost (works with exact match). Try --domain local (partial match) — returns empty array even if localhost cookies exist.

