# Index — Vanilla vs. Agentic Copilot Setup: Full Case & Test Plan

## Overview

Our QA automation team currently uses a custom agentic setup — agents, skills, and instructions organized in the `.github/` folder — built specifically to produce consistent, deterministic responses from GitHub Copilot. This approach has been challenged by a team member who argues that a simple, single `copilot-instructions.md` file uses fewer tokens than our agentic setup, raising the question of whether the added complexity is justified.

## Purpose

To validate both setups — the simple ("vanilla") instructions file and the current agentic setup — using GitHub Copilot in VS Code, and to compare their token usage directly through a controlled, repeatable test. The goal is a comprehensive, step-by-step method for running both setups side by side and measuring token consumption, producing evidence-based results that determine which setup is genuinely better for the team's long-term use — not just cheaper per call, but more reliable and cost-effective at scale.

---

Six documents total (this one plus five). Read in this order — each one sets up the next.

| # | Document | What it's for | Status |
|---|---|---|---|
| 1 | *(this index)* | Orientation — what exists, in what order, what's proven vs. not yet | — |
| 2 | `architecture-rationale.md` | The scale-independent argument — separation of concerns, coupling/cohesion, layered baseline-plus-skills structure, plus industry/vendor validation with sources | Sound independent of any test; now includes external validation |
| 3 | `case-for-agentic-setup-scenarios.md` | The practical case — 12 scenarios showing why agentic wins, mechanism by mechanism, with worked (illustrative) examples | Argument, not evidence |
| 4 | `vanilla-vs-agentic-copilot-comparison-methodology.md` | The test design — task corpus, acceptance specs, N-trial protocol, scoring | Design only, not yet run |
| 5 | `token-usage-measurement-setup.md` | The technical how-to — six methods for capturing token/cached-token data in VS Code Copilot, ranked by accuracy | Setup guide, not yet executed |
| 6 | `related-concepts-token-comparison-validity.md` | The checklist — eight concepts (premium requests, model routing, cache TTL, etc.) that can invalidate the result if skipped | Checklist, apply before/during the test |

---

## How these fit together

**Doc 2 is the "why, independent of measurement."** Separation of concerns, coupling/cohesion, Single Responsibility, blast-radius isolation, and the layered-architecture decision rule — this argument holds regardless of what the token study eventually shows.

**Doc 3 is the "why, in practice."** It makes the case on concrete scenarios (task diversity, drift, reuse, orchestration, governance, tool scoping) and illustrates the expected mechanism in each. It does not contain measured data — its worked example (Scenario 6) is explicitly a hypothetical table showing the *shape* of the tokens-per-accepted-task argument, not a result.

**Doc 4 is the "how you'd prove it."** It turns Doc 3's scenarios into a controlled, scoreable experiment: two branches, a real task corpus with acceptance specs, 5 trials per task per setup, and the tokens-per-accepted-task metric that Doc 3 argues should be the real comparison unit.

**Doc 5 is execution detail for one part of Doc 4** — specifically, how to actually capture the token numbers Doc 4's protocol calls for, including the cached-token distinction that materially changes the real cost comparison.

**Doc 6 is a pre-flight and mid-flight checklist** — read it before starting Doc 4's protocol (items 1, 2, 6 can invalidate the whole result if missed: premium-request multipliers, model auto-routing, and cache-warm bias from rapid-fire trials) and again before publishing (items 3, 4, 5, 7, 8 strengthen the report's defensibility).

---

## What's proven vs. not, right now

- **Proven / architecturally sound, independent of any test:** Doc 2's separation-of-concerns and coupling/cohesion reasoning for why agentic is the correct shape at scale.
- **Argued but not measured:** every scenario in Doc 3, including the illustrative tokens-per-accepted-task table.
- **Designed but not run:** the actual experiment in Doc 4.
- **Not yet started:** IT sign-off for proxy-based measurement (Doc 5, Method 2) if Chat Debug View (Method 0) doesn't expose `cached_tokens` in your VS Code version.

## Recommended sequence to present to the team

1. Lead with Doc 2's architecture reasoning (separation of concerns, blast radius, layered baseline-plus-skills structure) — this stands on its own regardless of the token study's outcome.
2. Concede the true part of the challenge up front: vanilla is cheaper per call. Reframe to tokens-per-accepted-task as the correct unit.
3. Present Doc 3's scenarios as the expected mechanism, clearly labeled as argument/hypothesis.
4. Propose Doc 4's protocol as how you'll confirm it, with Doc 6's pre-flight items (premium requests, model pinning, cache TTL) already accounted for.
5. Run the test. Report the real numbers — including if any come back weaker than expected, per Doc 3's own "where this argument is weaker" section.

## Still to do (not yet written)

- The actual test run (Doc 4 + Doc 5), and a results doc once it's done.
