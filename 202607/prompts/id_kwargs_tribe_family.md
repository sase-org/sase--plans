---
plan: 202607/id_kwargs_tribe_family.md
---
 Can you now help me (see the sase-7g epic bead for context) get rid of
the `%tribe` directive all together in favor of adding a new `tribe` kwarg to
`%id`?

- We should also make the 2nd argument of `%id` a `family` kwarg and start
  treating that arg as the name of the agant family that this agent should join.
- For example, the `%id(bar, family=foo)` directive should create a new agent
  family member named `foo--bar` that joins the `foo` agent family.
- We should then be able to enforce a few desirable constraints (throw a good
  validation error if clients try to violate these):
  - These kwargs are all mutually exclusive (i.e. if one of them is set neither
    of the other two are allowed to be set).
  - If `%clan(<clan>)` is used in a prompt, then none of these `%id` kwargs are
    allowed to be used, since `clan=<clan>` is set implicitly.
  - `%clan(<clan>)` should fail if `<clan>` already exists (I think we already
    do this).
- The user can use `%id(tribe=<tribe>)` to set the tribe of an agent without
  setting an explicit ID (i.e. let the ID be implicitly set to `@`, which is
  automatically transformed into a alphanumeric sequence that makes the agent
  name unique). We should define a built-in `#tribe` xprompt that takes
  `<tribe>` as an argument and defines `%id(tribe=<tribe>)` as its contents (to
  make this use-case easier).
- Make sure you update all references from all repos!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  