---
tier: tale
title: Back Ctrl+Space with the VCS xprompt MRU store
goal:
  Every successful agent launch with a VCS xprompt ref becomes the MRU head, and
  Ctrl+Space pre-fills the prompt input widget with that workflow and argument.
size: medium
proposed_by: bbugyi200.athena.062
create_time: 2026-08-18 09:59:02
status: wip
---

# Plan: Back `<ctrl+space>` with the VCS xprompt MRU store

## Goal

`<ctrl+space>` must pre-fill the prompt input widget with the VCS xprompt workflow and
argument of the **most recently launched** agent. Whenever a sase agent is successfully
launched with a VCS xprompt ref (`#gh:<project-or-patch>`, `#git:<project-or-patch>`,
…), that exact workflow+argument becomes the MRU head and is what the next
`<ctrl+space>` offers — no matter which surface launched it (ACE prompt bar,
`<ctrl+p>`-cycled ref, `<ctrl+g>` editor, `sase run` from a shell, or the `/sase_run`
agent skill).

## Current behavior (verified, not assumed)

There are two separate stores today and `<ctrl+space>` reads the wrong one.

**Store A — `~/.sase/vcs_xprompt_mru.json`** (`src/sase/history/vcs_xprompt_mru.py`).
Written by `record_vcs_xprompt_usage()`, called from
`sase/main/query_handler/_launch.py:186` (`_record_leading_vcs_xprompt_usage`) after a
successful `launch_query()`. Every launch surface funnels through `launch_query` (the
ACE prompt bar submits an argv-only `sase run` through the durable proc queue:
`_launch_start.py:196` → `_launch_procs.py:_submit_launch_proc` →
`agent_durable.py:submit_agent_launch` → `sase run`), so this store already has the
"updated on every launch" property. It is read only by `<ctrl+p>`/`<ctrl+n>` cycling
(`widgets/_vcs_mru_cycling.py:283`) and by `<ctrl+g>`
(`_entry_custom.py:62 action_start_last_vcs_xprompt_in_editor`).

**Store B — `~/.sase/last_agent_selection.json`**
(`src/sase/ace/last_agent_selection.py`). This is what `<ctrl+space>` actually reads.
`ctrl+space` canonicalizes to `ctrl+@` (`keymaps/key_validation.py:59`), binds to
`start_agent_from_patch` (`bindings.py:185`, `default_config.yml:426`), and dispatches
to `_entry_custom.py:35 action_start_agent_from_patch` →
`_entry_points.py:191 _load_last_custom_agent_selection` → Store B.

Store B is written at **selection** time, never at launch time, by exactly four TUI
entry points:

- `_entry_custom.py:122` — the `+` project-select modal
- `_entry_quick_launch.py:58` — leader-Space from the Patches tab
- `_entry_quick_launch.py:105` — leader-Space from the Agents tab
- `_entry_relaunch.py:391` — kill-and-edit relaunch

### Observed divergence

On the author's machine at planning time:

| store                       | head entry              | mtime |
| --------------------------- | ----------------------- | ----- |
| `vcs_xprompt_mru.json`      | `#gh:gh_sase-org__sase` | 09:50 |
| `last_agent_selection.json` | `gh_bobs-org__bob-cli`  | 06:32 |

`<ctrl+space>` pre-filled `#gh:bob-cli ` while the most recently launched agent was
`#gh:sase`.

### Defect 1 — launches that never move the `<ctrl+space>` target

- Open the bar on project A, cycle to ref B with `<ctrl+p>`, launch. B launches; Store B
  still says A.
- Type or paste `#gh:<ref>` into a home-mode bar and launch. Store B untouched.
- `<ctrl+g>` opens the MRU head in `$EDITOR` and launches from there. Store B untouched.
- `sase run '#gh:<ref> …'` from a shell, or a launch via the `/sase_run` skill. Store B
  untouched.
- The `,.` prompt-history replay path with a substituted prefix
  (`prompt/cli_run.py:90 replace_vcs_workflow_tags` → `launch_query`). Store B
  untouched.

### Defect 2 — selections that never launched _do_ move the target

Store B is written when the prompt bar opens. Pick a project from the `+` modal, press
`<escape>`, launch nothing — `<ctrl+space>` still retargets to it.

### Defect 3 — the MRU records a tag the launch did not use

`_leading_vcs_xprompt_prefix` (`_launch.py:196`) iterates `get_ref_patterns()` and
returns the first **registry-ordered** workflow whose pattern matches **anywhere** in
the query. It does not find the prompt's leading launch tag. Verified against the
installed package (`get_ref_patterns()` order is `['git', 'gh']`):

