---
tier: epic
title: Fold %tribe and the family form into %id kwargs
goal: 'The standalone %tribe|%t directive is removed and the two-positional %id family
  form is replaced by kwargs on %id: %id(bar, family=foo) creates family member foo--bar,
  %id(tribe=<t>) tribe-tags an auto-named agent (with a new built-in #tribe xprompt),
  clan=/family=/tribe= are mutually exclusive and all forbidden alongside %clan —
  with every emitter, TUI, Rust editor/LSP, docs, and cross-repo reference migrated
  so nothing breaks.

  '
phases:
- id: family-kwarg
  title: Replace the family positional form with family=
  depends_on: []
  description: '''Phase 1: Replace the family positional form with family='' section:
    rework parse_name_directive_args so %id(<suffix>, family=<parent>) replaces %id(parent,
    suffix) with a migration error for the old form, introduce the kwarg mutual-exclusion
    scaffold, reword every family error string, and update completion, docs, and tests.'
- id: tribe-kwarg
  title: Replace %tribe with the tribe= kwarg and add
  depends_on:
  - family-kwarg
  description: '''Phase 2: Replace %tribe with the tribe= kwarg and add #tribe'' section:
    remove %tribe|%t behind a migration error, parse %id(..., tribe=<t>) into the
    tag flow with an implicit auto id when the positional is omitted, migrate the
    TUI tag modal, retry rewrites, and axe chop emitter, ship the built-in #tribe
    xprompt, and update completion, docs, skill sources, and tests.'
- id: rust-editor
  title: Update the sase-core editor and LSP grammar
  depends_on:
  - family-kwarg
  - tribe-kwarg
  description: '''Phase 3: Update the sase-core editor and LSP grammar'' section:
    drop tribe|t from the Rust DIRECTIVES table, extend the %id kwarg candidates and
    snippets with family= and tribe=, and update hover, diagnostics, and the jsonrpc
    stdio integration tests in ../sase-core.'
- id: cross-repo
  title: Migrate chezmoi references and regenerate skills
  depends_on:
  - family-kwarg
  - tribe-kwarg
  description: '''Phase 4: Migrate chezmoi references and regenerate skills'' section:
    rewrite the chezmoi research-swarm xprompts and the stale %n snippet alias to
    the new grammar, regenerate and deploy the provider skill copies, and confirm
    sase-nvim needs no change.'
- id: smoke
  title: End-to-end exercises of the kwarg grammar
  depends_on:
  - family-kwarg
  - tribe-kwarg
  - rust-editor
  - cross-repo
  description: '''Phase 5: End-to-end exercises'' section: exercise family joins,
    tribe tagging, every new validation error, retries of tribed and family agents,
    the #tribe xprompt, and bead epic launches through the real launch machinery,
    plus the visual snapshot suite.'
create_time: 2026-07-19 15:40:37
status: wip
---

# Plan: Fold %tribe and the family form into %id kwargs

## Context

This is the follow-up to the sase-7g epic (`%name`→`%id` rename, create-only `%clan`, `clan=` kwarg). The user now wants
`%id` to be the single identity directive:

- The `%tribe:<t>` / `%t:<t>` directive is removed entirely in favor of a `tribe=` kwarg on `%id`.
- The two-positional family form `%id(parent, suffix)` is replaced by a `family=` kwarg with the argument order flipped:
  `%id(bar, family=foo)` creates a new family member named `foo--bar` that joins the `foo` agent family.
- The three kwargs `clan=`, `family=`, and `tribe=` are mutually exclusive, and none of them may be combined with a
  `%clan(...)` declaration in the same prompt (that declaration already sets clan membership implicitly).
- `%clan(<clan>)` failing when `<clan>` already exists is ALREADY enforced (sase-7g.3, create-only lifecycle) — verify,
  no new work.
- `%id(tribe=<t>)` with no positional id tags an implicitly auto-named agent (the id behaves like the `@` template,
  which resolves to a unique alphanumeric name), and a new built-in `#tribe` xprompt expands to exactly that form.
- All references in all repos are migrated: this repo, `../sase-core` (Rust editor/LSP grammar), and the chezmoi
  personal xprompts/skills. `sase-github`, `sase-telegram`, and `sase-nvim` contain no tribe/family directive references
  (sase-nvim is a pure LSP client of the Rust server) — verify-only.

The tribe CONCEPT is untouched: `@<tribe>` display prefixes and wait/fork targets (`parse_tribe_reference`),
`sase agent tribe {set,unset,list}` (meta-only, no prompt rewriting), `agent_tags.json` persistence, clan-level tribe
resolution (`resolve_clan_tribe`), and the `tribe=` kwarg on `%clan` all stay exactly as they are.

## Current behavior being changed

- `src/sase/xprompt/_directive_types.py` is the single source of truth: `_KNOWN_DIRECTIVES` contains `"tribe"`,
  `_DIRECTIVE_ALIASES` maps `"t" -> "tribe"`, and `_DEPRECATED_DIRECTIVE_MESSAGES` models hard-error migrations
  (`%name`, `%time`, `%edit`). `%tribe`'s value lands in `PromptDirectives.tag` (the field is deliberately named `tag`
  after the persisted concept — keep that).
- `parse_name_directive_args` (`src/sase/agent/_family_attach_directives.py`) accepts exactly one kwarg (`clan=`,
  rejection text says "Only clan= is supported"), plus the two-positional family form `%id(parent, suffix)` ->
  `ParsedNameDirective(family_parent, family_suffix)`; `normalize_family_suffix_arg` produces the `--<suffix>` role
  suffix (separator `AGENT_FAMILY_SEPARATOR` in `src/sase/plan_chain.py`), and `_family_attach_resolution.py` derives
  `parent_base--suffix`. Family attach travels launch→runner via the `SASE_AGENT_FAMILY_ATTACH` env payload; NO runtime
  path re-emits the positional form as prompt text, so the syntax swap is parser + error strings + docs, not emitter
  surgery.
- `%tribe` validation lives in `_validate_clan_directive_contract` (`src/sase/xprompt/_directive_collect.py`):
  `%tribe`+`%id(...,clan=...)` and `%tribe`+`%clan` conflicts; runtime analogs in `src/sase/axe/run_agent_directives.py`
  (tribe + joined clan, tribe + clan-inheriting family attach). Value parsing is `parse_tribe_tag` in
  `src/sase/xprompt/_directive_values.py`.
- Programmatic `%tribe` emitters: `set_prompt_tribe` in `src/sase/xprompt/directive_edit.py` (used by the TUI `N` tag
  modal via `src/sase/ace/tui/actions/agents/_tagging.py` + `_directive_persistence.py`, rewriting prompt.txt, history,
  and stash) and `_scaffolded_prompt` in `src/sase/axe/chop_proposals.py` (emits `%id:<name>` plus `%tribe:<tribe>` on
  separate tokens).
- Rust `../sase-core` hardcodes the directive grammar in `crates/sase_core/src/editor/directive.rs` (the `DIRECTIVES`
  table has `tribe`/`t`; `directive_argument_candidates` offers only `clan=` for `id`), surfaced through
  `editor/hover.rs`, `editor/diagnostics.rs` (unknown-directive check only), and the `sase_xprompt_lsp` server
  (completion routing + `%id(..., clan=...)` snippet) with the integration test
  `crates/sase_xprompt_lsp/tests/jsonrpc_stdio.rs`. Nothing directive-grammar-related is exposed through `sase_core_py`,
  so no Python binding or version coordination is required.
- The `#t` xprompt name is TAKEN by the time-floor built-in (`src/sase/xprompts/t.md` -> `%wait(time=...)`); the new
  built-in must be `#tribe` with no short alias (`#tribe` collides with nothing).

## Design overview

### New %id grammar

| Form                          | Meaning                                                                     |
| ----------------------------- | --------------------------------------------------------------------------- |
| `%id` / `%i` (bare)           | Auto-name (unchanged)                                                       |
| `%id:<full-name>` / `%id:!..` | Explicit full name / forced reuse (unchanged)                               |
| `%id(<id>, clan=<clan>)`      | Clan join, name `<clan>.<id>` (unchanged)                                   |
| `%id(<suffix>, family=<fam>)` | NEW: family member named `<fam>--<suffix>` joining the `<fam>` agent family |
| `%id(@, family=<fam>)`        | NEW: next free family suffix (replaces `%id(parent, @)`)                    |
| `%id(<id>, tribe=<t>)`        | NEW: explicit id plus tribe tag (replaces `%id:<id>` + `%tribe:<t>`)        |
| `%id(tribe=<t>)`              | NEW: tribe tag with implicit auto id (id behaves like the `@` template)     |
| `%id(a, b)`                   | Migration ERROR — "use %id(<suffix>, family=<parent>)"                      |
| `%tribe:<t>` / `%t:<t>`       | Migration ERROR (deprecated-directive precedent: `%name`, `%time`)          |

The `family=` value has exactly the semantics of today's first positional (a concrete existing agent house to attach to
— first attachment converts a standalone agent into a family); the positional takes today's second-arg semantics (bare
suffix token or `@`). Force-reuse `!` stays supported on the plain and `clan=` forms and on `%id(!<id>, tribe=...)`
(parity with `%id:!<id>` + `%tribe` today); it remains unsupported for family suffixes, where `@` allocation is the
escape hatch. `%id(tribe=<t>)` with no positional resolves its id exactly like bare `%id` (the empty-value auto-name
path in `_directive_extract.py` / `get_next_auto_name`, i.e. an `@` template allocation) while still carrying the tribe
into `PromptDirectives.tag`.

### Validation error catalog (all actionable `DirectiveError`s unless noted)

Prompt-local, in `parse_name_directive_args` / `_validate_clan_directive_contract`:

1. More than one of `clan=`/`family=`/`tribe=` on one `%id` — "the clan=, family=, and tribe= keywords on %id are
   mutually exclusive; set at most one."
2. Unknown kwarg on `%id` — existing error, text updated to "Only clan=, family=, and tribe= are supported."
3. Two positional args (with or without kwargs) — migration error pointing at `%id(<suffix>, family=<parent>)` (note the
   arg-order flip from the old `(parent, suffix)`).
4. `%clan` + `%id(..., clan=...)` — existing, keep.
5. `%clan` + `%id(..., family=...)` — clan and family membership stay mutually exclusive (replaces the old `%clan` +
   positional-family conflict).
6. `%clan` + `%id(..., tribe=...)` — "use %clan(<clan>, tribe=<tribe>) to set the clan's tribe" (replaces the old
   `%clan` + `%tribe` conflict).
7. `%id(clan=...)` with no positional — existing, keep (clan requires a member id).
8. `%id(family=...)` with no positional — the family kwarg requires a suffix; suggest `%id(<suffix>, family=<fam>)` or
   `%id(@, family=<fam>)`.
9. Empty/invalid `tribe=` value — non-empty required; names validated through the existing `sase.core.agent_tribe`
   validator (as `parse_tribe_tag` does today), with kwarg-form error text.
10. `%tribe`/`%t` anywhere — deprecated-directive migration error: removed; use `%id(tribe=<tribe>)` or the `#tribe`
    xprompt, and `%clan(<clan>, tribe=<tribe>)` for clans.

Runtime analogs in `run_agent_directives.py`: the two existing `ClanMembershipError`s (tag + joined clan; tag +
clan-inheriting family attach) become unreachable through a single `%id` thanks to kwarg exclusivity, but stay as
defense-in-depth with messages reworded to the kwarg spellings (the family-attach one still fires when a `tribe=`-tagged
agent's family parent inherits clan membership — that combination crosses two directives only via the env payload and
remains reachable).

### The #tribe built-in xprompt

New `src/sase/xprompts/tribe.md`, modeled on `t.md`: frontmatter `name: tribe`, one required input (`name: tribe`,
`type: word` — no `default:` key makes it required, per `InputArg`/`parse_inputs_from_front_matter`), body
`%id(tribe={{ tribe }})`. Loaded automatically by `load_xprompts_from_internal()`
(`src/sase/xprompt/loader_sources.py`); `#tribe:research` and `#tribe(research)` both work via the standard arg grammar.
Collision-checked: no existing xprompt, workflow, or `xprompt_aliases` entry is named `tribe`. Because a prompt may
contain only one `%id`, `#tribe` composes with prompts that have NO other `%id`; a prompt that wants an explicit id plus
a tribe writes `%id(<id>, tribe=<t>)` directly — extend the duplicate-`%id` error message to say so, since
`%id:<id> #tribe:<t>` is the natural mistake and it must fail with guidance, not confusion.

### What explicitly stays

`sase agent tribe` CLI (meta-only), `@tribe` wait/fork references, `%clan(<clan>, tribe=<t>)`, clan-tribe resolution and
all agent-meta/wire `tribe`/`tag` fields (Python and Rust), the `N` tag-modal keybinding, and the `#t` time xprompt.
`%clan` create-only semantics are untouched.

## Phase 1: Replace the family positional form with family=

- `parse_name_directive_args` (`src/sase/agent/_family_attach_directives.py`): accept `clan=`/`family=`/`tribe=` as the
  known kwarg set with the mutual-exclusion check (rule 1; `tribe=` is recognized here so the exclusion error is right
  from the start, and Phase 1 may keep it rejected downstream until Phase 2 wires it into the tag flow — cleanest is to
  land the full kwarg-set scaffold now and only the family behavior change in this phase). `family=` + exactly one
  positional maps to
  `ParsedNameDirective(family_parent=<family value>, family_suffix=normalize_family_suffix_arg(<positional>))`; zero
  positionals -> rule 8; two positionals -> migration error (rule 3). The downstream family machinery
  (`_family_attach_types.py`, `_family_attach_resolution.py`, `_family_attach_candidates.py`,
  `_family_attach_launch.py`, runner consumption in `run_agent_directives.py`) is untouched behaviorally — it keeps
  receiving `family_parent`/`family_suffix` and the env payload.
- Reword every error/help string that spells the positional form `%i(parent, suffix)`, `%i({parent}, @)`, or "two
  positional arguments": `_family_attach_directives.py` (including `normalize_family_suffix_arg`'s examples),
  `_family_attach_resolution.py` (reserved-root and name-taken errors), `_family_attach_candidates.py` (dismissed-parent
  error), `_family_attach_types.py` docstrings, `src/sase/agent/launch_validation.py`,
  `src/sase/agent/names/_registry.py`, `src/sase/xprompt/_directive_collect.py` (the `%clan`+family conflict becomes
  rule 5's kwarg spelling), `src/sase/xprompt/workflow_loader.py`.
- `extract_static_name_directive` (`src/sase/agent/multi_prompt_reference_directives.py`) keeps returning `None` for
  family forms (a family member's name is derived at attach time); confirm it does so for the kwarg form.
- Completion: `src/sase/ace/tui/widgets/directive_completion.py` — `_ID_KEYWORD_ARGUMENTS` gains `("family=", ...)`, and
  the `id` argument hint/description swap `(parent, suffix)` for the kwarg form. Verify `_extract_id_paren_arg_token` in
  `_directive_completion_tokens.py` completes a kwarg in the FIRST paren slot (`%id(family=` / `%id(tribe=` with no
  leading positional) — today it only fires after a comma; extend if needed.
- Bundled template/docs sweep for the positional form: `src/sase/xprompts/skills/sase_run.md` (lines using
  `%i(parent, suffix)` / `%i(parent, reviewer)` / `%i(parent, @)`), `docs/agent_families.md`, `docs/xprompt.md` (`%id`
  row + family narrative), `docs/ace.md`, `docs/prompt.md` as applicable. Do not touch generated `site/` mirrors.
- Tests: migrate the family spellings across `tests/test_directives_name.py`, `tests/test_directives_family.py`,
  `tests/test_directives_name_wait.py`, `tests/test_dynamic_agent_family_attach_*.py`,
  `tests/test_parallel_agent_family_*.py`, `tests/test_agent_launch_validation.py`,
  `tests/test_multi_prompt_reference_directives.py`, `tests/ace/tui/widgets/test_directive_completion_candidates.py`.
  New tests: `%id(a, b)` migration error; `%id(family=x)` missing-positional error; kwarg mutual-exclusion error;
  `%id(@, family=x)` auto suffix; `%clan` + `family=` conflict.

Exit criteria: `just check` green; family attach behaves identically to today through the new spelling; the old
positional form fails with the migration message.

## Phase 2: Replace %tribe with the tribe= kwarg and add #tribe

- `src/sase/xprompt/_directive_types.py`: remove `"tribe"` from `_KNOWN_DIRECTIVES`; keep `"t" -> "tribe"` in
  `_DIRECTIVE_ALIASES`; add the `tribe` entry to `_DEPRECATED_DIRECTIVE_MESSAGES` (rule 10) so both spellings
  canonicalize, raise on launch parse, and stay stripped/tolerated by display paths (`strip_known_directives`,
  `src/sase/history/prompt_metadata.py`, `chat_resume.py` — the `%name` precedent covers all three; add explicit
  regression tests). Update the `PromptDirectives.tag` field comment (now populated by the `tribe=` kwarg).
- Parsing: thread the `tribe=` kwarg from `parse_name_directive_args` through `_collect_name_paren_args`
  (`_directive_collect.py`) into extraction; repurpose `parse_tribe_tag` (`_directive_values.py`) to validate the kwarg
  value into `tag` (rule 9). An empty positional with `tribe=` follows the bare-`%id` auto-name path. Extend
  `_validate_clan_directive_contract` with rule 6 and drop the two obsolete `%tribe` conflicts; reword the runtime
  `ClanMembershipError`s in `run_agent_directives.py` (they otherwise keep their trigger conditions). Audit any `"%t"`
  substring fast-path guards while removing (a `"%t"` check matches both `%t` and `%tribe`; see the `has_*` helpers
  covered by `tests/test_directives_has_helpers.py`).
- Rewrite helpers (`src/sase/xprompt/directive_edit.py`): `set_prompt_tribe` now edits the `%id` directive —
  add/replace/remove a `tribe=` kwarg, converting `%id:<name>` to `%id(<name>, tribe=<t>)` when needed, appending
  `%id(tribe=<t>)` when the prompt has no `%id`, removing the whole directive again when clearing a tribe from an
  id-less `%id(tribe=<t>)`, and still stripping legacy `%tribe`/`%t`/`%g`/`%group` spellings from old prompts. The
  clan-vs-tribe dispatch in the TUI tag modal (`src/sase/ace/tui/actions/agents/_tagging.py` +
  `_directive_persistence.py`) is unchanged in shape; its prompt/history/stash rewrites must produce the kwarg form.
  CRITICAL: every `%id`-rewriting helper must preserve existing kwargs — `set_prompt_name`, `rewrite_retry_prompt_name`
  (TUI retries must not silently drop a `tribe=` when renaming), the alt fan-out strip/inject helpers
  (`_directive_alt_naming.py`), and `rewrite_prompt_clan_member_name` / `demote_prompt_clan_declaration`.
- Emitters: `_scaffolded_prompt` in `src/sase/axe/chop_proposals.py` merges its `%id:<name>` + `%tribe:<tribe>` pair
  into `%id(<name>, tribe=<tribe>)`. Confirm `prompt_has_name_directive` (mobile launch guard in
  `src/sase/integrations/_mobile_agent_launch.py`) treats `%id(tribe=...)` as a present name directive. Update the
  `runner_artifacts.py` docstring mention.
- The `#tribe` built-in: add `src/sase/xprompts/tribe.md` as designed above; extend the duplicate-`%id` error message
  per the design; document it beside `#t` in `docs/xprompt.md`.
- Completion: drop `tribe` from the directive catalogs in `directive_completion.py` (removal from `_KNOWN_DIRECTIVES`
  cascades; delete its hint/description/alias entries), add `("tribe=", ...)` to `_ID_KEYWORD_ARGUMENTS`, update the
  `%t`-prefix expectations.
- Docs: `docs/xprompt.md` (delete the `%tribe | %t` table row, add the kwarg + `#tribe` narrative),
  `docs/agent_families.md` (`%tribe`/`%t` examples), `docs/ace.md` (tag-modal description), `docs/axe.md`
  (`%tribe:chop`), `docs/prompt.md`, and the two blog posts under `docs/blog/posts/` that carry the directive table.
  `src/sase/xprompts/skills/sase_run.md` tribe block (keep `%clan(..., tribe=...)` guidance; standalone agents/families
  now use `%id(..., tribe=...)` / `#tribe`). Do not hand-edit `site/` mirrors or the five chezmoi SKILL.md copies (Phase
  4 regenerates them).
- Tests: migrate the literal `%tribe`/`%t` spellings listed in the research sweep (`tests/test_directives_extract.py`,
  `test_directive_edit.py`, `test_directives_time.py`/`_time_repeat.py`, `test_directives_family.py`,
  `test_axe_run_agent_phases_tags.py`, `test_axe_chop_result_protocol.py`, `test_multi_prompt_launcher_launch_env.py`,
  `test_dynamic_agent_family_attach_metadata.py`, `test_parallel_agent_family_launch.py`,
  `tests/history/test_chat_resume_sanitize.py`, `tests/ace/tui/test_agent_tagging.py`,
  `tests/ace/tui/widgets/test_directive_completion_candidates.py`, and friends). New tests: `%tribe`/`%t` raise the
  migration error while display/strip paths tolerate them; `%id(tribe=t)` auto-names and lands `tag`; `%id(x, tribe=t)`
  and `%id(!x, tribe=t)`; rules 1, 6, 9 errors; `#tribe:research` expansion (model: the `#t` tests in
  `test_directives_time.py`); retry rewrite preserves `tribe=`; tag modal round-trip.

Exit criteria: `just check` green; `%tribe`/`%t` fail with the migration message everywhere a launch parse happens;
tribe tagging via kwarg, modal, chop, and `#tribe` all land the same `tag` meta as before.

## Phase 3: Update the sase-core editor and LSP grammar

All in the sibling Rust repo (open with `/sase_repo`; commits through the standard sase flow). No `sase_core_py` binding
or Python-side version work is needed (the directive grammar is not exposed to Python).

- `crates/sase_core/src/editor/directive.rs`: delete the `tribe` entry from `DIRECTIVES` (the removed `family`/`group`
  directives set the precedent: old spellings become `unknown_directive` diagnostics); update the `id` description to
  cover clan/family/tribe roles; `directive_argument_candidates` for `"id"` becomes `clan=`/`family=`/`tribe=` with
  matching descriptions. Update the table-locking tests (documented aliases, id argument set, removed spellings now
  including `tribe`/`t`, `%t` prefix completion now empty).
- `crates/sase_core/src/editor/hover.rs` / `diagnostics.rs`: hover follows the metadata table automatically — update the
  exact-string tests (`%tribe:research` hover disappears; `%id(worker, family=...)` hovers as `%id`);
  `recognizes_current_directives_but_rejects_removed_spellings` moves `%tribe:research`/`%t:research` into the rejected
  set and adds the kwarg forms to the accepted set.
- `crates/sase_xprompt_lsp/src/server.rs`: snippet items — keep `%id(${1:id}, clan=${2:clan})`, add
  `%id(${1:suffix}, family=${2:family})` and `%id(tribe=${1:tribe})`; the `%tribe` name snippet disappears with the
  table entry. The agent-target completion kinds (`tribe|clan|family` values in `lsp_convert.rs`) are the metadata
  concept — untouched.
- `crates/sase_xprompt_lsp/tests/jsonrpc_stdio.rs`: update the integration test document and assertions
  (`%tribe:review`/`%t:review` now among the unknown-directive diagnostics; `tribe=`/`family=` completion items for
  `%id`; new snippets asserted).
- Update the crate CHANGELOGs per repo convention; run the sase-core test suite.
- sase-nvim consumes this server directly and has no hardcoded directive tables — after the server changes, run its LSP
  smoke checks (`tests/lsp_snippet_smoke.lua`) or eyeball the snippet list; no code change expected.

Exit criteria: sase-core tests green; the LSP surfaces exactly the new grammar.

## Phase 4: Migrate chezmoi references and regenerate skills

All chezmoi edits happen in the `/sase_repo`-opened checkout; commits through the standard sase flow.

- `home/sase/xprompts/research_swarm.md` last segment (already flagged as unmigrated in the sase-7g plan): replace
  `%name:research.@.image ... %clan(research.@, tribe=research)` with `%id(image, clan=research.@)` (drop the duplicate
  `%clan` declaration and the deprecated `%name`).
- `home/sase/xprompts/old_research_swarm.md`: each `%t:research` becomes `%id(tribe=research)` (or `#tribe:research` —
  pick one and use it consistently; the directive form avoids a second expansion hop in a file that already spells
  directives).
- `home/dot_config/sase/sase.yml` snippet alias `n: "%n:"` (stale since the rename): update to insert `%i:`.
- Regenerate the provider skill copies from the Phase 2 source edits: in the sase repo run `sase skill init --force`,
  then `chezmoi apply` (per the generated_skills long-term memory — never hand-edit the five `SKILL.md` copies). Verify
  the five copies are byte-identical again and tribe guidance shows the kwarg form.
- Sweep the rest of chezmoi for stragglers (`grep -ri '%tribe\|%t:'` and the family positional) — the research audit
  found none outside the files above.

Exit criteria: no `%tribe`/`%t` directive or positional family form anywhere in chezmoi; regenerated skills match their
sources.

## Phase 5: End-to-end exercises

Through the real launch machinery (and the TUI harness where applicable), exercise and fix anything found by:

- A family chain: launch a parent, attach `%id(reviewer, family=<parent>)`, then `%id(@, family=<parent>)` — assert `--`
  names, roles, and env-payload plumbing are byte-identical to the old positional behavior.
- Tribe tagging: `%id(tribe=t)` (auto name + tag meta + agent_tags.json), `%id(x, tribe=t)`, `#tribe:t`, the TUI `N` tag
  modal round-trip on standalone and clan-member agents, and a retry of a tribed agent (kwarg must survive the rename
  rewrite).
- Every error in the catalog (rules 1–10) surfacing with its intended message, including `%tribe:x` and `%id(a, b)`
  migration errors and the improved duplicate-`%id` message for `%id:<id> #tribe:<t>`.
- A research_swarm-shaped multi-prompt launch mixing a `%clan` declaration, `clan=` joiners, and a separate
  `%id(tribe=...)` standalone agent; `sase bead work` epic launch regression (emitters unchanged, must stay green).
- Axe chop proposal scaffolding produces the merged `%id(<name>, tribe=<tribe>)` and the runner lands the tag.
- Old-spelling regression: persisted prompts containing `%tribe:`/`%t:` still render in history/display surfaces without
  crashing while re-parsing for launch fails with the migration message.
- Full `just check` plus `just test-visual` — tribe/clan/family display output should be unchanged, so snapshot diffs
  indicate a regression, not an intended change.

## Risks and edge cases

- **Retry/rename rewrites dropping kwargs** is the top regression risk: any helper that replaces the `%id` directive
  wholesale (`rewrite_retry_prompt_name`, `set_prompt_name`, alt naming inject/strip) silently loses `tribe=`/`family=`
  unless taught to carry kwargs through. Cover each with a direct test.
- **Old persisted prompts**: anything stored with `%tribe`/`%t` (history, saved xprompts, running agents from before
  this change) raises the migration error when re-parsed for launch — intentional, but display paths must never crash on
  them, and the TUI tag modal must still be able to RE-tag such an agent (its legacy-strip path).
- **First-slot kwarg completion**: `%id(tribe=` with no positional is a new completion shape; the token extractor
  currently only recognizes kwargs after a comma.
- **`"%t"` substring guards** match `%tribe` too — audit fast paths before deleting the directive.
- **Arg-order flip** on the family form is a silent-behavior trap for muscle memory: `%id(a, b)` must hard-error
  (rule 3) rather than fall through to any positional interpretation.
- **`#tribe` + explicit `%id`** collides with the one-`%id`-per-prompt rule; the duplicate error message must point at
  `%id(<id>, tribe=<t>)`.
- **Cross-repo skew**: between Phase 2 and Phases 3–4 landing, the LSP still advertises `%tribe` while the Python parser
  rejects it (and vice-versa for chezmoi files). The phases minimize the window; the migration errors make the failure
  mode loud, not silent.

## Follow-ups requiring user action (not part of this repo's phases)

- **Long-term memory updates need explicit user approval in the implementing conversation**: `sase/memory/xprompts.md`
  (directive table still lists `%tribe:<name>` | `%t`) and `sase/memory/glossary.md` (Agent Tribe's "assigned with
  `%tribe:<name>` (alias `%t`)" and Agent Family's `%n(parent, suffix)` — the latter is stale from sase-7g already and
  is why the generated `OPENCODE.md`/`QWEN.md` shims still show `%n(parent, suffix)`). After approval: edit the
  canonical notes, then run `sase memory init` to regenerate `AGENTS.md` and the provider shims. Implementing agents
  must not edit memory files on the strength of this plan alone.
