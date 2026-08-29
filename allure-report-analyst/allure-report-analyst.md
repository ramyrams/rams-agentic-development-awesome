## Overview and Purpose

**Solution:** Allure Failure Analysis (Agent + Skill)


# Allure Failure Analysis (Agent + Skill)

## Overview
This solution adds an AI-assisted failure analysis layer to the existing
Cypress/Playwright + Allure test automation pipeline. It runs after the
weekend Azure DevOps automation job completes, reads the raw Allure results,
and produces a failure-focused report — distinguishing genuine test/product
defects from environment noise, locator drift, and flaky tests, and tracking
each failure's status across runs (new, recurring, flaky, resolved).

It is implemented as two GitHub Copilot `.github` primitives:

- **`allure-failure-analyst` (agent/chatmode)** — orchestrates the workflow:
  locates the results, asks minimal scoping questions, invokes the skill,
  and formats output for the intended audience (manager status report or
  engineering-detail report).
  See [`chatmodes/allure-failure-analyst.chatmode.md`](./chatmodes/allure-failure-analyst.chatmode.md)

- **`allure-report-analysis` (skill)** — holds the durable logic: how to
  parse Allure result files, the failure classification taxonomy, and the
  cross-run failure ledger schema.
  See [`skills/allure-report-analysis/SKILL.md`](./skills/allure-report-analysis/SKILL.md)

## Purpose
1. **Reduce manual triage time.** Today, failure review is a manual
   Monday-morning read-through of the Allure HTML report. This solution
   pre-classifies and prioritizes failures so the team reviews a ranked
   summary instead of a raw list.
2. **Separate signal from noise.** Not every failure is a defect —
   environment blips and flaky tests currently get mixed into the same
   pass-rate number as genuine regressions. Classification keeps the
   reported pass-rate trend meaningful.
3. **Make recurrence visible.** A failure appearing for the third
   consecutive week currently looks the same as a first-time failure. The
   ledger makes recurrence explicit and flags long-standing failures as
   candidates for the test-healing workflow rather than continued manual
   re-triage.
4. **Produce a consistent, audience-appropriate artifact.** One underlying
   analysis, two output shapes — a concise report for the manager status
   update, and a detailed table for the automation team to action — so the
   same run doesn't require separate manual write-ups.
5. **Stay low-governance-risk.** The solution only reads results and writes
   a ledger — it does not file defects, modify test code, or attempt
   auto-healing. Those remain separate, higher-governance workflows,
   consistent with an execution/reporting-first sequencing for AI adoption.

## Inputs
- `allure-results/*-result.json` — required, per-test status/steps
- `allure-results/*-container.json` — required, fixture/hook status
- `allure-results/history/history.json` — optional, improves flaky detection
- `allure-failure-ledger.json` — optional on first run, read + rewritten each run
- `categories.json` — optional, can seed/override classification taxonomy

## Open decision
Ledger (`allure-failure-ledger.json`) and `history/` persistence across
weekend Azure DevOps runs is not yet finalized — options under
consideration: commit to a state branch, Azure DevOps retained pipeline
artifacts, or an Azure Blob Storage account. See conversation history for
trade-offs.

## Out of scope
- Auto-filing defects in Azure DevOps
- Auto-healing or modifying test code



In Allure's raw results (before generating the HTML report), each test case produces a `*-result.json` file — that's where execution status lives.

**Key files in `allure-results/`:**

- **`<uuid>-result.json`** — one per test case. Contains:
  - `status` (passed/failed/broken/skipped/unknown) — overall test case status
  - `statusDetails` (message, trace) if failed/broken
  - `steps[]` array — each step has its own `name`, `status`, `start`/`stop` timestamps, and can nest sub-steps
  - `stage` (finished, running, etc.)

- **`<uuid>-container.json`** — one per fixture/container. Holds `befores[]`/`afters[]` (setup/teardown hooks), each with their own `status`.

- **`<uuid>-attachment.*`** — screenshots, logs, etc., referenced by result files.

**After generating the report** (`allure generate`), the aggregated data moves into `allure-report/data/`:
- `data/test-cases/<id>.json` — per-test detail including full step tree with statuses
- `data/suites.json`, `data/behaviors.json`, etc. — summarized views
- `widgets/summary.json` — overall pass/fail counts

**So concretely:** if you're parsing raw results, `*-result.json` → `steps[].status` gives you step-level status, and the top-level `status` field gives you the test case's overall outcome.

Want help writing a script to parse these (e.g., pulling all failed steps across a results folder)?

For GitHub Copilot specifically, here's what the agent actually reads at runtime — this matters because Copilot chatmodes don't have implicit filesystem access; they only see what's opened, referenced, or found via the `search`/`terminal` tools declared in the chatmode's frontmatter.

**Required inputs:**

| File | Where it comes from | Role |
|---|---|---|
| `allure-results/*-result.json` | Generated by your test run (Cypress + `allure-cypress` reporter) | Primary input — per-test status, steps, failure details |
| `allure-results/*-container.json` | Same test run | Fixture/hook (before/after) failures — environment vs. test-logic signal |

**Optional but used if present:**

