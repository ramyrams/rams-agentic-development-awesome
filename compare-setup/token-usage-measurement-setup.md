# Technical Setup: Capturing Token Usage for the Setup Comparison

Six methods, ordered by accuracy. Pick based on what your network/security policy allows — recommendation is at the end.

---

## Method 0 — VS Code Chat Debug View (simplest direct route — start here)

**What it gives you:** the exact `usage` JSON for each interaction, including `prompt_tokens`, `completion_tokens`, and `cached_tokens` — no proxy, no extension, no manual tokenization.

**Setup:**
1. Command Palette → `Developer: Show Chat Debug View` (or Copilot Chat panel → `⋯` overflow menu → "Show Chat Debug View").
2. Run a trial. Each entry lists system prompt, user prompt, context, and response.
3. Expand an entry's raw JSON under **Metadata** to see `prompt_tokens`, `completion_tokens`, `total_tokens`, and `cached_tokens` directly.
4. Copy the metadata block per trial into your results file.

**Limitation:** it's a raw debug panel, not an exportable dashboard — still manual copy-per-trial, and undocumented enough that field names/availability could shift between VS Code/Copilot versions. Verify it shows `cached_tokens` in your installed version before building your whole protocol around it; if it doesn't, fall back to Method 3.

---

## Method 1 — VS Code Language Model API (`vscode.lm.countTokens`)

**What it gives you:** exact token count against the *actual model's tokenizer*, via VS Code's built-in Language Model API — the most authoritative source for a single string's token count.

**Setup:**
1. Scaffold a minimal extension: `npm install -g yo generator-code` → `yo code` → choose "New Extension (TypeScript)".
2. In `extension.ts`, get a model reference:
   ```ts
   const [model] = await vscode.lm.selectChatModels({ vendor: 'copilot', family: 'gpt-4o' });
   const count = await model.countTokens(promptText);
   ```
3. Feed it text you've captured elsewhere (this API counts a string you give it — it doesn't automatically know what Copilot Chat actually assembled and sent).
4. Log to an output channel or write to a file per trial.

**Limitation:** this counts whatever text you hand it, not automatically the real injected prompt. You still need Method 3 to capture what Copilot actually assembled (instructions file content, skill content, retrieved context), then feed *that* text into `countTokens` for an exact count. Pair it with Method 3, not standalone.

---

## Method 2 — Network-level proxy capture (most accurate for real usage numbers)

**What it gives you:** the literal `usage` object (`prompt_tokens`, `completion_tokens`, `total_tokens`) from the API response itself — ground truth, no counting or estimation involved.

**Setup:**
1. Install mitmproxy (`brew install mitmproxy` or equivalent).
2. Run `mitmproxy` or `mitmweb`, configure your system/VS Code proxy settings to route through it (`http.proxy` / `https.proxy` in VS Code settings, or OS-level proxy).
3. Install and trust mitmproxy's CA certificate (required to decrypt TLS — see mitmproxy docs for your OS).
4. Run a trial in Copilot Chat; in mitmproxy's UI, find the request to the Copilot completions endpoint and inspect the JSON response body for the `usage` field.
5. Export request/response pairs per trial to JSON files for your results table.

**Before you do this:** check with IT/security first. This is a device-level TLS interception on an enterprise-managed machine and may violate policy, be blocked by MDM, or need explicit sign-off — flag it as a request, don't just run it. Also note Copilot's endpoint/response schema is undocumented and can change without notice, so don't build long-term tooling on it.

---

## Method 3 — VS Code Copilot Chat trace logging

**What it gives you:** the full assembled prompt Copilot Chat actually sent (system prompt + your instructions file / skill content + any injected file context) plus the completion text — this is what tells you *what's actually different between Setup A and Setup B*, which matters as much as the token count itself.

