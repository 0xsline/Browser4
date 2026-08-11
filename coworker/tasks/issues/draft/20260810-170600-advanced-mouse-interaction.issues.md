# Issues: advanced-mouse-interaction

> **Source:** `20260810-170600-advanced-mouse-interaction.full.md` | **Date:** 20260810-170600 | **Mode:** dev

## Scenario Background

### Task

The task was **partially successful**. All individual interaction steps (navigation, hover, drag, double-click, dialog handling, locator generation, text extraction) completed successfully with verification. However, a **phantom dialog state bug** forced a session restart before the final screenshot, meaning the interaction history was lost from the screenshot.

### Completed steps:
1. ✅ Navigation to interactive-5.html
2. ✅ Interactive snapshot (`-i`) — discovered all interactive elements
3. ✅ Hover over "Accessibility Tree" & "DOM Snapshot" tooltips — verified via eval (tooltip text hidden from AXTree)
4. ✅ Hover over "Wireless Headphones" product card — card-detail expansion verified
5. ✅ Drag "High Priority" to reorder — order changed from H→M→L→B to M→L→H→B
6. ✅ Double-click activation zone — status=ACTIVATED, counters incremented
7. ✅ Double-click reset zone — counters reset to 0/0, status=idle
8. ✅ `generate-locator e1272` → `#alertBtn`
9. ✅ `get text "#alertBtn"` → "🔔 Show Alert"
10. ✅ Alert dialog → `dialog-accept` → "[alert] User dismissed the alert dialog."
11. ✅ Confirm dialog → `--auto-dismiss-dialogs` → "[confirm] User clicked OK / accepted."
12. ✅ Prompt dialog → `dialog-accept "Testing prompt input from Browser4 CLI"` → "[prompt] User entered: ..."
13. ⚠️ Final screenshot captured but on fresh page due to phantom dialog bug

### Execution Context

| Step | Command | Result |
|------|---------|--------|
| Verify working dir | `pwd` | `/home/vincent/workspace/Browser4-4.13` ✅ |
| Read SKILL.md | `Read skills/browser4-cli/SKILL.md` | Full reference loaded |
| Help | `./b4w.ps1 help` | Command reference displayed |
| Start MockSite | `./bin/test.ps1 mock-site` | Already running on :18080 |
| Navigate | `./b4w.ps1 goto "http://localhost:18080/generated/interactive-5.html"` | ✅ "Advanced Interaction Playground" |
| Interactive snapshot | `./b4w.ps1 snapshot -i --stdout` | All interactive elements discovered |
| Hover tooltip 1 | `./b4w.ps1 hover e1237` | ✅ |
| Hover tooltip 2 | `./b4w.ps1 hover e1240` | ✅ |
| Verify tooltips | `./b4w.ps1 eval "..."` | visibility=visible/opacity=1 confirmed |
| Hover product card | `./b4w.ps1 hover e1242` ...

(truncated — see full.md for complete trace)

---

## Issues Found (10 issues)

### Issue 1: Phantom dialog state blocks page operations after prompt handling

**Severity:** Critical
**Category:** Reliability

#### Reproduction

1. click '#promptBtn' to trigger a prompt dialog
2. dialog-accept 'some text' to handle it
3. Attempt reload or screenshot — both report 'Page is blocked by a native prompt dialog: Enter a name for this test session:'
4. dialog-accept and dialog-dismiss both report 'No dialog is showing'

#### Expected Behavior

After dialog-accept successfully handles the prompt, the page should be in a normal state where reload and screenshot work.

#### Actual Behavior

The CDP backend reports a phantom dialog ('Enter a name for this test session:') that blocks page operations but cannot be dismissed via dialog-accept or dialog-dismiss. The only recovery is closing the session and starting a new one.

#### Root Cause Analysis

The CDP dialog tracking state became inconsistent with the actual browser state. The backend's Page.javascriptDialogOpening event may have fired for a second dialog (possibly from the session name prompt test page or a different context) but the dialog handler didn't track it. When screenshot/reload check for active dialogs via CDP, they see the stale dialog state, but dialog-accept/dialog-dismiss check a different tracking mechanism that shows no dialog.

