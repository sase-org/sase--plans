- **PLAN:** [../202607/agent_id_reference_syntax.md](../agent_id_reference_syntax.md)

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
  `<xprompt>.` to `<id>`, where `<xprompt>` is the name of the xprompt. We
  should be able to use the `{@<id>!}` syntax to override this behavior.
- Modify all existing xprompt swarms (including ones defined in my chezmoi repo)
  to use this new syntax.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 