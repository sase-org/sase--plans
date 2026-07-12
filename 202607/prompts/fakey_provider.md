---
plan: 202607/fakey_provider.md
---
 Can you help me design and implement a new fake agent CLI provider, named `fakey`?

- The CLI agent should have first-class support within sase and will be used to test sase agent launches and to test various expected agent CLI behavior, like failures (and sase agent retries), for example.
- This agent should have sane and useful defaults but should also be highly configurable. For example we should be able to configure it to fail with a specific failure message.
- As a first use case we should write some PNG screenshot tests using this agent CLI that demonstrate how agent retries work. Make sure that our ability to E2E test agent CLI failure retries is top-notch (if it's not already) by the end of this work (I plan to refine the way sase handles retrying failed agents soon, so good testing will be needed).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 