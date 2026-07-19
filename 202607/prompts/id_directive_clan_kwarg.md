---
plan: 202607/id_directive_clan_kwarg.md
---
 It should be impossible for an agent to join a clan unless the agent is
in the same hood as the one defined by the clan name. Therefore, it is redundant
to have to put `<foo>` in the input argument of `%name` if `%clan(<foo>)` is in
the prompt already. We should be able to just list the ID of the agent as
`<bar>` instead and have `<foo>.<bar>` derived as the name automatically. But it
would be confusing and conceptually wrong to say that an agent with `%name(bar)`
in its prompt is actually not named `bar` but `foo.bar`. Can you help me make
some changes to sase to address the concerns above?

- Let's rename the `%n|%name` directive to `%i|%id`.
- Let's also stop allowing `%clan` to be included in more than one prompt. We
  should throw an error if `%clan(foo)` is used when the `foo` clan already
  exists.
- Instead, all agents but the first (and possibly the first, if there is no need
  for the `%clan` directive's `tribe` kwarg) should use a new `clan` kwarg that
  we add to the `%id` directive.
- For example, assume that `%id(<bar>, clan=<foo>)` is used in a prompt. Then
  the agent that is launched will belong to a clan named `foo` and will be named
  `foo.bar`. If the `foo` clan did not already exist, we should create it (it
  will have no tribe).
- It should be a validation error to attempt to use the `%id` directive's `clan`
  kwarg in a prompt that also uses the `%tribe` directive (since to join a clan,
  we MUST also join its tribe).
- I've already migrated the
  ~/.local/share/chezmoi/home/sase/xprompts/research_swarm.md file (in my
  chezmoi repo) to use these new conventions (see that file for context on how
  this feature should work).
- The `%name` directive is pretty integral to sase's functionality so it likely
  has a lot of references. Make sure you catch all edge cases and don't break
  anything.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 