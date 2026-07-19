---
tier: epic
title: Rename %name to %id and add clan-scoped agent ids
goal: 'The %name|%n directive is renamed to %id|%i, %clan becomes a create-only declaration
  that errors when the clan already exists or is declared twice, and a new clan= kwarg
  on %id lets every other member join (or implicitly create) a clan with a derived
  <clan>.<id> name — with all launch, retry, bead-epic, TUI, docs, and template surfaces
  migrated so nothing breaks.

  '
phases:
- id: rename
  title: Rename %name|%n to %id|%i everywhere
  depends_on: []
  description: '''Phase 1: Rename %name|%n to %id|%i'' section: swap the canonical
    directive tables to id/i, add targeted migration errors for the old spellings,
    and update every consumer, emitter, completion surface, bundled template, doc,
    and test.'
- id: clan-kwarg
  title: Add the clan= kwarg to %id
  depends_on:
  - rename
  description: '''Phase 2: Add the clan= kwarg to %id'' section: parse %id(<id>, clan=<clan>),
    derive the <clan>.<id> agent name, add the clan-membership flags to PromptDirectives,
    and enforce the new directive-combination validation errors.'
- id: clan-lifecycle
  title: Single-declaration clan lifecycle
  depends_on:
  - clan-kwarg
  description: '''Phase 3: Single-declaration clan lifecycle'' section: make %clan
    create-only (error if the clan exists or is declared in more than one prompt),
    make clan= joiners join-or-create, and migrate retry/relaunch, bead epic, and
    TUI tagging flows to the join form.'
- id: smoke
  title: End-to-end exercises of the new grammar
  depends_on:
  - clan-lifecycle
  description: '''Phase 4: End-to-end exercises'' section: exercise swarm-style declare+join
    launches, every new validation error, clan-member retries, and bead epic launch/re-work
    through the real launch machinery.'
create_time: 2026-07-19 12:04:56
status: wip
---

# Plan: Rename %name to %id and add clan-scoped agent ids

## Context

An agent can only join a clan when its name is inside the hood defined by the clan name (an agent joining clan `foo`
must be named `foo.<suffix>`). That makes today's grammar redundant and conceptually wrong in two ways:

1. Every clan member must spell the full hood-qualified name in `%name` even though the clan prefix is implied by
   `%clan`.
2. `%name(bar)` claims the agent is "named bar" when clan membership actually forces the name to be `foo.bar` — the
   directive value is really an _id inside the clan_, not the name.

The fix, per the user's request:

- Rename `%name|%n` to `%id|%i`. The directive supplies the agent's _id_; for clan members the final name is derived.
- `%clan(<clan>)` becomes a **create-only declaration**: it is an error when the clan already exists, and an error when
  more than one prompt in a launch declares the same clan.
- A new `clan=` kwarg on `%id`: `%id(bar, clan=foo)` names the agent `foo.bar` and joins clan `foo`, creating it (with
  no tribe) if it does not exist yet.
- `%id(..., clan=...)` combined with `%tribe` is a validation error — joining a clan means joining its tribe, so a
  separate tribe declaration is contradictory.

The already-migrated reference example lives in the user's chezmoi repo at `home/sase/xprompts/research_swarm.md`:

```text
%clan(research.@, tribe=research) %id:research.@.cdx %model:@research_a {{ prompt }} #research
---
%id(cld, clan=research.@) %m:@research_b {{ prompt }} #research
---
%id(final, clan=research.@) %m:@research_lead %wait:research.@.cdx %wait:research.@.cld ...
```

Segment 1 declares the clan (it needs the `tribe=` kwarg, so it uses `%clan` plus a full `%id:<name>`); all later
segments join with `%id(<id>, clan=research.@)`. Note the clan name is an `@` name template — the whole design must keep
working with templates. (The file's _last_ segment is still unmigrated old-style `%name:` + a second `%clan(...)`; see
Follow-ups.)