| File | Where it comes from | Role |
|---|---|---|
| `allure-results/history/history.json` | Carried forward automatically if you copy the previous report's `history/` folder into the new `allure-results/` before generating | Allure's own built-in retry/flaky detection — the skill's flaky classification is more reliable if this exists, since it doesn't have to infer flakiness purely from your custom ledger |
| `allure-failure-ledger.json` | Produced by a prior run of this skill | Input **and** output — read at start of run to compute `state` (new/recurring/flaky/resolved), then rewritten at the end |
| `categories.json` | Optional, if your team defines Allure custom failure categories | Can seed/override the classification taxonomy in step 3 of the skill |

**How Copilot actually gets these into context — three options:**

1. **Manual reference:** you open the workspace with `allure-results/` present and type `@workspace` or drag the folder into chat, or reference `#allure-results` — Copilot's `search` tool then finds the JSON files itself.
2. **Terminal-mediated (recommended for this workflow):** since `tools: ['terminal']` is declared, the agent can run something like `find allure-results -name "*-result.json"` and `cat` them itself rather than requiring you to attach files — this is why I included `terminal` in the chatmode's tool list rather than relying only on `read_file`.
3. **Explicit path in your prompt:** the scope questions in the agent already ask for "path to `allure-results/`, or a specific Azure DevOps pipeline run/build number" — so the intended usage is you type something like `@allure-failure-analyst analyze ./allure-results` and it takes it from there.

One gap worth flagging: if your weekend Azure DevOps job doesn't persist and re-inject `allure-failure-ledger.json` and `history/` into the next run's `allure-results/` folder, the agent has no way to know "recurring" vs "new" — it'll only ever see one snapshot. Want me to add an explicit "ledger bootstrap" step to the chatmode that checks for and warns about a missing ledger, or is persistence something you're handling in the pipeline YAML separately?

Good question to settle first — let's do ledger-persistence, since the prompt file is trivial to write once we know where the ledger lives (it just needs to pass that location through), but the reverse isn't true: if we draft the prompt file first and then decide persistence, we'd likely have to rework it.

Three realistic options for persisting `allure-failure-ledger.json` and `history/` across weekend runs in Azure DevOps:

**1. Commit to the repo**
- Pipeline step commits the updated ledger back to a dedicated branch (e.g. `qa-reporting-state`) after each run.
- Pro: simple, versioned, diffable — you can literally see ledger history in git log.
- Con: adds a bot-commit into your repo history; needs a PAT/service account with write access.

**2. Azure DevOps Pipeline Artifacts (retained across runs)**
- Publish the ledger as a named artifact each run; next run downloads the artifact from the previous successful pipeline run before executing.
- Pro: no repo pollution, native ADO mechanism.
- Con: artifact retention policies can expire it; requires a "download last successful build's artifact" step in YAML, which is slightly fiddly but standard.

**3. External small store (Azure Blob Storage / Storage Account)**
- Pipeline reads/writes the ledger to a blob at a fixed path.
- Pro: most durable, no coupling to pipeline retention settings, easiest to also expose to the manager-report dashboard later if you build one.
- Con: one more Azure resource to provision and manage credentials for.

Given you're already in the Azure/Power Platform stack and this is a **governance-adjacent artifact** (failure trend history informs manager reporting), I'd lean toward **option 3** if you already have a storage account provisioned for other automation, otherwise **option 2** is the lowest-friction start — you can always migrate to blob storage later without changing the skill's logic, since the skill only cares that a ledger file exists at a known path when it starts.

Which one fits what you've already got provisioned?


Since you already have the allure-weekly-reporter/allure-report-analysis concept in your framework, I'll build this as the concrete pair — an agent that orchestrates the analysis and a skill that holds the reusable parsing/classification logic — following your `.github` primitive split (agent = "what to do and when," skill = "how to do it").

**Design rationale, in sequence:**

1. **Split by volatility, not by topic.** The *skill* holds things that rarely change once correct: how to parse `*-result.json`, how to classify a failure, the ledger schema. The *agent* holds things that change per run: scope (which pipeline, which date range), tone of the output, who it's for. This keeps you from having to edit parsing logic every time reporting cadence or audience changes.
2. **Classification logic lives in the skill, not in a prompt.** If failure-type buckets (assertion vs. locator vs. timeout vs. environment) only lived in a prompt file, every invocation would re-derive them inconsistently. Pinning them in the skill makes classification deterministic and testable — which matters since this will feed a failure ledger used for trend tracking.
3. **The ledger is the actual deliverable, the chat response is secondary.** New/recurring/flaky tracking only works if state persists across weekly runs. So the skill defines a ledger file format the agent reads-then-writes each invocation, rather than regenerating from scratch — otherwise "recurring" has no memory to compare against.
4. **Agent stays thin and asks one clarifying question up front (scope), then delegates.** This matches your Author/Architect ladder — a QA team member should be able to invoke this without knowing the classification internals.

I'll create both files now.Drop these directly into your `.github/chatmodes/` and `.github/skills/` folders — paths are already structured to match.

Two things worth deciding before you wire this into the weekend Azure DevOps job:

1. **Ledger storage location.** Right now the skill assumes `allure-failure-ledger.json` sits alongside results. If your pipeline agent is ephemeral, that file needs to persist somewhere durable (repo commit, Azure DevOps artifact, or a small blob store) or "recurring" tracking breaks every run.
2. **Where the `weekly-qa-report` prompt file plugs in.** This agent+skill pair produces the analysis; the prompt file you'd mentioned earlier is likely just the thin trigger ("run allure-failure-analyst against this week's build") — I kept that out of scope here so it stays a one-line prompt file rather than duplicating logic.

Want me to draft that prompt file next, or work through the ledger-persistence decision first?


