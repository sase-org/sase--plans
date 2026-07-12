---
plan: 202607/family_root_auto_zero_suffix.md
---
 Whenever there are two or more child agent rows beneath a single root agent entry, every child agent row should have a distinct agent name using a `--<id>` suffix. We seem to violate this constraint if a new family member is added dynamically by launching a new agent that specifies the 1st as the first argument of the `%name` directive and specified `<id>` for the 2nd argument of the `%name` directive (see #sshot, for example). Can you help me fix this by automatically assigning the `--0` suffix to the 1st agent (after the 2nd agent is added) in this case? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  