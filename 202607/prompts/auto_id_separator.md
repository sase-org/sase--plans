---
plan: 202607/auto_id_separator.md
---
 We currently use the `.f-@`, `.r-@`, and `.w-@` agent name suffices when launching sase agents that fork, wait for, or retry other agents. We add the dash here because when the `@` symbol syntax resolves to a alphanumeric sequence that starts with a letter, it's unclear to the user where the auto ID starts and ends. Can you help me remove this dash in the standard case and instead fix this in a generalized way by making it so the `@` symbol functionality automatically prepends a dash to itself if it was not used immediately to the right of a dash or a dot (`.`) and the first character of the auto-id sequence resolves to a letter? If the first character of the sequence resolves to a number, we should not add the dash automatically.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 