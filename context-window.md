**The context window isn't just "chat memory."** It's a shared token budget split across system instructions, tool/MCP definitions, injected file/repo context, conversation history, and reserved output space for the response itself. That last part is why a brand-new chat can already show 30–40% usage before you type anything — it's deliberate headroom, not a bug.

**Sizes today:** standard is ~128K–192K tokens across most surfaces; a 1M-token extended window rolled out June 4, 2026 for select frontier models (Claude Sonnet/Opus 4.6+, Claude Sonnet 5, GPT-5.3-Codex and newer), but **only in VS Code and Copilot CLI** — not GitHub.com Chat, the Copilot app, Visual Studio, JetBrains, Xcode, or Eclipse. It's opt-in per session and costs more AI credits, alongside a separate configurable-reasoning-level dial.

**Two friction points worth knowing for governance work:**
- Context state doesn't reliably carry over when you switch models mid-session — there's an active VS Code issue thread on this from July 2026.
- MCP servers and custom instruction stacks add fixed per-turn overhead regardless of what the user actually asks — worth treating as a design constraint when scoping enterprise rollouts, similar to how you'd budget any other fixed cost.

The doc breaks down surface-by-surface availability, what counts against the budget vs. what just changes *what's sourced into* it (Spaces, knowledge bases, indexing), and six practical takeaways for writing governance guidance that won't go stale as models retire (several Opus/Sonnet 4.x models are scheduled to retire September 1, 2026).


# GitHub Copilot Context Window — Reference Guide

*Last verified: August 2026*

---

## 1. What "context window" means in Copilot

The context window is the total amount of text (measured in tokens) that a model can "see" in a single request — everything Copilot sends to the underlying LLM plus everything the model is allowed to generate back. It is not just "how much chat history is kept." It is a shared budget split across several competing consumers:

1. **System instructions** — Copilot's own baseline prompt plus any custom instructions (repo-level, org-level, personal).
2. **Tool/function definitions** — schemas for every tool the agent can call (file edit, terminal, MCP tools, etc.). More tools connected = more budget consumed before you type anything.
3. **Injected context** — open files, selected code, workspace/repo indexing results, `@workspace` results, attached issues/PRs, Spaces content, knowledge base hits.
4. **Conversation history** — prior turns in the session.
5. **Reserved output** — space pre-allocated for the model's *response*, so a long generation isn't truncated mid-answer.

Because (1), (2), and (5) are fixed overhead, a brand-new chat session already shows meaningful usage before you send a real prompt — commonly reported around 30–40% on a 192K-token window, mostly reserved output and tool-definition overhead. This is by design, not a bug: it guarantees the model has room to finish its answer.

---

## 2. Context window sizes: what's actually available today

| Tier | Typical size | Notes |
|---|---|---|
| Default / standard | ~128K–192K tokens | Baseline for most GA models across IDEs |
| Extended (1M token) | 1,000,000 tokens | Opt-in, available on select frontier models only |

**Extended 1M-token context** was rolled out starting June 4, 2026, alongside configurable reasoning levels. Key constraints:

- Currently supported models include Claude Sonnet 4.6, Claude Opus 4.6/4.7/4.8, Claude Opus 5, Claude Sonnet 5, Claude Fable 5, and select GPT-5.x models (GPT-5.3-Codex, GPT-5.4, GPT-5.5, GPT-5.6 family) and Kimi K3.
- **Only available in VS Code and Copilot CLI** — not yet in Copilot Chat on GitHub.com, the GitHub Copilot app, Visual Studio, JetBrains, Xcode, or Eclipse.
- It's a per-session *choice*, not automatic — when you select a supported model, you explicitly pick standard vs. extended context.
- **Cost trade-off**: larger context and higher reasoning levels both consume more AI credits. GitHub's own guidance is to default to standard context/reasoning and reserve the 1M window for genuinely large multi-file or long-document work.

---

## 3. What consumes the budget in practice

| Consumer | Behavior |
|---|---|
| Custom instructions (repo/org/personal) | Loaded on every turn; the more instruction files you stack, the larger the fixed overhead |
| MCP tool definitions | Every connected MCP server adds its tool schemas to the budget *before* any user content — this is a known cause of unexpectedly high baseline usage |
| Repository indexing / `@workspace` | Pulls in ranked, relevant file excerpts, not the whole repo |
| Copilot Spaces / knowledge bases | Pre-curated context you've explicitly attached — persistent across the session |
| Reserved output | ~30% or more of the window is commonly held back for the response, especially on IDE chat |

---

## 4. Conversation history management: summarization vs. hard limits

Copilot does not simply let a long-running chat overflow the window. Two mechanisms interact:

1. **Auto-summarization** — Copilot (in VS Code) can automatically condense earlier turns of a long agent conversation to make room for new ones. This is controllable via the setting `github.copilot.chat.summarizeAgentConversationHistory.enabled`.
2. **Per-model budget re-evaluation on model switch** — this is a live, actively-discussed edge case: switching models *mid-session* does not reliably preserve your "percentage full" state. Developers have reported that switching from one 1M-context model to another 1M-context model mid-conversation can show a dramatically different (much smaller) effective window, then partially "recover" when switching back. The mechanism isn't fully transparent from the outside, but the practical takeaway is: **don't assume context state is portable across a model switch in the same session** — verify usage after switching, especially on long-running work.

---

## 5. Surfaces: context window behaves differently everywhere

Copilot's context window is not one universal number — it depends on where you're running it:

| Surface | Extended (1M) available | Configurable reasoning | Notes |
|---|---|---|---|
| VS Code | Yes | Yes | Most complete context tooling; per-session model + context + reasoning picker |
| Copilot CLI | Yes | Yes | Has its own dedicated "Context management" concept page — worth reading if you're scripting/automating |
| Copilot cloud agent | No (not listed) | Yes | Runs autonomously against a repo; context sourced from repo + issue/PR + custom agent config |
| GitHub Copilot app | Not yet (rolling out) | Yes | |
| Copilot Chat on GitHub.com | No | No | Standard window only |
| Visual Studio / JetBrains / Xcode / Eclipse | No | No | Standard window only |

Because governance and rollout usually span multiple surfaces, this matters: a policy or training plan written around VS Code behavior (1M context, reasoning levels) will not transfer to Copilot Chat on GitHub.com or JetBrains without adjustment.

---

## 6. Related context *sources* worth distinguishing from the window itself

These don't change the window's size, but they change what competes for it:

- **Copilot Spaces** — curated, reusable context bundles (files, docs, links) attached deliberately to ground responses for a task.
- **Copilot knowledge bases** — org-level Markdown documentation aggregated across repos, used as retrieval context for Copilot Chat in GitHub.
- **MCP (Model Context Protocol)** — extends what Copilot can *do*, but every connected server's tool schema is loaded into context overhead on every turn — a governance-relevant cost, not just a capability.
- **Content exclusion** — lets admins prevent specific files/paths from ever being sent as context (security/compliance control, not a size control).
- **Repository indexing / semantic indexing** — improves *what* gets pulled into context for code questions, not how much room there is.

---

## 7. Known friction points (from active GitHub Community / VS Code issue threads, 2026)

- **"Why can't I use the full context_window?"** — the API reports a large nominal `context_window` (e.g., 400K), but usable capacity is meaningfully smaller once system prompt, tool definitions, and reserved output are subtracted. This is consistent across multiple community threads and is treated as expected behavior, not a defect.
- **Reserved-output confusion** — new chats showing 30–40% usage with zero user content is the single most common point of confusion; GitHub's position is that this is deliberate headroom, not reclaimable.
- **Model-switch context volatility** — see Section 4. Actively being tracked as a VS Code issue as of mid-2026; treat as an open/evolving area rather than settled behavior.

---

## 8. Practical guidance (architect-level takeaways)

1. **Treat context budget as a design constraint, not an afterthought.** Every custom instruction file, every MCP server, every attached Space adds fixed overhead to *every single turn* — audit these the way you'd audit a service's memory footprint.
2. **Default to standard context/reasoning; escalate deliberately.** The 1M window and higher reasoning levels are cost multipliers on AI credits. Reserve them for genuinely large refactors, cross-file work, or long documents — not as a default setting.
3. **Don't assume context portability across model switches mid-session.** If a workflow depends on long-running context (e.g., an extended refactor), pin the model for that session rather than switching mid-stream.
4. **Separate "context window management" from "context sourcing" in any governance framework.** Spaces, knowledge bases, MCP, and repository indexing all affect *what* Copilot sees; only model choice and the standard/extended toggle affect *how much room* there is. These need separate policy levers.
5. **Surface-by-surface parity is not guaranteed.** If you're building enterprise guidance or training material, confirm which surface (VS Code vs. Chat vs. CLI vs. cloud agent vs. JetBrains) each recommendation applies to — extended context and reasoning controls are VS Code/CLI-first as of mid-2026 and expanding gradually.
6. **Watch the retirement calendar when governing model access.** Several context-relevant models (Claude Opus 4.5/4.6, Claude Sonnet 4.5/4.6, Gemini 3.1 Pro) are scheduled for retirement around September 1, 2026 — any governance documentation naming specific models should track GitHub's model retirement table rather than hardcoding model names.

---

## Sources

- GitHub Changelog — *Larger context windows and configurable reasoning levels for GitHub Copilot* (June 4, 2026)
- GitHub Docs — *Supported AI models in GitHub Copilot* (models-with-extended-capabilities, retirement history)
- GitHub Community Discussions — context_window utilization and reserved-output threads (Feb–Mar 2026)
- VS Code GitHub Issues — context window size discrepancy on model switch (Jul 2026)
