# A11y Sentinel

**Continuous accessibility testing and regression detection for web applications.**

> **Status: In Development** — This repository documents the architecture, design, and roadmap for A11y Sentinel. Core features are being actively built. Sections marked `[Planned]` describe functionality not yet implemented.

---

## Overview

Accessibility testing in most development workflows happens once — as a manual audit before launch, or a checkbox during QA. The result is that a site can pass an audit on a Thursday and introduce three keyboard navigation failures by Friday after a routine deployment.

A11y Sentinel is designed to treat accessibility the same way good engineering teams treat test coverage and performance budgets: as a continuous, automated signal integrated into the software development lifecycle.

The system crawls websites, runs deterministic accessibility analysis using axe-core and Lighthouse, simulates real browser interactions via Playwright, and compares results against a historical baseline to classify issues as **new**, **fixed**, or **persistent**. The goal is not to generate a one-time WCAG report — it is to detect regressions before they reach production users.

---

## The Problem

A typical accessibility audit is a point-in-time measurement. It tells you where you stand today. It tells you nothing about what changed between your last deployment and this one.

Web applications break accessibility in small, quiet ways:

- A developer adds a `div` where a `button` should be
- A modal stops returning focus to the trigger element
- A new form ships without associated labels
- A CSS change drops a color contrast ratio below 4.5:1
- A navigation refactor breaks tab order

None of these changes are intentional. Most of them won't be caught by a unit test. And unless someone runs a manual screen reader pass after every deployment — which teams rarely do — these regressions accumulate silently.

**The problem is not that developers don't care about accessibility. The problem is that there is no automated feedback loop telling them when they've broken it.**

---

## The Solution

A11y Sentinel provides that feedback loop.

```
Developer submits a URL
        ↓
Automated crawler discovers pages
        ↓
Playwright loads each page in a real browser
        ↓
axe-core + Lighthouse run accessibility analysis
        ↓
Interaction engine simulates keyboard navigation
        ↓
Results are normalized and persisted
        ↓
Regression engine compares against the previous scan
        ↓
Dashboard surfaces new issues, fixed issues, and trends
        ↓
[Planned] CI/CD integration blocks or flags regressions in PRs
```

The core output is not a compliance report. It is a **diff** — a structured comparison between the current state of your site and the last time it was measured.

---

## How It Works

### 1. URL Submission

A developer provides a starting URL. A11y Sentinel begins a scan job.

### 2. Crawling

A headless Playwright browser crawls the site, following internal links to discover pages. Each discovered page is added to the scan queue.

### 3. Accessibility Analysis

For each page, the system runs:

- **axe-core** — deterministic rule-based accessibility checks
- **Lighthouse** — accessibility audits including additional checks beyond axe-core
- **Custom WCAG-oriented rules** — project-specific checks for patterns axe-core does not cover

Issues are normalized into a consistent schema covering:

| Category | Example Issues |
|---|---|
| Images | Missing `alt` text, empty `alt` on informational images |
| Forms | Inputs without `<label>`, missing `aria-labelledby` |
| Interactive Elements | Buttons and links without accessible names |
| ARIA | Invalid roles, misused attributes, broken references |
| Heading Structure | Skipped heading levels, multiple `<h1>` elements |
| Color Contrast | Foreground/background ratio below WCAG threshold |
| Keyboard Accessibility | Unreachable elements, invisible focus indicators |
| Semantics | Div-based interactive elements, missing landmark regions |

### 4. Interaction Testing

Static HTML analysis finds what is wrong with the markup. Interaction testing finds what breaks when a user actually operates the page.

Playwright simulates keyboard interactions across each page:

- Tab and Shift+Tab traversal to identify focus order and reachable elements
- Enter and Space activation of interactive controls
- Escape to close modals and dismiss overlays
- Arrow key navigation within composite widgets (menus, tabs, dialogs)
- Focus management checks after dynamic content changes

This catches issues that axe-core cannot: focus traps, lost focus after modal close, interactive elements that are in the DOM but cannot receive keyboard focus, and focus indicators that are present in the stylesheet but invisible at runtime.

### 5. Results Storage

Scan results, including issue details, affected pages, severity classifications, and interaction test outcomes, are persisted to PostgreSQL. Each scan is associated with the previous scan for the same site, enabling historical comparison.

### 6. Regression Detection

The regression engine compares the current scan against the previous baseline and classifies every issue:

| Classification | Meaning |
|---|---|
| **New** | Present in the current scan, not present in the previous scan |
| **Fixed** | Present in the previous scan, not present in the current scan |
| **Persistent** | Present in both scans |