**Setup:**
1. Command Palette → `Developer: Set Log Level...` → select `GitHub Copilot Chat` → set to `Trace` (or `Debug` if Trace isn't offered in your version).
2. Open the Output panel (`View → Output`), select the `GitHub Copilot Chat` channel from the dropdown.
3. Run one chat trial. The full prompt construction and response get logged there.
4. Copy each trial's log block into its own transcript file (`task03_setupB_trial2.txt`), before running the next trial — the log is append-only and will otherwise blur together.

**This is your primary evidence source.** Feed these transcripts into Method 1 or Method 4 for token counts, and also keep them as the audit trail for your acceptance-criteria scoring.

---

## Method 4 — Offline tokenization with `tiktoken`

**What it gives you:** a fast way to batch-count tokens across many saved transcripts (from Method 3) without a live model reference.

**Setup:**
```bash
pip install tiktoken
```
```python
import tiktoken, glob, csv

enc = tiktoken.get_encoding("o200k_base")  # or cl100k_base — match closest to your pinned model

rows = []
for f in glob.glob("transcripts/*.txt"):
    prompt, completion = split_prompt_completion(f)  # your own parser for the trace format
    rows.append({
        "file": f,
        "prompt_tokens": len(enc.encode(prompt)),
        "completion_tokens": len(enc.encode(completion)),
    })

with open("token_results.csv", "w", newline="") as out:
    writer = csv.DictWriter(out, fieldnames=rows[0].keys())
    writer.writeheader()
    writer.writerows(rows)
```

**Caveat, state this explicitly in your report:** Copilot's backend model tokenizer may not be an exact match for the public `tiktoken` encodings. Label these numbers as **estimates**, not ground truth, unless corroborated by Method 2's `usage` field on at least a sample of trials. If Method 2 is available even for a handful of tasks, use it to validate/calibrate your `tiktoken` estimates before trusting them across the full corpus.

---

## Method 5 — GitHub Copilot Enterprise/Business admin metrics

**What it gives you:** org-level aggregates (active users, suggestions accepted, chat turns, lines of code) via the Copilot Metrics API or admin console.

**Setup:** requires org admin access; query the Copilot usage/metrics endpoint for your org.

**Not useful for this experiment** — no per-request token breakdown, only aggregate engagement stats. Use only as a sanity check on projected scale numbers after your controlled comparison is done, not as a measurement method for the comparison itself.

---

## Cached tokens — track this as its own column, not folded into the total

A **cached token** is a prompt token the backend already had in its KV-cache from a recent prior call (usually the stable prefix — system prompt, instructions file content), so it's reprocessed from cache instead of computed fresh. Cached tokens are typically billed at a steep discount (~90% cheaper) versus fresh input tokens, so comparing raw `total_tokens` between setups without splitting out cached vs. fresh **overstates the real cost difference**.

This matters directly for your comparison:
- **Vanilla setup** likely has a very stable prompt prefix (same instructions file every call) → high cache-hit rate → its effective cost is lower than raw totals suggest.
- **Agentic setup** may have a less stable prefix if routing logic or dynamically loaded skill/tool content varies by task → potentially lower cache-hit rate, or a different but still favorable rate if the same skill gets reused across similar tasks.

For every trial, capture and record separately: `prompt_tokens` (total), `cached_tokens` (subset of prompt_tokens served from cache), `completion_tokens`. Add a **fresh (billed) tokens = prompt_tokens − cached_tokens** column to your results table, and compute your tokens-per-accepted-task metric on *fresh* tokens, not raw totals — that's the number that maps to actual cost. Report the cache-hit ratio per setup as its own finding; it's likely to be one of the more interesting results, not just a footnote. Methods 0 and 2 give you `cached_tokens` directly; Method 4 (`tiktoken`) cannot compute this on its own since caching is a backend runtime behavior, not a property of the text — so if you're relying on Method 4, you need at least a sample of trials validated against Method 0 or 2 to know your real cache-hit rate.

---

## Recommendation

| Priority | Method | Why |
|---|---|---|
| 1 | Method 0 (Chat Debug View) | Simplest, gives exact `prompt_tokens`/`completion_tokens`/`cached_tokens` with no setup overhead — try this first |
| 2 | Method 3 (trace log) + Method 2 (proxy, if IT approves) | If Method 0 doesn't expose `cached_tokens` in your version — trace log gives real prompt content, proxy gives exact billed and cached counts to validate against |
| 3 | Method 3 (trace log) + Method 4 (tiktoken, calibrated) | If proxy isn't approved and Method 0 is unavailable — same real prompt content, estimated fresh-token counts; cache-hit rate must be sampled from Method 0/2 separately |
| Skip for this test | Method 1 alone, Method 5 | Method 1 needs pairing with Method 3 to be meaningful; Method 5 is aggregate-only, no cache breakdown |

Try Method 0 first before setting up anything heavier — if it exposes `cached_tokens` in your VS Code/Copilot version, it alone may cover the whole experiment. Get IT sign-off on Method 2 early only if Method 0 turns out insufficient — it's the one dependency that can block your timeline if raised late.
