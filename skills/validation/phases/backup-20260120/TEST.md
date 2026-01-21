# Phase: TEST

**Goal:** Test EVERY element with REAL verification - not just clicking, but confirming the action happened.

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

REQUIRED: Call read_console_messages and report:
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

**Coverage Check:**

Before generating report, verify:
```
const totalDiscovered = state.discovery.elements.length;
const totalTested = Object.keys(state.testing.results).length;

if (totalTested < totalDiscovered) {
  console.log(`INCOMPLETE: Only tested ${totalTested}/${totalDiscovered}`);
  console.log('Missing:', state.testing.pending);
  // DO NOT proceed to report until ALL are tested
}
```

---

## MANDATORY: Verification Report Template

**For EVERY element tested, you MUST produce this report:**

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
║   - Screenshot: {path}                                        ║
║   - DOM change: {before/after description}                    ║
║   - Count change: {X → Y items}                               ║
║   - Console: {errors: 0, warnings: 1}                         ║
║                                                               ║
║ VERDICT: [PASS] | [FAIL] | [SKIP]                            ║
║ RATIONALE: {Why this verdict - specific evidence}             ║
╚══════════════════════════════════════════════════════════════╝
```

**Without this report, the test is NOT complete.**

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

### Why VAL_ prefix?

1. **Identifiable** - Easy to spot test data in production
2. **Searchable** - Can find all test artifacts with one search
3. **Cleanable** - Blind cleanup sweep deletes all VAL_* items

### Form Fill Strategy

When filling forms, use predictable test data:

| Field Type | Test Value |
|------------|------------|
| Name/Title | `VAL_Test_{timestamp}` |
| Email | `val_test_{timestamp}@example.com` |
| Phone | `555-VAL-{last4}` |
| Number | `99999` (unlikely real value) |
| Date | Today's date |
| Select | First non-empty option |
| Checkbox | Check it |
| Text area | `VAL_Test description - {timestamp}` |

---

## Done Criteria Per Element Type

**An element test is DONE only when these criteria are met:**

| Element Type | Done Criteria |
|--------------|---------------|
| **Button (action)** | Clicked AND expected outcome verified AND state changed |
| **Button (modal)** | Modal opened AND contains expected content AND can close |
| **Button (delete)** | Confirmation shown AND item removed from list AND count decreased |
| **Button (submit)** | Form submitted AND success feedback AND data persisted |
| **Link (nav)** | URL changed AND target page loaded AND content visible |
| **Link (external)** | New tab opened (don't follow) |
| **Input (text)** | Can type AND value persists AND validation works |
| **Input (select)** | Can select option AND value persists |
| **Input (checkbox)** | Can toggle AND state persists |
| **Form (overall)** | All fields fillable AND submits AND data appears in list |

**If ANY criterion is not met, the test is NOT done.**

---

## Negative Testing (Error States)

**Don't just test happy paths. Also test:**

### Required Field Validation
```
1. Leave required field empty
2. Submit form
3. VERIFY: Error message shown
4. VERIFY: Form NOT submitted
```

### Invalid Input Validation
```
1. Enter invalid data (e.g., "abc" in number field)
2. Submit form
3. VERIFY: Validation error shown
4. VERIFY: Form NOT submitted
```

### Empty State Testing
```
1. Navigate to list page
2. If list is empty:
   - VERIFY: Empty state message shown
   - VERIFY: "Add" button still visible
3. If list has items:
   - Delete all VAL_* test items
   - VERIFY: Empty state appears OR list is empty
```

### Error Recovery
```
1. If an action fails:
   - VERIFY: Error message shown
   - VERIFY: User can retry
   - VERIFY: App doesn't crash
```

---

## Critical Flows Testing

After individual element tests, test identified critical flows:

```
For each flow in state.discovery.criticalFlows:
  1. Navigate to starting route
  2. Execute steps in sequence
  3. Verify each step succeeds
  4. Verify final outcome

Flow Result:
  - All steps passed → Flow PASS
  - Any step failed → Flow FAIL (note which step)
