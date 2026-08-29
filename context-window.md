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


Here's what actually matters day-to-day, not just for governance:

**1. Your first message in a session isn't "free."** New chat already burns ~30–40% of the window on system instructions + reserved output + any MCP tools you've got connected. If you're running several MCP servers, that overhead compounds — long sessions hit the wall faster than the raw window size suggests.

**2. Keep `@workspace`/repo-indexed questions specific.** Broader questions pull in more ranked file excerpts, eating budget faster and diluting relevance. "Explain how auth middleware handles token refresh" beats "explain this codebase."

**3. Don't switch models mid-task and assume continuity.** If you're deep into a large refactor and switch from one model to another, your effective context usage can jump around unpredictably (this is an actively reported issue, not settled behavior). For long-running work, pick a model and stick with it for that session.

**4. Turn on the 1M window deliberately, not by default.** It's opt-in in VS Code/Copilot CLI only, and it costs more AI credits. Reach for it specifically for: large multi-file refactors, working across an unfamiliar big codebase, or long docs/specs — not as your everyday setting.

**5. Watch for auto-summarization silently condensing history.** In VS Code, long agent conversations get auto-summarized (`github.copilot.chat.summarizeAgentConversationHistory.enabled`). Useful for staying under budget, but it means Copilot's "memory" of turn 3 by turn 40 may be a compressed version, not the original. If precision on an early decision matters, restate it rather than trusting it survived summarization.

**6. Custom instructions are loaded every single turn.** Repo + org + personal instruction files all stack as fixed overhead on every request. If your `.github/copilot-instructions.md` (or equivalent) is bloated, you're paying for it on every prompt, including trivial ones. Keep it tight and specific.

**7. If a response feels truncated or Copilot seems to "forget" something from 20 messages ago,** context pressure (not a bug) is the first thing to suspect — start a fresh session or explicitly re-paste the relevant snippet rather than assuming Copilot still has it in view.

**8. Attach context instead of re-explaining it.** Spaces and knowledge bases let you pre-load stable reference material (style guides, API contracts) once, so you're not re-pasting the same context every session — cheaper and more consistent than repeating it in prompts.


Yes — but it's specific to **Copilot CLI**, not the VS Code chat panel (which uses automatic summarization instead, no manual slash command exposed to you).

**In Copilot CLI:**
- `/compact` manually compresses your conversation history on demand — you decide when.
- **Auto-compaction kicks in automatically at ~95% of the token limit**, compressing history in the background without interrupting your session. GitHub's framing is that this enables "virtually infinite sessions" — you don't hit a hard wall, you just get progressively compressed history.
- You can press `Escape` to cancel a compaction if it kicks in at a bad moment.

**In VS Code chat**, there's no user-facing `/compact` — instead there's the setting `github.copilot.chat.summarizeAgentConversationHistory.enabled`, which does something similar automatically but isn't a command you trigger yourself.

**Practical implications for you:**

1. **In CLI, use `/compact` proactively before a natural checkpoint** — e.g., right after tests pass or a feature is done — rather than waiting for the 95% auto-trigger. Compacting on your terms preserves the details *you* think matter; auto-compaction preserves what the model judges matters, which isn't always the same thing.
2. **Don't rely on compacted history for exact details.** If a specific decision, config value, or error message needs to survive long-term, put it in a file (a scratch notes file, a comment, `AGENTS.md`/repo instructions) rather than trusting it'll survive compaction faithfully.
3. **If you're bouncing between VS Code and CLI for the same task, know the compaction behavior differs** — CLI gives you a manual lever, VS Code doesn't (yet). That's a real workflow difference if you're used to Claude Code's `/compact` and expect the same control in Copilot's IDE chat.
4. **Compaction is a symptom-treater, not a budget-increaser.** It buys you longer sessions on the same window size — it doesn't get you more effective context per turn. For genuinely large multi-file work, that's still the 1M extended-context toggle's job, not compaction's.


Real toolkit here, both in CLI and IDE/Chat. Here's the practical set:

**In Copilot CLI:**

