# Validation Skill

Comprehensive app validation with REAL verification - discovers all interactive elements, tests them with actual verification, and generates HTML reports.

---

## Overview

**The Validation Skill is REAL testing, not just documentation.** This skill ensures web applications are tested by actually:
- Navigating to your app in a real browser
- Discovering ALL interactive elements on every page
- Clicking buttons, filling forms, testing interactions
- Taking screenshots at multiple breakpoints (mobile, tablet, laptop, desktop)
- Checking for console errors and UI issues
- Generating persistent HTML reports with evidence

**Core principle:** If you didn't interact with the real element in the real app, you didn't test it.

---

## When to Use

- After implementing new UI features
- Before deploying to production
- When refactoring components
- After fixing bugs that need visual verification
- When you need comprehensive coverage of all interactive elements

**Do NOT use for:**
- Unit test writing (use test-driven-development skill)
- API endpoint testing (use feature-validation skill)
- Code review (use code-reviewer agents)

---

## Key Features

### 🔍 Comprehensive Discovery
- Automatically finds ALL interactive elements (buttons, links, forms, inputs)
- Tests across 4 breakpoints: mobile (375px), tablet (768px), laptop (1024px), desktop (1440px)
- Identifies critical user flows (login, signup, checkout, etc.)

### ✅ Real Verification
- Clicks buttons and verifies actions actually happened
- Takes screenshots and ANALYZES them (not just captures)
- Checks console for errors after every interaction
- Verifies state changes (list count, modal appearance, navigation)

### 📊 HTML Reports
- Persistent reports saved to `/test-manifest/reports/`
- Evidence-based: every verdict includes screenshot proof
- Shows passed/failed tests with actual outcomes
- Lists all console errors found

### 🔄 Session Persistence
- Saves progress after EVERY element test
- Resume across context resets without losing work
- State stored in `test-manifest/validation-state.json`

### 🧹 Cleanup
- Automatically removes test artifacts (VAL_* items)
- Cleans up test data after validation completes

---

## Architecture

This skill uses a **3-phase architecture** with state persistence:

```
Phase 1: DISCOVER
├── PRE-FLIGHT: Check dev server is running
├── Route Discovery: Find all app routes
├── Element Discovery: Find all interactive elements
└── Critical Flows: Identify key user journeys

Phase 2: TEST
├── For each element:
│   ├── Navigate to element's page
│   ├── Resize to breakpoint (mobile/tablet/laptop/desktop)
│   ├── Take screenshot & ANALYZE
│   ├── Interact (click/fill/submit)
│   ├── Verify outcome
│   ├── Check console errors
│   └── Save state
└── Cleanup: Remove VAL_* test artifacts

Phase 3: REPORT
├── Generate HTML report with evidence
├── Include all screenshots
├── List all console errors
└── Save to /test-manifest/reports/
```

### State Management

**State file:** `{project}/test-manifest/validation-state.json`

Tracks:
- Current phase (discover/test/cleanup/report)
- All discovered elements
- Test results for each element
- Pending/failed/quarantined items
- Summary statistics

**After EVERY element test:**
1. Result saved to state
2. Screenshot captured
3. Console checked
4. State file written to disk

This ensures **zero progress loss** if context overflows.

---

## Usage

### Basic Invocation

```bash
/validate http://localhost:3000
```

### Options

```bash
# With specific URL
/validate http://localhost:5173

# Resume from existing state
/validate --resume

# Start fresh (delete existing state)
/validate --fresh http://localhost:3000
```

---

## Browser Tools (MCP Chrome Extension)

This skill uses **MCP browser tools** via the Claude-in-Chrome extension:

| Tool | Purpose |
|------|---------|
| `mcp__claude-in-chrome__navigate` | Navigate to URL |
| `mcp__claude-in-chrome__read_page` | Get page elements |
| `mcp__claude-in-chrome__click` | Click element |
| `mcp__claude-in-chrome__form_input` | Fill form field |
| `mcp__claude-in-chrome__resize_window` | Change viewport size |
| `mcp__claude-in-chrome__computer` | Take screenshot |
| `mcp__claude-in-chrome__read_console_messages` | Check console errors |

