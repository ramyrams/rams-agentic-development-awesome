# Agent Engineering: Prompt · Context · Harness · Skill
### Plus two underlying patterns (looping & graph orchestration) — with one shared scenario

**Why this matters:** "prompt engineering," "context engineering," "harness engineering," and
"skill engineering" are the actual industry-recognized lineage — each one is a named
discipline that shows up in vendor blog posts, papers, and conference talks (OpenAI's
"Harness Engineering," LangChain's harness-engineering work, and 2026's emerging "skill
engineering" discipline built around SKILL.md-style packaging).

**Looping** and **graph orchestration** are real and important — but they're *patterns*
inside the harness layer, not separately branded disciplines. It's worth keeping that
distinction clear for the team so nobody goes looking for a "looping engineer" job title.

---

## The shared scenario

We'll use the same example throughout so you can see exactly what changes at each layer:

> **An enterprise agent that lets employees request PTO (time off) through chat.**
> Employee types: *"I want to take next Friday and Monday off."*

---

## 1. Prompt Engineering
**Layer: the single instruction you give the model for one call.**

This is about *wording* — how you phrase the system prompt / instructions so the model
behaves the way you want on a single turn. No tools, no memory, no multi-step logic — just
"what do I tell the model to get a good response."

**What you're tuning:** tone, output format, role definition, few-shot examples, constraints.

**Applied to the scenario:**
```
System prompt:
"You are an HR assistant. When an employee requests time off, confirm the exact
dates, ask if it's a full day or half day, and respond in a friendly, concise tone.
Never approve or deny requests yourself — only collect and confirm details."
```
Change the prompt → the tone, structure, or scope of a single response changes. That's it.
If the model doesn't know the employee's remaining PTO balance, prompt engineering alone
can't fix that — it doesn't have that information.

---

## 2. Context Engineering
**Layer: what information is actually placed in the model's context window before it answers.**

This is about *retrieval and assembly* — pulling in the right data (policy docs, employee
records, tool outputs, conversation history, memory) so the model has what it needs, without
overflowing the context window with irrelevant noise.

**What you're tuning:** what gets retrieved, how it's formatted, what's summarized vs. included
in full, what's excluded.

**Applied to the scenario:**
Before the model responds, the system injects:
```
[Employee record: 6 PTO days remaining]
[Team calendar: 2 other team members already off next Friday]
[Company policy excerpt: "Requests require 3 business days notice"]
```
Now the same prompt from Layer 1 produces a *grounded* answer — the model can say "you have
6 days left, but note two teammates are already out that Friday" instead of guessing or
hallucinating. Context engineering is why the answer is *correct*, not just well-phrased.

---

## 3. Harness Engineering
**Layer: the surrounding software that governs how the model is invoked — tools, guardrails, retries, sandboxing.**

This is the *runtime scaffolding* around the model: which tools it's allowed to call, how
tool calls are validated, what happens on error, what it's blocked from doing, how output is
checked before it reaches the user or a system of record. This is the layer where **looping**
and **graph orchestration** (below) actually live — they're patterns you implement *inside*
the harness, not separate disciplines alongside it.

**What you're tuning:** tool access/permissions, input/output validation, error handling,
retry logic, rate limits, safety checks, and the control flow (loops/graphs) that governs
how many steps the agent takes.

**Applied to the scenario:**
```
Harness rules for this agent:
- Can call: get_pto_balance(), get_team_calendar(), submit_pto_request()
- Cannot call: approve_pto_request() (reserved for manager-facing agent)
- If submit_pto_request() fails → retry once, then escalate to a human HR rep
- Output is scanned to ensure no other employee's PII appears in the response
```
Same prompt, same context — but now the *system* enforces that this agent can request time
off on the employee's behalf but can never approve it, and it fails safely if the HR system
API is down. This is what makes the agent trustworthy enough to deploy, not just articulate.

---

## 4. Skill Engineering
**Layer: packaging a repeatable agent procedure into a portable, versioned, reusable capability.**

This is the newest named rung in the lineage (prompt → context → harness → **skill**). Instead
of re-deriving "how to handle a PTO request" inside a prompt or hard-coding it into the
harness every time, you package the procedure itself — steps, policy checks, edge cases,
expected inputs/outputs — as a discrete, testable, version-controlled unit (e.g. a SKILL.md)
that any agent can load on demand. It's the difference between the agent "knowing how" once,
tribal-knowledge style, buried in a prompt, versus the org owning and maintaining that
procedure as an asset.

**What you're tuning:** what counts as one reusable "skill," how it's documented and tested,
how it's versioned as policy changes, how it's discovered/loaded by the agent, who owns it.

**Applied to the scenario:**
```
skill: pto-request-handling/SKILL.md
  description: "Handles employee PTO requests end-to-end: parse dates,
    check balance, check team calendar, apply notice-period policy,
    flag conflicts, submit or route to manager."
  version: 1.3.0
  owner: HR Systems team
  inputs: employee_id, requested_dates
  procedure: [steps + branch conditions, tested against a fixed eval set]
  changelog: "1.3.0 — added 3-business-day notice check per updated policy"
```
Now "how to handle PTO requests" isn't buried inside one agent's system prompt — it's a
maintained asset. If HR changes the notice-period policy from 3 to 5 days, you version-bump
the skill once, and every agent that loads it (the employee-facing bot, a Slack integration,
a manager dashboard agent) picks up the fix — instead of hunting down and editing five
different prompts.

---

## Two underlying patterns (not separately named disciplines)

These live *inside* harness engineering — worth knowing by name, but don't present them to
the team as peers of the four disciplines above.

### Pattern A: The agentic loop ("looping")
**How many steps the agent takes within one task, and when it stops or loops back.**

```
Loop for this request:
1. Parse "next Friday and Monday" → resolve to actual dates
2. Call get_team_calendar() → observe 2 teammates already out Friday
3. Decide: policy allows it, but flag the overlap → ask employee to confirm anyway
4. If employee confirms → call submit_pto_request()
5. Observe result → if success, confirm to employee; if failure, loop back to step 4 (retry once)
```
This plan → act → observe cycle is what lets the agent handle "actually, make it just Friday"
mid-conversation without restarting from scratch. It's a harness-layer implementation detail.

### Pattern B: Multi-agent / graph orchestration
**Routing work across multiple specialized agents or nodes via conditional edges.**

```
Graph:
[Intake Agent] --(request parsed)--> [Policy-Check Agent]
                                            |
                              (within policy) | (exceeds policy / conflict)
                                            v                     v
                                [Auto-Submit Node]      [Manager-Review Node]
                                            |                     |
                                            v                     v
                                   [Notification Agent] <---------
```
Used when a task is too broad or high-stakes for one agent/loop to own end-to-end — the
intake agent never touches approval; that routes to a different node when policy-check flags
a conflict. Frameworks like LangGraph use this "graph" terminology directly, which is likely
where "graph engineering" as a phrase comes from — but it's a design pattern *within*
orchestration/harness work, not an industry-branded discipline on its own.

---

## Side-by-side summary

| Layer | Governs | Unit of change | Scenario answer to "why did this work/fail?" |
|---|---|---|---|
| **Prompt** *(discipline)* | Wording of instructions | A single instruction | "The tone/format of the response" |
| **Context** *(discipline)* | What data the model sees | Retrieved/injected information | "Whether it knew the PTO balance and policy" |
| **Harness** *(discipline)* | Tools, permissions, control flow, error handling | The runtime around the model | "Whether it was *allowed* to submit vs. approve, and what happens if the API fails" |
| **Skill** *(discipline)* | Reusable, versioned procedures | A packaged capability (e.g. SKILL.md) | "Whether the notice-period policy update was applied everywhere at once" |
| ↳ Looping *(pattern, inside harness)* | Multi-step reasoning within one task | The plan→act→observe cycle | "Whether it caught the calendar conflict and asked before submitting" |
| ↳ Graph *(pattern, inside harness)* | Multi-agent workflow routing | Nodes and conditional edges | "Whether a conflicting request automatically routed to a manager instead of auto-submitting" |

**Teaching tip:** when something goes wrong in a deployed agent, walk the table top to bottom —
it's rarely "the prompt is bad." More often it's missing context, an over-permissioned harness,
a control-flow gap (loop or graph) that didn't handle the edge case, or a procedure that lives
only in one prompt instead of being owned as a versioned skill.