#### Code Pointer

`browser4-core/browser4-browser/ — PulsarWebDriver dialog handling and CDP state tracking`

#### AI Suggested Improvement

- Synchronize the dialog state detection between screenshot/reload operations and dialog-accept/dialog-dismiss — both should use the same underlying CDP dialog state
- Add a force-clear mechanism (e.g., `dialog-force-close`) that sends Page.handleJavaScriptDialog with accept:true even when internal tracking says no dialog exists
- Consider adding a CDP event listener for Page.javascriptDialogClosed to clear dialog state on unexpected dialog closures

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 2: Screenshot command silently fails when output path passed as positional argument instead of -o

**Severity:** High
**Category:** UX

#### Reproduction

1. Run `./b4w.ps1 screenshot .test-sessions/output.png` (without -o flag)
2. See success message: [Screenshot](/home/vincent/workspace/Browser4-4.13/.browser4-cli/snapshot/screenshot-TIMESTAMP.png)
3. Check the intended file: it doesn't exist
4. Check the snapshot dir: a corrupted 3-byte file was created

#### Expected Behavior

Either the positional argument should be treated as an output path, or an error should warn the user that the argument was interpreted as a ref selector and no valid element was found.

#### Actual Behavior

The path was silently interpreted as a CSS selector/ref. Since no element matched, the screenshot appears to succeed but produces a 3-byte corrupted file in the snapshot directory. The user's intended output file is never created.

#### Root Cause Analysis

The screenshot command accepts an optional [ref] positional argument for targeting an element. When a path-like string is passed positionally, it's treated as a selector with no warning. The -o/--filename flag must be used for file output paths. The lack of error when no element matches the 'selector' means the user gets a misleading success message.

#### Code Pointer

`cli/browser4-cli/src/ — screenshot command argument parsing`

#### AI Suggested Improvement

- If the positional argument looks like a file path (contains / or .png/.jpg extension), emit a warning suggesting -o flag
- When the ref selector matches no element, emit an error rather than producing a broken file
- Consider supporting screenshot <path> as a shorthand for screenshot -o <path> since this is the most common use case

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 3: htmlsnapshot command times out with HTTP 504 error

**Severity:** High
**Category:** Reliability

#### Reproduction

1. Navigate to any page
2. Run `./b4w.ps1 htmlsnapshot`
3. Command times out after 30-60s with 'Error: HTTP request timed out'

#### Expected Behavior

htmlsnapshot should capture the page HTML within a reasonable time (<10s).

#### Actual Behavior

The command times out with an HTTP timeout error. The endpoint http://localhost:8182/mcp/call-tool does not respond within the 60s timeout.

#### Root Cause Analysis

The html_snapshot_capture tool call to the MCP endpoint may be hanging due to a large page, resource loading issues, or a backend thread deadlock. The interactive-5.html page is not particularly large, suggesting a backend issue rather than page complexity.

#### Code Pointer

`browser4-rest/ — MCPToolController or html_snapshot_capture handler`

#### AI Suggested Improvement

- Add timeout configuration for htmlsnapshot capture specifically
- Investigate whether page resource loading is blocking the HTML capture
- Consider streaming or chunked capture for large pages
- Add better error messages that distinguish between 'page too large' and 'backend hang'

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 4: AXTree snapshot hides dynamic text content — counters and tooltips invisible

**Severity:** High
**Category:** Product

#### Reproduction

1. After double-clicking the activation zone, run `snapshot -v 0-1 --stdout`
2. Search for counter values or 'ACTIVATED' — they don't appear
3. Only eval reveals the actual values

#### Expected Behavior

The accessibility tree snapshot should expose dynamic text content changes so users can verify interaction results without resorting to eval.

#### Actual Behavior

Counter values (singleClickCount, dblClickCount) and status text appear as empty generic elements in the snapshot. Tooltip text (inside .tooltip-text spans) doesn't appear in the AXTree even when visible. Users must use eval for verification.

#### Root Cause Analysis