**No Playwright or Puppeteer needed!** Claude calls browser tools directly.

---

## Typical Workflow

### 1. PRE-FLIGHT Check

```
✓ Server responding at http://localhost:3000
✓ Load time: 1.2s
✓ Console: 0 errors, 2 warnings
✓ Auth: Not required
```

### 2. Discovery Phase

```
Discovering routes...
✓ Found 5 routes: /, /about, /dashboard, /settings, /profile

Discovering elements...
✓ Found 23 interactive elements:
  - 12 buttons
  - 5 links
  - 4 form inputs
  - 2 dropdowns

Identifying critical flows...
✓ Found 2 critical flows:
  - User registration flow
  - Dashboard navigation flow
```

### 3. Testing Phase

```
Testing element 1/23: "Login button"...
├── Mobile (375px):
│   ├── Screenshot captured
│   ├── Analysis: Button visible, properly sized
│   ├── Clicked button
│   ├── Verification: Login modal appeared ✓
│   └── Console: 0 errors
├── Tablet (768px): ✓
├── Laptop (1024px): ✓
└── Desktop (1440px): ✓

Testing element 2/23: "Signup link"...
[Similar breakdown for each element]
```

### 4. Report Generation

```
Generating HTML report...
✓ Report saved: test-manifest/reports/validation-2026-01-21.html

Summary:
- Routes tested: 5
- Elements tested: 23
- Passed: 21
- Failed: 2
- UI Issues: 3
- Console Errors: 1
```

---

## Anti-Laziness Rules

The skill enforces **REAL verification**, not fake claims:

### ❌ WRONG: "Screenshot taken successfully"
### ✅ RIGHT:
```
Screenshot analysis:
- Header: Visible, properly aligned
- Navigation: All items visible, no overflow
- Main content: Cards display correctly
- Mobile (375px): Menu collapses to hamburger
- Issue: Footer text cut off at 375px
```

### ❌ WRONG: "Clicked delete button successfully"
### ✅ RIGHT:
```
Delete button clicked:
- Confirmation modal appeared: YES
- Confirmed deletion
- Item 'Test Customer 12345' removed from list: VERIFIED
- List count changed from 5 to 4: VERIFIED
```

### ❌ WRONG: "No console errors"
### ✅ RIGHT:
```
Console check:
- Errors: 0
- Warnings: 2 (React key warning, deprecation notice)
- Actual messages:
  1. Warning: Each child in a list should have a unique "key" prop
  2. Warning: componentWillMount is deprecated
```

---

## Directory Structure

```
{project}/
  test-manifest/
    validation-state.json      # Persistent state
    screenshots/
      routes/                  # Route screenshots by breakpoint
        home-mobile.png
        home-tablet.png
        home-laptop.png
        home-desktop.png
      elements/                # Element interaction screenshots
        button-1-mobile.png
        button-1-clicked.png
    reports/
      validation-2026-01-21.html   # HTML report with evidence
```

---

## Breakpoints Tested

All elements are tested at **4 standard breakpoints:**

| Device | Width | Height | Purpose |
|--------|-------|--------|---------|
| Mobile | 375px | 812px | iPhone SE, small phones |
| Tablet | 768px | 1024px | iPad, standard tablets |
| Laptop | 1024px | 768px | Small laptops, 13" screens |
| Desktop | 1440px | 900px | Standard desktops, 15"+ screens |

---

## Example Report

