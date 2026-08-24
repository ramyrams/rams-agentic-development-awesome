
## SkillOpt

https://microsoft.github.io/SkillOpt/
https://venturebeat.com/orchestration/microsofts-open-source-skillopt-automatically-upgrades-ai-agent-skills-without-touching-model-weights


## Skill
* [The Book Pattern: Progressive Disclosure for AI Agents](https://idavidov.eu/orchestration-pattern-technical-book)
* [Skills: Domain Expertise on Demand](https://idavidov.eu/skills-domain-expertise-on-demand)
* [Explore First: Why the Agent Looks Before It Writes](https://idavidov.eu/explore-first-the-browser-use-workflow)
* [From Prompt to Passing Test: A Complete Agentic QA Session](https://idavidov.eu/from-prompt-to-passing-test)
* [AI-Native Workflow: The Operating Manual for Your Agent](https://idavidov.eu/ai-native-workflow)
* [CLAUDE.md: Teaching the AI Your Rules](https://idavidov.eu/claude-md-teaching-the-ai-your-rules)
* [The Scaffold: Playwright Project Structure Built for AI](https://idavidov.eu/the-scaffold-playwright-ai)






your AI agent gives your repo the same 30 seconds you give a book in a bookshop

hand a new hire a 400 page of documentation on day one and watch what happens. nothing. nobody reads such documents

one giant system prompt is just like that document. and it gets worse, rules near the bottom get applied less consistently than rules near the top. that failure has a name, context rot

now watch yourself buy a technical book

you flip to the back cover for the promise. you skim the preface for the author's rules. you scan the table of contents for the parts you care about

3 checks, then you pay. later you study one chapter. later still you open the appendix for the one worked example you need

write your orchestration the same way and your agent reads it the same way

layer 1 is CLAUDE. md. the role is the back cover, MUST/SHOULD/WON'T is the preface, the skills index is the table of contents. always loaded, around 100 tokens

layer 2 is one SKILL. md. WHY names the reasoning, HOW gives numbered phases, WHAT shows correct vs forbidden code. loaded only when the task matches, under 5k tokens

layer 3 is the references/ folder. full worked examples and runnable scripts. cost is zero until the moment of need

this structure has a name, progressive disclosure. Anthropic shipped it as an open standard in December 2025. by now it is the default for any agent setup that scales past a handful of rules

it works for the same reason a good book works. cheap on day 1, deep when you need it, infinite on the shelf

quick test. a colleague asks where the rule about X lives in your project. if you cannot answer in 5 seconds, the failure is the architecture, not the colleague. your agent fails the same way, it just never tells you


![Your repo is the technical book](https://media.licdn.com/dms/image/v2/D4D22AQHoWsFNgYIdbw/feedshare-shrink_800/B4DZ66uMo8HcAc-/0/1781249133424?e=1788998400&v=beta&t=6-YWMguQV5A2JIKcwJtUtiNYiAkLg9nJgT808_Z2Qjk)