The accessibility tree captures semantic roles and names from the browser's AXTree. Pure text nodes and elements styled with visibility:hidden/opacity:0 may not appear or may appear without their text content in the AXTree. Counters are likely `<span>` elements without ARIA roles that map to generic nodes without accessible names.

#### Code Pointer

`browser4-core/browser4-browser/ — PulsarWebDriver AXTree snapshot generation`

#### AI Suggested Improvement

- Enhance the snapshot to include text content for generic/inline elements when they have text children
- Add ARIA live region support so dynamic updates appear in snapshots
- Consider adding a `--include-text` flag that enriches the AXTree with text node values
- Document this limitation clearly in the snapshot command help and SKILL.md

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 5: dialog-accept produces no output on success — silent completion is confusing

**Severity:** Medium
**Category:** UX

#### Reproduction

1. Trigger a dialog (e.g., click '#alertBtn')
2. Run `./b4w.ps1 dialog-accept`
3. Command completes with no output at all

#### Expected Behavior

dialog-accept should produce a confirmation message like 'Dialog accepted' or 'Alert dismissed'.

#### Actual Behavior

The command exits with status 0 and produces zero output. Users cannot tell if the dialog was handled or if nothing happened.

#### Root Cause Analysis

The dialog-accept command does not print a success message to stdout. While error cases produce output, the success path is silent.

#### Code Pointer

`cli/browser4-cli/src/ — dialog-accept command handler`

#### AI Suggested Improvement

- Print a confirmation message on success: '✓ Dialog accepted' or '✓ Alert dismissed'
- Include the dialog type and message in the output for verification
- This matches the pattern used by other commands like hover ('✓ Hovered e1237')

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 6: --auto-dismiss-dialogs still causes long timeout on click

**Severity:** Medium
**Category:** UX

#### Reproduction

1. Run `./b4w.ps1 click '#confirmBtn' --auto-dismiss-dialogs`
2. Command takes 30-60s to complete

#### Expected Behavior

--auto-dismiss-dialogs should handle the dialog immediately without a noticeable delay.

#### Actual Behavior

The command still times out (30-60s) before completing, making it indistinguishable from a regular click that's blocked by a dialog. Users may kill the command thinking it hung.

#### Root Cause Analysis

The auto-dismiss likely waits for the click to complete (which blocks on the dialog) before dismissing the dialog, rather than dismissing it as soon as the dialog appears. The dialog check may happen on a polling interval rather than being event-driven.

#### Code Pointer

`browser4-rest/ — MCPToolController click handler with auto-dismiss logic`

#### AI Suggested Improvement

- Make auto-dismiss event-driven: listen for Page.javascriptDialogOpening and dismiss immediately
- Reduce the polling interval for dialog detection when --auto-dismiss-dialogs is active
- Add progress output during the wait: 'Dialog detected, auto-dismissing...'

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 7: Tooltip container elements appear as generic in interactive snapshot — not distinguishable from text

**Severity:** Medium
**Category:** Discoverability

#### Reproduction

1. Take an interactive snapshot with `snapshot -i --stdout`
2. Look for tooltip terms — they appear as 'generic "Accessibility Tree"' with no indication they trigger tooltips

#### Expected Behavior

Tooltip containers should have a distinguishable role or annotation in the snapshot indicating they are hoverable/interactive.

#### Actual Behavior

The tooltip-container spans appear as plain 'generic' elements in the AXTree. The tooltip text is embedded but hidden. Users cannot tell from the snapshot which elements have hover-triggered tooltips.

#### Root Cause Analysis

The tooltip containers are `<span>` elements without ARIA roles or interactive semantics, so they map to 'generic' in the accessibility tree. The hidden tooltip-text spans are hidden from the AXTree.

#### Code Pointer

`browser4-core/browser4-browser/ — AXTree node role/name extraction`

#### AI Suggested Improvement

- Consider annotating elements with CSS :hover rules or event listeners in the snapshot output
- Add a 'has-tooltip' annotation when elements have title attributes or data-tooltip attributes
- Document in SKILL.md that hover-triggered content requires eval for verification

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 8: Click-on-dialog times out with confusing error — dialog workflow not obvious to new users