```
'#gh:sase do work and see #git:other'     -> records '#git:other'   (launched #gh:sase)
'do a thing and compare with #gh:actstat' -> records '#gh:actstat'  (launched in home)
'#gh:sase seg one\n---\n#gh:actstat seg two' -> records '#gh:sase'  (2 agents, 1 recorded)
```

The launcher itself resolves the ref with leading-tag semantics
(`_parsing.extract_vcs_workflow_tag`, which also skips a `%id(...)`/`%wait(...)`
directive prefix), so the MRU disagrees with what actually ran. This poisons `<ctrl+p>`
cycling and `<ctrl+g>` today, and would poison `<ctrl+space>` after this change, so it
must be fixed in the same tale.

### Defect 4 — `<ctrl+g>` builds `history_sort_key` from a humanized ref

`load_launchable_vcs_xprompt_mru()` humanizes entries on return
(`_humanize_and_dedupe_mru`, `vcs_xprompt_mru.py:163`): on-disk `#gh:gh_sase-org__sase`
comes back as `#gh:sase`. `action_start_last_vcs_xprompt_in_editor`
(`_entry_custom.py:74-76`) then derives `history_sort_key` from that display ref
(`"sase"`), while every other prefill surface passes the canonical directory key
(`"gh_sase-org__sase"`) — see `_entry_custom.py:204` and `_entry_quick_launch.py:63`.
Prompt-history grouping diverges between the two paths.

## Design

Make Store A the single source of truth for "last used VCS xprompt workflow + argument",
and delete Store B.

This is the right direction rather than "keep Store B and also write it on launch"
because:

- Store A is already written at the one true point — a _successful_ launch, in
  `launch_query`, which every surface reaches. Store B would need a second writer in the
  CLI launch path, and `sase.ace.last_agent_selection` imports `sase.ace.tui.modals`,
  dragging Textual into every `sase run` process.
- Store A already prunes non-launchable entries on read
  (`_is_stale_known_project_prefix`, `_vcs_prefix_ref_is_gone`,
  `_vcs_prefix_provider_mismatched`) and already excludes the implicit `#git:home`
  default (`is_default_vcs_xprompt_prefix`), so the stale-selection clearing and
  default-home clearing that `_entry_points.py:124-146` implements by hand come free.
- With both readers moved off Store B, the entire module and its four save sites become
  dead code that Symvision will flag anyway.

**Intentional behavior changes to state in the commit message:**

1. Selecting a project/Patch and then cancelling the bar no longer retargets
   `<ctrl+space>`. Only launches do. This is the requested semantics.
2. When the last launch was home-mode, `<ctrl+space>` falls back to the most recent
   non-default VCS entry instead of clearing itself with a warning — consistent with
   `840e69794` ("exclude `#git:home` from Ctrl+Space replay history").
3. `<ctrl+space>` and `,.` now agree with `<ctrl+p>`'s first cycle target and with
   `<ctrl+g>`, because all four read one store.

**No feature flag.** This is a correction to existing behavior that is ready on landing,
not a disabled beta, a partially landed path, or a deprecation whose old branch must
stay reachable (`sase/memory/sase_flags.md`). The Store B branch is deleted, not kept
reachable.

## Implementation

### 1. Record the tag the launch actually used

`src/sase/main/query_handler/_launch.py`

- Replace `_leading_vcs_xprompt_prefix` so it uses the launcher's own leading-tag
  semantics instead of a registry-ordered `search()`:
  - `sase.xprompt._parsing.extract_vcs_workflow_tag(text.strip() + " ")` to get the
    leading tag (it already skips a `%…` directive prefix and returns `None` when the
    prompt has no leading tag), then
    `sase.xprompt._parsing.extract_project_from_vcs_tag(tag)` for the ref, then rebuild
    `#<workflow_type>:<ref>`. Derive `workflow_type` from the tag text rather than
    re-scanning the registry.
  - A prompt whose body merely _mentions_ a ref, with no leading tag, must record
    nothing.
- `_record_leading_vcs_xprompt_usage` records **one entry per launched multi-prompt
  segment**, in launch order, so the last-launched segment ends up at the MRU head.
  Split with `sase.agent.multi_prompt.parse_multi_prompt(query).segments`; a
  single-segment query is the existing behavior. Keep the call site where it is — after
  the success check — so failed and partial launches still record nothing.
- Rename the helpers to match what they now do (e.g.
  `_record_launched_vcs_xprompt_usage`); keep them module-private.

