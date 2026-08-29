# Vanilla Copilot Instructions vs. Agentic Setup — Comparison Methodology

**Purpose:** Settle the "simple instructions are cheaper" challenge with data, not opinion. Measures token cost AND outcome quality/determinism together, because token cost per call is meaningless without knowing how many calls it takes to get a correct result.

**Builds on:** existing `qa-eval-harness` (with_skill/without_skill comparison pattern, trajectory grading, failure taxonomy).

---

## 1. Reframe the hypothesis before testing

The challenge as posed ("simple instructions take fewer tokens") is true but incomplete — it's a per-call cost comparison, not a total-cost-of-outcome comparison. State the real hypothesis you're testing:

> "For QA automation script-generation tasks, the agentic setup (agents/skills/instructions) produces correct, standards-compliant output in fewer total tokens and fewer iterations than the vanilla setup, once retries/corrections are counted — and with lower output variance."

This framing is defensible in front of the team because it doesn't dodge the token question — it answers it at the right unit of measurement: **tokens per accepted (correct) output**, not tokens per call.

## 2. Define the two setups precisely (control everything else)

| Dimension | Setup A — Vanilla | Setup B — Agentic |
|---|---|---|
| Config | Single `.github/copilot-instructions.md` with all conventions/standards inline | Existing `.github/` structure: chatmode/agent, skills, scoped instructions, prompt files |
| Model | Same model, same version, pinned | Same model, same version, pinned |
| Repo | Same repo/branch, same starting file state | Same repo/branch, same starting file state |
| Context available | Whatever's in the single instructions file | Whatever the routed agent/skill loads for that task type |
| Tester | Same person runs both, same session hygiene (new chat per trial, no carryover) | Same |

Do this as two branches of the same repo (`experiment/setup-a-vanilla`, `experiment/setup-b-agentic`) so nothing else differs. If Setup A and Setup B aren't apples-to-apples on everything except the customization structure, the token comparison is meaningless regardless of the result.

## 3. Build a representative task corpus (10–15 tasks, tiered)

Pull from real backlog items, not synthetic ones — reviewers will trust it more. Tier by difficulty since that's usually where vanilla setups fall apart:

- **Tier 1 (simple, 3–4 tasks):** single Cypress test for a straightforward flow (e.g., login form validation)
- **Tier 2 (moderate, 4–5 tasks):** multi-step flow with data setup/teardown, needs to follow your team's page-object and naming conventions
- **Tier 3 (complex, 3–4 tasks):** flow requiring conditional logic, API mocking, or cross-cutting standards (accessibility checks, retry/wait patterns, Allure tagging)

Each task gets a written **acceptance spec** up front (this is your ground truth for scoring — same pattern as your `evals.json`): required assertions present, naming convention followed, no hardcoded waits, correct tagging, etc.

## 4. Token measurement — the part Copilot Chat doesn't give you for free

VS Code's Copilot Chat UI doesn't surface token counts natively. Three viable options, in order of effort/accuracy:

1. **Best: capture raw request/response and count with `tiktoken`.**
   Enable verbose logging: VS Code settings → `github.copilot.advanced.debug.overrideLogLevels` (or `Developer: Set Log Level` → GitHub Copilot Chat → Trace). This logs the full assembled prompt (system + instructions + injected file context) and completion per turn to the Output panel. Save each transcript to a file per trial, then run a small script post-hoc:
   ```python
   import tiktoken
   enc = tiktoken.encoding_for_model("gpt-4o")  # match whichever model you're pinned to
   prompt_tokens = len(enc.encode(prompt_text))
   completion_tokens = len(enc.encode(completion_text))
   ```
   This is the only method that captures the *actual* injected context (instructions file content, skill file content, retrieved chunks) — which is the whole point, since that's what differs between setups.

2. **If you have GitHub Copilot Enterprise/Business admin access:** the Copilot metrics/usage API gives aggregate suggestion and chat request counts, but not per-request token breakdowns — useful for corroborating scale patterns, not for this granular comparison. Treat as supplementary, not primary.

3. **Fallback:** manually copy prompt+response pairs from chat history into a token counter (e.g., OpenAI's tokenizer tool) if trace logging isn't available in your environment. More manual, same accuracy, just slower to run across 10-15 tasks × 5 reps × 2 setups.

Go with option 1. It's the only one that gives you the number your skeptical teammate will actually accept, because it's traceable back to the real logged prompt.

## 5. Determinism / consistency measurement

This is the piece the "fewer tokens" argument ignores entirely, and it's your strongest counter-evidence if the agentic setup wins on it.

For each task, run **N = 5 trials** per setup, fresh chat session each time (no conversation carryover — that would let the model "remember" and artificially stabilize vanilla results).

Score each trial's output against the acceptance spec (pass/fail per criterion), then compute per task:

- **Pass rate:** trials passing all acceptance criteria ÷ 5
- **Structural variance:** pairwise diff/similarity across the 5 outputs (e.g., normalized Levenshtein or AST-level diff for the generated test code) — low variance = deterministic, high variance = the setup is guessing
- **Convention-adherence rate:** did it follow naming/tagging/pattern standards without being re-prompted, across all 5

This reuses your existing trajectory-grading approach from the eval harness almost directly — same instrumentation, just run twice (once per setup) per task.

## 6. Step-by-step execution protocol

1. Freeze the task corpus and acceptance specs (step 3) — get sign-off from the challenger on the task list itself, so the result can't be dismissed as cherry-picked.
2. Set up the two branches (step 2), confirm no config bleed between them.
3. Enable trace logging (step 4) and do one dry-run task in each setup to confirm you're capturing full prompt+completion text, not truncated logs.
4. For each of the 10–15 tasks:
   - Run 5 trials in Setup A (fresh session each), save transcripts
   - Run 5 trials in Setup B (fresh session each), save transcripts
5. Post-process all transcripts: token count (step 4) + acceptance scoring (step 5) into a single results table (task × setup × trial → prompt tokens, completion tokens, total tokens, pass/fail, variance score).
6. Compute per-setup aggregates:
   - Mean tokens per **attempted** task
   - Mean tokens per **accepted (passing)** task — this divides total tokens by pass rate, which is the number that actually matters
   - Pass rate by tier
   - Variance score by tier
7. Project to scale: multiply mean-tokens-per-accepted-task by your team's actual weekly task volume, for both setups, to turn this into a cost number leadership/skeptics can compare directly.

## 7. Reporting the result honestly

Whichever way it comes out, present both numbers, not just the favorable one:

- If agentic wins on tokens-per-accepted-task but loses on raw tokens-per-call: say that explicitly — "higher per-call cost, offset by needing correction on X% fewer tasks."
- If vanilla wins outright on simple (Tier 1) tasks but loses on Tier 2/3: that's a legitimate finding — it argues for a **hybrid** setup (vanilla instructions for trivial tasks, agentic routing for complex ones), which is a stronger, more credible position than "agentic wins everywhere."
- Include the variance/determinism numbers even if they don't move the token argument — "consistent" was your original design goal, and it's a separate, defensible axis from cost.

## 8. Deliverable structure

Package the result as: task corpus + acceptance specs, raw transcripts (for auditability), results table, and a one-page summary with the tokens-per-accepted-task chart plus pass-rate-by-tier chart. This is the artifact that ends the debate — it lets anyone re-run a single task and check your numbers, which is what makes it credible rather than just asserted.
