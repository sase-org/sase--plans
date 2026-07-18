---
tier: epic
title: Tribe next-entity targeting for %wait and
goal: '%wait and #fork accept @<tribe> targets that bind to the very next agent or
  agent clan that joins that tribe, and forking an agent clan injects a lean context
  block (every member''s prompt plus reply statistics and metadata) instead of full
  member transcripts.

  '
phases:
- id: tribe-wait
  title: Tribe wait targeting
  depends_on: []
  description: '''Tribe wait targeting'' section: add @<tribe> parsing to %wait, tribe
    entity buckets and next-entity selection to WaitDependencyIndex, and thread the
    waiter-launch cutoff through all resolution callers.

    '
- id: clan-fork
  title: Clan fork context blocks
  depends_on: []
  description: '''Clan fork context blocks'' section: make #fork of an agent clan
    inject a per-member prompts-plus-reply-statistics block via a new Python history
    builder, replacing the current newest-member-only transcript behavior.

    '
- id: fork-tribe
  title: Fork tribe targets
  depends_on:
  - tribe-wait
  - clan-fork
  description: '''Fork tribe targets'' section: let #fork accept @<tribe>, imply the
    tribe wait, resolve the same next entity the wait bound to, and dispatch to the
    clan fork block or single-agent fork accordingly.

    '
- id: polish
  title: Completion, docs, and end-to-end exercises
  depends_on:
  - fork-tribe
  description: '''Completion, docs, and end-to-end exercises'' section: TUI directive
    completion for @tribe targets, skill/doc updates, and manual end-to-end exercises
    of the wait and fork tribe flows.

    '
create_time: 2026-07-18 18:08:26
status: wip
bead_id: sase-6x
---

# Plan: Tribe next-entity targeting for %wait and #fork, plus clan forking

## Context

`%wait:<name>` and `#fork:<name>` today target **already-known** entities: the wait resolver
(`src/sase/core/wait_dependency_resolution/_index.py`) checks clan, family, workflow, and exact-name buckets and
resolves against the newest existing candidate, and `#fork` (an xprompt workflow, `src/sase/xprompts/fork.yml`) resolves
a name to one transcript path and inlines the **full flattened transcript**. There is no way to say "wait for / fork
whatever gets launched next in tribe X". Epic approvals already launch clans that declare `%clan(<epic_id>, tribe=epic)`
(`src/sase/bead/work.py:286,301`), so the motivating `@epic` use case needs no changes on the launch side — only on the
targeting side.

Two gaps to close:

1. **Tribe targets.** A new argument form `@<tribe>` for `%wait`/`%w` and `#fork` that binds to the **next** agent or
   agent clan that joins that tribe — i.e. an entity whose launch is strictly newer than the waiting agent's own launch.
2. **Clan forking.** `#fork:<clan>` currently degrades to the newest successful member's full transcript
   (`most_recent_completed_clan_member` in `src/sase/agent/names/_lookup.py`). Forking a clan must instead inject every
   member's prompt plus **reply statistics and metadata only** — never the full reply text of every member — because
   every token in the child's context either helps or hurts.

### Existing machinery this builds on

- **Wait execution** is a marker-file barrier: the runner writes `waiting.json` (which already records the waiter's own
  `timestamp`) and polls for `ready.json` (`src/sase/axe/run_agent_wait.py`); the periodic chop script
  `src/sase/scripts/sase_chop_wait_checks.py` evaluates `dependency_resolution_status` over a `WaitDependencyIndex` and
  writes `ready.json`. A third caller lives in the TUI kill flow (`src/sase/ace/tui/actions/agents/_killing_utils.py`).
  All three pass `self_artifact_dir`, whose basename **is** the waiter's launch timestamp — the natural "next" cutoff,
  with no wire/signature changes required.
- **Tribe membership** has three sources: `agent_meta.json`'s `tag` (written from `%tribe` by
  `src/sase/axe/run_agent_directives.py`), `clan_tribe` on clan members (from `%clan(..., tribe=...)`, deterministically
  resolved by the existing Rust binding `sase.core.agent_clan_tribe.resolve_clan_tribe`), and post-hoc assignments in
  `~/.sase/agent_tags.json` keyed by `(agent_type, cl_name, raw_suffix)` where `raw_suffix` equals the artifact
  timestamp (`src/sase/ace/agent_tags.py`).