```

### Flow Verification Report

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
║ FLOW VERDICT: [PASS] | [FAIL at Step N]                      ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Test Execution Flow (MANDATORY FOR ALL ELEMENTS)

**⚠️ You MUST iterate through EVERY element. No shortcuts.**

### For Each Element in `testing.pending`:

```
1. Read current state
2. Get element details from state
3. Navigate to element's route (Playwright)
4. Screenshot at 3 breakpoints (ANALYZE each - save to disk)
5. Check console errors BEFORE action
6. Find element using Playwright selector
7. Click/interact with element
8. Check console errors AFTER action (report DELTA)
9. VERIFY expected outcome (not just "clicked")
10. Screenshot AFTER action (save to disk)
11. Record result with evidence
12. Update state (remove from pending, add to results)
13. Write state file IMMEDIATELY
14. REPEAT for next element until pending is EMPTY
```

### Progress Tracking

```
After each element:
  Tested: {N}/{total} ({percentage}%)
  Passed: {passed}
  Failed: {failed}
  Remaining: {pending.length}
```

**NEVER stop before `pending` is empty unless context overflow.**

---

## Step 1: Load Test Queue

Read state and get next pending element:

```javascript
const state = readState();
const elementId = state.testing.pending[0];
const element = state.discovery.elements.find(e => e.elementId === elementId);
```

---

## Step 2: Navigate to Route (PLAYWRIGHT)

```javascript
await page.goto(`${APP_URL}${element.route}`);
await page.waitForLoadState('networkidle');
// Additional wait to ensure dynamic content loads
await page.waitForTimeout(2000);
```

**Verify navigation succeeded:**

```javascript
const currentUrl = page.url();
if (!currentUrl.includes(element.route)) {
  console.log(`Navigation failed: expected ${element.route}, got ${currentUrl}`);
  return { status: 'skip', reason: 'Navigation failed' };
}
```

---

## Step 3: Screenshot at Breakpoints (with ANALYSIS) - PLAYWRIGHT

**⚠️ CRITICAL: Screenshots MUST be saved to disk, not just captured to memory.**

### Screenshot Function (Playwright)

```javascript
async function takeBreakpointScreenshots(page, elementId, route) {
  const breakpoints = [
    { width: 375, height: 812, name: 'mobile' },
    { width: 768, height: 1024, name: 'tablet' },
    { width: 1024, height: 768, name: 'desktop' }
  ];

  const screenshots = {};

  for (const bp of breakpoints) {
    await page.setViewportSize({ width: bp.width, height: bp.height });
    await page.waitForTimeout(500); // Let layout settle

    const path = `test-manifest/screenshots/elements/${elementId}-${bp.name}.png`;
    await page.screenshot({ path, fullPage: false });
    screenshots[bp.name] = path;

    console.log(`Saved: ${path}`);
  }

  return screenshots;
}
```

### 3.1 Mobile (375px)

```javascript
await page.setViewportSize({ width: 375, height: 812 });
await page.screenshot({
  path: `test-manifest/screenshots/elements/${elementId}-mobile.png`
});
```

**ANALYZE the screenshot (MANDATORY):**
```
Mobile (375px) Analysis:
- Navigation: [hamburger menu present? / items visible?]
- Main content: [readable? / buttons accessible?]
- Layout: [single column? / cards stacked?]
- Issues: [text overflow? / buttons cut off? / touch targets too small?]
- Screenshot path: test-manifest/screenshots/elements/{elementId}-mobile.png
```

### 3.2 Tablet (768px)

```javascript
await page.setViewportSize({ width: 768, height: 1024 });
await page.screenshot({
  path: `test-manifest/screenshots/elements/${elementId}-tablet.png`
});
```

**ANALYZE the screenshot:**
```
Tablet (768px) Analysis:
- Layout: [2-column? / sidebar visible?]
- Tables: [horizontal scroll needed? / columns visible?]
- Forms: [fields properly sized?]
- Issues: [awkward spacing? / elements overlapping?]
- Screenshot path: test-manifest/screenshots/elements/{elementId}-tablet.png
```

### 3.3 Desktop (1024px)

```javascript
await page.setViewportSize({ width: 1024, height: 768 });
await page.screenshot({
  path: `test-manifest/screenshots/elements/${elementId}-desktop.png`
});
```

**ANALYZE the screenshot:**
```
Desktop (1024px) Analysis:
- Full layout: [all sections visible?]
- Navigation: [expanded menu?]
- Data display: [tables readable? / charts rendered?]
- Issues: [excessive whitespace? / misalignment?]
- Screenshot path: test-manifest/screenshots/elements/{elementId}-desktop.png
```

### Screenshot Verification

**After screenshots are taken, verify files exist:**

```bash
ls -la test-manifest/screenshots/elements/{elementId}-*.png
# Should show 3 files: mobile, tablet, desktop
```

**If files don't exist, the screenshots failed. Do NOT continue without real screenshot files.**

---

## Step 4: Check Console Errors (AFTER EVERY ACTION)

**⚠️ CRITICAL: Console MUST be checked after EVERY interaction, not just once per route.**

**Using Playwright:**

```javascript
// Console capture setup (at browser init)
const consoleErrors = [];
const consoleWarnings = [];

