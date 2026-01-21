# Phase: DISCOVER

**Goal:** Find ALL routes and ALL interactive elements in the app.

---

## Step 0: PRE-FLIGHT Checks (MANDATORY)

**Before ANY discovery, verify the environment is ready.**

### Critical Checks (HALT on failure)

**Using Playwright:**

```javascript
// Pre-flight check script
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();

  // Capture console errors during load
  const consoleErrors = [];
  page.on('console', msg => {
    if (msg.type() === 'error') consoleErrors.push(msg.text());
  });

  try {
    // 1. Server responds
    const response = await page.goto('{appUrl}', { timeout: 30000 });
    if (!response.ok()) {
      console.log('HALT: Server returned', response.status());
      process.exit(1);
    }

    // 2. Check for blocking errors
    await page.waitForTimeout(3000); // Let errors accumulate
    if (consoleErrors.some(e => e.includes('fatal') || e.includes('crash'))) {
      console.log('HALT: Critical JS errors:', consoleErrors);
      process.exit(1);
    }

    console.log('PRE-FLIGHT PASSED');
    console.log('Console errors:', consoleErrors.length);
  } catch (e) {
    console.log('HALT:', e.message);
    process.exit(1);
  }
})();
```

**Run with:** `node test-manifest/preflight.js`

### Warning Checks (LOG and continue)

```
1. Console warnings on load:
   → Log count and continue

2. Slow initial load (>5s):
   → Log warning and continue

3. Missing assets (404s):
   → Log and continue
```

### PRE-FLIGHT Output

```
=== PRE-FLIGHT CHECK ===

Server: {appUrl}
  → Status: RESPONDING | FAILED
  → Load time: {seconds}s

Console on load:
  → Errors: {count} [CRITICAL if >0 blocking]
  → Warnings: {count}

Authentication:
  → Required: YES | NO
  → Available: YES | NO | N/A

PRE-FLIGHT: PASSED | FAILED

[If FAILED]
Cannot proceed. Fix environment issues:
- {issue 1}
- {issue 2}
```

**If PRE-FLIGHT fails, STOP. Do not proceed to discovery.**

---

## Step 1: Setup

### Create Directory Structure

```bash
mkdir -p test-manifest/screenshots/routes test-manifest/screenshots/elements test-manifest/reports
```

### Initialize Playwright Browser

```javascript
// test-manifest/discover.js
const { chromium } = require('playwright');

const APP_URL = process.env.APP_URL || 'http://localhost:3000';

(async () => {
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();

  // Console capture for entire session
  const consoleMessages = [];
  page.on('console', msg => consoleMessages.push({
    type: msg.type(),
    text: msg.text(),
    url: page.url()
  }));

  await page.goto(APP_URL);
  console.log('Browser initialized at:', APP_URL);

  // ... discovery continues
})();
```

**Run with:** `APP_URL=http://localhost:5001 node test-manifest/discover.js`

---

## Step 2: Discover Routes

### Option A: Scan Source Code (Preferred)

Look for router definitions:

```
Grep for: BrowserRouter|Routes|Route|createBrowserRouter|path:
In: src/
```

Extract routes from:
- React Router: `<Route path="/customers" />`
- Next.js: `pages/` or `app/` directory structure
- Vue Router: `routes: [{ path: '/customers' }]`

### Option B: Crawl Navigation (Fallback)

If source not accessible:

```
1. Navigate to home page
2. Read accessibility tree
3. Find all links with href
4. Record unique paths
5. For each discovered path, navigate and repeat
```

### Route Recording Format

For each route found:

```json
{
  "path": "/customers",
  "routeId": "customers",
  "requiresAuth": true,
  "component": "CustomersPage"
}
```

---

## Step 3: Discover Elements Per Route (MANDATORY - NO SHORTCUTS)

**⚠️ CRITICAL: You MUST visit EVERY route discovered, not just list them.**

**Anti-laziness check:** If you discovered 18 routes, you MUST navigate to all 18 routes and find elements on each one. Finding elements on 2 routes and saying "similar for others" is NOT acceptable.

### 3.0 Route Coverage Requirement

```
FORBIDDEN:
- "Similar patterns exist on other routes"
- "Skipping these routes as they follow the same pattern"
- "Only testing representative routes"

REQUIRED:
- Navigate to EVERY route
- Find elements on EVERY route
- Screenshot EVERY route at 3 breakpoints
- Record elements from EVERY route in state
```

For EACH route discovered:

### 3.1 Navigate to Route (EVERY SINGLE ONE)

**Using Playwright:**

```javascript
// For each route in discoveredRoutes
for (const route of discoveredRoutes) {
  await page.goto(`${APP_URL}${route.path}`);
  await page.waitForLoadState('networkidle');

  // Take screenshots at 3 breakpoints
  for (const viewport of [
    { width: 375, height: 812, name: 'mobile' },
    { width: 768, height: 1024, name: 'tablet' },
    { width: 1024, height: 768, name: 'desktop' }
  ]) {
    await page.setViewportSize(viewport);
    await page.screenshot({
      path: `test-manifest/screenshots/routes/${route.routeId}-${viewport.name}.png`
    });
  }

  // Find all interactive elements
  const elements = await findInteractiveElements(page, route);
  allElements.push(...elements);
}
```

### 3.2 Find ALL Interactive Elements

**Using Playwright to enumerate ALL elements:**