Leave `record_vcs_xprompt_usage()` alone: it already canonicalizes project aliases,
drops the `#git:home` default, and prunes stale/provider-mismatched prefixes.

### 2. Expose canonical MRU entries alongside display entries

`src/sase/history/vcs_xprompt_mru.py`

- Add a public accessor returning `(canonical_prefix, display_prefix)` pairs in MRU
  order — e.g. `load_launchable_vcs_xprompt_mru_pairs(projects_dir=None, prune=True)`.
  Factor the existing body of `load_launchable_vcs_xprompt_mru` into it and have
  `load_launchable_vcs_xprompt_mru` return `[display for _, display in pairs]` so
  `<ctrl+p>`/`<ctrl+n>` cycling is untouched.
- Dedupe stays keyed on the display form, first-wins, exactly as
  `_humanize_and_dedupe_mru` does today, so the two functions can never disagree on
  ordering or length.
- Disk stays canonical; the pruning write in `load_launchable_vcs_xprompt_mru` happens
  once, in the shared implementation.

Callers need both halves: the display prefix seeds the bar text and label (users must
never see a directory key — `CLAUDE.md` "Show Project Names, Never ProjectSpec Keys"),
while the canonical ref is the `history_sort_key`.

### 3. Point `<ctrl+space>` at the MRU

`src/sase/ace/tui/actions/agent_workflow/_entry_custom.py`

- Rewrite `action_start_agent_from_patch` to read the MRU head via the new pairs
  accessor and open the prompt bar through `_show_prompt_input_bar_for_home`:
  - `initial_text = f"{display_prefix} "`
  - `display_name = extract_project_from_vcs_tag(display_prefix) or display_prefix`
  - `history_sort_key = extract_project_from_vcs_tag(canonical_prefix) or display_name`
  - Empty MRU → `notify("No previously launched VCS xprompt", severity="warning")`.
- Keep the legacy `action_start_agent_from_changespec` alias and the `self.__dict__`
  override hooks at the top of the method exactly as they are; tests and legacy callers
  depend on them.
- Fix `action_start_last_vcs_xprompt_in_editor` (Defect 4) to take `history_sort_key`
  from the canonical half of the same pair. Factor the shared "resolve MRU head into
  `(initial_text, display_name, history_sort_key)`" step into one private helper used by
  both actions so they cannot drift again.
- Drop the Store B save block from `action_start_custom_agent`
  (`_entry_custom.py:117-123`) and the
  `self._clear_stale_last_custom_agent_selection(project_name)` call in
  `_start_custom_agent_from_selection` (`_entry_custom.py:158`) — replace the latter
  with a plain non-launchable `notify` that does not touch any persisted store.

### 4. Point `,.` prompt-history replay at the MRU

`src/sase/ace/tui/actions/agent_workflow/_entry_prompt_history.py`

`_start_prompt_history_from_last_selection` is documented as "same as Ctrl+Space" and
must stay that way. It needs `project_file` to build its prefix today; with an MRU
prefix the workflow tag is already known, so:

- Resolve the MRU head to `(display_prefix, canonical_prefix)`.
- `vcs_prefix = display_prefix` (already `#<workflow>:<name>`, no
  `_vcs_prompt_prefix_or_notify` call needed).
- `history_key` = canonical ref, as in step 3.
- The `PromptContext` it builds is already home-mode with the `home` project spec
  (`_entry_prompt_history.py:73-79`); keep that unchanged.
- Empty MRU → the same warning text as `<ctrl+space>`.

### 5. Delete Store B

- Delete `src/sase/ace/last_agent_selection.py`.
- Remove the save blocks in `_entry_quick_launch.py:53-59` and
  `_entry_quick_launch.py:100-106`, and in `_entry_relaunch.py:386-392`. Those methods
  keep everything else they do (prefix building, bar mount).
- Remove from `_entry_points.py`: `_clear_stale_last_custom_agent_selection`,
  `_clear_default_home_replay_selection`, `_selection_replays_default_vcs_prefix`,
  `_last_selection_is_replayable`, `_load_last_custom_agent_selection`, and the
  `_last_custom_agent_selection` class attribute.
- Remove the now-unused `_last_custom_agent_selection` declarations in
  `_agent_launch.py:33`, `_entry_custom.py:17`, `_entry_quick_launch.py:20`,
  `_entry_relaunch.py:127`, and the `_load_last_custom_agent_selection` /
  `_clear_stale_last_custom_agent_selection` `TYPE_CHECKING` stubs in
  `_entry_custom.py:21-29` and `_entry_prompt_history.py:25-27`.
