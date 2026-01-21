# Phase: DISCOVER

**Goal:** Find ALL routes and ALL interactive elements in the app using MCP browser tools.

---

## MCP Tools Reference

| Action | MCP Tool |
|--------|----------|
| Navigate to URL | `mcp__claude-in-chrome__navigate url="{url}"` |
| Get page snapshot | `mcp__claude-in-chrome__read_page` |
| Resize viewport | `mcp__claude-in-chrome__resize_window width={w} height={h}` |
| Take screenshot | `mcp__claude-in-chrome__computer action="screenshot"` |
| Read console | `mcp__claude-in-chrome__read_console_messages` |
| Click element | `mcp__claude-in-chrome__click ref="{ref}"` |

---

## Step 0: PRE-FLIGHT Checks (MANDATORY)

**Before ANY discovery, verify the environment is ready.**

### 0.1 Navigate to App

```
mcp__claude-in-chrome__navigate url="{appUrl}"
```

Wait for page to load fully.

### 0.2 Check Console for Blocking Errors

```
mcp__claude-in-chrome__read_console_messages pattern="error"
```

**Analyze output:**
- If FATAL errors or crashes → HALT
- If normal console warnings → LOG and continue

### 0.3 Get Initial Page Snapshot

```
mcp__claude-in-chrome__read_page
```

**Verify:**
- Page rendered content (not blank)
- No login redirect (unless expected)
- Main UI elements visible in accessibility tree

### PRE-FLIGHT Output

```
=== PRE-FLIGHT CHECK ===

Server: {appUrl}
  → Status: RESPONDING | FAILED
  → Page loaded: YES | NO

Console on load:
  → Errors: {count} [CRITICAL if blocking]
  → Warnings: {count}

Authentication:
  → Required: YES | NO
  → Login page detected: YES | NO

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

If source not accessible, crawl via MCP:

```
1. mcp__claude-in-chrome__read_page
2. Find all links with href in accessibility tree
3. Record unique paths
4. For each discovered path:
   - mcp__claude-in-chrome__navigate url="{appUrl}{path}"
   - mcp__claude-in-chrome__read_page
   - Find more links
   - Repeat until no new routes
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

**CRITICAL: You MUST visit EVERY route discovered, not just list them.**

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
- Screenshot EVERY route at 4 breakpoints
- Record elements from EVERY route in state
```

For EACH route discovered:

### 3.1 Navigate to Route

```
mcp__claude-in-chrome__navigate url="{appUrl}{route.path}"
```

Wait for page to load.

### 3.2 Screenshot at 4 Breakpoints

**CRITICAL: All 4 breakpoints are required.**

| Breakpoint | Width | Height | Name |
|------------|-------|--------|------|
| Mobile | 375 | 812 | mobile |
| Tablet | 768 | 1024 | tablet |
| Laptop | 1024 | 768 | laptop |
| Desktop | 1440 | 900 | desktop |

For EACH breakpoint:

```
1. mcp__claude-in-chrome__resize_window width={width} height={height}
2. Wait 500ms for layout to settle
3. mcp__claude-in-chrome__computer action="screenshot"
4. ANALYZE the screenshot immediately (Claude sees the image)
5. Note the screenshot for the report
```

**Screenshot Analysis (MANDATORY):**

```
{breakpoint}px Analysis:
- Navigation: [visible? hamburger? items accessible?]
- Main content: [readable? properly sized? cut off?]
- Layout: [single column? grid? overflow?]
- Issues: [text truncated? buttons hidden? touch targets too small?]
```

### 3.3 Get Accessibility Tree

```
mcp__claude-in-chrome__read_page
```

### 3.4 Find ALL Interactive Elements

From the accessibility tree, identify:

| Element Type | How to Find |
|--------------|-------------|
| Buttons | `button`, `[role="button"]`, elements with onClick |
| Links | `link`, `a[href]` |
| Inputs | `textbox`, `input`, `combobox`, `checkbox` |
| Forms | `form`, fieldsets with submit buttons |

For EVERY interactive element found:

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

### 3.5 Infer Expected Behavior

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

### 3.6 Check Console Errors on Each Route

```
mcp__claude-in-chrome__read_console_messages
```

Record any errors found on this route.

### 3.7 Handle Authentication

If route redirects to login:
- Mark route as `requiresAuth: true`
- Add to `authRequired` list
- User will need to authenticate before testing

### 3.8 Identify Critical Flows

While discovering elements, identify **smoke-level user flows**:

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
    "criticalFlows": [/* identified flows */],
    "routeScreenshots": {
      "/customers": {
        "mobile": "analyzed - no issues",
        "tablet": "analyzed - no issues",
        "laptop": "analyzed - no issues",
        "desktop": "analyzed - no issues"
      }
    }
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

Critical flows identified: {count}

Screenshots taken: {routes * 4} (4 breakpoints per route)

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
- [ ] EVERY page had accessibility tree read via `mcp__claude-in-chrome__read_page`
- [ ] ALL buttons recorded (compare visible count vs recorded)
- [ ] Expected behaviors assigned to ALL elements
- [ ] ALL 4 breakpoints screenshotted for EVERY route
- [ ] Each screenshot was ANALYZED (not just taken)
- [ ] Console checked on EVERY route
- [ ] State file written with complete data

**If a page has visible buttons but 0 elements recorded, you missed them. Go back and check.**

---

## Discovery Output Files

| File | Purpose |
|------|---------|
| `validation-state.json` | Main state with all discovery data |

Discovery data is stored IN the state file, not separate files.