**Severity:** Medium
**Category:** Documentation

#### Reproduction

1. As a first-time user, run `./b4w.ps1 click '#alertBtn'`
2. Command hangs for 60s then times out
3. No indication that a dialog is blocking the page

#### Expected Behavior

The timeout message should indicate that a dialog is blocking and suggest using dialog-accept or --auto-dismiss-dialogs.

#### Actual Behavior

The command simply times out with no dialog-specific guidance. A new user wouldn't know to use dialog-accept as a separate command unless they had read the Dialog Handling section of SKILL.md carefully.

#### Root Cause Analysis

The click command's timeout error doesn't inspect whether a dialog is blocking. The error message is generic.

#### Code Pointer

`cli/browser4-cli/src/ — click command error handling / timeout message`

#### AI Suggested Improvement

- Detect when a dialog is blocking during click timeout and include specific guidance in the error message
- Suggest: 'A browser dialog may be blocking the page. Try: dialog-accept or dialog-dismiss'
- Add a 'Common workflows' tip about dialog handling in the click --help output

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 9: Product card hover expansion detail not visible in interactive snapshot (-i)

**Severity:** Low
**Category:** Product

#### Reproduction

1. Hover over product card (ref for '📦 Wireless Headphones' container)
2. Take interactive snapshot (-i)
3. The expanded detail text ('Features: Noise cancelling...') doesn't appear

#### Expected Behavior

The expanded card detail text should appear in the snapshot after hovering.

#### Actual Behavior

The card detail text only appears in the full (non-interactive) snapshot. The -i flag might filter out the detail paragraph as non-interactive.

#### Root Cause Analysis

The -i flag filters to interactive elements only (buttons, links, inputs, etc.), which excludes the expanded detail paragraph since it's a static text element.

#### Code Pointer

`cli/browser4-cli/src/snapshot.rs — interactive mode filtering logic`

#### AI Suggested Improvement

- Document that -i mode excludes expanded/hovered content that isn't directly interactive
- Consider including elements that became visible due to interaction in the -i output
- Suggest using -v 0 instead of -i when verifying hover-triggered content

#### Human Review

- [ ] **ACCEPT** — issue confirmed valid; suggested improvement is correct
- [ ] **ACCEPT with improvements** — issue valid but fix needs refinement (add details in Notes)
- [ ] **DEFER** — issue acknowledged but intentionally deferred (add rationale in Notes)
- [ ] **WONTFIX** — issue acknowledged but will not be fixed (add rationale in Notes)
- [ ] **REJECT** — issue invalid, not a problem, or already addressed
- [ ] **DUPLICATE** — issue duplicates another existing issue (reference in Notes)
- **Notes:**

---

### Issue 10: Priority list drag reorder places item above target instead of below

**Severity:** Low
**Category:** Product

#### Reproduction

1. Drag e1254 (High Priority, original position 1) onto e1257 (Backlog, position 4)
2. Result order: Medium → Low → High → Backlog
3. High Priority ends up at position 3, not position 4 (bottom)

#### Expected Behavior

Dragging an item onto the last item should place it after that item (at the bottom).

#### Actual Behavior

High Priority was placed above Backlog (position 3 of 4), not below it (position 4 of 4). This is insertion-before-target behavior rather than insertion-after-target.

#### Root Cause Analysis

The drag implementation uses insertion-before-target semantics (insertBefore) rather than checking if the target is the last element to use appendChild. This is standard HTML5 drag behavior but may not match user expectations for 'drag to bottom'.

#### Code Pointer

`browser4-core/browser4-browser/ — PulsarWebDriver drag implementation`

#### AI Suggested Improvement

- Document the drag insertion semantics (before vs after target) in the command help
- Consider adding a --position flag (before|after|replace) for drag operations
- The current behavior is consistent with HTML5 drag-and-drop spec but the help should clarify this

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

**Completion Status:** Partially Successful — all interaction steps completed and verified, but a phantom dialog bug forced session restart before the final screenshot, losing the interaction history from the captured image.

