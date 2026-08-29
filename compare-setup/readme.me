
**Overview**

Our QA automation team currently uses a custom agentic setup — agents, skills, and instructions organized in the `.github/` folder — built specifically to produce consistent, deterministic responses from GitHub Copilot. This approach has been challenged by a team member who argues that a simple, single `copilot-instructions.md` file uses fewer tokens than our agentic setup, raising the question of whether the added complexity is justified.

**Purpose**

To validate both setups — the simple ("vanilla") instructions file and the current agentic setup — using GitHub Copilot in VS Code, and to compare their token usage directly through a controlled, repeatable test. The goal is a comprehensive, step-by-step method for running both setups side by side and measuring token consumption, producing evidence-based results that determine which setup is genuinely better for the team's long-term use — not just cheaper per call, but more reliable and cost-effective at scale.

Six documents total, all in `/mnt/user-data/outputs/`:

1. **`00-index-vanilla-vs-agentic-case.md`** — Overview, Purpose, reading order for the other five, and a status summary of what's proven vs. still to do.
2. **`case-for-agentic-setup-scenarios.md`** — The argument: 12 practical scenarios showing the mechanism for why agentic wins, plus a "where this argument is weaker" section.
3. **`vanilla-vs-agentic-copilot-comparison-methodology.md`** — The test design: task corpus, acceptance specs, N-trial protocol, scoring, reporting structure.
4. **`token-usage-measurement-setup.md`** — Technical how-to: six methods for capturing token and cached-token data in VS Code/Copilot, ranked by accuracy, with a recommendation.
5. **`related-concepts-token-comparison-validity.md`** — Eight concepts (premium requests, model routing, cache TTL, reasoning tokens, etc.) that can invalidate the comparison if skipped.
6. **`00-index-vanilla-vs-agentic-case.md`** *(same as #1 — listed once)*

So five distinct working documents plus the index. Reading order per the index: overview/purpose → architecture reasoning (currently in this chat, not yet its own doc — see index's "still to do") → scenarios → methodology → measurement setup → validity checklist.

One open item from earlier still stands: the separation-of-concerns/coupling-cohesion architecture argument from this conversation isn't written into any file yet. Want me to add it as its own short document, or fold it into the top of the scenarios doc?


Short answer: yes, your setup matches both the vendor-recommended architecture and where the wider industry has converged — with citations below, plus one gap worth knowing about.

**GitHub's own documented model is exactly your layered pattern.** Their official guidance describes it in the same three tiers you're using: custom instructions are the house rules, skills are playbooks, and custom agents are specialized teammates — you can use all three together, but they solve different problems. More specifically, repository custom instructions provide always-on background guidance that sets project-wide norms, which Copilot should generally follow for any task in that repo, while skills are modular knowledge packages that Copilot selectively activates based on the relevance of the request, unlike custom instructions which are always loaded into context for every interaction. That's precisely the baseline-plus-skills split from your architecture-rationale doc — you didn't invent a nonstandard pattern, you implemented the one GitHub itself documents.

**Skills are a cross-vendor open standard, not a GitHub-only pattern.** Skills follow the open Agent Skills standard, so the same skill works across VS Code, GitHub Copilot CLI, and the GitHub Copilot coding agent. Notably, this standard originated at Anthropic and is now vendor-neutral — Copilot Enterprise's own docs note personal skills can live at the recommended `~/.copilot/skills/` path, with a legacy-compatible `~/.claude/skills/` path, confirming Claude Code and Copilot converged on the same folder/file convention independently arriving at what you're doing.

**Real enterprise deployments use the same decomposition principle you're applying.** One financial-services example uses a senior code agent that decomposes complex engineering tasks and spawns specialized sub-agents that learn skills and complete work iteratively, with evaluation loops at each stage — same shape as your plan→explore→script→test pipeline with per-skill eval gates. And this isn't a niche bet: Gartner predicts 40% of enterprise applications will embed task-specific AI agents by the end of 2026, up from less than 5% at the start of the year.

**One gap worth flagging, not a red flag but a roadmap item:** as of early 2026, skills can only be created at the repository and personal levels — organization-level and enterprise-level skill support is on GitHub's roadmap. If your longer-term plan is a shared org-wide skill catalog across multiple repos/teams (which your reuse scenario assumes), that's not fully native yet — worth tracking as a platform dependency rather than something to build a workaround for prematurely.



