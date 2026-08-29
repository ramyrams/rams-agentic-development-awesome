# Developer Checklist: Reducing Hallucination in Copilot Code Generation

*Purpose: practical, pre-merge checklist for developers using GitHub Copilot (Chat, inline, CLI, coding agent). Grounded in GitHub's own guidance plus published research on LLM long-context and code-generation failure modes.*

---

## Why this matters (context, one paragraph)

Copilot doesn't "know" your codebase — it infers from whatever context is loaded in a given request, then generates the statistically most plausible continuation. When it lacks real information (an internal API's actual signature, a package that doesn't exist, a business rule never stated), it doesn't reliably say "I don't know" — it produces code that *looks* credible instead. That's the core failure mode this checklist defends against.

---

## A. Before you prompt (prevention)

- [ ] **Break the task down.** Don't ask for an entire controller/service/workflow in one shot. Smaller, scoped requests reduce both hallucination surface area and wasted context.
- [ ] **Be specific about requirements, inputs, outputs, and constraints** — vague comments/prompts produce unpredictable results; explicit ones constrain the model toward what actually exists.
- [ ] **Give Copilot the real context it needs, don't assume it has it.** Neither inline suggestions nor Copilot Chat automatically understands your domain model, business rules, or architecture — attach the relevant file(s), interface definitions, or docs explicitly (`@` mentions, `#` chat variables, `@ FILENAME` in CLI).
- [ ] **For anything non-trivial, ask it to explain its plan before generating code** ("explain your approach before writing this"). This surfaces incorrect assumptions before they're baked into code, and is cheaper to correct in prose than in a diff.
- [ ] **State known-good API names, package names, and versions explicitly in the prompt** when you know them — don't make Copilot guess at library specifics from memory.

## B. While reviewing generated code (detection)

- [ ] **Read and understand every suggestion before accepting it.** Never accept code you can't explain.
- [ ] **Check every API, method, and function call actually exists** — hallucinated APIs (plausible-looking calls to methods/endpoints that don't exist) are a named, common failure mode.
- [ ] **Be skeptical of code that "looks right" but doesn't match your actual intent.** Fluent, well-formatted code is not the same as correct code — confidence of output has no correlation with correctness.
- [ ] **Watch for tests that got deleted or skipped instead of fixed.** A model under pressure to "make tests pass" will sometimes remove the test rather than fix the bug — always inspect diffs to test files, not just source files.
- [ ] **Check that project/organizational constraints weren't silently ignored** (style guides, security policies, architectural patterns) — the model optimizes for a plausible answer, not necessarily your team's rules, unless those rules were in context.

## C. Dependency and package verification (the sharpest hallucination risk)

- [ ] **Never install or import a package Copilot suggests without verifying it actually exists on the real registry** (npm, PyPI, etc.) — this is the single highest-risk category.
- [ ] **Watch for "slopsquatting"** — attackers register real-looking package names that LLMs are statistically prone to hallucinate, then ship malicious code under that name. A hallucinated dependency isn't just broken — it's an active supply-chain attack vector.
- [ ] **Verify each dependency is actively maintained** (not archived, has recent commits/maintainer activity) before trusting it — Copilot itself can help with this check if asked directly against an attached `package.json`/`requirements.txt`, but the verification step must happen.
- [ ] **Check license compatibility** for any newly suggested dependency, not just functionality.

## D. Automated verification (don't rely on reading alone)

- [ ] **Run automated tests and confirm the code actually compiles/builds** — don't trust a suggestion just because it reads cleanly.
- [ ] **Run static analysis / linting** on generated code, same as human-written code — no exemption.
- [ ] **Use security scanning (SAST/DAST equivalents, e.g., CodeQL) and dependency scanning (e.g., Dependabot)** on any PR containing AI-generated code.
- [ ] **Check for new warnings or errors introduced**, not just whether the build passes.
- [ ] **For security-sensitive areas (auth, data handling, encryption), apply stricter review** — treat these as never-skip-human-review zones regardless of how confident the suggestion looks.

## E. Human process (the actual backstop)

- [ ] **Route AI-generated code through the same PR/review process as human-written code** — no fast-lane merges because "Copilot wrote it."
- [ ] **Get a second reviewer for larger or legacy-codebase changes**, where hallucination risk and blast radius are both higher.
- [ ] **Escalate to a senior engineer when uncertain**, rather than accepting a plausible-looking answer to move faster — the Copilot best-practice literature is explicit that supervision is not optional, it's structural.
- [ ] **If Copilot's output degrades noticeably in quality mid-session** (vaguer, less on-topic, ignoring earlier constraints), treat that as a context-window signal, not a one-off bad answer — start a fresh session or `/compact`/`/clear` rather than continuing to push the same degraded context (see context-window reference doc).

---

## Quick reference: the 3 highest-leverage habits

1. **Small, specific, context-attached prompts** beat large vague ones — this is the single biggest lever on hallucination *rate*.
2. **Verify packages against the real registry, every time** — this is the highest-*severity* risk category (supply-chain, not just a bug).
3. **Automated tests + static/security analysis on every AI-touched diff** — this is the backstop that catches what review misses.

---

## Sources
- GitHub Docs — *Best practices for using GitHub Copilot*
- GitHub Docs — *Review AI-generated code*
- Snyk — *5 security best practices for adopting generative AI code assistants*
- Checkmarx — *GitHub Copilot Security Risks* (2026) — slopsquatting/hallucination squatting
- Wipro Tech Blogs — *Mastering GitHub Copilot: Best Practices* (2026)