page.on('console', msg => {
  if (msg.type() === 'error') consoleErrors.push({
    text: msg.text(),
    url: page.url(),
    timestamp: new Date().toISOString()
  });
  if (msg.type() === 'warning') consoleWarnings.push({
    text: msg.text(),
    url: page.url()
  });
});

// Function to get errors since last check
function getNewConsoleErrors(lastIndex) {
  const newErrors = consoleErrors.slice(lastIndex);
  return {
    errors: newErrors,
    count: newErrors.length,
    lastIndex: consoleErrors.length
  };
}
```

### Console Check Timing

```
FORBIDDEN:
- Check console once at page load, then ignore
- "No new errors" without actually checking

REQUIRED:
- Record console state BEFORE action
- Record console state AFTER action
- Report DELTA (new errors caused by action)
```

**Example verification:**

```javascript
// Before clicking
const errorsBefore = consoleErrors.length;

// Click action
await page.click(selector);
await page.waitForTimeout(500); // Let errors propagate

// After clicking
const errorsAfter = consoleErrors.length;
const newErrors = consoleErrors.slice(errorsBefore);

console.log(`Console check: ${newErrors.length} new errors`);
if (newErrors.length > 0) {
  console.log('New errors:', newErrors.map(e => e.text));
}
```

**Record actual output:**
```
Console Check (after clicking "Add Customer"):
- Errors before: {count}
- Errors after: {count}
- NEW errors caused by action: {count}
- Messages: [list actual error text]
- Source: [file and line if available]
```

---

## Step 5: Find and Click Element (PLAYWRIGHT)

### 5.1 Locate Target Element

**Using Playwright selectors:**

```javascript
// By text content
const button = page.getByRole('button', { name: element.text });

// By selector (if stored during discovery)
const buttonBySelector = page.locator(element.selector);

// By test ID (if available)
const buttonByTestId = page.getByTestId(element.testId);
```

### 5.2 Verify Element is Visible and Clickable

```javascript
// Wait for element to be visible
await button.waitFor({ state: 'visible', timeout: 5000 });

// Check if enabled
const isEnabled = await button.isEnabled();
if (!isEnabled) {
  console.log('Element disabled, cannot click');
  return { status: 'skip', reason: 'Element disabled' };
}
```

### 5.3 Click Element

```javascript
// Record console errors before click
const errorsBefore = consoleErrors.length;

// Click the element
await button.click();

// Wait for response (network or DOM change)
await page.waitForTimeout(1000);

// Record console errors after click
const errorsAfter = consoleErrors.length;
const newErrors = consoleErrors.slice(errorsBefore);

console.log('Click result:', {
  element: element.elementId,
  newConsoleErrors: newErrors.length
});
```

### 5.4 Screenshot After Click

```javascript
// Take screenshot to document the result
await page.screenshot({
  path: `test-manifest/screenshots/elements/${element.elementId}-after-click.png`
});
```

---

## Step 6: VERIFY Expected Outcome

This is the CRITICAL step. Based on `element.expectedBehavior`:

### If "opens-modal"

```
mcp__claude-in-chrome__read_page
```

**VERIFY:** Look for dialog/modal in accessibility tree
```
Verification:
- Modal visible: YES/NO
- Modal title: "{title}"
- Form fields present: [list fields]
- Close button available: YES/NO
```

### If "deletes-item"

Before clicking, count items:
```
Item count before: {X}
```

After clicking and confirming:
```
mcp__claude-in-chrome__read_page
```

**VERIFY:** Count items again
```
Verification:
- Item count before: {X}
- Item count after: {Y}
- Specific item "{name}" removed: YES/NO
- Success message shown: YES/NO
```

### If "creates-item"

After form submission:
```
mcp__claude-in-chrome__navigate url="{listPage}"
mcp__claude-in-chrome__read_page
```

**VERIFY:** New item appears
```
Verification:
- Navigated to list
- Search for "Test Item {timestamp}"
- Item found: YES/NO
- Data correct: YES/NO
```

### If "updates-data"

After saving:
```
Verification:
- Success message: "{message}"
- Data persisted (refresh and check): YES/NO
- Changed field shows new value: YES/NO
```

### If "navigates"

```
Verification:
- URL changed to: {newUrl}
- Expected page loaded: YES/NO
- Content visible: [describe]
```

---

## Step 7: Record Result

```javascript
const result = {
  elementId: element.elementId,
  testedAt: new Date().toISOString(),
  route: element.route,
  screenshots: {
    mobile: "screenshots/elements/{id}-375.png",
    tablet: "screenshots/elements/{id}-768.png",
    desktop: "screenshots/elements/{id}-1024.png"
  },
  screenshotAnalysis: {
    mobile: "{analysis}",
    tablet: "{analysis}",
    desktop: "{analysis}"
  },
  consoleErrors: [],
  clickResult: "success|failed",
  verification: {
    expected: element.expectedBehavior,
    method: element.verificationMethod,
    passed: true|false,
    evidence: "{description of what was checked}"
  },
  uiIssues: [],
  status: "pass|fail"
};
```

---

## Step 8: Update State (IMMEDIATELY)

```javascript
// Read current state
const state = readState();