The accessibility health score is recalculated and the delta is surfaced prominently.

### 7. Dashboard

The dashboard presents scan results, regression summaries, historical health trends, and per-issue remediation guidance.

---

## Key Features

### Regression Detection (Core Feature)

The primary purpose of A11y Sentinel is not to generate a list of accessibility issues — every accessibility scanner does that. The primary purpose is to tell you what changed.

```
Previous Scan  →  Current Scan  →  Regression Engine
    (baseline)       (latest)

                                   ┌─────────────┐
                                   │  NEW (3)    │  ← introduced since baseline
                                   ├─────────────┤
                                   │  FIXED (1)  │  ← resolved since baseline
                                   ├─────────────┤
                                   │ PERSISTENT  │  ← unchanged
                                   │    (12)     │
                                   └─────────────┘
```

A team can use this to answer: *Did the deployment we just shipped make accessibility better or worse?*

### Real Browser Interaction Testing

Accessibility issues that only appear at runtime — broken focus management, keyboard traps, inaccessible dynamic content — require a real browser to detect. Playwright runs tests in Chromium, giving the analysis engine access to the fully rendered DOM, computed styles, and JavaScript-driven behavior.

### `[Planned]` User Journey Testing

Developers define application workflows as sequences of interactions. A11y Sentinel executes those workflows and reports any accessibility failures encountered along the way.

```yaml
# Example journey definition
name: "Admissions Application Flow"
steps:
  - navigate: "/"
  - click: "[data-testid='admissions-link']"
  - click: "text=Undergraduate"
  - click: "text=Apply Now"
  - assert_focus_visible: true
  - fill: "#email-input"
    value: "test@example.com"
  - assert_accessible_name: "#submit-button"
```

This produces a report of accessibility failures that occur during the actual user workflow, not just on static page loads.

### `[Planned]` CI/CD Integration

A11y Sentinel can be integrated into GitHub Actions to run accessibility checks on every pull request and compare results against the main branch baseline.

```yaml
# .github/workflows/a11y.yml
- name: Run A11y Sentinel
  run: |
    npx a11y-sentinel scan $PREVIEW_URL \
      --baseline main \
      --fail-on-regression
```

A regression causes the check to fail and surfaces the diff in the PR.

### `[Planned]` AI-Assisted Remediation

Where helpful, an LLM layer explains issues in plain language and suggests concrete code fixes. This is explicitly an assistance layer, not the detection mechanism.

```
Detection:  Playwright + axe-core + custom rules  (deterministic)
                              ↓
                       Issue identified
                              ↓
Explanation: LLM generates plain-language description + fix suggestion
```

The accessibility analysis does not rely on a language model to identify issues. LLMs are only used to explain what the deterministic engine has already found.

---

## System Architecture

```
┌─────────────────────────────────┐
│        Next.js Frontend         │
│  Dashboard · Scans · Issues     │
└────────────────┬────────────────┘
                 │  REST API
┌────────────────▼────────────────┐
│         FastAPI Backend         │
│   Scan management · Auth · API  │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────┐
│        Scan Job System          │
│  Job queue · Worker dispatch    │
└──────┬──────────────────┬───────┘
       │                  │
┌──────▼──────┐    ┌──────▼──────┐
│  Playwright │    │  axe-core   │
│   Browser   │◄──►│  Lighthouse │
│  Automation │    │  Analysis   │
└──────┬──────┘    └──────┬──────┘
       │                  │
┌──────▼──────────────────▼───────┐
│       Regression Engine         │
│  Issue diff · Health score      │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────┐
│           PostgreSQL            │
│  Scans · Issues · History       │
└─────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility |
|---|---|
| **Next.js Frontend** | Dashboard UI, scan submission, issue browser, trend charts |
| **FastAPI Backend** | REST API, scan lifecycle management, authentication, job dispatch |
| **Scan Job System** | Async job queue, worker coordination, progress tracking |
| **Playwright** | Headless browser, page loading, interaction simulation, DOM access |
| **axe-core** | In-browser accessibility rule evaluation |
| **Lighthouse** | Programmatic accessibility and performance auditing |
| **Regression Engine** | Issue diffing, new/fixed/persistent classification, health scoring |
| **PostgreSQL** | Persistent storage for scans, pages, issues, and historical data |

---

## Accessibility Testing Pipeline

```
1. Scan job created
        │
        ▼
2. Crawler discovers internal pages
   (Playwright follows links from the root URL)
        │
        ▼
3. Each page is loaded in a headless Chromium instance
        │
        ▼
4. axe-core is injected and executed in the page context
   → Returns structured violations with element references
        │
        ▼
