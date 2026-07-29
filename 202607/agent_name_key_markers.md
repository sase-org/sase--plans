---
tier: epic
title: Keyed `{@<id>}` agent-name markers for xprompt swarms
goal: 'Every `@` reference inside one xprompt swarm launch resolves to the same concrete
  agent-name token, chosen once per launch and substituted into the prompt text before
  any agent spawns, so a later swarm launch can never steal an earlier swarm''s hood.

  '
phases:
- id: grammar
  title: Keyed marker grammar in sase-core
  depends_on: []
  size: medium
  description: 'grammar: teach `sase_core::agent_name_template` the `{@<id>}` / `{@<id>!}`
    marker, add a lexical marker scanner and key accessor, and expose both through
    the `sase_core_py` bindings.

    '
- id: facade
  title: Python facade and prompt-grammar plumbing
  depends_on:
  - grammar
  size: small
  description: 'facade: wrap the new Rust entry points in `sase.agent.names`, widen
    the directive and xprompt argument character classes so braced markers survive
    parsing, and bump the `sase-core-rs` requirement window.

    '
- id: resolve
  title: Launch-time key resolution and text substitution
  depends_on:
  - facade
  size: medium
  description: 'resolve: add a keyed marker resolver that allocates one token per
    key under the agent-name lock and rewrites every occurrence in the dispatch''s
    segment text, wire it into the launch funnel, and reject markers that reach a
    runner.

    '
- id: qualify
  title: Per-invocation key qualification at swarm expansion
  depends_on:
  - resolve
  size: medium
  description: 'qualify: rewrite unqualified `{@<id>}` markers to `{@<xprompt>.<stamp>.<id>!}`
    while expanding an xprompt swarm so each invocation gets its own key space, leaving
    `!` markers untouched.

    '
- id: migrate
  title: Migrate existing xprompt swarms
  depends_on:
  - qualify
  size: small
  description: 'migrate: convert the bare `@` markers in this repo''s `reads` swarm
    and the chezmoi `research_swarm` swarm to the keyed syntax, including the prose
    references.

    '
- id: docs
  title: Document the keyed marker syntax
  depends_on:
  - qualify
  size: small
  description: 'docs: describe the keyed marker, its qualification rule, and the `!`
    override in the xprompt documentation and the xprompt catalog help surfaces.

    '
create_time: 2026-07-29 09:07:16
status: wip
bead_id: sase-aq
---

- **PROMPT:** [202607/prompts/agent_name_key_markers.md](prompts/agent_name_key_markers.md)
- **BEAD:** [sase-aq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-aq/README.md)

# Plan: Keyed `{@<id>}` agent-name markers for xprompt swarms

## Confirmed root cause

The reported failure is real and the diagnosis in the prompt is correct.

The `#research_swarm` xprompt (chezmoi `home/sase/xprompts/research_swarm.md`) names its four agents with the bare `@`
agent-name template marker: `%clan(research.@, ...)`, `%id:research.@.cdx`, `%id(cld, clan=research.@)`,
`%id(final, clan=research.@)`, `%id(image, clan=research.@)`.

Two swarm launches overlapped:

| Swarm | cdx              | cld              | final            | image                   |
| ----- | ---------------- | ---------------- | ---------------- | ----------------------- |
| A     | `20260729064015` | `20260729064019` | `20260729064020` | deferred behind `final` |
| B     | `20260729065513` | `20260729065515` | `20260729065520` | `20260729065525`        |

Swarm A allocated the token `o` (hood `research.o`); swarm B, launched 15 minutes later, allocated `p` (hood
`research.p`). Swarm A's `image` segment waits on `research.@.final`, so it did not bootstrap until 07:13:27. By then
the registry contained a `research.p` clan.

`clan=research.@` is not rewritten by the launcher's template-reference pass — `rewrite_template_references` in
`src/sase/agent/multi_prompt_reference_rewriting.py` only rewrites `%wait` and `#fork`/`#resume` arguments — so the
child process received the literal template. With no clan payload in its environment,
`src/sase/axe/run_agent_directive_identity.py:188` fell through to `_resolve_clan_membership`, which calls
`_resolve_clan_target` in `src/sase/agent/clan_membership.py:174`. That helper resolves a template target with
`resolve_agent_name_template_reference(target, names=get_reserved_clan_names())`, and that resolver is defined as
_latest wins_ (`require_latest_agent_name_template` in `src/sase/agent/names/_templates.py:278`). The latest
`research.@` clan was swarm B's, so `research.@` rendered `p`, and the agent failed with:

```
ClanMembershipError: Agent 'research.o.image' cannot join clan 'research.p':
clan members must use the 'research.p.<suffix>' hood
```

The manual recovery relaunch at `20260729072341` confirms the fix shape: its `raw_xprompt.md` is byte-for-byte the same
segment with every `research.@` hand-substituted to `research.o`, and it ran to completion.

Two secondary defects follow from the same root cause and are in scope:

1. Prose references are never resolved at all. Step 1 of the lead-researcher segment says
   ``Read both transcripts to learn which report file each researcher wrote (`research.@.cdx` -> `__a`, ...)``. Nothing
   rewrites prose, so the lead researcher reads a literal `research.@.cdx` that names no agent.
2. `%clan` / `clan=` arguments reach the child unresolved, which is what made a late-bootstrapping agent depend on
   registry state 33 minutes after its launch was planned.

## Design

### Syntax

A keyed marker is `{@<id>}` or `{@<id>!}`.

- `<id>` is one or more alphanumeric segments joined by `.`: `[A-Za-z0-9]+(?:\.[A-Za-z0-9]+)*`. Examples: `{@1}`,
  `{@foobar}`, `{@lead.a}`.
- A trailing `!` marks the id as **already qualified**: no implicit prefix is added.
- The bare `@` marker keeps working exactly as it does today. It is not deprecated by this epic; the keyed form is the
  path toward eventually requiring a keyed marker, but no warning or removal is part of this work.
- A single agent-name value carries exactly one marker. `{@1}.{@2}` and a value mixing a braced marker with a stray bare
  `@` are both errors.

### Key qualification

While an xprompt swarm is expanded, every **unqualified** marker in every produced segment is rewritten to

```
{@<xprompt>.<stamp>.<id>!}
```

where `<xprompt>` is the xprompt's name and `<stamp>` is unique per xprompt invocation within the dispatch. Markers that
already carry `!` pass through untouched, so a nested swarm can deliberately share a key with its caller.

Keys are **dispatch-scoped**: the resolution table lives for one launch. The `<xprompt>.<stamp>.` prefix keeps two
invocations of the same swarm in one dispatch from colliding and makes every key globally unique, which is what a future
durable key ledger would need. Cross-dispatch persistence is explicitly **not** part of this epic; a marker in a new
dispatch allocates a fresh token. This is sufficient for the reported bug, because resolution now happens once in the
parent and is baked into the prompt text.

A marker that appears outside any xprompt swarm expansion (a literal user prompt, an embedded `#name` reference in a
plain prompt) is resolved with its literal id as the key, exactly as if it had been written with `!`.

### Resolution

Resolution happens once per dispatch, in the parent, after swarm expansion and before anything parses directives or
spawns a process. For each distinct key:

1. Collect every **name shape** containing that key across all segments. A name shape is the marker plus the maximal run
   of name-legal characters (`[A-Za-z0-9_.-]`) immediately before and after it. `%id:research.{@1}.cdx` yields
   `research.{@1}.cdx`; `clan=research.{@1})` yields `research.{@1}`; the prose `` `research.{@1}.cdx` `` yields the
   same shape as the first. This is purely lexical — no directive parsing is involved, so `%id`, `%clan`, `clan=`,
   `%wait`, `#fork`, and prose are all covered by one pass.
2. Under `agent_name_allocation_lock()`, walk `iter_agent_name_template_tokens()` and pick the lowest token for which
   every rendered name shape and every dotted namespace prefix it implies is available. Availability uses
   `AgentNameNamespaceReservationIndex.from_registry_names(get_reserved_agent_names(), namespace_containers=get_reserved_clan_names())`,
   the same index `PlannedNameAllocator` already uses.
3. Substitute the token for every occurrence of that key's marker in every segment's full text.

