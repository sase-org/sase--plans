- **PLAN:** [../202607/clan_member_kill_and_edit_crash.md](../clan_member_kill_and_edit_crash.md)

 When I attempt to kill and edit the `sase-8k.2` agent, which lives in the `sase-8k` agent clan, I receive the below error when `sase ace` crashes. Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 
```
│ /home/bryan/projects/github/sase-org/sase/src/sase/xprompt/directive_edit.py:145 in rewrite_prompt_clan_member_name                                                              │
│                                                                                                                                                                                  │
│   142 │                                                                                                                                                                          │
│   143 │   prefix = f"{clan_name}."                                                                                                                                               │
│   144 │   if not agent_name.startswith(prefix) or not agent_name.removeprefix(prefix):                                                                                           │
│ ❱ 145 │   │   raise ValueError(                                                                                                                                                  │
│   146 │   │   │   f"Cannot relaunch agent '{agent_name}' as a member of clan "                                                                                                   │
│   147 │   │   │   f"'{clan_name}': expected the '{prefix}<suffix>' hood."                                                                                                        │
│   148 │   │   )                                                                                                                                                                  │
│                                                                                                                                                                                  │
│ ╭─────────────────────────────────────────────────── locals ───────────────────────────────────────────────────╮                                                                 │
│ │                  _ = '#gh:gh_sase-org__sase\n\n\n\n#bd/work_phase_bead:sase-8k.2'                            │                                                                 │
│ │         agent_name = 'sase-8k'                                                                               │                                                                 │
│ │          clan_name = 'sase-8k'                                                                               │                                                                 │
│ │ current_agent_name = 'sase-8k'                                                                               │                                                                 │
│ │         directives = PromptDirectives(                                                                       │                                                                 │
│ │                      │   auto_mode='plan',                                                                   │                                                                 │
│ │                      │   auto_enabled=True,                                                                  │                                                                 │
│ │                      │   auto_argument=None,                                                                 │                                                                 │
│ │                      │   hide=False,                                                                         │                                                                 │
│ │                      │   model='small_phase_worker',                                                         │                                                                 │
│ │                      │   model_alias_overrides=mappingproxy({}),                                             │                                                                 │
│ │                      │   reasoning_effort=None,                                                              │                                                                 │
│ │                      │   name='sase-8k.2',                                                                   │                                                                 │
│ │                      │   bead_id='sase-8k.2',                                                                │                                                                 │
│ │                      │   name_explicit=True,                                                                 │                                                                 │
│ │                      │   name_force_reuse=False,                                                             │                                                                 │
│ │                      │   clan='sase-8k',                                                                     │                                                                 │
│ │                      │   clan_declared=False,                                                                │                                                                 │
│ │                      │   clan_tribe=None,                                                                    │                                                                 │
│ │                      │   clan_summary=None,                                                                  │                                                                 │
│ │                      │   clan_summary_script=None,                                                           │                                                                 │
│ │                      │   family_attach_parent=None,                                                          │                                                                 │
│ │                      │   family_attach_suffix=None,                                                          │                                                                 │
│ │                      │   name_template=None,                                                                 │                                                                 │
│ │                      │   name_template_base=None,                                                            │                                                                 │
│ │                      │   name_indexed_template=False,                                                        │                                                                 │
│ │                      │   name_indexed_base=None,                                                             │                                                                 │
│ │                      │   repeat_count=None,                                                                  │                                                                 │
│ │                      │   tribe=None,                                                                         │                                                                 │
│ │                      │   wait=[],                                                                            │                                                                 │
│ │                      │   wait_beads=[],                                                                      │                                                                 │
│ │                      │   wait_duration=None,                                                                 │                                                                 │
│ │                      │   wait_until=None,                                                                    │                                                                 │
│ │                      │   wait_runners=None,                                                                  │                                                                 │
│ │                      │   wait_priority=None                                                                  │                                                                 │
│ │                      )                                                                                       │                                                                 │
│ │        force_reuse = True                                                                                    │                                                                 │
│ │             prefix = 'sase-8k.'                                                                              │                                                                 │
│ │             prompt = '#gh:gh_sase-org__sase\n%id(2, clan=sase-8k, bead=sase-8k.2)\n%model:@small_phase_w'+41 │                                                                 │
│ ╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────╯                                                                 │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
ValueError: Cannot relaunch agent 'sase-8k' as a member of clan 'sase-8k': expected the 'sase-8k.<suffix>' hood.
```