5. Lighthouse audit runs against the loaded page
   → Returns scored accessibility audit items
        │
        ▼
6. Interaction engine runs keyboard traversal
   → Tab order, focus visibility, modal behavior, dynamic content
        │
        ▼
7. Results are normalized into a unified issue schema
   {id, rule, severity, wcag_criterion, element, page, description}
        │
        ▼
8. Issues are persisted to PostgreSQL with scan association
        │
        ▼
9. Regression engine loads the previous scan for this site
   and computes the three-way diff (new / fixed / persistent)
        │
        ▼
10. Health score is calculated and the scan is marked complete
        │
        ▼
11. Dashboard reflects updated results
```

---

## Regression Detection

Each scan is compared against the immediately preceding scan for the same site to produce a regression report.

### Issue Identity

Two issues are considered the same issue if they share the same rule identifier, WCAG criterion, and affected element selector. This allows the engine to track whether a specific violation on a specific element was introduced, fixed, or carried over.

### Classification Logic

```python
# Simplified classification logic
previous_issues = {issue.fingerprint for issue in baseline_scan.issues}
current_issues  = {issue.fingerprint for issue in current_scan.issues}

new_issues        = current_issues - previous_issues
fixed_issues      = previous_issues - current_issues
persistent_issues = current_issues & previous_issues
```

### Health Score

The accessibility health score is a project-specific metric calculated from the number and severity of issues relative to the pages scanned. It is **not** an official WCAG compliance score and does not represent legal accessibility conformance.

```
Score = 100 − (critical × 10 + serious × 5 + moderate × 2 + minor × 1)
        ────────────────────────────────────────────────────────────────
        Capped at 0, normalized per page count
```

### Example Regression Report

```
Scan: 2025-10-14  →  Scan: 2025-10-21

Accessibility Health:  94  →  81   (−13)

NEW ISSUES (3)
──────────────────────────────────────────────────────────────
  [SERIOUS]   Missing accessible name          /admissions
              <button class="menu-toggle">

  [SERIOUS]   Focus not returned after modal close  /apply
              #application-modal

  [MODERATE]  Color contrast ratio 3.1:1       /contact
              .secondary-cta

FIXED ISSUES (1)
──────────────────────────────────────────────────────────────
  [MODERATE]  Image missing alt text            /index
              <img src="campus.jpg">

PERSISTENT ISSUES (8)
──────────────────────────────────────────────────────────────
  ... 8 issues carried from previous scan
```

---

## `[Planned]` User Journey Testing

Static page analysis and keyboard traversal cover most scenarios. User journey testing handles the rest: multi-step workflows where accessibility failures only appear after a sequence of interactions.

A developer defines a journey as an ordered set of steps. The engine executes those steps in a real browser and reports any accessibility violation, focus failure, or ARIA state problem encountered along the way.

**Why this matters:** A form field may have a correct label on initial page load. After a validation error is shown dynamically, the error message may not be programmatically associated with the field. Static analysis will not catch this. A journey test will.

### Example Journey

```
Journey: Admissions Application

  Step 1  Navigate to homepage
  Step 2  Click primary navigation → "Admissions"
          ✓ Focus moves to Admissions page heading
  Step 3  Click "Undergraduate Programs"
          ✓ Page accessible, landmark structure intact
  Step 4  Click "Apply Now"
          ✓ Application form loads
          ✗ Email input has no accessible label
          ✗ Form error region missing aria-live attribute
  Step 5  Submit form with empty fields
          ✗ Error messages not announced to screen readers
          ✗ Focus not moved to first error

Journey Result: 3 failures across 2 steps
```

---

## `[Planned]` Developer Workflow / CI/CD

The intended integration model makes accessibility a first-class check in pull request workflows.

```
Developer opens a pull request
              ↓
GitHub Actions workflow triggers
              ↓
Application is built and deployed to a preview environment
              ↓
A11y Sentinel scans the preview URL
              ↓
Results are compared against the main branch baseline
              ↓
         No regressions?                 Regressions found?
              ↓                                  ↓
    ✓ Check passes                   ✗ Check fails
                                     PR comment with diff
