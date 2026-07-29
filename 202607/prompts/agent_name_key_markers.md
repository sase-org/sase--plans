- **PLAN:** [../202607/agent_name_key_markers.md](../agent_name_key_markers.md)

 This agent failed (see ~/tmp/screenshots/20260729_071508.png) because
the `@` sase agent syntax supported by the `%id` directive's input argument (and
other places, like in the `#fork` xprompt's input argument and the `%wait`
directive's input argument, for example) rendered `p` instead of `o` (since all
of the other research agents launched by the `#research_swarm` xprompt used the
`research.o` sase agent hood). Can you help me verify/deny my suspicion and fix
this by adding support for a new syntax?

- Let's start allowing (with the goal of eventually requiring this since the `@`
  syntax will always produce a sequence that produces a unique agent name in the
  future) a new `{@<id>}` syntax, where `<id>` is some alphanumeric (which can
  also contain the `.` character) sequence (e.g. `{@1}`, `{@foobar}`, etc...),
  such that `{@<id>}` always resolves to the same unique alphanumeric (with a
  prepended dash in one case I think) sequence.
- To ensure that all of these references in the same xprompt swarm get assigned
  a unique alphanumeric sequence, we should always implicitly append
  `<xprompt>.<timestamp>.` to `<id>`, where `<xprompt>` is the name of the
  xprompt (in case `<timestamp>` is not enough to disambiguate) and
  `<timestamp>` is a unique timestamp (so agents from from one launched
  `#research_swarm` xprompt swarm do not use a `research.<hood>` agent hood
  where `<hood>` actually corresponds with a different `#research_swarm` agent
  swarm launch, for example).
- We should be able to use the `{@<id>!}` syntax to override the behavior
  described by the previous bullet (so no prefix is added to `<id>`).
- Modify all existing xprompt swarms (including ones defined in my chezmoi repo)
  to use this new syntax.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 