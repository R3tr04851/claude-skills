# Phase: TEST

**Goal:** Test EVERY element using MCP browser tools with REAL verification - not just clicking, but confirming the action happened.

---

## MCP Tools Reference

| Action | MCP Tool |
|--------|----------|
| Navigate to URL | `mcp__claude-in-chrome__navigate url="{url}"` |
| Get page snapshot | `mcp__claude-in-chrome__read_page` |
| Click element | `mcp__claude-in-chrome__click ref="{ref}"` |
| Fill form field | `mcp__claude-in-chrome__form_input ref="{ref}" value="{value}"` |
| Resize viewport | `mcp__claude-in-chrome__resize_window width={w} height={h}` |
| Take screenshot | `mcp__claude-in-chrome__computer action="screenshot"` |
| Read console | `mcp__claude-in-chrome__read_console_messages` |

---

## CRITICAL: Anti-Laziness Rules

### Rule 1: Screenshots MUST be ANALYZED

```
FORBIDDEN: "Screenshot taken"
FORBIDDEN: "Looks good"
FORBIDDEN: "No issues"

REQUIRED: Actual analysis describing:
- What is visible in the screenshot
- Layout assessment (aligned? overflow? cut-off?)
- Responsive behavior (does content reflow correctly?)
- If no issues: WHY it's correct (e.g., "All 5 buttons visible and properly spaced")
```

### Rule 2: Button Actions MUST be VERIFIED

```
FORBIDDEN: "Clicked successfully"
FORBIDDEN: "Button responded"

REQUIRED: Verify the EXPECTED OUTCOME:
- Delete button → Check item is GONE from list
- Create button → Check new item APPEARS in list
- Modal button → Check modal is VISIBLE in accessibility tree
- Save button → Check success message OR data persisted
- Refresh button → Check timestamp changed OR content updated
```

### Rule 3: Console MUST be Actually READ

```
FORBIDDEN: "No errors"
FORBIDDEN: "Console clean"

REQUIRED: Call mcp__claude-in-chrome__read_console_messages and report:
- Exact count of errors
- Exact count of warnings
- Actual message text if any
```

### Rule 4: ALL Elements MUST be Tested

```
FORBIDDEN:
- "Testing representative sample of buttons"
- "Similar buttons on other pages would behave the same"
- "Skipping duplicate functionality"
- Testing 2 out of 50 elements and calling it done

REQUIRED:
- Every element in discovery.elements MUST be tested
- Progress: {tested}/{total} must reach {total}/{total}
- Each element gets its own VERIFICATION REPORT
- State file tracks completion of EVERY element
```

---

## Step 0: Resume Check

### Load State

Read `test-manifest/validation-state.json`

Check:
- `session.currentPhase` should be "test"
- `testing.currentIndex` tells you where to resume
- `testing.pending` has elements not yet tested
- `testing.results` has completed test results

### If Resuming

```
=== RESUMING TEST PHASE ===

Session: {session.id}
Context resets: {contextResets + 1}
Previously tested: {Object.keys(results).length}
Remaining: {pending.length}

Continuing from element: {pending[currentIndex]}
```

Update `session.contextResets++` and save state.

---

## Step 1: Test Each Element (MANDATORY - ALL ELEMENTS)

**CRITICAL: You MUST test EVERY element in `testing.pending`, not a sample.**

For EACH element in `testing.pending`:

### 1.1 Navigate to Element's Route

```
mcp__claude-in-chrome__navigate url="{appUrl}{element.route}"
```

Wait for page load.

### 1.2 Get Fresh Page Snapshot

```
mcp__claude-in-chrome__read_page
```

Find the element by its `ref` or by matching text/role in the accessibility tree.

**If element not found:**
- Page may have changed since discovery
- Record: `status: "skip"`, `reason: "element-not-found"`
- Continue to next element

### 1.3 Check Console Before Action

```
mcp__claude-in-chrome__read_console_messages
```

Record baseline console state (count of errors/warnings).

### 1.4 Take Pre-Action Screenshot

```
mcp__claude-in-chrome__computer action="screenshot"
```

**ANALYZE the screenshot (Claude sees the image directly):**
- Current page state
- Element visible and clickable?
- Any existing modals/overlays blocking?

### 1.5 Perform the Action

Based on element type:

#### For Buttons
```
mcp__claude-in-chrome__click ref="{element.ref}"
```

#### For Form Inputs
```
mcp__claude-in-chrome__form_input ref="{element.ref}" value="VAL_Test_{timestamp}"
```

Use test data prefix `VAL_` so cleanup is easy.

#### For Links
```
mcp__claude-in-chrome__click ref="{element.ref}"
```

### 1.6 Wait and Get Post-Action State

Wait 500-1000ms for UI to update, then:

```
mcp__claude-in-chrome__read_page
```

### 1.7 Check Console After Action

```
mcp__claude-in-chrome__read_console_messages
```

Compare to baseline. Report DELTA (new errors caused by action).

### 1.8 Take Post-Action Screenshot

```
mcp__claude-in-chrome__computer action="screenshot"
```

**ANALYZE the screenshot:**
- What changed?
- Did expected behavior occur?
- Any error messages visible?

### 1.9 VERIFY Outcome (CRITICAL - NOT OPTIONAL)

**This is the most important step. "Clicked successfully" is NOT verification.**

Based on `element.expectedBehavior`:

| Expected Behavior | How to VERIFY with MCP |
|-------------------|------------------------|
| `opens-modal` | `read_page` → Modal element NOW in accessibility tree |
| `closes-modal` | `read_page` → Modal element NO LONGER in tree |
| `navigates` | URL changed, `read_page` shows new page content |
| `creates-item` | Navigate to list, `read_page` → new item in list |
| `deletes-item` | `read_page` → item NO LONGER in list |
| `updates-item` | `read_page` → item data changed |
| `shows-message` | `read_page` → success/error message in tree |
| `triggers-download` | Download started (check for download dialog) |
| `filters-data` | `read_page` → list content changed |
| `submits-form` | Form cleared OR redirect OR success message |

### 1.10 Generate Verification Report

**For EVERY element tested, produce this report:**

```
╔══════════════════════════════════════════════════════════════╗
║                    VERIFICATION REPORT                        ║
╠══════════════════════════════════════════════════════════════╣
║ Element:     {elementId}                                      ║
║ Route:       {route}                                          ║
║ Type:        {button|link|input|form}                         ║
╠══════════════════════════════════════════════════════════════╣
║ ACTION TAKEN:                                                 ║
║   {Specific action - e.g., "Clicked 'Add Customer' button"}   ║
║                                                               ║
║ EXPECTED OUTCOME:                                             ║
║   {What should happen - e.g., "Modal should open with form"}  ║
║                                                               ║
║ OBSERVED OUTCOME:                                             ║
║   {What actually happened - e.g., "Modal opened, form has     ║
║    5 fields: Name, Email, Phone, Address, Notes"}             ║
║                                                               ║
║ EVIDENCE:                                                     ║
║   - Screenshot: analyzed - {description}                      ║
║   - DOM change: {before/after description}                    ║
║   - Count change: {X → Y items}                               ║
║   - Console: {errors: 0, warnings: 1}                         ║
║                                                               ║
║ VERDICT: [PASS] | [FAIL] | [SKIP]                            ║
║ RATIONALE: {Why this verdict - specific evidence}             ║
╚══════════════════════════════════════════════════════════════╝
```

### 1.11 Record Result

```json
{
  "elementId": "customers-add-btn",
  "route": "/customers",
  "status": "pass|fail|skip",
  "testedAt": "{ISO}",
  "action": "click",
  "verification": {
    "expected": "opens-modal",
    "verified": true,
    "evidence": "Modal 'Add Customer' appeared with form fields: name, email, phone"
  },
  "consoleErrors": [],
  "uiIssues": [],
  "screenshots": {
    "before": "analyzed - button visible at top right",
    "after": "analyzed - modal opened with 3 form fields"
  }
}
```

### 1.12 Update and Save State (AFTER EVERY ELEMENT)

```javascript
state.testing.results[elementId] = result;
state.testing.currentIndex++;
state.testing.pending = state.testing.pending.filter(id => id !== elementId);
if (result.status === 'fail') state.testing.failed.push(elementId);
state.summary.tested++;
state.summary[result.status === 'pass' ? 'passed' : result.status === 'fail' ? 'failed' : 'skipped']++;
state.session.lastUpdatedAt = new Date().toISOString();
```

**Write state to file immediately.** This allows resume if context resets.

