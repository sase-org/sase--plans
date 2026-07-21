---
plan: 202607/phase_worker_alias_bucket.md
---
 Can you help me split the `@phase_worker` model alias up into a new `phase_worker` model alias bucket?

- Let's continue to define the `@phase_worker` model alias as is, but define it in the new `phase_worker` bucket.
- Let's also start defining the new `@small_phase_worker`, `@medium_phase_worker`, and `@large_phase_worker` model aliases.
- All of these should default to using the `@phase_worker` model alias as their value.
- We should use these new model aliases for the phase agents that we launch (which alias we use should depend on the size of the phase).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