The generated HTML report includes:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Validation Report - 2026-01-21</title>
  <style>
    .pass { color: #22c55e; }
    .fail { color: #ef4444; }
    .screenshot { max-width: 100%; border: 1px solid #ddd; }
  </style>
</head>
<body>
  <h1>Validation Report</h1>

  <h2>Summary</h2>
  <ul>
    <li>Total Elements: 23</li>
    <li class="pass">Passed: 21</li>
    <li class="fail">Failed: 2</li>
    <li>Console Errors: 1</li>
  </ul>

  <h2>Test Results</h2>

  <!-- For each element -->
  <div class="test-case">
    <h3>Login Button - <span class="pass">PASS</span></h3>
    <p><strong>Location:</strong> Homepage, top-right</p>
    <p><strong>Breakpoints:</strong> All passed (mobile, tablet, laptop, desktop)</p>
    <img class="screenshot" src="../screenshots/elements/button-1-mobile.png">
    <p><strong>Action:</strong> Clicked button</p>
    <p><strong>Verification:</strong> Login modal appeared ✓</p>
    <p><strong>Console:</strong> 0 errors</p>
  </div>

  <div class="test-case">
    <h3>Delete Button - <span class="fail">FAIL</span></h3>
    <p><strong>Location:</strong> Dashboard, user list</p>
    <p><strong>Issue:</strong> Button not visible at mobile breakpoint</p>
    <img class="screenshot" src="../screenshots/elements/button-5-mobile.png">
    <p><strong>Console Error:</strong> TypeError: Cannot read property 'id' of undefined</p>
  </div>
</body>
</html>
```

---

## Phase Files

The skill is split into 3 phase files for context efficiency:

| File | Lines | Purpose |
|------|-------|---------|
| `SKILL.md` | ~370 | Orchestrator, loads phases |
| `phases/DISCOVER.md` | ~300 | PRE-FLIGHT + discovery logic |
| `phases/TEST.md` | ~650 | Testing + verification + cleanup |
| `phases/REPORT.md` | ~280 | HTML report generation |
| `templates/report.html` | ~460 | Report HTML template |

**Total:** ~2,060 lines across 5 files

**Context per phase:** 700-1,000 lines max (orchestrator + one phase)

---

## Common Mistakes to Avoid

| Mistake | Correct Approach |
|---------|------------------|
| Skip PRE-FLIGHT check | ALWAYS verify server running first |
| Test only desktop | Test all 4 breakpoints (mobile, tablet, laptop, desktop) |
| Screenshot without analysis | ANALYZE each screenshot for UI issues |
| Click without verification | VERIFY the action actually happened |
| Ignore console errors | CHECK console after EVERY interaction |
| Delete test reports | Reports are PERMANENT, saved to `/test-manifest/` |
| Skip cleanup phase | ALWAYS remove VAL_* test artifacts |

---

## Red Flags - STOP and Fix

- **"Screenshot looks good"** → NO, ANALYZE it: What's visible? Any issues?
- **"Button clicked successfully"** → NO, VERIFY: What happened after click?
- **"No errors"** → NO, READ console: What are the actual messages?
- **"Test complete"** → NO, CHECK: Did you test all 4 breakpoints?
- **"Element works"** → VERIFY with evidence: Screenshots? Console? State changes?

---

## Integration with Other Skills

- **Before using:** verification-before-completion (claims work is done)
- **After using:** systematic-debugging (if validation finds issues)
- **Complements:** feature-validation (validates CRUD operations specifically)

---

## Requirements

- Claude Code CLI
- Claude-in-Chrome MCP extension
- Running dev server (or static site)
- Project with web interface

---

## Troubleshooting

### Dev Server Not Running
```
✗ PRE-FLIGHT FAILED: Server not responding at http://localhost:3000
→ Start your dev server first:
  npm run dev
  # or
  yarn dev
  # or
  python -m http.server 3000
```

### Console Errors Found
```
✗ Console Error: TypeError: Cannot read property 'id' of undefined
→ This is a TEST FAILURE. Fix the error before marking validation complete.
```

### Element Not Found
```
✗ Element "Login button" not found on page
→ Page may have loaded incorrectly, or element selector changed
→ Check page source, verify element exists
```

---

## Version History

- **v1.0** - Initial release
  - 3-phase architecture
  - State persistence
  - 4 breakpoint testing
  - HTML report generation
  - MCP browser tools integration

---

## License

MIT

---

## Contributing

Issues and improvements welcome! This skill is part of the [ai-debate-hub](https://github.com/wolverin0/ai-debate-hub) project.

---

**Built for comprehensive, evidence-based app validation with Claude Code.**