// Add result
state.testing.results[element.elementId] = result;

// Remove from pending
state.testing.pending = state.testing.pending.filter(id => id !== element.elementId);

// Update summary
state.summary.tested++;
if (result.status === 'pass') state.summary.passed++;
if (result.status === 'fail') {
  state.summary.failed++;
  state.testing.failed.push(element.elementId);
}
state.summary.uiIssues += result.uiIssues.length;
state.summary.consoleErrors += result.consoleErrors.length;

// Update timestamp
state.session.lastUpdatedAt = new Date().toISOString();

// Calculate completion
const total = state.summary.totalElements;
const done = state.summary.tested;
state.summary.completionPercentage = Math.round((done / total) * 100);

// WRITE IMMEDIATELY
writeState(state);
```

---

## Step 9: Retry on Failure

If a test fails:

1. Log the failure reason
2. Wait 2 seconds
3. Retry ONCE
4. If still fails, mark as failed and continue

```
Retry attempt for {elementId}:
- Original failure: {reason}
- Retry result: {pass|fail}
```

---

## Step 10: Handle Modal Cleanup

If test opened a modal, close it before continuing:

```
1. Look for close button (X, Cancel, Close)
2. Click it
3. Verify modal dismissed
4. If stuck, press Escape key
5. If still stuck, refresh page
```

---

## Step 11: Context Check

After every 5 elements tested:

Check if conversation is getting long. If yes:

```
=== VALIDATION PAUSED ===

Progress: {tested}/{total} elements ({percentage}%)
Passed: {passed}
Failed: {failed}

State saved. To continue: /validate
```

Then STOP.

---

## Step 12: Transition to REPORT

When `testing.pending` is empty:

1. Update `currentPhase: "report"`
2. Save state
3. Read `phases/REPORT.md`
4. Execute REPORT phase

---

## Result Status Criteria

| Status | Criteria |
|--------|----------|
| PASS | Clicked successfully AND verification passed AND no UI issues |
| FAIL | Click failed OR verification failed OR critical UI issue |
| SKIP | Element not found OR route inaccessible |

---

## Common Issues and Handling

### Element Not Found

```
Element "{text}" not found in accessibility tree.
- Screenshot current page state
- Mark as SKIP with reason
- Continue to next element
```

### Modal Won't Close

```
1. Try clicking X button
2. Try clicking Cancel
3. Try pressing Escape
4. If all fail, navigate away and back
```

### Session Expired

```
Detected redirect to login.
- Pause testing
- Notify user: "Authentication expired"
- Wait for user to re-login
- Continue from current element
```

---

## Step 13: Cleanup Phase (Before Report)

**After all tests complete, clean up test artifacts:**

### 13.1 Find All VAL_* Items

For each module/route tested:

```
1. Navigate to list page
2. Search/filter for "VAL_"
3. Record all matching items
```

### 13.2 Delete Test Artifacts

```
For each VAL_* item found:
  1. Click delete button
  2. Confirm deletion
  3. Verify item removed
  4. Log: "Cleaned: {item name}"
```

### 13.3 Cleanup Report

```
=== CLEANUP REPORT ===

Items found: {count}
Items deleted: {count}
Items failed to delete: {count}

Cleanup status: COMPLETE | PARTIAL | FAILED

[If PARTIAL or FAILED]
Remaining items:
- {item 1} - {reason}
- {item 2} - {reason}

Note: Manual cleanup may be required.
```

### 13.4 Update State

```javascript
state.cleanup = {
  completed: true,
  itemsFound: count,
  itemsDeleted: count,
  itemsFailed: count,
  failedItems: [/* list */]
};
```

**Cleanup is BEST EFFORT.** If some items can't be deleted, log them and continue to REPORT phase.
