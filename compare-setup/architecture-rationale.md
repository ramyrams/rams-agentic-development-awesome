# Architecture Rationale — Why Agentic Is the Correct Shape at Scale

This argument stands on its own, independent of the token measurement study — it's a standard software-architecture case, not an AI-specific one, and it holds regardless of what the token numbers eventually show.

## The core argument

1. **Separation of concerns.** Things that change for different reasons should be separated. A large project has genuinely distinct task types (script generation, healing, reporting, exploratory testing). A monolithic instructions file violates this — an Allure-tagging rule and a Cypress wait-pattern rule live in the same file and get deployed together, whether or not they're related.

2. **Coupling and cohesion.** A vanilla setup is maximally coupled — every task type's behavior is coupled to the same file, so nothing can change independently. An agentic setup gets high cohesion per skill (everything in one skill relates to one task type) and low coupling between skills (changing one doesn't risk the others). This is the same reasoning that justifies microservices over a monolith, or modules over a single script — it's not new to AI, it's the same principle applied here.

3. **Single Responsibility.** Each skill/agent should have one reason to change. A vanilla file has as many reasons to change as it has task types — architecturally, that's the definition of a god object.

4. **Scalability under growth.** An architecture is judged by how it behaves at scale, not just at current size. Vanilla's per-call cost and attention-reliability both degrade as task types are added; agentic's marginal cost per new task type is close to flat (new skill, no added cost to existing ones). The standard test for architectural soundness is what happens at 10x current scope, not what it costs today.

5. **Independent testability and blast-radius control.** At small scale, a bug in a shared file is annoying. At large scale, with many contributors and many task types, an unreviewed change to a shared file is a production incident waiting to happen. Isolated, independently eval-gated components are the standard mitigation for this — and it's what the per-skill eval-suite policy already assumes.

## Where pure agentic goes too far

A large project doesn't need every scrap of behavior split into a skill. Cross-cutting concerns — repo structure, general code style, security baseline rules — genuinely belong in one place, applied uniformly, the same way a linter config isn't split per module. Splitting things that don't vary by task type adds coordination overhead for no benefit — that's over-engineering, an architecture smell in the other direction.

## The correct shape: layered, not pure-agentic

A lean, stable baseline `copilot-instructions.md` for genuinely universal conventions (rules that apply to all task types and rarely change) + a skill/agent catalog for anything task-specific, variable, or independently evolving. Same layering pattern as a shared base config plus service-specific configs in any modular system.

**Decision rule:** if a rule applies to *all* task types and rarely changes → baseline instructions file. If a rule is specific to one task type, likely to change independently, or needs its own eval/versioning → its own skill.

This rule is what makes the architecture defensible in review — the position isn't "agentic because it's newer," it's "each piece of behavior lives at the layer matched to how often and independently it changes." That's a standard, well-understood architecture argument, and it holds up regardless of which side of the AI-tooling debate the reviewer sits on.

---

## Industry validation

This isn't a nonstandard pattern — it's the vendor-documented and cross-industry-converged architecture.

**GitHub's own documentation describes this exact three-tier model.** Their guidance frames custom instructions as always-on house rules, skills as on-demand playbooks, and custom agents as specialized teammates, each solving a different problem. Repository-level instructions provide always-on background guidance and project-wide norms that Copilot generally follows for every task in the repo, while skills are modular knowledge packages that Copilot selectively activates only when relevant — unlike instructions, which load into context on every interaction. That's precisely the baseline-plus-skills split argued above; it's the pattern GitHub itself documents, not something built outside vendor guidance.

**Skills are a cross-vendor open standard, not GitHub-specific.** The Agent Skills format is portable — the same skill works across VS Code, GitHub Copilot CLI, and the GitHub Copilot coding agent. The standard originated at Anthropic and is now vendor-neutral; GitHub Copilot Enterprise's own docs note personal skills can live at a path with a legacy-compatible Claude Code path alongside it, confirming multiple major AI vendors converged independently on the same skill-folder convention.

**Enterprise deployments use the same decomposition principle.** At least one large financial-services deployment runs a senior code agent that decomposes complex engineering tasks and spawns specialized sub-agents that learn skills and complete work iteratively, with evaluation loops at each stage — the same shape as a staged pipeline (e.g., plan → explore → script → test) with per-skill eval gates. This direction is also broadly forecast at scale: Gartner projects 40% of enterprise applications will embed task-specific AI agents by the end of 2026, up from under 5% at the start of the year.

**One current platform gap, worth tracking rather than working around prematurely:** as of early 2026, GitHub skills support only repository- and personal-level storage — organization-level and enterprise-level skill sharing is on GitHub's roadmap but not yet native. Any plan for a shared, org-wide skill catalog across multiple repos or teams should account for this as a dependency, not assume it's already solved.

**Sources:**
- GitHub Copilot CLI custom agents/skills overview — devleader.ca, July 2026
- "GitHub Copilot Instructions vs Prompts vs Custom Agents vs Skills vs X vs WHY?" — DEV Community, April 2026
- GitHub Copilot Enterprise docs, Chapter 14: Agent Skills — Writing & Managing
- "Understanding Enterprise AI Agents: The 2026 Guide to Deployment, Governance, and Scale" — Lyzr, 2026
- Gartner enterprise AI agent adoption forecast, cited in Neontri "Enterprise AI Agents: The 2026 Strategy, Selection, and Deployment Guide," June 2026