- Leave `_vcs_prompt_prefix_or_notify` and `_is_launchable_project` in place if other
  callers remain; delete whichever becomes unreferenced. Run `just lint` — Symvision
  will name anything missed; read `sase/memory/symvision.md` before adding any pragma.

Do **not** touch `SelectionItem` / `ProjectSelectResult`
(`modals/project_selection_types.py`) — the project-select modal still uses them for
in-flight selection; only the _persistence_ of a selection goes away.

### 6. Labels and help text

- `bindings.py:185`, `keymaps/metadata.py:110`: `start_agent_from_patch` is no longer
  "Run Agent (Patch)"; retitle to something like "Run Agent (Last VCS XPrompt)". Keep
  the action name and the `ctrl+@` default binding unchanged so user keymap configs and
  `default_config.yml:426` keep working.
- `help_modal/agents_bindings.py:473`, `help_modal/patches_bindings.py:164`,
  `help_modal/axe_bindings.py:153`: "Repeat last +/Ctrl+Space selection" → "Repeat last
  launched VCS xprompt".
- `_app_action_availability.py:87-96`: the comment says "replays the last launch
  selection"; refresh the wording. The guard itself (disable `start_agent_from_patch`
  while a prompt bar is mounted) is still correct and must stay — it is what stops
  `<ctrl+space>` from tearing down a draft.

## Tests

New / rewritten:

- `tests/test_launch_query_feedback.py` (or a new focused module) — table-drive the
  step-1 recording cases, at minimum the four verified above plus a `%id(foo) #gh:sase`
  directive-prefixed prompt and a multi-segment `---` prompt asserting _both_ segments
  recorded with the last segment at the head.
- `tests/test_vcs_xprompt_mru.py` — the pairs accessor: canonical/display halves for a
  project whose `PROJECT_NAME` differs from its directory key, dedupe agreement with
  `load_launchable_vcs_xprompt_mru`, and that one call still performs at most one
  pruning write.
- A `<ctrl+space>` behavior test alongside
  `tests/ace/tui/test_entry_points_vcs_prefix_selection.py`: after
  `record_vcs_xprompt_usage("#gh:<canonical>")`, `action_start_agent_from_patch` mounts
  the bar with `initial_text == "#gh:<display> "`, `display_name` humanized, and
  `history_sort_key` canonical. Empty MRU warns and mounts nothing.
- A regression test for the headline bug: record ref A, then record ref B, then assert
  `<ctrl+space>` offers B (this is the "cycle then launch" case reduced to its store
  effect).
- `<ctrl+g>` keeps its behavior but now passes the canonical `history_sort_key`.

Update / delete:

- Delete `tests/ace/test_last_agent_selection.py`.
- `tests/ace/tui/_entry_points_vcs_prefix_helpers.py`,
  `tests/ace/tui/test_entry_points_vcs_prefix_selection.py`,
  `tests/ace/tui/test_agent_bulk_kill_edit.py`,
  `tests/ace/tui/test_family_member_relaunch.py` — drop `last_agent_selection`
  assertions and fixtures; keep the prefix/mount assertions.

Both `vcs_xprompt_mru` and any test that launches must keep using the existing SASE home
isolation fixtures — never let a test write the real `~/.sase/vcs_xprompt_mru.json` (see
`cce40d885` "isolate VCS xprompt MRU path resolution" and the `_MRU_FILE` module hook).

## Verification

- `just install` first (ephemeral workspace).
- `just check` while iterating.
- `just check-full` before landing, via `/sase_monitor` with a `--next` action — this
  touches the ACE TUI action surface and deletes a module, so scoped selection is not
  enough.
- Manual smoke in `sase ace`: launch `#gh:<projA>`, then open the bar and `<ctrl+p>` to
  `#gh:<projB>` and launch, then press `<ctrl+space>` and confirm it offers
  `#gh:<projB>`. Then `sase run '#gh:<projA> noop'` from a shell and confirm
  `<ctrl+space>` offers `#gh:<projA>`.

## Out of scope

- Recording the _resolved_ ref from `AgentLaunchResult` rather than the prompt text.
  Rejected: external refs (`#gh:owner/repo`) and Patch names do not map back to a single
  prefix spelling, and `record_vcs_xprompt_usage` already canonicalizes aliases. Revisit
  only if alias drift shows up in practice.
- Any change to `<ctrl+p>`/`<ctrl+n>` cycling semantics or to the MRU's pruning rules.
- Any change to the `+` project-select modal itself.
