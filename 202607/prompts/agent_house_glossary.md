---
plan: 202607/agent_house_glossary.md
---
 Can you add a new term to the memory/glossary.md file called "agent house"? This term is just used as a generic name for agents/agent families. This way we don't need to explicitly state that we're talking about both of them, which is often the case since an agent family hijacks an agent's name when it is created. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 

%xprompts_enabled:false
### Questions and Answers

#### Q1: README scope

> May I update the four generated line/token totals in sase/memory/README.md so sase init can regenerate the five instruction shims and just check can pass?

- [x] **Authorize README update** — Recommended: apply only the generator-produced +4/-4 metadata update, regenerate the five shims, validate, and commit.
- [ ] **Keep glossary only** — Leave the committed glossary change as-is; generated shims remain stale and just check remains failing.

%xprompts_enabled:true