- **Critical collision:** `@` is the agent-name **template marker** (`is_agent_name_template` = "contains exactly one
  `@`"), so today `@epic` parses as a template rendering `<token>epic`. Every wait/fork path that resolves templates
  must detect tribe references first. Derived naming is also affected: a single-target wait derives
  `allocate_wait_name("<target>.w@")` and a single-parent fork derives resume names via template resolution — both crash
  or misbehave on `@`-prefixed targets and must skip tribe refs.

## Design

### Syntax and validation

- A **tribe reference** is a wait/fork target argument of the form `@<tribe>` where `<tribe>` matches the existing tag
  grammar (`^[A-Za-z0-9_.-]+$`, validated by stripping `@` then calling `validate_tag_name`). Examples: `%wait:@epic`,
  `%w:@review,builder-2`, `#fork:@epic`, `#fork(@epic, other-agent)`.
- Add one shared primitive, e.g. `parse_tribe_reference(value) -> str | None` (returns the validated bare tribe name for
  `@`-prefixed values, `None` otherwise), in a core-safe module importable from both `sase.xprompt` and
  `sase.core.wait_dependency_resolution` without pulling in TUI code. All wait/fork surfaces consult it **before** any
  template logic.
- A malformed tribe ref (e.g. `%wait:@bad name`) raises a `DirectiveError` at parse time. Tribe existence is
  deliberately not validated — tribes are created on first use and the target is a future entity by definition.
- The `@` prefix is used verbatim as the stored dependency string (in `wait_for`, `waiting.json`, `ready.json`
  `resolved_deps`), so display surfaces that already render these lists (Agents-tab waiting enrichment) show `@epic`
  with zero wire changes.

### "Next entity" semantics

- **Cutoff:** the waiting/forking agent's own launch timestamp (its artifact directory basename). An entity qualifies
  only if its launch timestamp is **strictly newer** than the cutoff. Timestamps are the existing sortable
  `YYYYMMDDHHMMSS` artifact names; lexicographic comparison suffices.
- **Entity:** a clan generation when the tribe evidence sits on a clan member (any member's resolved `clan_tribe`, meta
  `tag`, or tags-store tag enrolls the whole generation); otherwise the standalone agent itself. Family members tagged
  with a tribe count as standalone agents. A clan entity's launch timestamp is the minimum member timestamp of that
  generation.
- **Resolution:** a `@tribe` dependency is resolved when at least one qualifying entity is complete-successful (for
  clans: every member of the generation completed successfully, matching existing `clan_candidate` aggregation). The
  canonical bound entity is the **earliest-launched** complete one, which keeps re-evaluation deterministic across chop
  runs and keeps wait and fork agreeing. A failed entity never resolves the wait, but a newer qualifying entity still
  can — consistent with existing name-wait failure semantics.
- **Self-exclusion:** entities that contain the waiting agent itself (its own artifact dir, or a clan generation it
  belongs to) never qualify, so an `@epic`-tagged agent waiting on `@epic` cannot deadlock on itself.
- **Scoping** follows the existing index scoping asymmetry: the launch-time fast path checks the current project only,
  while the chop script indexes all projects. This only delays cross-project resolution by one chop cycle; it never
  resolves wrongly.

### Clan fork content (token-lean by design)

Forking a clan injects one block containing, per member (sorted by launch timestamp): the member's **sanitized
prompt(s)** (reusing `_parse_chat_turns` + `_sanitize_resume_prompt` from `src/sase/history/chat.py`) and a **reply
summary line** — outcome, model, launch time, approximate reply size (words/lines), and the transcript path so the child
can read a specific full reply on demand. A clan header carries clan name, generation, tribe (when declared), and member
count, plus one instruction sentence telling the child that full replies were intentionally omitted and where to find
them. Full reply text is never inlined for clan members.

## Tribe wait targeting

Everything needed for `%wait:@epic` to block until the next `@epic` entity completes.

1. **Directive layer** (`src/sase/xprompt/`): in `_directive_values.py`, `resolve_wait_agent_args` validates tribe refs
   (raising `DirectiveError` on bad grammar) and `resolve_wait_templates` skips them before template resolution. Bare
   `%wait` most-recent resolution and `%wait(time=/runners=)` are untouched. `directive_edit.py`'s canonical rewrite
   must round-trip `@tribe` args unchanged.
2. **Derived naming** (`src/sase/axe/run_agent_directives.py`): when the sole wait target is a tribe ref, skip
   `allocate_wait_name` (it would build an invalid two-marker template) and fall through to neutral auto-naming.
   Optional nicety if cheap: derive `<tribe>.w@` instead.
3. **Index** (`src/sase/core/wait_dependency_resolution/`): `add()` grows a tribe bucket populated from meta `tag`,
   per-member `clan_tribe` (effective clan tribe resolved via the existing Rust facade
   `sase.core.agent_clan_tribe.resolve_clan_tribe` — do not reimplement its precedence), and `~/.sase/agent_tags.json`
   joined by `(cl_name, timestamp)` from meta. Add a core-safe raw reader for the tags file (the existing
   `load_agent_tags` imports TUI `AgentType`; core must not) — either a small reader inside `sase.core` or a refactor
   that moves identity-neutral loading down and keeps `sase.ace.agent_tags` as the typed wrapper. New
   `tribe_candidate(tribe, *, newer_than, exclude_artifact_dir)` implements the entity model and selection rule above;
   `is_resolved` routes `@`-prefixed names exclusively to it (no fallthrough to clan/family/ workflow/named buckets),
   deriving `newer_than` from `self_artifact_dir`'s basename inside `dependency_resolution_status`. Tribe refs with no
   available cutoff resolve as waiting.
4. **Non-index resolver parity** (`src/sase/agent/names/_lookup.py`): `resolve_wait_dependency` gains the same tribe
   branch (or its tribe-unaware callers are confirmed unreachable for tribe refs and documented as such).
5. **Runner** (`src/sase/axe/run_agent_wait.py`): no structural changes — verify tribe strings flow through
   `waiting.json`/`ready.json` untouched and the `wait_completed_at` fast path still short-circuits re-execs.

**Tests** (extend `tests/test_directives_wait.py`, `tests/test_directives_name_wait.py`, wait-check suites): tribe ref
parsing (valid, malformed, mixed lists, backtick/paren forms, alias `%w`, `%t` non-collision), template non-collision
(`@epic` vs `base-@`), index selection (no entity yet → waiting; entity newer than cutoff completes → resolved;
older-than-cutoff entity ignored; failed entity ignored until a newer one completes; clan generation aggregation;
self-exclusion; earliest-complete determinism), chop-script end-to-end over synthetic artifact dirs, derived naming
skip, and enrichment display of `@epic` in waiting lists.

## Clan fork context blocks

Everything needed for `#fork:<clan>` to inject the lean clan block. No tribe syntax in this phase.

1. **Resolution** (`src/sase/scripts/agent_chat_from_name.py`): before the current per-name path resolution, detect that
   a requested name is a clan (`find_agent_clan`) whose newest generation is complete, and emit a typed source — extend
   `sources_json` entries with a `kind` field (`"agent"` today's shape, `"clan"` carrying clan name, generation, tribe
   if declared, and per-member `{name, path, artifact_dir}`). The single-path compatibility field keeps pointing at the
   newest successful member.
2. **History builder** (`src/sase/history/`): new builder, e.g. `build_fork_injected_history(sources) -> str`, that
   assembles the complete injected block for any mix of agent and clan sources — single agent (existing
   `load_chat_for_resume` output and framing, unchanged), multiple parents (existing multi-parent framing), and clan
   sources (new block per the design above). Reply statistics come from the member transcript's parsed turns;
   outcome/model/timestamps come from each member's `done.json`/`agent_meta.json`. Recursive ancestor inlining
   (`load_chat_for_resume`'s `#fork` re-expansion) applies to member **prompts** only — sanitized like any other prompt
   — never to member replies.
3. **Workflow** (`src/sase/xprompts/fork.yml`): collapse the `load_history` step's inline Python to a thin call into the
   new builder so format logic lives in tested Python, not YAML. `#fork_by_chat` is untouched. Confirm the `type: agent`
   input accepts clan names (it already resolves them today).
4. **Naming**: fork-derived child naming for a clan parent keeps deriving from the clan name (`<clan>.f@` template path
   already works off the requested name string).

**Tests** (extend `tests/test_fork_workflow.py`, `tests/test_agent_chat_from_name.py`, new history-builder tests): clan
source emission (complete clan, incomplete clan → error consistent with today's unresolved-fork behavior, clan+agent
mixed parents), block content (all member prompts present and sanitized, no member reply text inlined, stats and paths
present, deterministic member order), and fork.yml smoke via the existing workflow test harness.

## Fork tribe targets

Ties the two prior phases together for `#fork:@epic`.

1. **Reference layer** (`src/sase/agent/names/_resume.py`): `fork_agent_names` passes tribe refs through verbatim
   (skipping template resolution) so `run_agent_directives.py` derives the implied `%wait:@epic`;
   `sole_resume_agent_name` returns `None` for tribe refs so the child gets neutral auto-naming instead of crashing in
   template resolution. `has_fork_reference` and deferred-start detection already match.
2. **Fork-time resolution** (`src/sase/scripts/agent_chat_from_name.py`): a tribe-ref argument resolves via the same
   shared next-entity selection used by the wait index (build the index the way the chop script does), with the cutoff
   taken from `SASE_ARTIFACTS_DIR`'s basename; a missing environment or no qualifying complete entity yields a clear
   `RuntimeError` (the implied wait makes this unreachable in normal launches). A clan entity dispatches to the clan
   source kind; a standalone agent to the plain agent kind.
3. **Chat-layer regexes** (`src/sase/history/chat.py`, `_RESUME_REF_RE` in `_resume.py`): confirm `@`-prefixed args
   survive the argument grammar (they are ordinary non-space tokens) and that stored prompts containing `#fork:@epic`
   sanitize away cleanly on re-fork.
4. **Launcher surfaces**: multi-prompt launches and `launch_cwd_agents` defer/queue on `#fork:@epic` exactly as they do
   for named forks (the detection predicates are name-agnostic; add coverage rather than code where possible).

**Tests** (extend `tests/test_agent_names_resume.py`, `tests/test_multi_prompt_launcher_fork.py`,
`tests/test_agent_chat_from_name.py`): implied-wait derivation for tribe refs, neutral naming, tribe→clan and
tribe→agent fork resolution against synthetic artifacts (including earliest-complete tie-breaking agreement with the
wait side), launch-time behavior when the tribe entity does not exist yet (defers rather than errors), and mixed
`#fork:@epic,named-agent` parents.

## Completion, docs, and end-to-end exercises

1. **TUI completion** (`src/sase/ace/tui/widgets/directive_completion.py`): offer `@<tribe>` completions (from known
   tags) for `%wait:`/`%w:` and `#fork:` argument positions, mirroring the existing tribe completion for
   `%tribe`/`%clan(tribe=)`.
2. **Docs**: update `src/sase/xprompts/skills/sase_run.md` (generated xprompt skill source — the implementing agent must
   first read the `generated_skills.md` long-term memory via `sase memory read`) to document `@tribe` wait/fork targets
   and clan forking. The `%wait` row in the canonical memory `sase/memory/xprompts.md` should also mention `@tribe`
   targets, but memory files must not be edited without explicit user permission granted in the implementing
   conversation — ask the user first, and skip the memory edit if permission is not granted.
3. **End-to-end exercises**: with throwaway agents in a scratch project, exercise (a) `%wait:@<test-tribe>` launched
   first, then a tribe-tagged agent, observing block → chop resolve → proceed; (b) the same with a
   `%clan(..., tribe=...)` clan and a `#fork:@<test-tribe>` child, verifying the injected block contains every member
   prompt, statistics, and no member reply text; (c) `sase agent tribe set` post-hoc tagging unblocking a waiting agent.

## Risks and notes

- **`@` template-marker collision** is the sharpest edge: any path that calls
  `is_agent_name_template`/`require_latest_agent_name_template` on a wait or fork target must check
  `parse_tribe_reference` first. The phase test lists above pin this in the directive layer, `_resume.py`, and
  `agent_chat_from_name.py`; grep for remaining template-resolution call sites on wait/fork targets before closing each
  phase.
- **Determinism/races**: binding to the earliest-launched complete entity makes chop re-evaluation and later fork
  resolution agree in all but one benign edge (an earlier-launched entity completing after `ready.json` was written may
  be picked by the fork; both candidates are qualifying post-cutoff tribe members). Document this in the builder's
  docstring.
- **Layering**: the tags-store read used by the index must not import TUI modules into `sase.core`. Clan-tribe
  precedence stays in the existing `sase_core_rs` binding; no new Rust wire is required. If wait resolution later
  migrates into the Rust core, tribe selection should move with it.
- **Uniform runtimes**: all behavior lives in directive parsing, the wait barrier, and the fork workflow — no
  agent-CLI-specific branches anywhere.
- **Backward compatibility**: `#fork:<clan>` changes behavior from newest-member transcript to the clan block by design;
  plain `%wait:<clan>`, named forks, families, and workflows are untouched. Existing `waiting.json`/`ready.json`
  consumers see tribe refs as opaque strings.
- Symvision lint may flag new helper symbols; consult the `symvision.md` long-term memory before adding pragmas or
  whitelists.
