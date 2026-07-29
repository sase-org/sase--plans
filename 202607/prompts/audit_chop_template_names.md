- **PLAN:** [../202607/audit_chop_template_names.md](../audit_chop_template_names.md)

 We currently use sase agent names like `audit_bugs.sase.7270b986bf6f` for the `recent_bug_audit` and `recent_improvement_audit` chops (configured in my chezmoi repo and defined in my bugyi-chops GitHub repo). Can you help me start using the special `@` character functionality for this instead, so these agents are given names of the form `audit_<type>.sase.@` instead (where `@` is replaced with an alphanumeric sequence)? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 