Substitution applies the same separator rule as `render_agent_name_template`: insert `-` before the token only when the
name-legal run immediately preceding the marker is non-empty, its last character is neither `-` nor `.`, and the token
starts with a lowercase letter. `research.{@1}` becomes `research.o`; `foo{@1}` becomes `foo-o`; a marker at the start
of a line or after a space or `(` has an empty preceding run and becomes a bare `o`. Fenced code blocks and
`%xprompts_enabled:false` disabled regions are protected, matching every other prompt rewriter in
`src/sase/xprompt/_fenced_blocks.py` and `src/sase/xprompt/_disabled_regions.py`.

After substitution the segments contain only concrete names. `prepare_clan_launches` sees a literal `research.o`,
`validate_launch_name_requests` sees literal names, and — decisively for the reported bug — a child that bootstraps 33
minutes later reads `clan=research.o` from its own prompt and can no longer consult registry state.

### Why not resolve lazily at runtime

Keeping the marker unresolved and looking the key up in a durable ledger at agent-bootstrap time was considered and
rejected for this epic. It requires a new persistent store that must survive `rebuild_name_registry`, it leaves the
prose problem unsolved, and it keeps the failure mode alive for any code path that loses the ledger. Parent-side
substitution is what the user's own manual recovery did, and it removes the dependency on late registry state entirely.

---

## Phase `grammar`: Keyed marker grammar in sase-core

Work in the `sase-core` repo. Open it with `/sase_repo` (`sase repo open sase-core`) and use the printed path.

All edits are in `crates/sase_core/src/agent_name_template.rs` and `crates/sase_core_py/src/lib.rs`.

