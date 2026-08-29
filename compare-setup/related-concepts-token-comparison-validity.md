# Related Concepts — Things That Can Invalidate the Token Comparison

Background concepts to understand before running the vanilla-vs-agentic test. Several of these directly affect whether the result is defensible — flagged where that's the case.

---

## 1. Premium requests vs. tokens (billing unit mismatch)

**Can invalidate the result if skipped.**

GitHub Copilot plans typically meter/bill in **premium requests**, not raw tokens, and different models carry different request multipliers. "Fewer tokens" and "cheaper" are not automatically the same claim — a model with a higher per-request multiplier can cost more overall even at a lower token count.

**Action:** pull your org's premium-request-multiplier table for whichever model you pin. Report cost in both units — tokens (technical argument) and premium requests (the number finance/leadership actually bills against).

---

## 2. Model routing ("Auto")

**Can invalidate the result if skipped.**

If Copilot Chat is set to auto-select a model rather than a pinned one, different turns can silently route to different models mid-experiment — different tokenizers, different pricing, different multipliers — without it being visible in the transcript unless you check.

**Action:** explicitly pin one model for the entire test window, in both setups. Confirm the pinned model in the debug/trace metadata for a sample of trials rather than assuming the setting held.

---

## 3. Context window limits and context rot

Two failure modes as the vanilla instructions file grows: hitting the context limit outright, and "lost in the middle" — the model attending less reliably to content buried in a long, undifferentiated block versus content near the start or end. This is a named mechanism, not just an assertion, and it's worth citing explicitly as the reason a monolithic file degrades with scale rather than just failing on token count.

---

## 4. Tool-definition overhead / tool search

Agentic setups typically expose more tools (skills, MCP tools, workspace actions), and every tool's schema sent up front adds fixed token cost per turn regardless of whether it's used. Newer Copilot versions defer full tool schemas until the model searches for one (lightweight name/description sent first, full schema loaded on demand).

**Action:** check whether your environment has tool search enabled — it materially changes the agentic setup's baseline token cost and is a moving target across Copilot releases. Note the Copilot/VS Code version you tested against in the report, since this behavior isn't stable across versions.

---

## 5. Reasoning tokens

If the pinned model is a reasoning model, completion usage can include hidden `reasoning_tokens` that are billed but not shown in the visible response text.

**Action:** check `completion_tokens_details` in the usage JSON (via Chat Debug View or proxy capture) — otherwise completion-token counts will look artificially cheap for reasoning models.

---

## 6. Cache TTL / cold vs. warm cache

**Can invalidate the result if skipped.**

Prompt caches expire after a period of inactivity (provider-dependent, often on the order of minutes). A controlled experiment run as rapid-fire back-to-back trials will show a much higher cache-hit rate than real team usage — developers working in bursts across a day, cache going cold between sessions.

**Action:** note this gap explicitly in the report, or add a deliberate cold-start trial variant per setup (pause past the cache TTL before running it) to bound the real-world range rather than reporting only the best-case warm-cache number.

---

## 7. Statistical rigor at N=5

Five trials per task is enough to see gross determinism differences but is a thin sample for a confidence interval. If a task's pass rate lands at a boundary (e.g., 3/5 vs. 4/5) and that task feeds the headline result, rerun it at N=10 before publishing — cheap insurance against a challenger picking apart the sample size.

---

## 8. Tokenizer non-portability

Token counts aren't comparable across different model families — a GPT tokenizer and a Claude tokenizer count the same text differently. Only matters if the numbers are later reused to compare across models rather than within the one pinned model used for this test.

**Action:** state the pinned model and tokenizer explicitly in the report so the numbers aren't later misapplied to a different model without re-measuring.

---

## Priority if time is short

Items 1, 2, and 6 are the ones most likely to invalidate the result outright if skipped — lock those down before the test window starts. Items 3, 4, 5, 7, 8 strengthen the report's credibility but won't by themselves produce a wrong headline number.
