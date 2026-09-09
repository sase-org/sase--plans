---
tier: epic
status: done
title: Soft-disabled LLM providers and a launch guard for hard-disabled ones
goal: "A provider can be put in a soft disable that spares it in load-balanced pools
  while another member can cover, never triggers a `||` fallback, and still runs when an
  agent asks for it by name — and any launch that truly needs a hard-disabled provider
  is caught before an agent spawns, with an ACE panel that offers a one-keypress way out
  (enable, soft-enable, pick another model, abort this agent, abort the launch).

  "
phases:
  - id: core-mode
    title: Provider-disable mode on the Rust wire
    depends_on: []
    size: medium
    description:
      "core-mode: add the hard/soft `mode` field to the sase-core provider-disable
      record, bump the wire schema to 2 with a v1 migration that keeps in-flight
      disables as hard, and thread the mode through the set/try-set bindings."
  - id: routing
    title: Mode-aware routing policy
    depends_on:
      - core-mode
    size: medium
    description:
      "routing: teach the Python routing layer the three-state member availability
      (preferred / sparing / unavailable), spare soft members in `|` pools only while a
      preferred member exists, keep soft members winning `||` fallbacks, and stop soft
      disables from raising, pausing overrides, or blocking retries."
  - id: provider-ui
    title: Launch Control soft-disable workflow
    depends_on:
      - routing
    size: medium
    description:
      "provider-ui: add the `s` soft-disable action and mode-aware rendering to Provider
      Routing, keep the current window when flipping mode, show sparing state in the
      Launch Control rows, top-bar pill, model picker, and `%model` completion, and
      document all of it."
  - id: guard-core
    title: Fail-closed launch guard
    depends_on:
      - routing
    size: medium
    description:
      "guard-core: enumerate the launch units one prompt will spawn, refuse before any
      spawn when a unit can only run on a hard-disabled provider, and accept an
      ACE-resolved unit bundle through the `sase run` request payload."
  - id: guard-panel
    title: The ACE disabled-provider launch panel
    depends_on:
      - provider-ui
      - guard-core
    size: medium
    description:
      "guard-panel: run the guard before ACE submits a launch and resolve each blocked
      unit in sequence through a single-keypress panel offering enable, soft-enable,
      per-provider enable, another model, abort this agent, and abort the whole launch."
proposed_by: bbugyi200.athena.07o
bead_id: sase-qx
create_time: 2026-09-09 19:51:36
---

- **PROMPT:**
  [prompts/202608/soft_provider_disables.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/soft_provider_disables.md)
- **BEAD:**
  [sase-qx](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qx/README.md)

# Plan: Soft-disabled LLM providers and a launch guard for hard-disabled ones

## Problem

Today a provider disable is all-or-nothing. `sase.llm_provider.provider_disable` stores
one record per provider, and every routing decision treats its presence as "this
provider does not exist right now": `|` pools skip it, `||` fallback chains fall past
it, autodetect ignores it, the model picker hides its models, `%model` completion drops
its rows, and `get_provider()` raises `ProviderTemporarilyDisabledError`.

That is the right behavior when a provider is unusable. It is the wrong behavior for the
case this epic is about: a usage limit is _approaching_ but not hit. The user wants to
stop spending that provider on work any other provider could do, while keeping it for
the agents that specifically need it. A hard disable cannot express that — it also
diverts `||` fallback chains, which is a much bigger behavior change than intended.

The second half of the problem is what happens when a launch really does need a
hard-disabled provider. Nothing checks before spawning. `%model:claude/opus` with Claude
disabled resolves fine, claims a workspace, spawns the runner, and only then fails
inside `get_provider()` at invoke time. A pool whose every member is disabled behaves
the same way: `select_model_alias_pool_member` deliberately keeps member zero "so normal
provider diagnostics can explain why the selected launch cannot run" — but that
diagnostic arrives after the agent exists.

## Current behavior (verified)

Provider-disable state:

- `sase-core` owns the store: `crates/sase_core/src/provider_disable.rs`, wire schema
  version 1, one record per provider under `~/.sase/llm_provider_disables.json`, with
  fields `version`, `provider`, `created_at`, `expires_at`, `source` (`serde` with
  `deny_unknown_fields`). Unknown/invalid/expired records are pruned on read; an
  unreadable or wrong-version file is deleted.
- `crates/sase_core_py/src/lib.rs` exposes `provider_disable_wire_schema_version`,
  `provider_disable_get`, `provider_disable_set_relative`, `provider_disable_set_until`,
  `provider_disable_try_set_relative`, `provider_disable_try_set_until`,
  `provider_disable_clear`.
- `src/sase/llm_provider/provider_disable.py` is the strict Python facade
  (`TemporaryProviderDisable.from_wire` requires exactly the five fields).
  `provider_disable_peek.py` is the lock-free display read used by keystroke and top-bar
  paths.
- Writers: ACE's Provider Routing modal (`source="ace"`) and usage-limit detection
  (`source="usage_limit"`, first-writer-wins through `try_disable_provider*`).

Routing:

- `load_balancing.py` owns selector parsing plus the two selection functions, both of
  which take a `Sequence[bool]` availability mask:
  `select_model_alias_pool_member(alias, selector, availability, consume=)` (weighted
  round-robin over a machine-global cursor in `~/.sase/llm_lb.json`) and
  `select_model_alias_fallback_member(availability)` (first `True`, else index 0).
- `model_alias_resolution.py` builds that mask from `resolved_target_is_available()` →
  `registry.provider_routing_available()`, which returns `False` when the provider is
  unregistered, when `provider_disable_for()` finds any record, or when the declared CLI
  is missing. The same module builds `ModelAliasSelectorMember(..., available=bool)` for
  display through `model_alias_selector_details()`, consumed by
  `doctor/checks_config_model_aliases.py`, `model_launch_settings.py`, `alias_view.py`,
  and the Models panel rows.
- Any active disable also: makes `resolve_default_alias_target()` return
  `<provider>/unknown`; makes `get_default_provider_name()` ignore an otherwise-live
  temporary override; suspends a temporary alias override in
  `_resolve_model_alias_result` and marks
  `AliasView.override_paused_by_provider_disable`; removes the provider from autodetect
  in `get_configured_default_provider_name()`; raises from
  `raise_if_provider_temporarily_disabled()`; and disqualifies a retry fallback in
  `axe/run_agent_exec_retry.py::_usage_limit_retry_precedence`.
