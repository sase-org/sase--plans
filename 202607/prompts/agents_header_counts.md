- **PLAN:** [../202607/agents_header_counts.md](../agents_header_counts.md)

 Can you help me improve the agent counts shown at the top of the "Agents" tab of the `sase ace` TUI?

- Let's start showing the `<N>/<M>`, where `<N>` is the number of agents running and `<M>` is the number of configured maxiumum allowed running agents to use the same `<N>` used by the `running` count on that same line.
- Let's start showing the `<Q>` queued, where `<Q>` is the number of agents that are currently queued to run inside the same square brackets, but only when there are a non-zero number of sase agents that are currently queued (which should only happen when `<N>` is equal to `<M>`).
- For example, in #sshot, `[5/10 · 0 queued]  [5 running · 4 waiting · 31 done]` should be replaced with `[5/10 running · 4 waiting · 31 done]`.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