Directive parsing is entirely Python-side (`src/sase/xprompt/`); the Rust core only performs downstream name-template
rendering and clan-tribe resolution over metadata, so no `../sase-core` changes are needed. The `i`/`id` spellings
collide with nothing (no `%if`; the alt fan-out's internal `alt_id` and workflow-gate `"id"` keys are unrelated).

## Current behavior being changed

- `src/sase/xprompt/_directive_types.py` is the single source of truth: `_KNOWN_DIRECTIVES` contains `"name"`,
  `_DIRECTIVE_ALIASES` maps `"n" -> "name"`, and `_DEPRECATED_DIRECTIVE_MESSAGES` already models hard-error migrations
  (`%time`, `%edit`).
- `%name` forms today: bare `%name` (auto-name), `%name:<full-name>`, `%name:!<name>` (forced reuse),
  `%name(parent, suffix)` (family attach — currently rejects **all** kwargs in `parse_name_directive_args`,
  `src/sase/agent/_family_attach_directives.py`), and one-marker `@` templates (`research.@`).
- `%clan` today both creates and joins: the multi-prompt prepass (`src/sase/agent/multi_prompt_launch_plan.py`,
  `prepare_clan_launches`) groups members by raw clan value, resolves `@` templates, validates the `<clan>.` hood
  prefix, and **reuses an existing clan's generation** when `find_agent_clan` hits; the runner-side fallback
  (`src/sase/axe/run_agent_directives.py` + `resolve_existing_clan_membership` in `src/sase/agent/clan_membership.py`)
  joins an existing clan when no launcher env payload is present. Both "join" behaviors move to the `clan=` kwarg.
- The clan's tribe is not stored on the clan container; each member that carried `%clan(..., tribe=...)` stamps
  `clan_tribe` into its own agent meta, and display/grouping resolves the effective tribe across members via the
  Rust-backed `resolve_clan_tribe` (`src/sase/core/agent_clan_tribe.py`). That is why old-style prompts repeated the
  full `%clan(<clan>, tribe=<tribe>)` on every member.

## Design overview

### New grammar

| Form                         | Meaning                                                                  |
| ---------------------------- | ------------------------------------------------------------------------ |
| `%id` / `%i` (bare)          | Auto-name (unchanged from bare `%name`)                                  |
| `%id:<full-name>`            | Explicit full agent name (unchanged semantics)                           |
| `%id:!<name>`                | Forced name reuse (unchanged)                                            |
| `%id(parent, suffix)`        | Family attach (unchanged; still rejects any kwargs)                      |
| `%id(<id>, clan=<clan>)`     | New: agent named `<clan>.<id>`, member of clan `<clan>` (join-or-create) |
| `%id(!<id>, clan=<clan>)`    | New: same, with forced reuse of the derived `<clan>.<id>` name           |
| `%clan(<clan>[, tribe=<t>])` | Declaration only: creates the clan; error if it already exists           |

The `clan=` value may be an `@` name template. The derived name is produced by joining the raw clan value and the id
with `.` **before** template resolution, so `%id(cld, clan=research.@)` yields the template name `research.@.cld` and
flows through the existing template machinery untouched. Dotted ids (`%id(a.b, clan=foo)` -> `foo.a.b`) are allowed —
the hood is prefix-based today and stays that way.

### New validation errors (all `DirectiveError`s with actionable messages)

1. `%id(..., clan=...)` + `%tribe` in one prompt — "joining a clan joins its tribe; put `tribe=` on the clan's `%clan`
   declaration instead".
2. `%id(..., clan=...)` + `%clan` in one prompt — redundant/conflicting; a declaring prompt spells the full name
   (`%clan(foo, tribe=t) %id:foo.bar`), a joining prompt uses only the kwarg.
3. `%id(parent, suffix, clan=...)` — family attach and clan membership stay mutually exclusive (mirrors the existing
   `%clan` + `%n(parent, suffix)` error).
4. `%id(clan=foo)` with no positional id — the kwarg requires a member id.
5. `%clan(foo)` when clan `foo` already exists in the name registry — "clan 'foo' already exists; join it with
   `%id(<id>, clan=foo)`".
6. `%clan(foo)` (same resolved clan) in more than one prompt of a launch — "only one prompt may declare a clan; other
   members join with `%id(<id>, clan=foo)`".

Rules 1–4 are prompt-local and belong beside the existing conflict checks (`_validate_clan_directive_contract` in
`src/sase/xprompt/_directive_collect.py` and the family-attach parser). Rules 5–6 are launch-scope and belong in the
clan prepass (plus the runner-side fallback for launches that bypass the prepass).

### Clan lifecycle semantics

- **Declaration** (`%clan`): creates the clan and may set its tribe. Exactly one declaring prompt per clan per launch;
  error if the clan already exists from an earlier launch.
- **Joiner** (`%id(..., clan=)`): joins the clan declared in the same launch, or an existing registered clan, and
  otherwise creates the clan implicitly (no tribe, generation from the first member timestamp exactly as the prepass
  computes today). Joiners never carry `clan_tribe`; the existing member-metadata tribe resolution already treats
  members without an explicit tribe correctly, so a joiner of a tribed clan displays under that tribe with no new
  plumbing.
- A launch may freely mix: one declaration plus N joiners (research_swarm shape), joiners only (implicit creation or
  joining a pre-existing clan), or several distinct clans each with its own declaration.

### Old-spelling migration

`%name`/`%n` become targeted migration errors, following the `%time`/`%edit` precedent: keep the `n -> name` alias so
both spellings canonicalize, and add `name` to `_DEPRECATED_DIRECTIVE_MESSAGES` ("The '%name'/'%n' directive has been
renamed; use %id/%i, and %id(<id>, clan=<clan>) to join a clan."). Display/strip paths (`strip_known_directives`,
history metadata parsing in `src/sase/history/prompt_metadata.py`) must keep tolerating old persisted prompts —
deprecated spellings are stripped/ignored for display and only raise when a prompt is actually parsed for launch.
Internally, the canonical directive key becomes `"id"` throughout the parsing tables and `== "name"` branches, while the
`PromptDirectives.name` dataclass field keeps its name (it holds the agent _name_, and there is precedent: `%tribe`
populates the internal `tag` field with an explanatory comment).

## Phase 1: Rename %name|%n to %id|%i

Mechanical but wide; the central tables make most consumers follow automatically, and the work is hunting the hard-coded
literals.

- `src/sase/xprompt/_directive_types.py`: `_KNOWN_DIRECTIVES` `name -> id`; `_DIRECTIVE_ALIASES` add `i -> id`, keep
  `n -> name`; add the `name` migration message to `_DEPRECATED_DIRECTIVE_MESSAGES`; update docstrings and the
  `PromptDirectives` field comments.
- Canonical-key rename (`"name"` -> `"id"` in `seen` dicts and alias-resolution comparisons) through:
  `_directive_collect.py` (including `_collect_name_paren_args` and the duplicate error text), `_directive_extract.py`,
  `_directive_values.py`, `_directive_scan.py`, `_directive_alt_naming.py` (strip/inject helpers must emit `%id:`),
  `directive_edit.py` (`set_prompt_name`), `jinja_inspect.py`, and the out-of-package consumers that compare against the
  canonical name: `src/sase/agent/retry_prompt.py` (its `directive_alias` literal becomes `"id" | "i"`),
  `src/sase/agent/launch_validation.py` (including `rewrite_force_reuse_name_directives` and its `"%n"/"%name"`
  substring fast-paths — the new fast-path is `"%i" not in prompt`, which also covers `%id`),
  `src/sase/agent/multi_prompt_reference_directives.py`, `src/sase/agent/_family_attach_directives.py`,
  `src/sase/history/prompt_metadata.py`, `src/sase/agent/names/_resume.py`,
  `src/sase/agent/multi_prompt_reference_allocator.py`.
- Literal `%name:`/`%n:` emitters now emit `%id:`/`%i:`: `src/sase/agent/repeat_launcher.py`,
  `src/sase/integrations/_mobile_agent_launch.py`, `src/sase/bead/work.py` (and
  `cli_work_plan.py`/`cli_work_cleanup.py`), `src/sase/agent/launch_cwd_agents.py`, `launch_cwd_bead_work.py`,
  `launch_cwd_common.py`, `src/sase/llm_provider/preprocessing.py`, TUI relaunch/name-entry helpers under
  `src/sase/ace/tui/actions/agent_workflow/`.
- Error and help strings that spell `%name`, `%n(parent, suffix)`, or `%n(parent, @)`: `launch_validation.py`,
  `_family_attach_types.py`, `_family_attach_candidates.py`, `_family_attach_resolution.py`,
  `src/sase/xprompt/workflow_loader.py`, `_parsing_vcs_tags.py`, `src/sase/agent/names/__init__.py` docstring,
  `src/sase/core/agent_scan_wire_markers.py` docstring, `src/sase/xprompt/directives.py` module docstring,
  `src/sase/agent/xprompt_swarm.py` docstring, `src/sase/prompt/search/sources.py` comment.
- User-facing catalogs: `src/sase/ace/tui/widgets/directive_completion.py` (`name` entry becomes `id` with updated
  description/argument hints) and `_directive_completion_tokens.py`.
- Bundled/generated templates (read the `generated_skills.md` long-term memory via the `/sase_memory_read` procedure
  before touching these): `src/sase/xprompts/skills/sase_run.md`, `sase_var.md`, `sase_agents_status.md`; project
  xprompt `sase/xprompts/reads.md`.
- Docs: `docs/xprompt.md` (directive table and narrative), `docs/agent_families.md`, `docs/ace.md`, `docs/prompt.md`. Do
  not hand-edit the generated `site/` mirrors.
- Do NOT touch the unrelated literal `%n`/`%c` copy-key formatting tokens in `src/sase/ace/tui/actions/clipboard/`,
  `commands/_formatting.py`, `commands/types.py`, `modals/command_palette_modal.py`, or `models/_agent_state.py` — those
  are keybinding format glyphs, not directives.
- Tests: rename spellings across the directive/name/launch/family/bead/TUI test files (`tests/test_directives_*.py`,
  `tests/test_agent_launch_validation.py`, `tests/test_dynamic_agent_family_attach_*.py`,
  `tests/test_repeat_launcher.py`, `tests/test_agent_names_extract_*.py`, `tests/test_bead/*`,
  `tests/ace/tui/widgets/test_directive_completion_candidates.py`, and friends). Add new tests: `%name`/`%n` raise the
  migration error; `strip_known_directives` and history metadata still strip/tolerate old spellings.

Exit criteria: `just check` green; no functional behavior change beyond the spelling swap and the migration errors.

## Phase 2: Add the clan= kwarg to %id

- `parse_name_directive_args` (`src/sase/agent/_family_attach_directives.py`): accept an optional `clan=` kwarg (all
  other kwargs still rejected). Exactly one positional id is required with `clan=`; the two-positional family form with
  `clan=` errors (rule 3); a bare `clan=` errors (rule 4). A leading `!` on the id is stripped into the force-reuse flag
  before derivation. Return the clan value alongside the parsed name so callers can derive `<clan>.<id>`.
- `_directive_collect.py` / `_directive_extract.py`: thread the parsed clan ref into the collected state; derive the
  full name (`<clan>.<id>`); populate `PromptDirectives` with the derived `name`, `name_explicit=True`, `clan=<clan>`
  and a new field distinguishing how membership was requested (e.g. `clan_declared: bool` — True only for an explicit
  `%clan` directive). Template metadata (`name_template*`) must be resolved from the derived name so `clan=research.@`
  behaves exactly like an explicit `%id:research.@.cld` template.
- Extend `_validate_clan_directive_contract` with rules 1 and 2, and add the runtime analog beside the existing
  `%tribe`-vs-inherited-clan check in `src/sase/axe/run_agent_directives.py`.
- Update the static extraction helpers used by the launch prepass (`extract_static_name_directive`,
  `extract_static_clan_directive` in `src/sase/agent/multi_prompt_reference_directives.py`) so a
  `%id(<id>, clan=<clan>)` segment reports both its derived name and its clan membership (marked as a joiner, never
  carrying a tribe). `StaticClanDirective` grows the joiner/declaration distinction. Launch-name validation
  (`_extract_explicit_name` in `src/sase/agent/launch_validation.py`) must validate the derived name.
- Completion: add `clan=` to the `%id` argument hints in `src/sase/ace/tui/widgets/directive_completion.py` (alongside
  the existing `%clan` `tribe=` keyword plumbing) and token extraction in `_directive_completion_tokens.py`.
- Docs: extend the `%id` row/section in `docs/xprompt.md` and the examples in `docs/agent_families.md` with the kwarg
  form.
- Tests: parsing of every form and every new prompt-local error; derived-name correctness with plain names, dotted ids,
  `!` reuse, and `@` templates; completion candidates.

In this phase the kwarg is parsed and validated but the launch prepass may still treat a joiner exactly like today's
`%clan:<name>` member (join-or-reuse generation); the lifecycle tightening lands in Phase 3.

## Phase 3: Single-declaration clan lifecycle

- Prepass (`prepare_clan_launches` in `src/sase/agent/multi_prompt_launch_plan.py`): group members from both
  declarations and joiners as today, then enforce rules 5–6 — at most one declaring segment per resolved clan, and a
  declaring segment errors when `find_agent_clan` already knows the clan. Joiners keep today's behavior (reuse the
  existing generation, or create with the first member's timestamp when the clan is new). Only the declaring segment's
  `tribe=` flows into members' `clan_tribe`. Perform the exists-check and reservation under the existing name-allocation
  lock so two concurrent declaring launches cannot both create the same clan.
- Runner-side fallback (no launcher env payload, `src/sase/axe/run_agent_directives.py`): joiners resolve-or-create (a
  create-if-missing variant next to `resolve_existing_clan_membership` in `src/sase/agent/clan_membership.py`);
  declarations create-only and error when the clan exists.
- Retry/relaunch of clan members: TUI retries rewrite the id directive to a fresh `<clan>.<suffix>.N` name but keep the
  original `%clan(...)` text, which would now trip rule 5. Add a `directive_edit.py` helper that demotes a clan
  declaration to the join form (drop `%clan(<clan>[, tribe=...])`, rewrite the id directive to
  `%id(<suffix>, clan=<clan>)`) and apply it in the relaunch entry points under
  `src/sase/ace/tui/actions/agent_workflow/` when the declared clan already exists. Verify resume
  (`src/sase/agent/names/_resume.py`) and chat-fork (`src/sase/history/chat_fork.py`) member flows stay green.
- Bead epics (`render_multi_prompt` in `src/sase/bead/work.py`): emit the `%clan(<epic_id>, tribe=epic)` declaration on
  the first segment only and `%id(!<suffix>, clan=<epic_id>)` joiners elsewhere; when re-running `sase bead work` for an
  epic whose clan already exists (failed/killed prior launch, later-wave phases, cleanup relaunches in
  `cli_work_cleanup.py`), emit the join form everywhere. Update the work rendering tests accordingly.
- TUI clan re-tagging (`src/sase/ace/tui/actions/agents/_tagging.py` + `set_prompt_clan_tribe`): tribe edits now only
  rewrite prompts that actually contain a `%clan` declaration; joiner prompts get their `clan_tribe` meta updated
  without inventing a `%clan` directive.
- Docs: rewrite the clan narrative in `docs/xprompt.md` and `docs/agent_families.md` around declare-once/join-many,
  including the research_swarm-shaped example.
- Tests: prepass declaration/joiner matrix (declaration+joiners, joiners-only implicit creation, join-existing across
  launches, duplicate declaration error, exists error, concurrent-launch reservation), runner fallback paths, demote
  helper, retry e2e, bead work rendering and re-work, tagging flows.

## Phase 4: End-to-end exercises

Through the real launch machinery (and the TUI harness where applicable), exercise and fix anything found by:

- A research_swarm-shaped multi-prompt launch: `%clan(x.@, tribe=t) %id:x.@.a` + two `%id(<id>, clan=x.@)` joiners with
  `%wait` chains — assert derived names, single clan with correct generation and tribe, and template resolution.
- Joiners-only launch creating a clan implicitly with no tribe; a later separate launch joining it with
  `%id(new, clan=...)`.
- Every new validation error (rules 1–6) surfacing with its intended message, including the cross-launch `%clan` exists
  error.
- Retry of a clan member (declaring member and joiner) via the TUI relaunch path.
- `sase bead work` epic launch and a re-work after simulated failure.
- Old-spelling regression: a `%name:` prompt fails with the migration message; an old persisted prompt still renders in
  history/display surfaces.
- Full `just check` plus the PNG visual snapshot suite (`just test-visual`) — clan/tribe display output should be
  unchanged, so snapshot diffs indicate a regression, not an intended change.

## Risks and edge cases

- **Old persisted prompts**: any stored prompt containing `%name`/`%n` (history, saved xprompts, running agents from
  before the rename) raises the migration error when re-parsed for launch. This is intentional (precedent: `%time`,
  `%edit`) but display paths must never crash on them.
- **Retry/relaunch flows** are the easiest place to silently break rule 5 — the demote helper must be wired into every
  relaunch entry point that replays a stored prompt.
- **Bead epic re-work** relies on `%clan` join semantics today; Phase 3's emitter changes are load-bearing, not
  cosmetic.
- **Templates**: clan refs and derived names with `@` must keep exactly one marker; deriving `<clan>.<id>` where both
  halves contain `@` would produce an invalid two-marker template — reject with a clear error.
- **Concurrency**: the declaration exists-check must be atomic with clan reservation under the name-allocation lock.
- **`%i` substring checks**: fast-path guards like `"%n" not in prompt` must be rewritten carefully (`"%i"` matches both
  `%i` and `%id`).
- The unrelated `%n`/`%c` copy-key formatting tokens in TUI clipboard/command code must not be renamed.

## Follow-ups requiring user action (not part of this repo's phases)

- **Long-term memory updates need explicit user approval**: `sase/memory/xprompts.md` (the directive table lists
  `%name:<n>` | `%n` and the old `%clan` semantics) and `sase/memory/glossary.md` (Agent Family's `%n(parent, suffix)`,
  Agent Clan's description) should be updated after this lands, followed by `sase memory init`. Implementing agents must
  not edit memory files on the strength of this plan alone.
- **chezmoi `home/sase/xprompts/research_swarm.md` last segment is unmigrated**: it still uses `%name:research.@.image`
  plus a second `%clan(research.@, tribe=research)`, which will hard-error under the new rules; it should become
  `%id(image, clan=research.@)` with the `%clan` directive removed. Other personal xprompts using `%name`/`%n` will
  surface via the migration error.
- **sase-nvim**: the Neovim plugin's directive syntax/completion should learn `%id|%i` and the `clan=` kwarg.