- `xprompt/model_completion.py::_apply_provider_disables` drops concrete model and
  provider rows for disabled providers; `modals/model_picker_rows.py::build_model_rows`
  skips a disabled provider's models entirely.

ACE surfaces:

- `models_panel.py` binds `p` → `action_providers()` (`models_panel_providers.py`),
  which pushes `ProviderRoutingModal` (`models_panel_provider_modal.py`). Its bindings
  are `d`/`enter` = disable or change duration, `x` = enable, `j/k` navigate, `esc/q`
  back. Both write paths run in `run_worker(..., thread=True, exclusive=True)` and
  reload a `ProviderRoutingSnapshot`.
- `models_panel_provider_rendering.py` renders the row
  (`_DISABLED_STYLE = "bold #FFAF5F"`, `_AVAILABLE_STYLE = "bold #87D787"`,
  `_CLI_MISSING_STYLE = "dim #A8A8A8"`), the description strip, the Launch-Control title
  summary, and builds the `DurationPickerModal` (`models_panel_duration.py`, keys
  `1`-`6`, `t`, `c`; the shared `DurationChoiceModal` also has an unused `x` "extra
  choice" binding).
- `widgets/provider_disables_indicator.py` is the top-bar pill
  (`PROVIDER_DISABLE_PALETTE = _PillPalette(accent="#FFAF5F", secondary="#5F3518")`),
  polling `peek_active_provider_disables()` every 30 s.
- Visual goldens exist: `models_panel_provider_routing_modal_120x40.png`,
  `models_panel_provider_routing_modal_narrow_70x32.png`,
  `models_panel_provider_duration_picker_120x40.png`,
  `models_panel_provider_routing_until_cleared_120x40.png`,
  `models_panel_provider_disabled_120x40.png`,
  `provider_disables_indicator_{single,multiple}_120x40.png`.

Launch path:

- ACE: `_launch_start.py::_launch_resolved_prompt` reserves a timestamp, **unmounts the
  prompt bar**, then submits a durable `sase run` proc through
  `actions/agent_durable.py::submit_agent_launch`, whose request payload already carries
  `prompt`, `workflow`, `display_name`, `project_name`, `workflow_name`, and the
  trusted-surface `allow_force_reuse` flag.
- `sase run`: `main/query_handler/_launch.py::launch_query` reads that payload
  (`load_request(RUN_LAUNCH)`), optionally applies a force-reuse rewrite (which replaces
  the prompt and supplies `segment_extra_env`), then calls
  `launch_agents_from_cwd(query, segment_extra_env=...)`.
- `agent/launch_cwd_agents.py::launch_agents_from_cwd_impl` resolves project context,
  runs `parse_multi_prompt()` then `expand_xprompt_swarms_with_metadata()` into
  `expanded_segments` + `expanded_segment_template_groups` +
  `expanded_segment_swarm_xprompts` (+ `expanded_segment_extra_env`), then branches:
  more than one segment → `launch_multi_prompt_agents()`; otherwise `%repeat` recursion,
  then `%alt`/`%model` fan-out via `plan_prompt_fanout_variants()` →
  `launch_multi_prompt_agents(segments=[query], preplanned_fanout_plans=[alt_plan])`,
  else a single `execute_launch_plan(plan_fake_fanout("single", [query]), ...)`.
- `agent/launch_request_planning.py::build_preview_plan(prompt)` already performs
  exactly the read-only half of that enumeration (canonicalize → `parse_multi_prompt` →
  `expand_xprompt_swarms_with_metadata` → multi-segment fake fan-out, or
  repeat/alt/single plan for one segment) for launch-approval previews.
- `llm_provider/launch_selection.py::resolve_launch_selection(directives, ..., consume=False)`
  is the existing non-consuming way to learn the provider/model/effort a prompt would
  use; `xprompt/directives.py::extract_prompt_directives(prompt)` yields the
  `PromptDirectives` it needs.

## Design decisions (settled)

Implementing agents should treat these as decided and not re-litigate them.

**One record, one mode.** A provider has at most one disable, and that record gains a
`mode` of `hard` or `soft`. A second parallel store was rejected: it would duplicate
expiry, locking, provenance, the peek cache, the top-bar pill, and notifications, and it
would invent a "both at once" state that has no meaning. Flipping mode is a rewrite of
the one record.

**Wire schema goes to 2, with a real migration.** Adding a field to a
`deny_unknown_fields` record that the Python facade rehydrates strictly is a wire
change. Bumping to 2 and dropping v1 state was rejected because usage-limit disables are
hours-long: an upgrade mid-window would silently re-enable a rate-limited provider.
Reading a v1 file migrates each record to `mode: "hard"` and rewrites at v2.

**The two rules are selection policy, not new state.** Member availability becomes three
states, and the two selectors keep taking a boolean mask:

| Selector        | Mask                                                           |
| --------------- | -------------------------------------------------------------- |
| `\|` pool       | preferred members only; if none are preferred, sparing members |
| `\|\|` fallback | preferred **and** sparing members                              |

That is precisely the requested behavior: a pool spares a soft member while another
member can cover, and a soft member still wins a fallback chain so no `||` candidate is
diverted. It also leaves `load_balancing.py`'s cursor, fingerprints, and weighting
untouched, so no rotation state migrates.

**A soft disable never fails a launch.** `raise_if_provider_temporarily_disabled()` and
therefore `get_provider()` raise for hard only. Explicit intent (`%model:claude/opus`, a
bare Claude-owned model, `provider_name="claude"`, `SASE_LLM_EXEC_PROVIDER`) keeps
working. Nothing about a soft disable is fail-closed.

**The launch guard is hard-only, and its fast path costs nothing.** Both the launcher
guard and the ACE preflight begin by asking `peek_active_provider_disables()` whether
any **hard** disable is active; when none is, they return immediately and no
enumeration, planning, or alias resolution happens. Launch behavior on a machine with no
hard disable is byte-for-byte what it is today.

**The guard fails closed only on a confirmed block, and fails open on surprises.** A
unit whose every candidate resolves to a hard-disabled provider refuses the launch
before any spawn. Any _unexpected_ exception inside the guard (a planning error, a
resolution bug, an unreadable xprompt) logs a warning and lets the launch proceed
exactly as today. A bug in this subsystem must never be able to block launching.

**CLI-missing providers stay out of scope.** The guard blocks on hard disables only. A
member skipped because its CLI is not installed is reported as context in the panel ("2
of 3 pool members are unavailable") but is not something the guard blocks on or offers
to fix, and no new behavior is added for it.

**The decision unit is one expanded segment.** The user-visible unit of "one agent" for
the panel is one post-swarm-expansion segment — which is exactly what an xprompt swarm's
`---` separators produce, and what the user's four-agent example means. A unit may
itself fan out (`%alt`, `%{%m:a | %m:b}`, `%repeat:k`) into several slots; the guard
resolves **every** slot of the unit so a model-bearing fan-out cannot hide a blocked
branch, but abort and model decisions apply to the whole unit. Slot-level abort was
rejected: it is not expressible in prompt text, and expressing it would mean threading a
slot-keyed decision table through four launcher layers whose plans are re-derived out of
process.

**"Pick a different model" is offered only when the unit has one model.** When a unit's
slots resolve to more than one distinct provider/model — i.e. the prompt itself fans out
models — a single `%model` rewrite would silently collapse the fan-out, so that row is
replaced by a dim line telling the user to edit the prompt (aborting keeps the draft in
the prompt bar). Every other action stays available for such a unit.

**ACE decisions travel as a resolved unit bundle, and only when needed.** Enable,
soft-enable, and abort-everything need no transport: they are a write plus "submit
unchanged" or "submit nothing". A single-unit launch needs none either — a model change
is a rewrite of the one prompt the TUI already holds. Only a multi-unit launch that
drops a unit or re-models one submits a `launch_units` list (prompt + template group +
swarm xprompts per surviving unit) in the `sase run` request payload, which
`launch_agents_from_cwd_impl` consumes in place of its own expansion. This mirrors the
existing `segment_extra_env` mechanism and lands at the seam that already produces those
three parallel lists.

**Force-reuse wins over a unit bundle.** If the payload also triggers a force-reuse
rewrite (`%id:!name` from a kill-and-edit relaunch), `launch_query` applies force-reuse
and ignores `launch_units`; the guard then refuses with its normal actionable error
rather than launching something the user aborted. Kill-and-edit relaunches are
single-unit prompts, where the panel never needs the bundle at all.

**Preflight before unmount.** ACE runs the preflight while the prompt bar is still
mounted and only unmounts when a launch is actually submitted, so aborting everything
leaves the user's draft exactly where it was.

**No feature flag.** Every phase ships a coherent, honest behavior: `core-mode` and
`routing` change nothing until someone writes a soft disable; `provider-ui` is the first
way to write one; `guard-core` upgrades "spawn an agent that dies at invoke time" to "a
clear refusal before spawning", which is already strictly better and promises nothing
more; `guard-panel` upgrades ACE's refusal into an interactive resolution. Per
`sase/memory/sase_flags.md` a flag exists for a disabled beta or an early landed path
that does not yet work, and no phase here ships an unready user-reaching promise.

**Rust core boundary.** Only the disable record's shape and migration belong in
`../sase-core` (`sase/memory/`-declared boundary: a web app or another frontend would
need the same record). The selection policy, the guard, and every ACE surface are Python
and Textual glue and stay in this repo. The guard is deliberately not in the Rust core:
it composes Python-side prompt parsing, xprompt expansion, and alias resolution.

**Out of scope.** No new `sase` CLI subcommand (the store's only interactive writer
stays Launch Control), no glossary term, no memory-file edits, no change to usage-limit
detection (auto-disables stay hard), and no change to `CHANGELOG.md` (release-please
owns it; use Conventional Commits).

## Vocabulary

Use these words consistently in code, UI, and docs:

- **hard disable** — today's disable. UI state text stays `disabled`.
- **soft disable** — the new mode. UI state text is `soft`, and the verb in prose is
  "spares" ("pools spare CLAUDE while another member can cover").
- **preferred / sparing / unavailable** — the three member-availability states.
- **unit** — one expanded segment; the thing the panel calls "this agent".

## Verification for every phase

Run `just install` first: these `sase_<N>` workspaces are ephemeral and dependencies
drift. Then run `just check` after making changes. Before the epic's combined tree
lands, run `just check-full` through `/sase_monitor` (never inline), handing it a
`--next` action.

Phases that add a public symbol a later phase consumes should add
`--epic-symbol <epic_bead_id>(<symbol>)` to the Symvision invocation in the `Justfile`
and remove that entry in the phase that wires up the real consumer (see
`sase/memory/symvision.md`). The phase order below is chosen so this should rarely be
needed.

Every ACE change obeys `sase/memory/tui_perf.md`: no disk I/O, JSON parsing, or alias
resolution on the event loop or in a render path; keystroke paths read through the
lock-free `peek_active_provider_disables()`; every provider-disable write and every
guard preflight runs in `run_worker(..., thread=True, exclusive=True)`; programmatic
`OptionList.highlighted` assignments keep their guard flag.

---

## Provider-disable mode on the Rust wire

All work in this phase is in the `sase-core` repo. Open it the sanctioned way first:

```bash
sase repo open sase-core -r "Add hard/soft mode to the provider-disable wire record"
```

Use the printed path for every read and write. Follow that repo's `AGENTS.md`: never
edit `[workspace.package].version` or crate versions (release-plz owns them), use
Conventional Commits (this is an additive `feat:` with a migration, not a `feat!:` — v1
state is migrated, not rejected), and verify with `just check` (or `./scripts/check.sh`)
from the repo root, never `cargo test -p sase_core` alone.

### Changes

1. `crates/sase_core/src/provider_disable.rs`:
   - `pub const PROVIDER_DISABLE_WIRE_SCHEMA_VERSION: u32 = 2;`
   - Add a serialized mode enum next to the wire structs:

     ```rust
     #[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
     #[serde(rename_all = "lowercase")]
     pub enum ProviderDisableMode {
         Hard,
         Soft,
     }
     ```

     Give it `as_str()` returning `"hard"`/`"soft"` and a `parse` helper that maps those
     two exact strings and rejects anything else with
     `ProviderDisableError::Validation`.

   - `ProviderDisableWire` gains `pub mode: ProviderDisableMode` (keep
     `deny_unknown_fields`; the field is required at v2).
   - `set_provider_disable_relative`, `set_provider_disable_until`,
     `try_set_provider_disable_relative`, and `try_set_provider_disable_until` each take
     a `mode: ProviderDisableMode` and pass it to `candidate_record`.
   - `read_records_locked` migrates: deserialize into `RawProviderDisableStateWire`;
     when `raw.version == 1`, parse each record with a v1-shaped struct, build the v2
     record with `mode: Hard`, mark `changed = true` so the existing atomic rewrite
     persists the migration, and continue with the normal validity/expiry filtering.
     Keep today's "delete the file" behavior for any other unsupported version and for
     unparseable bytes.
   - `is_valid_record_for_key` keeps its current checks; the mode is validated by
     deserialization.

2. `crates/sase_core/src/lib.rs` — export `ProviderDisableMode` alongside the existing
   provider-disable re-exports.

3. `crates/sase_core_py/src/lib.rs`:
   - The four setter bindings gain a `mode: &str` argument, parsed through the new
     helper so an invalid value raises `ValueError` from
     `provider_disable_error_to_pyerr`. Place it after `source` in each
     `#[pyo3(signature = ...)]` and default it to `"hard"`, so an older caller keeps
     writing hard disables and the argument order stays additive.
   - Update the module docstring's binding list at the top of the file to show the new
     signatures.

4. Tests in `crates/sase_core/src/provider_disable.rs` (unit) and the `sase_core_py`
   binding tests:
   - A v2 round trip for each mode through get/set/try-set/clear.
   - A v1 file on disk (fixture JSON without `mode`) is read as one hard record **and**
     rewritten at v2 on disk, with `created_at`/`expires_at`/`source` preserved.
   - A v1 record whose `expires_at` has passed is pruned during migration.
   - An unknown `mode` string in an otherwise valid v2 record is pruned like any other
     invalid record (the file survives if another record is valid).
   - An unsupported version (3) still deletes the file.
   - An invalid `mode` argument to a setter raises rather than writing.
   - `provider_disable_wire_schema_version()` returns 2.

### Done when

`just check` is green in `sase-core`, a v1 state file is migrated in place to v2 with
every record hard, and the four setters refuse an unknown mode. Nothing in the `sase`
repo changes in this phase.

---

## Mode-aware routing policy

Teach this repo's routing layer the mode. After this phase a soft disable is fully
functional through configuration and the Rust store; only the UI to create one is
missing.

Run `just install` first: it rebuilds `sase_core_rs` from the local `sase-core`
checkout, which is how the new binding signature becomes importable. If `just install`
reports the checkout is behind the published floor, follow its printed instructions — do
**not** hand-edit the `sase-core-rs` window in `pyproject.toml` (the release-branch
reconciler ratchets it at release time).

### Changes

1. `src/sase/llm_provider/provider_disable.py`:
   - `PROVIDER_DISABLE_WIRE_SCHEMA_VERSION = 2`.
   - Module constants `PROVIDER_DISABLE_MODE_HARD = "hard"`,
     `PROVIDER_DISABLE_MODE_SOFT = "soft"`, and
     `PROVIDER_DISABLE_MODES = (PROVIDER_DISABLE_MODE_HARD, PROVIDER_DISABLE_MODE_SOFT)`.
   - `TemporaryProviderDisable` gains `mode: str`, included in the strict `required` set
     in `from_wire` and validated against `PROVIDER_DISABLE_MODES`. Add
     `is_hard`/`is_soft` properties; every consumer compares through those, never
     against a raw string.
   - `disable_provider`, `disable_provider_until`, `try_disable_provider`, and
     `try_disable_provider_until` gain a keyword-only
     `mode: str = PROVIDER_DISABLE_MODE_HARD`, validated locally before the binding
     call, and pass it through.
2. `src/sase/llm_provider/provider_disable_peek.py` — accept v2 records; when the file
   is still at version 1, read each record with `mode` defaulted to hard instead of
   returning an empty mapping, so the top bar and completion stay honest in the window
   between an upgrade and the first authoritative read. Never write.
3. `src/sase/llm_provider/load_balancing.py` — add the selection policy next to the two
   selectors that consume it:

   ```python
   class MemberAvailability(StrEnum):
       PREFERRED = "preferred"
       SPARING = "sparing"
       UNAVAILABLE = "unavailable"


   def pool_availability_mask(states: Sequence[MemberAvailability]) -> list[bool]:
       """Spare sparing members while any preferred member can cover."""


   def fallback_availability_mask(states: Sequence[MemberAvailability]) -> list[bool]:
       """Treat sparing members as available so a soft disable never diverts `||`."""
   ```

   Both docstrings state the rule in full. Neither selector's signature, cursor
   handling, fingerprinting, nor weighting changes.

4. `src/sase/llm_provider/registry.py`:
   - `provider_routing_available()` now returns `False` only for an unregistered
     provider, a **hard** disable, or a missing CLI. Update its docstring to say a soft
     disable stays routable but is deprioritized, and name `provider_routing_state()`.
   - Add
     `provider_routing_state(provider_name, provider_disables=None) -> MemberAvailability`:
     `UNAVAILABLE` when `provider_routing_available()` is `False`, `SPARING` when the
     captured disable `is_soft`, else `PREFERRED`.
   - `raise_if_provider_temporarily_disabled()` raises only when the disable `is_hard`.
   - `get_configured_default_provider_name()` walks `autodetect_candidates` twice: first
     accepting only `PREFERRED` candidates, then accepting `SPARING` ones. The existing
     `RuntimeError` stays for "nothing at all".
   - `get_default_provider_name()` keeps an active temporary override unless its
     provider has a **hard** disable.
5. `src/sase/llm_provider/model_alias_resolution.py`:
   - Add `resolved_target_availability(target, provider_disables, *, available)` which
     layers the mode on top of the existing boolean: `UNAVAILABLE` when `available` is
     false, `SPARING` when the target's provider has a soft disable, else `PREFERRED`.
     Keep `resolved_target_is_available()` and the
     `config.__dict__["_resolved_target_is_available"]` monkeypatch seam exactly as they
     are — the tri-state is derived from that boolean plus the disables snapshot, so
     existing test seams keep working.
   - Both selector sites (the plain one and the suspended-override one) build `states`
     instead of `availability`, then pass `pool_availability_mask(states)` to
     `select_model_alias_pool_member` and `fallback_availability_mask(states)` to
     `select_model_alias_fallback_member`.
   - `_resolve_model_alias_result` suspends a temporary alias override only for a
     **hard** disable (`suspended_disable` is `None` for soft, so a soft-disabled
     override target stays applied).
   - `resolve_default_alias_target()` returns `<provider>/unknown` only for a hard
     disable.
   - `ModelAliasSelectorMember` gains `sparing: bool = False` (populated from the
     member's state); `available` keeps meaning "selectable at all" and is therefore
     `True` for a sparing member. `model_alias_selector_details()` computes the selected
     index through the same two mask helpers so display and routing cannot disagree.
6. `src/sase/llm_provider/alias_view.py` — the override-pause check at the
   `provider_disables` lookup must key on a hard disable, matching decision 5;
   `override_paused_by_provider_disable` stays `None` for a soft disable.
7. `src/sase/llm_provider/model_launch_settings.py` — the local availability list built
   for a raw (unresolved) selector uses the same tri-state plus mask helpers, and the
   `paused_disable` short-circuit keys on hard.
8. `src/sase/axe/run_agent_exec_retry.py` — `_usage_limit_retry_precedence` disqualifies
   a retry fallback only for a hard disable, so a soft-disabled fallback provider still
   rescues a usage-limit failure. Update the surrounding docstring, which currently says
   "resolves to the same now-disabled provider".
9. `src/sase/doctor/checks_config_model_aliases.py` — a sparing pool member gets its own
   note ("... is soft-disabled and will be spared while another member is available")
   distinct from the unavailable-member note; a fallback chain whose winner is sparing
   is not reported as a problem.
10. `docs/llms.md` — extend the "Provider disables are an availability layer" table and
    the selector prose with the hard/soft split (rows for `round-robin alias` /
    `ordered fallback` / `direct provider/model` / `temporary alias override` /
    `autodetect` under a soft disable), and note that `source` and `mode` are
    independent (a usage-limit auto-disable is always hard).

### Tests

- `tests/test_provider_disable.py` — `mode` round trips, defaults to hard, rejects an
  unknown value, and `is_hard`/`is_soft` agree.
- `tests/llm_provider/test_provider_disable_peek.py` — a v1 file reads as hard records
  and is not rewritten by the peek.
- New `tests/llm_provider/test_provider_disable_soft_routing.py`, building on
  `tests/llm_provider/_load_balanced_alias_helpers.py`:
  - a 3-member pool with one soft member skips it while another is preferred, and the
    cursor advances from the winner;
  - a pool whose members are **all** soft rotates among them (nothing is skipped) and
    the cursor still advances;
  - a pool with one soft and one CLI-missing member selects the soft member;
  - a `||` chain whose first candidate is soft selects that first candidate (no
    diversion), while a hard first candidate is still skipped;
  - `get_provider()` on a soft-disabled provider returns a provider; on a hard one it
    raises `ProviderTemporarilyDisabledError`;
  - autodetect prefers a preferred candidate over a soft one and falls back to the soft
    one when it is the only candidate;
  - a temporary alias override on a soft-disabled provider stays applied, and on a hard
    one is still suspended.
- `tests/llm_provider/test_provider_disable_routing.py` — keep every existing
  hard-disable assertion passing unchanged; that file is the regression net for this
  phase.
- Doctor and retry-precedence unit tests for items 8 and 9.

### Done when

Soft and hard disables written directly through the facade produce the two documented
routing behaviors, every pre-existing hard-disable test still passes, and `just check`
is green.

---

## Launch Control soft-disable workflow

Give the user the workflow to create, flip, and see a soft disable, in the same place
they disable a provider today.

### The interaction

Provider Routing keeps `d`/`enter` for a hard disable and `x` for enable, and adds `s`
for a soft disable. The key chooses the mode; the duration picker always follows, so
intent is one keypress deep and there is no mode sub-modal:

- `d`/`enter` on any row → "Disable CLAUDE" duration picker → writes a hard disable.
- `s` on any row → "Soft-disable CLAUDE" duration picker → writes a soft disable.
- `x` → clears whichever disable is active.
- When the highlighted row already has an active disable **in the other mode**, the
  duration picker's first row becomes `x  Keep current window (1h 42m left)` — one
  keypress to flip hard ↔ soft without re-choosing a window. It is omitted when the mode
  matches, so no row is ever a no-op.

That last row is why the mode lives on a key rather than inside the duration picker: the
picker stays a pure "how long" question, and mode conversion appears exactly when it is
meaningful.

### Changes

1. `src/sase/ace/tui/modals/models_panel_duration.py` — `DurationPickerModal` gains a
   keyword-only `keep_current: KeepCurrentWindow | None = None`. When set, prepend a
   `DurationChoice(key="x", title=f"Keep current window ({remaining})", tone="accent", value=keep_current)`
   — reusing `DurationChoiceModal`'s existing unused `x` binding. `KeepCurrentWindow` is
   a new frozen dataclass carrying `expires_at: float | None`, so the write path can
   reuse the stored expiry verbatim.
2. `src/sase/ace/tui/modals/models_panel_provider_rendering.py`:
   - `provider_duration_modal(provider, *, mode, keep_current)` — mode-specific title
     (`Disable CLAUDE` / `Soft-disable CLAUDE`) and subtitles; the soft subtitles say
     what soft means ("Spare CLAUDE in pools that have another option; explicit `%model`
     still runs").
   - `render_provider_row` gains a `soft` branch: state text
     `soft · <provenance> · <remaining>` in a new
     `_SOFT_DISABLED_STYLE = "bold #FFD75F"`, visually between available-green and
     disabled-orange.
   - `provider_description_text` gains a soft branch spelling out both rules in two
     lines: pools spare it while another member can cover, `||` fallbacks and explicit
     `%model` still use it, and running processes are untouched.
   - `provider_title_line` renders soft entries with the soft style and a `soft` marker
     so the Launch Control title distinguishes the two kinds.
   - `duration_suffix` handles `KeepCurrentWindow` ("with its current window").
3. `src/sase/ace/tui/modals/models_panel_provider_modal.py`:
   - Add `("s", "soft_disable", "Soft disable")` to `BINDINGS`;
     `action_disable_or_change` and a new `action_soft_disable` share one
     `_begin_disable(mode)` helper that stores the pending mode, computes `keep_current`
     from the highlighted row's active disable when the mode differs, and pushes the
     picker.
   - `ProviderWriteOutcome` carries the written mode; `_submit_disable` passes `mode=`
     to `disable_provider`/`disable_provider_until` and, for `KeepCurrentWindow`, calls
     `disable_provider_until(expires_at)` (or `disable_provider(None)` when the stored
     window was "until cleared").
   - Toasts name the mode: `CLAUDE soft-disabled for 2h; alias routing refreshed.` and
     `CLAUDE disabled 2h (was soft); alias routing refreshed.`
   - Footer:
     `[green]d/enter[/green]=Disable  [yellow]s[/yellow]=Soft disable [green]x[/green]=Enable  [dim]j/k[/dim]=Navigate  [dim]esc[/dim]=Back`.
     `#provider-routing-container` is already `width: 86; max-width: 95%`, so no style
     change should be needed — check the existing `..._modal_narrow_70x32` golden and
     shorten the footer labels rather than widening the container if it wraps.
4. `src/sase/ace/tui/widgets/_override_pill.py` and
   `widgets/provider_disables_indicator.py` — add
   `PROVIDER_SOFT_DISABLE_PALETTE = _PillPalette(accent="#FFD75F", secondary="#4A3A12")`.
   The pill shows `CLAUDE off 1h` for hard and `CLAUDE soft 1h` for soft; with several
   disables active it renders the hard one first (more severe) with `+N`, and uses the
   soft palette only when every active disable is soft. The tooltip lists each
   provider's mode and ends with the two-line explanation of what soft means.
5. `src/sase/ace/tui/modals/model_picker_rows.py::build_model_rows` — skip a provider's
   models only for a hard disable. A soft-disabled provider keeps its header and models,
   with the header labelled `CLAUDE  4 models  soft` and rows dimmed one step. Picking
   one is allowed: that is the whole point of soft.
6. `src/sase/xprompt/model_completion.py::_apply_provider_disables` — drop concrete
   model and provider rows for hard disables only; keep soft rows and annotate them
   `soft` in the same place alias rows show provenance.
7. `src/sase/ace/tui/modals/models_panel_provider_state.py::disabled_explicit_provider_message`
   returns `None` for a soft disable (an explicit soft target is valid, not a
   rejection); add a sibling `soft_explicit_provider_note()` used by the alias-edit and
   override inputs to show an informational line instead of a rejection.
8. Selector member rendering — a sparing member renders a `soft` chip in the soft style
   and is still counted in the `pool <available>/<total>` chip, because it remains
   selectable. The three sites are `models_panel_rendering_rows.py` (the
   `append_pool_chip(text, sum(member.available ...), len(members))` count),
   `models_panel_rendering_descriptions.py` (the per-member `✓`/`×` marker and style),
   and `models_panel_selector_builder.py` (the builder's own `✓`/`×` from
   `resolved_target_is_available`).
9. `docs/ace.md` — extend the "Provider routing controls" state table with
   `soft · manual · <time> left` and `soft · usage-limit automatic · <time> left`,
   document the `s` key, the keep-current-window row, and that a soft-disabled provider
   stays in the model picker and `%model` completion. `docs/llms.md` — cross-reference
   from the availability-layer section added in `routing`.

### Tests

- `tests/test_models_panel_provider_modal.py` — `s` writes a soft disable with the
  picked duration; `d` on a soft row offers the keep-current-window row and flipping
  preserves `expires_at`; `s` on a soft row does **not** offer it; `x` clears either
  mode; the mode appears in the toast.
- `tests/test_models_panel_provider_rendering.py` — row text, description strip, and
  title summary for each of available / soft / hard / CLI-missing.
- `tests/test_provider_disables_indicator.py` — soft-only, hard-only, and mixed pills
  plus tooltips.
- Model picker and `%model` completion tests: a hard-disabled provider's models are
  absent; a soft-disabled provider's models are present, marked, and selectable.
- PNG snapshots (`just test-visual`, accept with `--sase-update-visual-snapshots`): add
  `models_panel_provider_soft_disabled_120x40`,
  `models_panel_provider_duration_picker_keep_window_120x40`, and
  `provider_disables_indicator_soft_120x40`; refresh the existing provider-routing and
  indicator goldens for the new footer/state text.

### Done when

A user can soft-disable a provider from Launch Control in two keypresses, flip an
existing disable's mode without losing its window, see the mode everywhere the hard
state is shown today, and still pick that provider's models; `just check` and
`just test-visual` are green.

---

## Fail-closed launch guard

Stop a launch that can only run on a hard-disabled provider **before** anything spawns,
on every surface, and give ACE the transport it needs in the next phase.

### 1. Launch units

New module `src/sase/agent/launch_guard.py`:

```python
@dataclass(frozen=True)
class LaunchUnitCandidate:
    """One fan-out slot of a unit and the provider/model it would use."""

    slot_index: int
    prompt: str
    provider: str | None
    model: str | None
    blocked_by: TemporaryProviderDisable | None   # hard disables only
    unavailable: bool                             # unregistered or CLI missing


@dataclass(frozen=True)
class LaunchUnit:
    """One expanded prompt segment: what the panel calls "this agent"."""

    index: int
    total: int
    prompt: str
    template_group: str | None
    swarm_xprompts: tuple[str, ...]
    candidates: tuple[LaunchUnitCandidate, ...]

    @property
    def blocked(self) -> bool: ...          # no candidate can run
    @property
    def blocking_providers(self) -> tuple[str, ...]: ...
    @property
    def single_model(self) -> str | None: ...  # None when candidates disagree


def plan_launch_units(prompt: str) -> tuple[LaunchUnit, ...]: ...
def blocked_launch_units(prompt: str) -> tuple[LaunchUnit, ...]: ...
```

`plan_launch_units` mirrors `build_preview_plan`'s read-only enumeration exactly:
`canonicalize_project_aliases_in_prompt` → `parse_multi_prompt` →
`expand_xprompt_swarms_with_metadata` → per segment, the same repeat/alt planning
(`plan_agent_launch_fanout(..., launch_kind="repeat")`, then
`plan_prompt_fanout_variants`, with the `"#" in segment` xprompt-expansion retry, else
`plan_fake_fanout`). Each slot's `PromptDirectives` come from
`extract_prompt_directives(slot.prompt)` and each candidate's provider/model from
`resolve_launch_selection(directives, consume=False, provider_disables=<one captured snapshot>)`
— `consume=False` is mandatory: the guard must never advance a pool cursor.

A candidate is `blocked_by` when its resolved provider has a **hard** disable in that
snapshot. A unit is `blocked` when every candidate is either blocked or unavailable, and
at least one is blocked (a unit that is only CLI-unavailable is not this guard's
business).

`blocked_launch_units` is the entry point everything else calls. It starts with the fast
path: if `peek_active_provider_disables()` contains no hard disable, return `()` without
planning anything.

### 2. Refusal

Also in `launch_guard.py`:

```python
class DisabledProviderLaunchError(RuntimeError):
    """Raised before any spawn when a unit needs a hard-disabled provider."""
```

Its message names, for the first blocked unit, the provider(s), the remaining window,
why the unit has no alternative (explicit `%model`, an exhausted pool, an exhausted
fallback, or the launch default), the agent index when the launch has more than one
unit, and the remedy:
`Enable it in ACE Launch Control (,m → p) or choose another model.` Reuse
`format_provider_disable_expiry` and `provider_disable_provenance_label` so the wording
matches the rest of the system.

`src/sase/agent/launch_cwd_agents.py::launch_agents_from_cwd_impl` calls
`guard_launch_units(...)` immediately after `expanded_segments` and its metadata lists
are built and before `resolve_agent_name_key_markers`, i.e. before any branch, any
workspace claim, and any spawn. It records the failed prompt through the existing
`record_failed_launch_prompt(...)` so the draft stays recoverable, then raises.

The guard is wrapped so that only a `DisabledProviderLaunchError` escapes: any other
exception is logged at warning with `exc_info=True` and swallowed, and the launch
proceeds exactly as it does today. This fail-open rule is not optional — write it as a
test.

Because the check runs once over the whole submission before the first spawn, a blocked
multi-segment launch cannot half-launch, so no rollback path is involved.

### 3. ACE-resolved unit bundle

`main/query_handler/_launch.py::launch_query` reads an optional `launch_units` list from
the request payload: each entry is
`{"prompt": str, "template_group": str | null, "swarm_xprompts": [str]}`. Validate
strictly (a list of objects with exactly those keys, non-empty prompts) and reject a
malformed payload with the existing `emit_run_launch_result(success=False, ...)` +
`sys.exit(1)` shape. When a force-reuse rewrite applies, ignore `launch_units` entirely
(decision above).

`launch_agents_from_cwd()` / `launch_agents_from_cwd_impl()` gain a keyword-only
`launch_units: Sequence[LaunchUnitInput] | None = None` (mirroring `segment_extra_env`).
When supplied, `launch_agents_from_cwd_impl` uses it in place of its own
`parse_multi_prompt` + `expand_xprompt_swarms_with_metadata` result — the three parallel
lists (`expanded_segments`, `expanded_segment_template_groups`,
`expanded_segment_swarm_xprompts`) come straight from the bundle, and
`multi.local_xprompts` still comes from `parse_multi_prompt(query)` so
frontmatter-defined local xprompts survive. Everything downstream (naming, VCS refs,
fan-out, clans) is unchanged. `segment_extra_env`, when also supplied, must have one
entry per bundle entry; mismatched lengths raise the existing `ValueError`.

The guard still runs on the bundle, so a bundle that somehow still contains a blocked
unit is refused rather than launched.

### Tests

New `tests/agent/test_launch_guard.py`:

- No hard disable → `blocked_launch_units()` returns `()` and does not plan (assert by
  monkeypatching `parse_multi_prompt` to raise).
- `%model:<provider>/<model>` on a hard-disabled provider → one blocked unit whose
  reason is the explicit model.
- The same with a **soft** disable → no blocked unit.
- A four-segment prompt with two segments on a hard-disabled provider → exactly those
  two units blocked, with `index`/`total` of 2/4 and 4/4.
- A pool alias whose every member is hard-disabled → blocked, `blocking_providers` lists
  each distinct provider once; with one member enabled → not blocked.
- A `||` fallback whose first candidate is hard-disabled and second is available → not
  blocked.
- `%{%m:<blocked> | %m:<ok>}` → the unit is **not** blocked (one candidate can run) but
  `single_model` is `None`; with both branches blocked → blocked.
- `%repeat:3` on a blocked provider → one unit, not three.
- Consistency: for a matrix of prompts (single, `---` swarm, `%alt`, `%repeat`, an
  xprompt swarm fixture), `plan_launch_units` produces the same slot prompts in the same
  order as `build_preview_plan`. This is the guard against the two enumerators drifting.
- No pool cursor moves: assert `~/.sase/llm_lb.json` is byte-identical across a guard
  run.

`tests/agent/test_launch_cwd_guard.py`:

- A blocked prompt raises `DisabledProviderLaunchError` with no workspace claimed and no
  spawn (assert the spawn hook was never called and the prompt was recorded as failed).
- An internal guard failure (monkeypatch `blocked_launch_units` to raise `ValueError`)
  logs a warning and launches normally.
- A supplied `launch_units` bundle launches exactly those units with their template
  groups and swarm xprompts, and a shorter bundle launches fewer agents than the raw
  prompt would.
- `launch_query` rejects a malformed `launch_units` payload and prefers force-reuse over
  a bundle.

### Done when

`sase run` (and every non-ACE surface) refuses a launch that needs a hard-disabled
provider with an actionable message and no spawned agent, a soft disable never refuses,
the guard costs nothing when no hard disable is active, an internal guard error cannot
block a launch, and `just check` is green.

---

## The ACE disabled-provider launch panel

Turn ACE's refusal into a resolution. The panel is the user-facing centerpiece of this
epic, so its wording, keys, and layout are specified in full.

### Layout

A single-screen, single-keypress chooser reusing the established
`.duration-choice-container` idiom (`prompt_submit_choice_modal.py` is the closest
reference), in a new `src/sase/ace/tui/modals/disabled_provider_launch_modal.py` with
`#disabled-provider-launch-container` in `styles.tcss` (`width: 76`,
`border: double #FF875F`).

```
                    Provider disabled · agent 2 of 4

  CLAUDE   disabled · manual · 1h 42m left

  This prompt asks for claude/opus with %model, so there is no fallback to
  route to.

  » #gh:sase %m:opus Fix the flaky selector test in tests/ace/tui/…

  e   Enable CLAUDE, then launch this agent
        Clears the disable for every later launch too.
  s   Soft-enable CLAUDE, then launch this agent
        Keeps sparing it in pools; this agent still runs on it.
  m   Pick a different model for this agent…
  a   Abort this agent
        The other 3 agents in this launch still start.
  A   Abort all 4 agents

  esc = abort this agent
```

Rules for the rows:

- `e` clears every provider blocking **this** unit, then re-checks the unit.
- `s` rewrites each blocking provider's disable to `soft` with its **current window**
  preserved, then re-checks. This is the row that pays off the rest of the epic: it is
  the exact answer to "I am conserving this provider but this agent needs it".
- `1`…`9` appear **only** when two or more providers block the unit, one row per
  provider ("Enable CODEX only, then re-check"), after `e`/`s`.
- `m` appears only when `unit.single_model` is not `None`. Otherwise it is replaced by a
  dim line: `This prompt fans out models; press esc and edit it to change a branch.`
- `A` appears only when the launch has more than one unit; its label always states the
  real total.
- `esc`/`q` are aliases for `a`.

The context block is built from the unit: the provider list with mode, provenance, and
remaining window; a one-sentence reason (`explicit %model`,
`every member of @large is disabled`, `the @xlarge fallback has no available candidate`,
`the launch default resolves to …`), including how many pool members were unavailable
for a non-disable reason when that is part of the story; and a single-line elided prompt
preview.

### Flow

In `src/sase/ace/tui/actions/agent_workflow/_launch_start.py`:

1. `_launch_resolved_prompt` first calls a new
   `_preflight_provider_disables(prompt, keep_bar)`. Its fast path is synchronous and
   free: if `peek_active_provider_disables()` has no hard disable, fall through to
   today's code unchanged. **The prompt bar is not unmounted yet.**
2. Otherwise start a
   `run_worker(..., thread=True, exclusive=True, group="launch-provider-guard")` that
   returns `blocked_launch_units(prompt)` plus the full unit list. Nothing about the
   enumeration touches the event loop.
3. On success with no blocked units, continue into the existing submit path unchanged.
4. With blocked units, resolve them one at a time in unit order: push the panel for the
   first blocked unit, and in its callback act on the decision and move to the next. A
   provider enabled while resolving unit 2 can unblock unit 3, so re-run
   `blocked_launch_units` (in the same worker group) after any write before choosing the
   next panel — that is also what makes the per-provider `1`…`9` loop work.
5. Provider writes (`enable_provider`, `disable_provider_until(..., mode="soft")`) run
   in the worker, never on the event loop, and reuse the modes/helpers from
   `provider-ui`.
6. `m` pushes the existing
   `ModelPickerModal(title="Model for this agent", include_default_option=False, provider_disables=<current snapshot>)`,
   which already excludes hard-disabled providers and (after `provider-ui`) includes
   soft ones. The chosen target replaces or inserts `%model:<target>` in that unit's
   prompt.
7. Finally:
   - every unit aborted → notify `Launch aborted; your prompt is still here.`, leave the
     bar mounted with the draft, submit nothing;
   - no unit modified and none aborted → submit exactly as today;
   - one unit of one modified → submit the rewritten prompt through today's path (no
     bundle);
   - a multi-unit launch with any unit dropped or re-modelled → submit with the
     `launch_units` bundle from `guard-core` in `extra_payload`, and use the surviving
     units' prompts joined by `\n---\n` as the `prompt` field so history, the toast, and
     the proc row describe what actually launched.
8. Only when something is submitted does the bar unmount and `_prompt_context` clear —
   the existing `keep_bar` semantics are preserved on both paths.
9. Re-read the prompt bar's live state after the worker returns rather than trusting
   what was captured before it (`sase/memory/tui_perf.md` rule 4); if the bar was
   unmounted or the context released while the panel was open, abort with a warning
   toast instead of launching a stale prompt.

`_prompt_bar_requests.py`, `_entry_relaunch.py`, `_entry_quick_launch.py`, and
`_mentor_review.py` all reach launches through `_finish_agent_launch` /
`_launch_resolved_prompt`, so they inherit the guard with no changes. Verify that during
implementation and add a test for the relaunch path.

### Tests

`tests/ace/tui/test_disabled_provider_launch_panel.py` (Textual pilot, following
`tests/ace/tui/_agent_launch_helpers.py`):

- No hard disable → no panel, no worker, submit signature unchanged (guards the
  zero-cost fast path).
- Explicit `%model` on a hard-disabled provider → panel appears with one provider, no
  digit rows, no `A` row; `e` writes the enable and submits the original prompt.
- `s` writes a soft disable preserving `expires_at` and submits.
- `m` → picker → submits a prompt whose `%model` is the picked target; an existing
  `%model` is replaced, not duplicated.
- Four-unit swarm with units 2 and 4 blocked → the panel appears twice in sequence; `a`
  then `a` submits a two-unit `launch_units` bundle; `A` on the first panel submits
  nothing and leaves the draft in the bar.
- Two providers blocking one unit (an exhausted pool) → digit rows present; enabling one
  provider re-checks and dismisses the panel when that is enough, and re-renders with
  the remaining blocker when it is not.
- A model-fan-out unit shows the dim edit-your-prompt line and no `m` row.
- Aborting every unit leaves the prompt bar mounted with the original text.
- The relaunch entry point (`_entry_relaunch`) hits the same panel.

PNG snapshots: `disabled_provider_launch_panel_120x40`,
`disabled_provider_launch_panel_swarm_120x40` (with the `A` row and digit rows),
`disabled_provider_launch_panel_narrow_70x32`.

`docs/ace.md` — a new subsection under the Launch Control / provider material describing
the panel, every key, when each row appears, and the fact that aborting keeps the
prompt.

### Done when

Launching an agent that needs a hard-disabled provider opens the panel instead of
failing; each of the six actions behaves as specified; a swarm shows one panel per
blocked agent in sequence and launches exactly the agents the user kept; aborting
everything preserves the draft; and `just check` plus `just test-visual` are green.
