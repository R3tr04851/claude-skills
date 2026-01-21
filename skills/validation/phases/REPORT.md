# Phase: REPORT

**Goal:** Generate comprehensive HTML report with all evidence.

---

## Step 1: Read Final State

```javascript
const state = readState('test-manifest/validation-state.json');
```

Extract:
- All results from `state.testing.results`
- Summary from `state.summary`
- Discovery data from `state.discovery`

---

## Step 2: Categorize Results

Group results by:

### By Status
```javascript
const passed = Object.values(results).filter(r => r.status === 'pass');
const failed = Object.values(results).filter(r => r.status === 'fail');
const skipped = Object.values(results).filter(r => r.status === 'skip');
```

### By Route
```javascript
const byRoute = {};
Object.values(results).forEach(r => {
  if (!byRoute[r.route]) byRoute[r.route] = [];
  byRoute[r.route].push(r);
});
```

### Issues
```javascript
const uiIssues = Object.values(results).flatMap(r => r.uiIssues || []);
const consoleErrors = Object.values(results).flatMap(r => r.consoleErrors || []);
const verificationFailures = failed.map(r => ({
  element: r.elementId,
  expected: r.verification.expected,
  evidence: r.verification.evidence
}));
```

---

## Step 3: Load Report Template

Read template from: `templates/report.html`

Or use embedded template if file not found.

---

## Step 4: Generate Report Content

### Summary Section

```html
<div class="summary-grid">
  <div class="summary-card total">
    <span class="number">{totalElements}</span>
    <span class="label">Total Elements</span>
  </div>
  <div class="summary-card pass">
    <span class="number">{passed}</span>
    <span class="label">Passed</span>
  </div>
  <div class="summary-card fail">
    <span class="number">{failed}</span>
    <span class="label">Failed</span>
  </div>
  <div class="summary-card warn">
    <span class="number">{uiIssues}</span>
    <span class="label">UI Issues</span>
  </div>
</div>
```

### Per-Route Sections

For each route:

```html
<section class="route-section">
  <h2>{route.path}</h2>

  <div class="screenshots-row">
    <figure>
      <img src="{mobile-screenshot}" alt="Mobile 375px">
      <figcaption>Mobile (375px)</figcaption>
    </figure>
    <figure>
      <img src="{tablet-screenshot}" alt="Tablet 768px">
      <figcaption>Tablet (768px)</figcaption>
    </figure>
    <figure>
      <img src="{laptop-screenshot}" alt="Laptop 1024px">
      <figcaption>Laptop (1024px)</figcaption>
    </figure>
    <figure>
      <img src="{desktop-screenshot}" alt="Desktop 1440px">
      <figcaption>Desktop (1440px)</figcaption>
    </figure>
  </div>

  <h3>Elements Tested</h3>
  <table>
    <thead>
      <tr>
        <th>Element</th>
        <th>Expected</th>
        <th>Verified</th>
        <th>Status</th>
      </tr>
    </thead>
    <tbody>
      <!-- For each element in route -->
      <tr class="{pass|fail}">
        <td>{element.text}</td>
        <td>{element.expectedBehavior}</td>
        <td>{verification.evidence}</td>
        <td><span class="badge {status}">{status}</span></td>
      </tr>
    </tbody>
  </table>

  <!-- If UI issues found -->
  <div class="issues-box">
    <h4>UI Issues</h4>
    <ul>
      <li>{issue description}</li>
    </ul>
  </div>

  <!-- If console errors found -->
  <div class="console-box">
    <h4>Console Errors</h4>
    <pre>{error messages}</pre>
  </div>
</section>
```

### Issues Summary Section

```html
<section class="issues-summary">
  <h2>Issues Found</h2>

  <h3>Verification Failures ({count})</h3>
  <ul>
    <li>
      <strong>{elementId}</strong> on {route}
      <br>Expected: {expected}
      <br>Evidence: {what actually happened}
    </li>
  </ul>

  <h3>UI Issues ({count})</h3>
  <ul>
    <li>{route}: {issue description}</li>
  </ul>

  <h3>Console Errors ({count})</h3>
  <ul>
    <li>{route}: {error message}</li>
  </ul>
</section>
```

---

## Step 5: Write Report File

Save to: `test-manifest/reports/validation-{YYYY-MM-DD}.html`

```javascript
const filename = `validation-${new Date().toISOString().split('T')[0]}.html`;
writeFile(`test-manifest/reports/${filename}`, reportHtml);
```

---

## Step 6: Update State

```javascript
state.session.status = 'completed';
state.session.completedAt = new Date().toISOString();
state.reportPath = `test-manifest/reports/${filename}`;
writeState(state);
```

---

## Step 7: Output Summary

```
=== VALIDATION COMPLETE ===

Session: {session.id}
Duration: {startedAt} to {completedAt}
Context resets: {contextResets}

Summary:
- Total elements: {totalElements}
- Passed: {passed} ({passRate}%)
- Failed: {failed}
- Skipped: {skipped}
- UI Issues: {uiIssueCount}
- Console Errors: {consoleErrorCount}

Report saved to: test-manifest/reports/validation-{date}.html

Top Issues:
1. {most critical issue}
2. {second issue}
3. {third issue}

Recommendations:
- {recommendation based on failures}
```

---

## Report HTML Template

The report should include:

### Header
- Project name
- Date/time
- App URL tested
- Session duration

### Summary Cards
- Total elements
- Passed count (green)
- Failed count (red)
- UI issues (yellow)
- Console errors (orange)

### Per-Route Sections
- Route path
- Screenshot gallery (4 breakpoints: Mobile 375px, Tablet 768px, Laptop 1024px, Desktop 1440px)
- Screenshot analysis notes
- Elements tested table
- Issues for that route
- Console errors for that route

### Global Issues Section
- All verification failures with evidence
- All UI issues grouped by type
- All console errors grouped by type

### Test Evidence
- Clickable screenshots (lightbox)
- Actual verification logs
- Before/after comparisons where applicable

### Footer
- Generation timestamp
- Skill version
- Link to state file for debugging

---

## Anti-Laziness: Report Validation

Before marking complete:

- [ ] Report file actually exists
- [ ] Screenshots are embedded/linked correctly
- [ ] Every tested element has a row in the table
- [ ] Failed tests have evidence descriptions
- [ ] Issues section populated if any issues exist
- [ ] Summary numbers match actual counts

**If report shows 0 failures but testing had failures, something is wrong.**
