---
plan: 202607/gate_primary_enter.md
---
 When a sase gate is shown in the TUI's notification panel, the `<enter>` key should always trigger the submission of the primary button/option offered by the gate (make sure that every sase gate has to provide this and update all existing sase gates if necessary--e.g. this should be the "Tale" option for tale plan gates and the "Epic" option for epic plan gates), but it currently toggles the "Launch coder agent" option when a tale plan is proposed (the `<space>` keymap can be used for this; `<enter>` should submit the primary option). Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 