1. **`/usage`** — shows premium requests used this session, session duration, lines of code edited, and token usage broken down per model. This is your actual "am I close to the wall" check — use it before big tasks, not after you hit a limit.
2. **`/compact`** — manual compaction on demand (covered above).
3. **`@ FILENAME`** — inject a specific file's contents directly into context, instead of hoping the agent finds it via indexing. More precise, cheaper than a vague natural-language reference.
4. **`! COMMAND`** — run a shell command directly, bypassing the model entirely. Free — doesn't touch your token budget. Use this for anything you don't actually need reasoning on (checking git status, listing files).
5. **`/clear`** — wipes session context outright when you're switching tasks, rather than letting an unrelated task's history sit in your window as dead weight.
6. **`/session`** — session metrics, similar spirit to `/usage`.
7. **`/add-dir`** and **`/list-dirs`** — control and audit which directories Copilot can see. Useful for keeping context scoped to what's relevant (and for compliance trails, if that matters where you work).
8. **`/cwd /path`** — switch working directory without starting a new session, so you don't lose context you still need while moving to a different part of the repo.
9. **`Shift+Tab`** — cycles between standard, plan, and autopilot mode. Plan mode is worth using deliberately before a big multi-file change — it front-loads reasoning about *what* to do before burning context on *doing* it.
10. **`/delegate <prompt>`** — hands your current CLI session off to Copilot's cloud coding agent, preserving context, so a long task can keep running in the background instead of tying up your local session/window.
11. **`/mcp add`** — add MCP servers on demand rather than having them all loaded by default. Since every connected MCP server's tool schema eats context on every turn (see earlier), only attach what a given session actually needs.

**In VS Code / Copilot Chat:**

12. **`/clear` and `/new`** — same idea as CLI: start fresh rather than letting a stale, unrelated conversation keep consuming budget.
13. **`@` mentions** (`@workspace`, specific files, issues, PRs, repositories) — scope what gets pulled in, same principle as CLI's `@ FILENAME`. The narrower the mention, the less irrelevant content competes for context.
14. **`#` chat variables** — attach specific, named context (a symbol, a selection) rather than relying on the model to infer what you mean from prose.

**The general pattern across all of this:** every tool above is really one of three things — *see what you're using* (`/usage`, `/session`), *narrow what's loaded* (`@`, `/add-dir`, `/mcp add`, `/cwd`), or *reclaim space* (`/clear`, `/compact`). Building a habit around those three categories will get you further than memorizing individual commands.


Short answer: **there's no verified, universal "70% cliff" specific to Copilot** — I couldn't find that number in GitHub's own docs. But the underlying phenomenon you're pointing at is real and well-documented in LLM research generally, just not with a fixed number that applies everywhere.

**What's actually established:**

- Research (Liu et al., "Lost in the Middle") shows models are most accurate when relevant info sits at the *start* or *end* of context, and meaningfully worse when it's buried in the middle — regardless of how full the window is overall.
- Degradation isn't linear. Multiple studies describe a **plateau-then-cliff pattern**: quality holds up reasonably well, then drops sharply past some threshold — but that threshold varies a lot by model and task. One 2026 study found a critical collapse point around 40–50% of max context for one model family; another found 1M-token models degrading over 50% in some tasks by just 100K tokens (10% full). There's no single agreed percentage across models.
- Tool-calling/agentic tasks (which is what Copilot is doing constantly) degrade *faster* than plain Q&A as context fills — one study found 13–91% degradation depending on how many tools and how much tool-response text was in play.

**So the honest takeaway:** don't treat "70%" as a real threshold to watch on a meter. Treat "the conversation is getting long and unfocused" as the actual signal, regardless of what percentage it maps to.

**Practical best practices, building on what we've covered:**

1. **Start a fresh session per task, not per day.** The single biggest quality lever isn't compaction or thresholds — it's not letting unrelated history accumulate in the first place. If you're moving from "fix this bug" to "now write tests for something else," `/clear` or `/new` rather than continuing.

2. **Put the most important instruction last, not first.** Given the lost-in-the-middle effect, if you have a long prompt with a lot of context plus one critical constraint, state the constraint again at the end, right before you hit enter — it lands in the highest-attention zone.

3. **Don't paste the same large file/context repeatedly across turns.** It buries earlier turns further into the "middle" and makes the whole session more vulnerable to degradation. Reference it once, then refer back to it by name.

4. **For agentic/multi-file tasks, prefer narrow, sequential asks over one giant prompt.** Given how much tool-calling accuracy specifically degrades with context size, "refactor this one function, then the next" beats "refactor this entire module" as a single instruction.

5. **Treat a sudden drop in response quality as a context signal, not a model getting worse.** If Copilot starts giving vague, generic, or slightly-off-topic answers after a long session, that's the practical version of what the research calls degradation — the fix is `/compact`, `/clear`, or a fresh session, not rephrasing the same prompt harder.

6. **Trust `/usage`, not vibes, for the actual number.** Since there's no universal quality threshold, use the CLI's real token/request breakdown to decide when to compact — not a percentage you read somewhere.

7. **For anything that must be exact (a specific error message, a config value, an API contract), don't rely on it surviving a long session.** Given lost-in-the-middle plus summarization, restate or re-paste precision-critical details close to when you need them acted on, rather than trusting they're still "in view" from 30 turns ago.
8. 