Do not hand-edit any `version` field in `Cargo.toml`; release-plz owns those (see the repo's `AGENTS.md`).

### `crates/sase_core/src/agent_name_template.rs`

- Add `AgentNameTemplateKey { id: String, qualified: bool }`.
- Add a marker scanner over arbitrary text that returns each marker's byte span, `id`, and `qualified` flag. Both the
  braced form and the bare `@` must be reported, so callers can detect mixed usage. Braced ids validate against
  `[A-Za-z0-9]+(?:\.[A-Za-z0-9]+)*`; a `{@` that is not followed by a valid id and a closing `}` is not a marker (it is
  ordinary text).
- Extend `AgentNameTemplate` with `marker: String` (the literal matched marker text) and
  `key: Option<AgentNameTemplateKey>`.
- `AgentNameTemplate::parse` now finds exactly one marker of either form. Two markers, or a braced marker plus a stray
  bare `@`, produce the existing `InvalidMarkerCount` error. `prefix` and `suffix` are the text before and after the
  whole marker span, so the existing `render`, `match_token`, and separator logic keep working unchanged.
- `namespace_template()` must re-emit `self.marker` verbatim instead of the hard-coded `AGENT_NAME_TEMPLATE_MARKER`, so
  `research.{@1}.cdx` yields `research.{@1}` rather than `research.@`.
- `is_agent_name_template` switches from counting `'@'` to counting markers.
- Add `agent_name_template_key(template) -> Result<Option<AgentNameTemplateKey>, _>`.

### `crates/sase_core_py/src/lib.rs`

Expose `agent_name_template_key` (returning `None` or a dict with `id` / `qualified`) and the marker scanner (returning
a list of dicts with `start`, `end`, `id`, `qualified`, `braced`). Update the module doc comment's binding inventory at
the top of the file to list the new names.

### Tests

Extend the `#[cfg(test)]` module in `agent_name_template.rs`:

- `parse` accepts `research.{@1}.cdx`, `{@a}`, `foo.{@lead.a}`, and `research.{@x!}`, and reports the right key and
  qualification flag for each.
- `parse` rejects `{@1}.{@2}`, `a@b{@1}`, `{@}` used as a template, and `{@-bad}`.
- `render` and `match_token` are exact inverses for keyed templates across the same token matrix the existing
  `render_and_match_are_exact_inverses` test uses.
- `namespace_template` returns `research.{@1}` for `research.{@1}.cdx` and `foo.{@1!}x` for `foo.{@1!}x.bar`.
- The scanner finds markers in prose, reports byte spans that round-trip through slicing, and does not treat `{@}`,
  `{ @1 }`, or a Jinja `{{ prompt }}` as a marker.

Run `cargo fmt`, `cargo clippy`, and `cargo test` in the sase-core workspace.

---

## Phase `facade`: Python facade and prompt-grammar plumbing

Work in this repo.

### Rust dependency

Bump the `sase-core-rs` requirement window in `pyproject.toml:46` to the version that carries the `grammar` phase's
bindings. `just install` builds an editable wheel from `../sase-core` when that checkout exists (see the `rust-install`
target in the `Justfile`), so local development picks the change up automatically; the pyproject window still has to
move for released installs.

### `src/sase/agent/names/_templates.py`

- Add `AgentNameTemplateKey` (frozen dataclass mirroring the Rust struct), `agent_name_template_key(template)`, and
  `iter_agent_name_key_markers(text)` wrappers over the new bindings, following the existing `_core("...")` pattern.
- Add `marker` and `key` fields to the `AgentNameTemplate` dataclass and populate them from the parse payload.
- Export the new names from `src/sase/agent/names/__init__.py`.

### Prompt grammar

Braced markers must survive the existing lexers so preflight, preview, completion, and `sase xprompt explain` see a
template rather than a truncated argument. Add `{`, `}`, and `!` where they are missing:

- `src/sase/xprompt/_directive_types.py:15` — `_DIRECTIVE_PATTERN`'s colon-argument class is
  `[!a-zA-Z0-9_#/.,()@=-]*[a-zA-Z0-9_#/,()@=-]`. Today `%id:research.{@1}.cdx` truncates at the brace. Add `{}` to both
  the body class and the trailing class (the trailing class must accept `}` so `%clan:research.{@1}` parses whole).
- `src/sase/xprompt/processor.py:63` — `_XPROMPT_PATTERN`'s colon-argument class
  `[a-zA-Z0-9_.~,+/@-]*[a-zA-Z0-9_~,+/@-]` truncates `#fork:research.{@1}.final`. Add `{}` and `!` to both halves.
- Verify `_DIRECTIVE_PREFIX_RE` in `src/sase/xprompt/_parsing_vcs_tags.py:20` (`%[^\s(]+...`) and `parse_args` in
  `src/sase/xprompt/_parsing.py` already pass braces through; add a regression test rather than a change if they do.
- `_RESUME_REF_RE` in `src/sase/agent/multi_prompt_reference_resume.py` uses `[^\s)]+` for its colon argument and needs
  no change; cover it with a test.

### Validation

`validate_user_agent_name` and the `_LaunchNameRequest` path in `src/sase/agent/launch_validation.py` must treat a value
containing a keyed marker as a template (it already routes templates through `is_agent_name_template`, which the
`grammar` phase updated) and must never see one at claim time. Add an explicit, actionable error for an unresolved
marker reaching a claim: name the marker and say the launch pipeline failed to resolve it.

### Tests

Add cases to `tests/test_agent_name_templates.py` for the facade wrappers and to
`tests/test_xprompt_processor_pattern.py` plus the directive-pattern tests for the widened character classes. Assert
that `%id:research.{@1}.cdx`, `%clan(research.{@1}, tribe=research)`, `%wait:research.{@1}.final`, and
`#fork:research.{@1}.final` each round-trip through their respective extractors with the marker intact.

---

## Phase `resolve`: Launch-time key resolution and text substitution

Work in this repo. This is the phase that fixes the reported failure.

### New module `src/sase/agent/agent_name_keys.py`

Public entry point:

```python
def resolve_agent_name_key_markers(segments: Sequence[str]) -> list[str]: ...
```

Behavior:

1. Return `segments` unchanged when no segment contains `{@` (fast path — this runs on every dispatch).
2. Protect fenced blocks and disabled regions in each segment before scanning, and unprotect after substituting.
3. Scan every segment for keyed markers with `iter_agent_name_key_markers`. Group by key id. Bare `@` markers are
   ignored by this pass and keep their existing behavior.
4. For each key, collect the distinct **name shapes** as described in the design: expand each marker span left and right
   over `[A-Za-z0-9_.-]`, and keep the resulting substring with the marker still embedded.
5. Under `agent_name_allocation_lock()`, build one
   `AgentNameNamespaceReservationIndex.from_registry_names(get_reserved_agent_names(), namespace_containers=get_reserved_clan_names())`
   and iterate `iter_agent_name_template_tokens()`. A token is acceptable for a key when, for every collected name
   shape, `candidate_available(rendered_name, rendered_namespace)` holds. Keys are resolved in first-appearance order
   and each accepted token's names are added to the index so two keys in one dispatch cannot pick the same hood.
6. Substitute every marker occurrence with its key's token, applying the separator rule from the design section.

Keep the whole resolution inside one `agent_name_allocation_lock()` block so all keys in a dispatch see one consistent
snapshot.

Also expose a small `has_unresolved_agent_name_key_marker(text) -> bool` helper for the runner guard below.

### Wiring

Call the resolver in `src/sase/agent/launch_cwd_agents.py`, immediately after `expanded_segments` is built from
`expand_xprompt_swarms_with_metadata` and **before** `validate_launch_name_requests` and the
`len(expanded_segments) > 1` branch, so both the multi-segment and single-segment paths get resolved segments.

Audit the other dispatch entry points before deciding this is sufficient: the ace TUI launch path, the mobile gateway
path (`src/sase/integrations/_mobile_agent_launch.py`), and `sase agent`-style CLI launches. If any of them reaches
`launch_multi_prompt_agents` without passing through `launch_cwd_agents`, resolve there too, or move the call into the
shared funnel those paths share. Resolution must be idempotent so a double call is harmless.

### Runner guard

In `src/sase/axe/run_agent_directives.py`, before identity resolution, raise a clear error when the resolved prompt
still contains a keyed marker. A marker that reaches a runner means the parent-side resolution was skipped, and failing
loudly there is far better than the silent wrong-hood behavior this epic removes.

### Tests

New `tests/test_agent_name_key_markers.py`:

- Two segments sharing `{@1!}` resolve to the same token; two different keys resolve to different tokens.
- The chosen token skips a hood already reserved in the registry, including the namespace case where `research.o` is
  free as an exact name but `research.o.cdx` is taken.
- The separator rule: `research.{@1}` -> `research.o`, `foo{@1}` -> `foo-o`, a line-leading `{@1}` -> `o`, and a digit
  token never gains a dash.
- Prose, `%id`, `%clan`, `clan=`, `%wait`, and `#fork` occurrences of one key all receive the same token in one pass.
- Fenced blocks and disabled regions are left untouched.
- Idempotence: resolving already-resolved segments is a no-op.

Regression test reproducing the reported failure: build a two-launch scenario where launch A reserves hood `research.o`,
launch B reserves `research.p`, and launch A's deferred `image` segment is bootstrapped afterwards. With bare `@` the
clan target resolves to `research.p`; with `{@1}` the segment text already reads `research.o` and
`_resolve_clan_membership` receives a concrete name. Place the runner-side half near the existing clan tests
(`tests/_clan_summary_persistence_helpers.py` and its users show the established fixtures).

---

## Phase `qualify`: Per-invocation key qualification at swarm expansion

Work in this repo, in `src/sase/agent/xprompt_swarm.py`.

`_render_xprompt_swarm` (line 382) is the single place that renders one xprompt swarm body and splits it into segments,
and every expansion path funnels through it. Qualify there:

- Mint a fresh stamp on every `_render_xprompt_swarm` call, combining a launch timestamp with a monotonic per-dispatch
  counter so two invocations of the same xprompt cannot collide even at identical clock resolution. Do **not** key the
  stamp off `template_group`: `_expand_xprompt_swarms_with_metadata` (line 568) and
  `_expand_embedded_xprompt_swarm_reference` (line 412) both inherit an outer `template_group` on recursion, so a nested
  swarm would otherwise reuse its caller's stamp. Thread a dedicated counter alongside the existing `group_counter`
  (line 645) through the same call sites. The stamp must contain only characters legal in an id (alphanumerics and `.`).
- Rewrite every **unqualified** `{@<id>}` in each produced sub-segment to `{@<xprompt>.<stamp>.<id>!}`. Leave markers
  that already carry `!` untouched, and leave bare `@` untouched.
- Protect fenced blocks and disabled regions, consistent with the resolver.

Recursion: `_expand_xprompt_swarms_with_metadata` re-enters with already-qualified text. Because qualification only
touches unqualified markers, an outer swarm's keys are stable while an inner swarm's own `{@1}` picks up the inner
xprompt's prefix. A marker written `{@shared!}` in both is the documented way to share one hood across nesting levels.

`sase.core` note: qualification is prompt-expansion plumbing that depends on Python-side fence protection and the
existing group counter, so it stays in this repo. The marker grammar it relies on lives in `sase-core` (phase
`grammar`).

### Tests

Extend `tests/test_xprompt_swarm_expansion.py`:

- A swarm body containing `{@1}` in a `%id`, a `%clan`, and prose emits the same qualified key in all three places.
- Two `#swarm` references in one dispatch produce two distinct qualified keys.
- `{@x!}` survives expansion byte-for-byte.
- A nested swarm's unqualified `{@1}` differs from its caller's, while `{@shared!}` matches across both.

Then add an end-to-end test through the launcher (see `tests/test_multi_prompt_launcher_xprompt_groups.py` for the
established harness) asserting that a `research_swarm`-shaped fixture produces four segments whose `%id`, `%clan`,
`%wait`, `#fork`, and prose all name one hood, and that a second launch of the same fixture picks a different hood
without disturbing the first.

---

## Phase `migrate`: Migrate existing xprompt swarms

Two swarm xprompts use the bare `@` marker. A repo-wide sweep for other swarm bodies (a top-level `---` separator
outside fenced blocks plus an `@` in a name position) found no others; re-verify before finishing, including
`src/sase/xprompts/`, `src/sase/default_xprompts/`, `sase/xprompts/`, and the home xprompt directories.

Do not touch `%model:@alias` / `%m:@alias` values. Those are model aliases and are unrelated to agent-name markers;
`old_research_swarm.md` contains one and needs no change.

### `sase/xprompts/reads.md` (this repo)

Replace `reads.@` with `reads.{@1}` in all six places: the four `%id` values (`.agy`, `.cld`, `.cdx`, `.final`) and the
three `%wait` values in the final segment.

### `home/sase/xprompts/research_swarm.md` (chezmoi)

Open the chezmoi repo with `/sase_repo` (`sase repo open chezmoi`) and use the printed path. Replace `research.@` with
`research.{@1}` in every occurrence:

- line 11: `%clan(research.@, ...)` and `%id:research.@.cdx`
- line 16: `clan=research.@`
- line 20: `clan=research.@`, `%wait:research.@.cdx`, `%wait:research.@.cld`
- line 39: the prose ``(`research.@.cdx` -> `__a`, `research.@.cld` -> `__b`)`` — this is the reference that never
  resolved before and now will
- line 60: `clan=research.@`, `%wait:research.@.final`, `#fork:research.@.final`

Per the chezmoi repo's `CLAUDE.md`, after committing there you must run `chezmoi update -a --force` to apply the change
to the home directory.

### Verification

Launch the migrated `#research_swarm` twice in quick succession and confirm from `sase ace` that each launch's four
agents share one hood and that the deferred `image` segment joins its own launch's clan. This is the acceptance check
for the whole epic.

---

## Phase `docs`: Document the keyed marker syntax

Work in this repo.

- `docs/xprompt.md`: document `{@<id>}` and `{@<id>!}` alongside the existing swarm and `%id`/`%clan` material. Cover
  the id grammar, the implicit `<xprompt>.<stamp>.` qualification, dispatch scoping, the `!` override and when to reach
  for it (sharing a hood across nested swarms), the dash-insertion rule with a worked example, and the fact that prose
  occurrences are substituted too. State plainly that bare `@` still works but resolves latest-wins and is therefore
  unsafe for a swarm whose members can start late.
- Check `docs/ace.md` and `docs/agent_families.md` for agent-name-template text that should cross-reference the new
  syntax.
- Refresh any xprompt catalog or completion help strings that enumerate marker syntax so the TUI and the catalog agree
  with the docs.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims: the user has not granted
memory-file permission for this work.

---

## Out of scope

- Deprecating or warning on the bare `@` marker. The keyed form is additive here.
- A durable, cross-dispatch key-to-token ledger. Keys are dispatch-scoped; the qualified key shape is chosen so a ledger
  could be added later without a syntax change.
- Changing how `%clan` resolves a _concrete_ target, or the latest-wins semantics of
  `resolve_agent_name_template_reference` for bare `@`.
