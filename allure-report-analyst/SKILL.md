---
name: allure-report-analysis
description: Parses raw Allure results, classifies failures by root-cause category, and maintains a cross-run failure ledger (new/recurring/flaky/resolved). Use when analyzing Allure test output for failure trends, not just current-run status.
---

# Allure Report Analysis

## 1. Input

Prefer raw results over the generated HTML report:

```
allure-results/
  <uuid>-result.json      # one per test case — status, steps[], statusDetails
  <uuid>-container.json   # one per fixture — befores[]/afters[] with own status
```

Rationale: the generated `allure-report/data/*.json` is a denormalized,
re-aggregated view meant for the web UI. Parsing raw results avoids a
lossy round-trip and gives direct access to `steps[]` and `statusDetails.trace`.

## 2. Extraction

For each `*-result.json` where `status` is `failed` or `broken`, extract:

| Field | Path | Purpose |
|---|---|---|
| Test name | `.name` | identity |
| Suite | `.labels[] where name=="suite"` | grouping |
| Status | `.status` | failed vs broken (broken = unexpected exception, not assertion) |
| Failed step | first `steps[]` entry where `status != "passed"` | localizes failure within the test, not just "test failed" |
| Message | `.statusDetails.message` | classification input |
| Trace | `.statusDetails.trace` | classification input, first ~5 lines only |
| History ID | `.historyId` | stable identity across runs — required for ledger matching |

Also check `.status == "broken"` for fixture-level failures in
`*-container.json` (`befores[]`/`afters[]`) — these indicate environment/setup
problems, not test logic problems, and must be classified separately.

## 3. Classification taxonomy

Classify each failure into exactly one category, in this priority order
(check top-down, first match wins):

1. **Environment/Infra** — trace contains connection refused, timeout
   connecting to a host, 5xx from a non-application-under-test service,
   container/browser launch failure. Not an application bug.
2. **Locator/Selector** — Cypress/Playwright "element not found",
   "element not visible", "element detached from DOM". Common in UI
   suites after a front-end change.
3. **Timing/Flaky** — assertion failure on a value that is timing-
   dependent (race condition language in message), OR — more reliably —
   this exact `historyId` has an inconsistent status across the last 5
   ledger entries (passed, failed, passed...). Flaky is a *pattern*
   classification, not a single-run classification — don't label a
   first-time failure "flaky."
4. **Assertion/Logic** — a genuine expected-vs-actual mismatch with no
   environment or locator signature. This is the only category that
   implies an actual product or test-logic defect.
5. **Unknown** — doesn't cleanly match above; flag for manual review
   rather than force-fitting a category.

## 4. Failure ledger

Maintain a ledger file (`allure-failure-ledger.json`) alongside the results,
one entry per `historyId`, updated every run:

```json
{
  "historyId": "abc123...",
  "testName": "POST /orders returns 201 on valid payload",
  "category": "assertion",
  "firstSeenRun": "2026-08-15",
  "lastSeenRun": "2026-08-29",
  "consecutiveFailures": 3,
  "statusHistory": ["failed", "failed", "failed"],
  "state": "recurring"
}
```

`state` derivation rules:
- **new** — `historyId` not in prior ledger, now failing
- **recurring** — failed in prior ledger entry AND this run, `consecutiveFailures` increments
- **flaky** — appears in `statusHistory` with mixed pass/fail over last 5 runs
- **resolved** — was in prior ledger as failing, now passing → remove from active ledger, log to a `resolved` archive list (don't just delete silently — the manager report should be able to say "3 resolved since last week")

If no prior ledger exists (first run), every failure is `new` by definition
— state this explicitly in output rather than letting it look like a spike.

## 5. Output templates

### Manager status report (concise)
```markdown
## Weekly Test Report — [date range]

**Summary:** X passed / Y failed / Z broken (N% pass rate, [+/-]M pts vs last week)

**Needs attention:**
1. [Recurring, 3rd week] <test name> — <one-line category + cause>
2. ...

**New failures this week:** N (see appendix)
**Resolved since last week:** N
**Flaky (unreliable, not code issues):** N — excluded from pass-rate trend
```

### Engineering-detail report
Full table: test name | suite | category | state | consecutive count |
failed step | message excerpt (≤1 line) | trace pointer. Sorted by
state priority: recurring → new → flaky → one-off.

## 6. Explicitly out of scope
This skill produces analysis and the ledger only. It does not:
- Auto-file defects in Azure DevOps (separate, higher-governance-risk workflow)
- Attempt to auto-heal or auto-fix failing tests
- Modify test code

These are intentionally left to the test-healing workflow so this skill
stays low-risk and safe to run unattended in the weekend automation job.