**Success Rate:** 85% — 12 of 13 steps fully succeeded with verification. The final screenshot step was technically completed but on a fresh page state.

**Issues Found:** 10

**Major Blockers:** Phantom dialog state (Critical) prevented screenshot after prompt dialog handling — only recovery was session restart. htmlsnapshot HTTP timeout prevented CSS-based content extraction.

**Most Confusing Aspects:** 1. Dialog workflow is unintuitive — click times out, then you need a separate dialog-accept command; 2. AXTree snapshots don't show dynamic content like counter values or tooltip text, forcing reliance on eval; 3. Screenshot -o vs positional argument distinction is easy to miss; 4. Silent success on dialog-accept gives no feedback that the dialog was handled; 5. The --auto-dismiss-dialogs flag causes the same long timeout, making it unclear if it worked.

**Most Valuable Improvements:** 1. Fix the phantom dialog state bug — this is a session-breaking issue; 2. Add confirmation output to dialog-accept/dialog-dismiss; 3. Improve click timeout error messages to detect and report dialog blocking; 4. Enrich AXTree snapshots with dynamic text content; 5. Warn when screenshot path is misinterpreted as a selector.

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

#### Issue 1: Phantom dialog state blocks page operations after prompt handling

1. click '#promptBtn' to trigger a prompt dialog
2. dialog-accept 'some text' to handle it
3. Attempt reload or screenshot — both report 'Page is blocked by a native prompt dialog: Enter a name for this test session:'
4. dialog-accept and dialog-dismiss both report 'No dialog is showing'

#### Issue 2: Screenshot command silently fails when output path passed as positional argument instead of -o

1. Run `./b4w.ps1 screenshot .test-sessions/output.png` (without -o flag)
2. See success message: [Screenshot](/home/vincent/workspace/Browser4-4.13/.browser4-cli/snapshot/screenshot-TIMESTAMP.png)
3. Check the intended file: it doesn't exist
4. Check the snapshot dir: a corrupted 3-byte file was created

#### Issue 3: htmlsnapshot command times out with HTTP 504 error

1. Navigate to any page
2. Run `./b4w.ps1 htmlsnapshot`
3. Command times out after 30-60s with 'Error: HTTP request timed out'

#### Issue 4: AXTree snapshot hides dynamic text content — counters and tooltips invisible

1. After double-clicking the activation zone, run `snapshot -v 0-1 --stdout`
2. Search for counter values or 'ACTIVATED' — they don't appear
3. Only eval reveals the actual values

#### Issue 5: dialog-accept produces no output on success — silent completion is confusing

1. Trigger a dialog (e.g., click '#alertBtn')
2. Run `./b4w.ps1 dialog-accept`
3. Command completes with no output at all

#### Issue 6: --auto-dismiss-dialogs still causes long timeout on click

1. Run `./b4w.ps1 click '#confirmBtn' --auto-dismiss-dialogs`
2. Command takes 30-60s to complete

#### Issue 7: Tooltip container elements appear as generic in interactive snapshot — not distinguishable from text

1. Take an interactive snapshot with `snapshot -i --stdout`
2. Look for tooltip terms — they appear as 'generic "Accessibility Tree"' with no indication they trigger tooltips

#### Issue 8: Click-on-dialog times out with confusing error — dialog workflow not obvious to new users

1. As a first-time user, run `./b4w.ps1 click '#alertBtn'`
2. Command hangs for 60s then times out
3. No indication that a dialog is blocking the page

#### Issue 9: Product card hover expansion detail not visible in interactive snapshot (-i)

1. Hover over product card (ref for '📦 Wireless Headphones' container)
2. Take interactive snapshot (-i)
3. The expanded detail text ('Features: Noise cancelling...') doesn't appear

#### Issue 10: Priority list drag reorder places item above target instead of below

1. Drag e1254 (High Priority, original position 1) onto e1257 (Backlog, position 4)
2. Result order: Medium → Low → High → Backlog
3. High Priority ends up at position 3, not position 4 (bottom)