### 1.13 Reset to Clean State

After testing element, especially if it opened a modal:

1. Look for close button (X or Cancel) in accessibility tree
2. `mcp__claude-in-chrome__click ref="{closeButton.ref}"`
3. `mcp__claude-in-chrome__read_page` to verify modal closed
4. If stuck, navigate away: `mcp__claude-in-chrome__navigate url="{appUrl}{route}"`

---

## Step 2: Responsive Testing (4 Breakpoints)

For each route, verify layout at 4 breakpoints:

| Breakpoint | Width | Height | Name |
|------------|-------|--------|------|
| Mobile | 375 | 812 | mobile |
| Tablet | 768 | 1024 | tablet |
| Laptop | 1024 | 768 | laptop |
| Desktop | 1440 | 900 | desktop |

### 2.1 For Each Route + Breakpoint

```
1. mcp__claude-in-chrome__navigate url="{appUrl}{route}"
2. mcp__claude-in-chrome__resize_window width={width} height={height}
3. Wait 500ms for layout to settle
4. mcp__claude-in-chrome__computer action="screenshot"
5. ANALYZE screenshot immediately (Claude sees the image)
6. mcp__claude-in-chrome__read_console_messages
```

### 2.2 Screenshot Analysis (MANDATORY)

**"Looks good" is NOT valid analysis.**

For each screenshot, check and report:

```
{route} @ {breakpoint}px:

Navigation:
  - Visible: YES | HAMBURGER | HIDDEN
  - All items accessible: YES | SOME_HIDDEN | NO
  - Touch targets: ADEQUATE | TOO_SMALL

Content:
  - Text readable: YES | TRUNCATED | OVERLAPPING
  - Images: SIZED_CORRECTLY | OVERFLOWING | MISSING
  - Tables: SCROLLABLE | SQUISHED | BROKEN

Layout:
  - Structure: CORRECT | OVERLAPPING | BROKEN
  - Spacing: CONSISTENT | CRAMPED | EXCESSIVE
  - Alignment: CORRECT | MISALIGNED

Issues Found:
  - {specific issue 1}
  - {specific issue 2}
```

### 2.3 Record UI Issues

```json
{
  "route": "/customers",
  "breakpoint": 375,
  "issues": [
    {
      "type": "overflow",
      "description": "Table overflows horizontally, no scroll",
      "severity": "medium"
    }
  ]
}
```

---

## Step 3: Test Critical Flows

After individual element tests, test multi-step flows from `discovery.criticalFlows`.

For EACH flow:

### 3.1 Navigate to Flow Starting Point

```
mcp__claude-in-chrome__navigate url="{appUrl}{flow.route}"
```

### 3.2 Execute Each Step

For each step in `flow.steps`:

```
1. mcp__claude-in-chrome__read_page - find element by elementId
2. mcp__claude-in-chrome__click ref="{ref}" OR form_input
3. Wait for UI update
4. mcp__claude-in-chrome__read_page - verify intermediate state
5. mcp__claude-in-chrome__computer action="screenshot" - document each step
```

### 3.3 Generate Flow Report

```
╔══════════════════════════════════════════════════════════════╗
║                   CRITICAL FLOW REPORT                        ║
╠══════════════════════════════════════════════════════════════╣
║ Flow:        {flowId}                                         ║
║ Description: {description}                                    ║
╠══════════════════════════════════════════════════════════════╣
║ Step 1: {action} → {result} [PASS/FAIL]                      ║
║ Step 2: {action} → {result} [PASS/FAIL]                      ║
║ Step 3: {action} → {result} [PASS/FAIL]                      ║
╠══════════════════════════════════════════════════════════════╣
║ FINAL VERIFICATION:                                           ║
║   Expected: {flow goal}                                       ║
║   Observed: {actual result}                                   ║
║                                                               ║
║ FLOW VERDICT: [PASS] | [FAIL at Step N]                      ║
╚══════════════════════════════════════════════════════════════╝
```

### 3.4 Cleanup Test Data

If flow created data (like a test customer):
1. Delete the test item (find delete button, click it)
2. Verify cleanup worked via `read_page`

---

## Step 4: Console Error Audit

### 4.1 Aggregate All Console Errors

From all testing, compile unique console errors:

```json
{
  "consoleErrors": [
    {
      "message": "Failed to fetch /api/customers",
      "routes": ["/customers", "/dashboard"],
      "count": 5,
      "severity": "error"
    }
  ]
}
```

### 4.2 Categorize Errors

| Category | Examples | Severity |
|----------|----------|----------|
| API Errors | Failed fetch, 404, 500 | High |
| React Errors | Unhandled rejection, key warnings | Medium |
| Deprecation | Console warnings about deprecated APIs | Low |
| Third-party | Analytics, tracking script errors | Info |

---

## Step 5: Update Final State

After all testing complete:

```javascript
state.session.currentPhase = 'report';
state.testing.completedAt = new Date().toISOString();
state.summary = {
  totalElements: discovery.elements.length,
  totalFlows: discovery.criticalFlows.length,
  tested: Object.keys(results).length,
  passed: passCount,
  failed: failCount,
  skipped: skipCount,
  flowsPassed: flowPassCount,
  flowsFailed: flowFailCount,
  uiIssues: allUIIssues.length,
  consoleErrors: uniqueConsoleErrors.length
};
```

Save state file.

---

## Step 6: Output Summary

```
=== TEST PHASE COMPLETE ===

Session: {session.id}
Duration: {startedAt} to {completedAt}
Context resets: {contextResets}

Element Tests:
  - Total: {totalElements}
  - Passed: {passed} ({passRate}%)
  - Failed: {failed}
  - Skipped: {skipped}

Flow Tests:
  - Total: {totalFlows}
  - Passed: {flowsPassed}
  - Failed: {flowsFailed}

UI Issues: {uiIssues}
Console Errors: {consoleErrors}

Failed Elements:
{for each failed}
  - {elementId} on {route}: {expected} but {evidence}
{end}

Proceeding to REPORT phase...
```

---

## Step 7: Transition to REPORT Phase

1. Update state `currentPhase: "report"`
2. Save state file
3. Read `phases/REPORT.md`
4. Execute REPORT phase

---

## Test Data Strategy (VAL_ Prefix)

### All test data MUST use the VAL_ prefix

```
Pattern: VAL_Test_{timestamp}

Examples:
- Customer name:  VAL_Test_1705753042
- Email:          val_test_1705753042@example.com
- Phone:          555-VAL-0001
- Description:    VAL_Test created by validation skill
```

### Form Fill Strategy

When filling forms, use predictable test data:

| Field Type | Test Value |
|------------|------------|
| Name/Title | `VAL_Test_{timestamp}` |
| Email | `val_test_{timestamp}@example.com` |
| Phone | `555-VAL-{last4}` |
| Number | `99999` |
| Date | Today's date |
| Select | First non-empty option |
| Checkbox | Check it |
| Text area | `VAL_Test description - {timestamp}` |

---

## Error Handling

### Element Not Found
```json
{
  "status": "skip",
  "reason": "element-not-found",
  "evidence": "Element ref_5 not present in current accessibility tree"
}
```

### Click Failed
```json
{
  "status": "fail",
  "reason": "action-failed",
  "evidence": "Click on ref_5 returned error: element not interactable"
}
```

### Unexpected Behavior
```json
{
  "status": "fail",
  "reason": "verification-failed",
  "expected": "opens-modal",
  "evidence": "No modal appeared after click. Page state unchanged."
}
```

### Console Error on Action
```json
{
  "status": "fail",
  "reason": "console-error",
  "evidence": "Action triggered: TypeError: Cannot read property 'id' of undefined"
}
```

---

## Quarantine Rules

If an element fails 3 times across context resets:

1. Move to `testing.quarantined`
2. Don't retry again
3. Mark as `status: "quarantined"`
4. Include in report as known flaky

---

## Anti-Laziness: Testing Validation

Before marking testing complete:

- [ ] EVERY element in pending list was tested (check counts match)
- [ ] EVERY test has verification evidence (not just "clicked")
- [ ] ALL 4 breakpoints were screenshotted per route
- [ ] ALL critical flows were tested end-to-end
- [ ] Console was checked on EVERY route with `read_console_messages`
- [ ] Screenshots were ANALYZED, not just taken
- [ ] State file saved after EVERY element (check timestamp)
- [ ] Failed tests have specific failure evidence

**If summary shows 0 failures but you saw errors, something is wrong. Re-check.**