```

This removes accessibility from the "pre-launch checklist" category and puts it in the same automated feedback loop as unit tests and type checks.

---

## Proposed Tech Stack

| Technology | Role |
|---|---|
| **Next.js** | Frontend framework, dashboard UI, server-side rendering |
| **TypeScript** | Type safety across the frontend codebase |
| **Tailwind CSS** | Utility-based styling |
| **Recharts** | Accessibility health trend charts |
| **Python** | Backend language |
| **FastAPI** | REST API framework, async request handling |
| **Pydantic** | Request/response validation and data modeling |
| **SQLAlchemy** | ORM for database access |
| **PostgreSQL** | Primary data store for scans, issues, and history |
| **Playwright** | Headless browser automation and interaction testing |
| **axe-core** | In-browser accessibility rule evaluation |
| **Lighthouse** | Programmatic accessibility and performance audits |
| **GitHub Actions** | `[Planned]` CI/CD integration |
| **Redis** | `[Planned]` Background job queue |

---

## Project Roadmap

The project is being built in phases, moving from core infrastructure toward developer workflow integration.

### Phase 1 — Core Scanner
- [ ] URL submission and crawl management
- [ ] Playwright-based page loading
- [ ] axe-core integration and result normalization
- [ ] Lighthouse integration
- [ ] Issue schema and PostgreSQL persistence
- [ ] Basic API endpoints for scan management

### Phase 2 — Dashboard
- [ ] Scan history view
- [ ] Per-page issue breakdown
- [ ] Issue severity classification
- [ ] Accessibility health score display
- [ ] Project/site management

### Phase 3 — Regression Detection
- [ ] Baseline scan storage
- [ ] New / Fixed / Persistent issue classification
- [ ] Health score delta display
- [ ] Historical trend visualization

### Phase 4 — Interaction Testing
- [ ] Tab order traversal engine
- [ ] Focus visibility checks
- [ ] Modal focus management testing
- [ ] Keyboard operability of interactive controls
- [ ] Dynamic content accessibility

### Phase 5 — User Journey Testing
- [ ] Journey definition schema
- [ ] Journey execution engine
- [ ] Per-step accessibility reporting
- [ ] Journey-level regression detection

### Phase 6 — Developer Workflow
- [ ] GitHub Actions integration
- [ ] Pull request regression checks
- [ ] Automated baseline management
- [ ] Slack / email notifications

### Phase 7 — AI Assistance Layer
- [ ] LLM-generated plain-language issue explanations
- [ ] Remediation code suggestions
- [ ] Context-aware fix examples

---

## Limitations

Automated accessibility testing is a useful signal. It is not a complete accessibility assessment.

**What automated testing can do:**
- Identify a large portion of common accessibility violations
- Detect regressions between deployments
- Surface keyboard navigation failures
- Check markup and ARIA correctness
- Track trends over time

**What automated testing cannot do:**
- Replace testing with actual assistive technologies (NVDA, JAWS, VoiceOver)
- Replace testing with people with disabilities
- Evaluate the quality of alternative text, not just its presence
- Assess whether the logical reading order makes sense to a real user
- Interpret complex WCAG requirements that require human judgment
- Certify WCAG 2.1 or 2.2 conformance

The accessibility health score generated by A11y Sentinel is a **project-specific metric**. It is not an official score, not a legal certification, and not a claim of WCAG conformance. Passing an A11y Sentinel scan does not mean a website is fully accessible.

A complete accessibility program requires automated testing as one part of a broader process that includes manual review, assistive technology testing, and feedback from users with disabilities.

---

## Why I Built This

While working as a Web Developer Assistant on production university websites, I spent significant time on accessibility work: auditing pages for WCAG compliance, fixing semantic navigation issues, correcting heading hierarchies, and improving Core Web Vitals. Most of that work happened manually, with periodic audits rather than continuous feedback.

The process had a recurring problem: changes made by other contributors would sometimes undo previous accessibility work between audit cycles. There was no automated system to catch those regressions before they reached users.

A11y Sentinel is an attempt to build the tooling I wanted in that workflow — a system that makes accessibility failures visible at the moment they are introduced, not weeks later when someone runs a manual audit.

The engineering interest here is not accessibility checking per se. It is the harder problem of **continuous regression detection over a live, changing system** — crawling, browser automation, result diffing, historical modeling, and developer workflow integration.

---

## Future Improvements

Beyond the planned roadmap phases, longer-term directions include:

- **Distributed crawling** for large sites with hundreds of pages
- **Multi-browser testing** (Firefox, WebKit) to catch browser-specific issues
- **Mobile viewport simulation** for responsive accessibility testing
- **Issue replay** — reproducing a specific failure on demand in the dashboard
- **Source-code-aware remediation** — LLM suggestions tied to the actual component code
- **Multi-user organizations** with role-based access control
- **Scheduled scans** with configurable intervals
- **Webhook delivery** for integration with external monitoring systems
- **Export formats** — structured reports for accessibility auditing workflows

---

## License

License not yet specified.

---

*A11y Sentinel is an independent project. It is not affiliated with or endorsed by any university, organization, or institution.*