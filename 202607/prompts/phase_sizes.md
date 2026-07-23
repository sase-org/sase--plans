- **PLAN:** [../202607/phase_sizes.md](../phase_sizes.md)

 Can you help me add support for two new phase sizes that can be used in
epic sase plans?: `xsmall` and `xlarge`

- Make sure that we tell agents, via the explanation text that is output by the
  `sase plan validate` command's `--explain` CLI option, how these new sizes are
  meant to be used:
  - The `xsmall` phase size should generally only be used for the simplest of
    tasks (for example, when testing a sase agent feature by launching sase
    agents to observe their output--make sure you remove the similar exception
    that is described by the plan validation explanation text regarding
    explicitly overriding a phase's model; tell agents to prefer to use the
    `xsmall` phase size for this).
  - The `xlarge` size should rarely be used and is an admission by the planning
    agent that the given task/feature is too large for them to effectively plan
    on their own. This phase size might also be suitable if it is genuinely
    desirable to plan part of a feature only once other parts have been
    implemented. This phase should generally only be used when the agent is
    fairly certain that this phase agent will create an epic plan file.
- We should make the following changes to the agent prompts for epic phase
  worker agents based on these new phase sizes:
  - Make sure to add new phase worker model aliases to the `phase_worker` model
    alias bucket for these new phase sizes.
  - We should add a new `@smart` model alias that defaults to using the
    `@default` model alias for its value. This is the model alias we should
    start using for `large` phase worker prompts; we should reserve `@smartest`
    for `xlarge` phases.
  - We should rename `@cheaper` to `@cheap` and rename `@cheapest` to
    `@cheaper`. Once this is done you should add a new `@cheapest` model alias
    that uses a default value of `claude/haiku || codex/gpt-5.3-codex-spark`
    (Haiku with a fallback to codex's spark). We should continue to use (the
    newly renamed) `@cheap` for the `@small_phase_worker` and start using
    `@cheaper` for the new `@xsmall_phase_worker` model alias value (used by
    phase workers that are working `xsmall` phases--the new Haiku `@cheapest`
    model alias should not be used yet).
  - We should change the default model used by the `@medium_phase_worker` model
    alias to `codex/gpt-5.6-sol@high` instead of `@default` (the net change here
    is that we start using the `high` model effort level for `medium` phases
    instead of `xhigh`).
  - Phase workers that use the `xlarge` phase size should include `#plan` at the
    end of their prompt (just like phase workers for `large` phases do); phase
    workers for `xsmall` phases should not.
- Make sure you add great support for these new phase sizes anywhere that we
  currently display phase sizes (for example, in an epic clan's summary).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 