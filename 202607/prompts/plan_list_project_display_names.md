---
plan: 202607/plan_list_project_display_names.md
---
 The `sase plan list` command (see output below) is not outputing the correct sase project names. For example, it shows `gh_sase-org__sase` instead of `sase` and `gh_bobs-org__bob-cli` instead of `bob-cli`. Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 
```
❌1 ❯ sase plan list
╭───────────────────────────────────────────────────────────────────────────────── Plan Pipeline ──────────────────────────────────────────────────────────────────────────────────╮
│               0                                     10                                               10                                               344                        │
│           Proposed                            Approved shown                                   Rejected shown                                   Archived total                   │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭────────────────────────────────────────────────────────────────────────────────── Proposed (0) ──────────────────────────────────────────────────────────────────────────────────╮
│ No pending plan proposals.                                                                                                                                                       │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭───────────────────────────────────────────────────────────────────────────────── Approved (10) ──────────────────────────────────────────────────────────────────────────────────╮
│  Approved   Action   Agent/Project                        Tier   Plan path                                                                                                       │
│ ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── │
│  12m ago    tale     8v / gh_sase-org__sase               tale   ~/.sase/plans/202607/workspace_local_core_fallback.md                                                           │
│  12m ago    tale     8s / gh_sase-org__sase               tale   ~/.sase/plans/202607/agent_panel_plan_goal.md                                                                   │
│  27m ago    epic     8u / gh_sase-org__sase               epic   ~/.sase/plans/202607/per_project_research_sidecars.md                                                           │
│  17h ago    tale     sase-61 / gh_sase-org__sase          epic   ~/.sase/plans/202607/sase61_landing_gaps.md                                                                     │
│  19h ago    tale     sase-60 / gh_sase-org__sase          -      ~/.sase/plans/202607/research_sidecar_cutover_fix.md                                                            │
│  20h ago    epic     8p--plan-0 / gh_sase-org__sase       -      ~/.sase/plans/202607/plan_validate_command_1.md                                                                 │
│  21h ago    tale     8n--plan-0 / gh_sase-org__sase       -      ~/.sase/plans/202607/bulk_kill_edit_waiting_name_reuse.md                                                       │
│  21h ago    tale     8b.f0.w1.w1.f2 / gh_sase-org__sase   -      ~/.sase/plans/202607/codeblock_card_highlight.md                                                                │
│  22h ago    tale     8m / gh_sase-org__sase               -      ~/.sase/plans/202607/agent_panel_fold_scope.md                                                                  │
│  23h ago    tale     8l / gh_bobs-org__bob-cli            -      ~/.sase/plans/202607/dependency_status_propagation.md                                                           │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭───────────────────────────────────────────────────────────────────────────────── Rejected (10) ──────────────────────────────────────────────────────────────────────────────────╮
│  Archived   Tier   Plan path                                                                                                 Note                                                │
│ ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── │
│  21h ago    epic   ~/.sase/plans/202607/plan_validate_command.md                                                             inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  21h ago    -      ~/.sase/plans/202607/bulk_retry_name_reuse.md                                                             inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  23h ago    -      ~/.sase/plans/202607/prompt_codeblock_highlight.md                                                        inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  23h ago    -      ~/.sase/plans/202607/sdd_cli_retirement_and_sidecar_repos.md                                              inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  1d ago     -      ~/.sase/plans/202607/yank_highlight.md                                                                    inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  1d ago     -      ~/.sase/plans/202607/fix_project_agent_revert_checkout.md                                                 inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  1d ago     -      ~/.sase/plans/202607/block_link_alias_removal.md                                                          inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  1d ago     -      ~/.sase/plans/202607/mark_next_hidden_transclusions.md                                                    inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  1d ago     -      ~/.sase/plans/202607/toggle_pomodoro_move_only_hash.md                                                    inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
│  1d ago     -      ~/.sase/plans/202607/xprompt_skill_highlight.md                                                           inferred from archived proposal not represented by  │
│                                                                                                                              proposed/approved state                             │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

```