```javascript
async function findInteractiveElements(page, route) {
  const elements = [];

  // Get ALL buttons
  const buttons = await page.locator('button, [role="button"], input[type="submit"]').all();
  for (const btn of buttons) {
    const text = await btn.textContent();
    const ariaLabel = await btn.getAttribute('aria-label');
    elements.push({
      elementId: `${route.routeId}-btn-${slugify(text || ariaLabel)}`,
      route: route.path,
      type: 'button',
      text: text || ariaLabel,
      selector: await btn.evaluate(el => getUniqueSelector(el))
    });
  }

  // Get ALL links
  const links = await page.locator('a[href]').all();
  // ... similar for links

  // Get ALL inputs
  const inputs = await page.locator('input, select, textarea').all();
  // ... similar for inputs

  return elements;
}
```

### 3.3 Record Each Element

For EVERY button, link, input found:

```json
{
  "elementId": "customers-add-btn",
  "route": "/customers",
  "type": "button",
  "text": "Add Customer",
  "ref": "ref_5",
  "expectedBehavior": "opens-modal",
  "verificationMethod": "check-for-modal"
}
```

### 3.4 Infer Expected Behavior

Analyze button text/aria-label to determine expected behavior:

| Text Pattern | Expected Behavior | Verification |
|-------------|-------------------|--------------|
| Add, Create, New, + | Opens form/modal | Modal visible in tree |
| Edit, Update, Modify | Opens edit form | Form with existing data |
| Delete, Remove, Trash | Confirms then deletes | Item removed from list |
| Save, Submit, Confirm | Persists data | Success message or redirect |
| Cancel, Close, X | Dismisses without save | Modal/form closes |
| Search, Filter | Filters data | List content changes |
| View, Details, Open | Shows detail page | Navigation or modal |
| Export, Download | Triggers download | File download starts |
| Refresh, Reload | Reloads data | Page/section updates |

### 3.5 Handle Authentication

If route redirects to login:
- Mark route as `requiresAuth: true`
- Add to `authRequired` list
- User will need to authenticate before testing

### 3.6 Identify Critical Flows

While discovering elements, identify **smoke-level user flows** - sequences of elements that represent core user journeys:

| Flow Type | Example Sequence | Why Critical |
|-----------|------------------|--------------|
| Authentication | login-submit → dashboard-visible | Can users access the app? |
| CRUD Create | add-btn → form-fill → save-btn → item-in-list | Can users create data? |
| CRUD Delete | delete-btn → confirm-btn → item-removed | Can users remove data? |
| Navigation | nav-link → page-loads → content-visible | Do routes work? |

**Record critical flows:**

```json
{
  "flowId": "create-customer",
  "description": "User can create a new customer",
  "route": "/customers",
  "steps": [
    {"elementId": "customers-add-btn", "action": "click"},
    {"elementId": "customer-form", "action": "fill", "data": "VAL_Test_{timestamp}"},
    {"elementId": "customer-save-btn", "action": "click"},
    {"verify": "item-appears-in-list"}
  ]
}
```

**Minimum critical flows to identify:**
- [ ] Login flow (if auth exists)
- [ ] One CREATE flow per major module
- [ ] One DELETE flow per major module
- [ ] Main navigation paths

---

## Step 4: Build State File

After discovering all routes and elements:

```json
{
  "session": {
    "id": "val-{timestamp}",
    "startedAt": "{ISO}",
    "lastUpdatedAt": "{ISO}",
    "status": "in_progress",
    "currentPhase": "test",
    "appUrl": "{url}",
    "contextResets": 0
  },
  "preflight": {
    "passed": true,
    "serverStatus": "responding",
    "loadTime": 1.2,
    "consoleErrors": 0,
    "consoleWarnings": 2,
    "authRequired": false
  },
  "discovery": {
    "completedAt": "{ISO}",
    "routes": [/* all routes */],
    "elements": [/* all elements */],
    "criticalFlows": [/* identified flows */]
  },
  "testing": {
    "currentIndex": 0,
    "results": {},
    "pending": [/* all elementIds */],
    "failed": [],
    "quarantined": []
  },
  "summary": {
    "totalElements": {count},
    "totalFlows": {count},
    "tested": 0,
    "passed": 0,
    "failed": 0,
    "skipped": 0,
    "flowsPassed": 0,
    "flowsFailed": 0,
    "uiIssues": 0,
    "consoleErrors": 0
  }
}
```

Write to: `test-manifest/validation-state.json`

---

## Step 5: Output Summary

```
=== DISCOVERY COMPLETE ===

Routes found: {count}
  - Public: {count}
  - Protected: {count}

Interactive elements: {count}
  - Buttons: {count}
  - Links: {count}
  - Inputs: {count}
  - Other: {count}

State saved to: test-manifest/validation-state.json

Proceeding to TEST phase...
```

---

## Step 6: Transition to TEST Phase

After discovery completes:

1. Update state `currentPhase: "test"`
2. Save state file
3. Read `phases/TEST.md`
4. Execute TEST phase

---

## Anti-Laziness: Discovery Validation

Before marking discovery complete, verify:

- [ ] ALL routes visited (not just obvious ones)
- [ ] EVERY page had accessibility tree read
- [ ] ALL buttons recorded (compare visible count vs recorded)
- [ ] Expected behaviors assigned to ALL elements
- [ ] State file written with complete data

**If a page has visible buttons but 0 elements recorded, you missed them. Go back and check.**

---

## Discovery Output Files

| File | Purpose |
|------|---------|
| `validation-state.json` | Main state with all discovery data |

Discovery data is stored IN the state file, not separate files.
