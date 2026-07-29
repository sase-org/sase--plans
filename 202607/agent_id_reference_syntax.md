---
tier: epic
title: Explicit `{@<id>}` agent-name reference syntax
goal: 'Every agent-name reference in one launch batch that shares an `{@<id>}` label
  resolves to exactly one concrete name, chosen once at launch-plan time and substituted
  into every segment''s prompt text, so no agent can ever re-resolve a template later
  and drift onto a different clan or peer.

  '
phases:
- id: core
  title: Rust core `{@<id>}` template primitives
  depends_on: []
  size: small
  description: 'core: add braced-marker parsing, canonicalization, and id extraction
    to sase-core''s agent_name_template module, expose them through the PyO3 bindings,
    and cover the render/match round-trip for the new form.

    '
- id: lexer
  title: Python lexical plumbing for braced markers
  depends_on:
  - core
  size: medium
  description: 'lexer: widen the directive colon-argument pattern to accept braces,
    surface the new core helpers through sase.agent.names, and make agent-name claiming
    reject a stray unresolved brace instead of registering it into a name.

    '
- id: scope
  title: Xprompt-scoped id namespacing
  depends_on:
  - lexer
  size: medium
  description: 'scope: seal unqualified ids in an expanded xprompt body to `<xprompt>.<id>`
    at substitution time, honor the trailing `!` opt-out, and keep nested and repeated
    swarm invocations from colliding.

    '
- id: bind
  title: Batch-scoped id-to-token binding
  depends_on:
  - scope
  size: medium
  description: 'bind: key PlannedNameAllocator token groups off resolved ids instead
    of the heuristic template-base key, and allocate a token for every id in the batch
    including ids that no `%id` directive defines.

    '
- id: substitute
  title: Whole-prompt id substitution before spawn
  depends_on:
  - bind
  size: medium
  description: 'substitute: rewrite every braced marker in every segment to its bound
    concrete text before any child spawns, covering directives, fork references, and
    prose, on both the clan and non-clan launch paths.

    '
- id: clanenv
  title: Preserve clan membership across the runner code refresh
  depends_on: []
  size: small
  description: 'clanenv: stop the launcher-supplied clan payload from being lost when
    a waiting runner re-execs for a code refresh, which is the trigger that turned
    the drifting template into a hard failure.

    '
- id: migrate
  title: Migrate first-party xprompt swarms
  depends_on:
  - substitute
  size: small
  description: 'migrate: convert the reads and research swarms in this repo and the
    chezmoi repo to the braced form, including the prose references the old form never
    resolved.

    '
- id: docs
  title: Document the braced reference form
  depends_on:
  - substitute
  size: small
  description: 'docs: describe the new syntax, its scoping rules, and how it differs
    from a bare marker across the xprompt, axe, and ace docs and the bundled skill
    sources.

    '
create_time: 2026-07-29 08:25:31
status: wip
bead_id: sase-ap
---

- **BEAD:** [sase-ap](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ap/README.md)

# Plan: Explicit `{@<id>}` agent-name reference syntax

## Verification of the reported failure

The suspicion is **confirmed**, and the exact mechanism is now known.

`research.o.image` failed with:

```
ClanMembershipError: Agent 'research.o.image' cannot join clan 'research.p':
clan members must use the 'research.p.<suffix>' hood
```

The `research_swarm` xprompt's fourth segment is:

```
%id(image, clan=research.@) %wait(priority=20) %model:codex/gpt-5.6-sol %wait:research.@.final #fork:research.@.final
```

Two independent defects combined.

**1. The `@` template survives into the child prompt for clan arguments.**
`PlannedNameAllocator.rewrite_template_references` (`src/sase/agent/multi_prompt_reference_rewriting.py:14`) rewrites
templates only inside `%wait` directives and `#fork`/`#resume` references. It does not touch `%clan(...)` or the `clan=`
keyword on `%id`. So the launcher pinned `%wait:research.o.final` and `#fork:research.o.final`, but the child's prompt
text still read `clan=research.@`. Prose references such as line 39's `` `research.@.cdx` `` were likewise never
rewritten, so the lead researcher's prompt literally contained the unresolved marker.

**2. The pinned clan payload does not survive the runner's code-refresh re-exec.** The launcher does resolve the clan
for the whole batch in `prepare_clan_launches` (`src/sase/agent/multi_prompt_launch_plan.py:278-345`) and passes it to
the child in `SASE_AGENT_CLAN_MEMBERSHIP`. But `consume_clan_membership_plan_from_env`
(`src/sase/agent/clan_membership.py:32`) **pops** that variable during the first bootstrap pass. The `image` agent then
blocked for 33 minutes on `%wait:research.o.final`; sase's editable checkout moved during that wait (several agents
visible in the same screenshot were committing to this repo), so `refresh_runner_code_after_wait`
(`src/sase/axe/run_agent_runner_refresh.py:63`) re-exec'd the runner with the already-popped environment. The refreshed
pass re-ran `bootstrap_agent_run` → `extract_directives_and_write_meta` → `resolve_agent_identity` with
`clan_membership_plan = None`, which took the `directives.clan is not None` branch at
`src/sase/axe/run_agent_directive_identity.py:187` and re-resolved `research.@` from scratch.

That re-resolution goes through `resolve_agent_name_template_reference` (`src/sase/agent/names/_templates.py:308`),
whose semantics are **"the highest existing concrete name matching this template"**. By 07:13 a second research swarm
had created the `research.p` clan — visible as a live `research.p` hood in the same screenshot — so `research.@`
resolved to `research.p`. The agent's own name was still `research.o.image`, taken from the durable
`SASE_AGENT_PLANNED_NAME` reservation, which does survive the exec. Name and clan disagreed, and the prefix check at
`src/sase/axe/run_agent_directive_identity.py:249-266` raised.

Timing corroborates it exactly: `research.o.image` shows a 33m05s duration ending at 07:13:27, ten seconds after
`research.o.final` completed at 07:13:17 — the failure landed the instant the wait cleared and the refreshed bootstrap
re-parsed the prompt.

The braced syntax fixes defect 1 categorically: once every marker is substituted into the prompt text before spawn,
re-resolution is idempotent no matter how many times bootstrap runs. Defect 2 is a real latent bug in its own right —
the clan **declarer** re-exec'ing after a wait would hit `declare_clan_membership`'s create-only path and fail with
"already exists" — so it gets its own phase.

## Design

### Syntax

`{@<id>}` is a drop-in replacement for the bare `@` marker inside an agent-name template. `<id>` is a non-empty run of
`[A-Za-z0-9]` and `.`. A trailing `!` inside the braces (`{@<id>!}`) marks the id as already fully qualified.

```
%clan(research.{@1}, tribe=research) %id:research.{@1}.cdx
%id(final, clan=research.{@1}) %wait:research.{@1}.cdx
```

Rendering is identical to the bare marker: the token replaces the marker, with a `-` inserted when the marker directly
follows an alphanumeric or `_` and the token is letter-leading (`build{@1}` → `build-o`, `research.{@1}` →
`research.o`). That is the "prepended dash in one case" — it is `AgentNameTemplate::render`'s existing separator rule
and must be reused rather than reimplemented.

### Semantic difference from the bare marker

This distinction is the whole point of the change and must be documented and enforced:

| Form         | Meaning                                                                                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@` (legacy) | _Reference_: resolved late against the durable registry to the **latest existing** matching name. Drifts when a newer match appears.                                       |
| `{@<id>}`    | _Binding_: one token allocated for this launch batch, shared by every occurrence of the same id in the batch, substituted into the prompt text before spawn. Cannot drift. |

Because `{@<id>}` is batch-local, an id that appears in the batch **only** in reference positions (`%wait`, `#fork`)
with no `%id` or `%clan` in the same batch binding it is an error, with a message pointing at the bare `@` form for
referring to a previously launched agent. Bare `@` keeps working unchanged; the stated goal is to require the braced
form eventually, so this epic makes it available and converts all first-party swarms, leaving the hard requirement as a
later follow-up. Do not add a deprecation warning for bare `@` in this epic.

### Scoping

An id written inside an xprompt body is implicitly qualified with that xprompt's name, so two xprompts that both use
`{@1}` never share a token. Qualification happens when the body is substituted, by rewriting each unqualified `{@<id>}`
to the sealed form `{@<xprompt>.<id>!}`. Sealed ids are left alone by any outer expansion pass, so nesting composes
correctly: an inner xprompt's ids are qualified with the inner name even when its body is spliced into an outer swarm.

An author writes `{@<id>!}` to opt out entirely — the id stays exactly as written and is batch-global, which is the
escape hatch for coordinating ids across xprompts.

**Assumption to flag:** the request says to "append `<xprompt>.` to `<id>`". Since that fragment carries a trailing dot,
this plan reads it as qualifying the id with the xprompt name as a prefix — `{@1}` inside `research_swarm` becomes id
`research_swarm.1`. If a suffix was intended, only the string built in the `scope` phase changes; nothing else in the
design depends on the order.

Qualification by name alone is not sufficient for uniqueness when the same swarm is invoked twice in one prompt
(`#research_swarm:a --- #research_swarm:b`). The expander already solves this with a per-invocation template group
(`_next_template_group` in `src/sase/agent/xprompt_swarm.py:645` yields `xprompt:<name>:<n>`), and that group is already
threaded per segment as `_ExpandedXpromptSwarmSegment.template_group`. Token binding therefore keys off the pair
`(segment template group, resolved id)`, not the id alone, so repeated invocations stay distinct while the id text the
author sees and writes stays readable. Segments with no template group (a literal user prompt) key off the id alone.

### Why this fixes the failure

After the `substitute` phase, a child receives `%id(image, clan=research.o)` rather than `%id(image, clan=research.@)`.
Any number of bootstrap passes — including one after a code-refresh re-exec — re-resolve to the same concrete
`research.o`, so `resolve_or_create_clan_membership` simply rejoins the existing clan and the name/clan prefix check
passes.

## Rust core `{@<id>}` template primitives

Work in the `sase-core` linked repo, opened with `/sase_repo`. All of this is shared domain behavior that the TUI, CLI,
completion, and editor integration must agree on, so it belongs in `crates/sase_core/src/agent_name_template.rs` per the
Rust core backend boundary rule.

Add to that module:

- A braced-marker grammar: `{@` + id + optional `!` + `}`, where id is `[A-Za-z0-9.]+`. Reject an empty id and reject a
  leading or trailing `.` in the id.
- A lookup that finds braced markers in an arbitrary string and reports each marker's id, sealed flag, and byte span.
- `canonicalize_agent_name_template(source) -> Result<(String, Option<String>)>` returning the bare-`@` template plus
  the marker id when the source used the braced form. `research.{@1}.cdx` yields `("research.@.cdx", Some("1"))`;
  `research.@.cdx` yields `("research.@.cdx", None)`.
- A marker-count rule that treats bare `@` and braced markers uniformly: exactly one marker total. Reject `{@1}.{@2}`
  and `foo.@.{@1}` with a distinct error variant naming both markers, because a two-marker name cannot be rendered.
- `is_agent_name_template` must accept the braced form as a valid one-marker template. Audit its current implementation
  (`value.matches('@').count() == 1`) — a braced marker contains exactly one `@`, so it already returns `true`, but
  `parse` would currently split on the brace and produce the garbage prefix `research.{` and suffix `}`. That
  silent-garbage path is the specific regression risk here.

Export the new functions from `crates/sase_core/src/lib.rs` and bind them in `crates/sase_core_py/src/lib.rs` following
the existing `agent_name_template` binding style.

Rust tests in the module's `mod tests`: canonicalization for each shape already covered by `renders_template_shapes`,
the sealed (`!`) form parsing, render/match round-trip through canonicalization so `{@1}` renders identically to `@` for
every token in the existing token list, and rejection cases for empty ids, malformed braces, and multiple markers.

Bump the published `sase-core-rs` version window in this repo's `pyproject.toml` only if the sase-core release process
requires it; dev installs build from the local checkout, so a follow-up phase running `just install` picks the new
bindings up automatically.

## Python lexical plumbing for braced markers

Three separate lexical surfaces need to admit braces.

**Directive colon arguments.** `_DIRECTIVE_PATTERN` in `src/sase/xprompt/_directive_types.py:15` restricts colon
arguments to `[!a-zA-Z0-9_#/.,()@=-]`, which excludes `{` and `}`. `%wait:research.{@1}.cdx` currently truncates at the
brace and silently produces the wrong dependency — verify that failure mode with a test first, then add `{` and `}` to
both character classes in group 3. This pattern is shared by the alt splitter, the extractor, completion, and every
rewriting pass, so re-run the full directive test suite after the change. Confirm no interaction with the `%{a | b}`
alt-directive syntax, which is matched by a leading `%{` and so is lexically distinct from a `{@` inside a directive
value.

The other two surfaces already tolerate braces and need only test coverage confirming so: `_RESUME_REF_RE`
(`src/sase/agent/names/_resume.py:21`) accepts `[^\s)]+` for a colon argument, and `_DIRECTIVE_PREFIX_RE`
(`src/sase/xprompt/_parsing_vcs_tags.py:20`) accepts `%[^\s(]+`.

**Names package API.** Surface the new core helpers through `src/sase/agent/names/_templates.py` and
`src/sase/agent/names/__init__.py`, mirroring the existing thin-wrapper style that calls `_core(...)` and translates
`ValueError` into the typed `AgentNameTemplateError` hierarchy. Add a helper that, given an arbitrary text span, expands
to the maximal surrounding run of agent-name characters `[A-Za-z0-9_.-]`, canonicalizes that run, and renders it with a
token — this is the primitive the `substitute` phase needs, and putting it here keeps the dash rule in one place.

**Guard the unresolved form.** No `{` may reach a claimed agent name. Add a check to `validate_user_agent_name`
(`src/sase/agent/launch_validation.py:123`) rejecting `{` or `}` with a message naming the unresolved marker, so a
substitution gap fails loudly at claim time instead of registering an agent literally named `research.{@1}.cdx`.

## Xprompt-scoped id namespacing

Qualification runs when an xprompt body is substituted, so each body is sealed with its own name before any outer or
inner expansion sees it.

Add a pass that rewrites every unsealed `{@<id>}` in a substituted body to `{@<xprompt>.<id>!}` and leaves sealed
`{@<id>!}` markers untouched. Apply it inside `expand_single_xprompt` (`src/sase/xprompt/processor.py`), not only in the
swarm path — a plain `.md` xprompt part spliced into a segment can carry markers too, and `_render_xprompt_swarm`
(`src/sase/agent/xprompt_swarm.py:382`) already routes through `expand_single_xprompt`.

Requirements:

- Idempotent. Running the pass twice over the same text must not double-qualify.
- Fenced code blocks and `%xprompts_enabled:false` regions are protected, using the existing `protect_fenced_blocks` and
  `protect_disabled_regions` helpers. Inline backtick spans are **not** protected — a prose reference such as
  `` `research.{@1}.cdx` `` is exactly the case the old form got wrong and must resolve.
- Markers in a user's literal prompt (not inside any xprompt body) are never qualified; they behave as if sealed.

Tests: nested swarms where inner and outer both use `{@1}` and must end up with different ids; a sealed `{@1!}` shared
deliberately between two xprompts; markers inside fenced blocks left verbatim; a marker in an xprompt whose name
contains `/` (namespaced xprompts such as `#research/image`) producing a usable id.

## Batch-scoped id-to-token binding

`PlannedNameAllocator` (`src/sase/agent/multi_prompt_reference_allocator.py`) already implements exactly the required
semantics — `_template_group_tokens` maps a group key to one token, and every template in that group renders with it,
choosing a token only when _all_ rendered names and namespaces in the group are free. This phase replaces the heuristic
group key with the explicit id.

- `normalize_template_group` (`src/sase/agent/multi_prompt_reference_allocation.py:65`) currently derives a key from the
  template base by stripping the last dotted segment. When the incoming template carries a braced marker, derive the key
  from `(segment template group, resolved id)` instead. Keep the heuristic path intact for bare-`@` templates so legacy
  prompts are unaffected.
- `planned_name_for_prompt` and `planned_names_for_template_group` must canonicalize a braced `%id` value to its
  bare-`@` form before handing it to the existing allocation machinery, so allocation, reservation, and
  namespace-occupancy logic all continue to operate on the shape they already understand.
- Record the chosen token per id on the allocator so the `substitute` phase can read it back. `_template_latest` is
  keyed by exact template string and is not sufficient; add an explicit id-to-token map.
- An id may appear with no `%id` template binding it — `%clan(research.{@1})` whose members all join via `clan=`, for
  instance. Collect every id in the batch up front, together with every name template and clan target it appears in, and
  allocate a token for each id from that full set. An id that appears only in `%wait` or `#fork` positions must raise a
  `DirectiveError` naming the id and suggesting bare `@` for cross-batch references.

Tests: two segments sharing an id get one token and non-colliding names; two different ids in one batch are allocated
independently; a batch whose first candidate token collides with an existing registry name skips to the next token for
_every_ member of the group, not just the colliding one; an unbound reference-only id errors.

## Whole-prompt id substitution before spawn

This is the phase that fixes the reported failure. After name planning completes and before any child spawns, every
braced marker in every segment must be replaced with its bound concrete text.

Add a substitution pass alongside `rewrite_template_references` in `src/sase/agent/multi_prompt_reference_rewriting.py`.
Unlike that function, it is **not** directive-aware: it scans the whole prompt text and replaces every braced marker
occurrence, so `%clan(...)`, the `clan=` keyword on `%id`, `%wait`, `#fork`, and plain prose are all covered by
construction. For each occurrence:

1. Expand to the maximal surrounding run of `[A-Za-z0-9_.-]` characters.
2. Reject the run if it contains more than one marker, or a bare `@` alongside a braced marker.
3. Canonicalize the run to a bare-`@` template and render it with the id's bound token, reusing the helper added in the
   `lexer` phase so the dash rule stays in one implementation.
4. Splice the rendered text back.

Protect fenced blocks and disabled regions; leave inline backtick spans unprotected.

Wire it in at both launch paths, each of which already has the right ordering hook:

- Clan path: `prepare_clan_launches` (`src/sase/agent/multi_prompt_launch_plan.py:250-260`) already rewrites every slot
  prompt after name planning completes, precisely so the shared namespace is settled first. Substitute there. Because
  the clan target is now concrete text, the templated-clan resolution branch at lines 284-322 becomes a no-op for
  braced-form prompts; leave it in place for bare-`@` prompts.
- Non-clan path: `src/sase/agent/multi_prompt_launch_execution.py` runs the same rewrite per segment; apply the
  substitution at the same point.

Also audit and cover the paths that re-derive a prompt after launch, since each is a chance for a marker to leak:
`src/sase/agent/retry_prompt.py`, `src/sase/agent/repeat_launcher.py`, and `src/sase/agent/output_variable_context.py`.
A retry re-runs an already-substituted prompt, so it should see concrete names and need no change — confirm that with a
test rather than by inspection.

**Acceptance test for the reported bug.** Add a regression test that launches a batch modeled on `research_swarm` — a
`%clan(x.{@1}, ...)` declarer plus a `%id(image, clan=x.{@1})` member — asserts the member's spawned prompt text
contains the concrete clan name and no `{`, then registers an interfering newer clan (`x.<next-token>`) and re-runs
`extract_directives_and_write_meta` against that spawned prompt with no `SASE_AGENT_CLAN_MEMBERSHIP` in the environment.
It must resolve back to the original clan. Under the current code that scenario reproduces the `ClanMembershipError`;
assert it no longer does.

## Preserve clan membership across the runner code refresh

Independent of the syntax work, and worth fixing because it is what turned a latent drift into a hard failure.

`consume_clan_membership_plan_from_env` (`src/sase/agent/clan_membership.py:32`) pops `SASE_AGENT_CLAN_MEMBERSHIP` so
nested launches cannot inherit it. That pop is correct for nested launches but wrong for the runner's own re-exec:
`refresh_runner_code_after_wait` (`src/sase/axe/run_agent_runner_refresh.py:63`) calls `os.execv` with the current
`os.environ`, so the refreshed pass loses the launcher's pinned clan identity and re-resolves from scratch.

Restore the payload for the re-exec only. The refresh function already re-materializes the one-shot prompt file before
exec and documents that contract in its module docstring — "any future argv field that names a one-shot resource must
therefore be re-materialized here before exec". The clan payload is the same class of one-shot resource, so
re-materializing it there is the change the existing design calls for. Capture the decoded plan during bootstrap and
re-export it into `os.environ` immediately before `os.execv`, gated on the same conditions as the exec itself so nothing
else in the process ever sees it restored.

Extend the module docstring to name the clan payload alongside the prompt file, and add a test that a refreshed pass
still receives the launcher's clan plan. Also add a test for the declarer case: an agent whose segment carries
`%clan(...)` and a `%wait`, re-bootstrapped after a refresh, must not fail with a create-only "already exists" error.

## Migrate first-party xprompt swarms

Convert every first-party swarm to the braced form. Bare `@` remains supported, so this is a behavior fix for these
swarms rather than a forced migration.

**This repo** — `sase/xprompts/reads.md`. Four segments using `reads.@.agy`, `reads.@.cld`, `reads.@.cdx`, and
`reads.@.final`, plus three `%wait` directives on lines 66-68. Replace each `@` with `{@1}`; the implicit qualification
makes the effective id `reads.1`.

**Chezmoi repo** — open with `/sase_repo` and edit only the path it prints.

- `home/sase/xprompts/research_swarm.md`. Replace `research.@` throughout: the `%clan` declaration and
  `%id:research.@.cdx` on line 11, the `clan=` keywords on lines 16, 20, and 60, the `%wait` targets on lines 20 and 60,
  and the `#fork` target on line 60. Critically, also convert the **prose** references on lines 39-40 —
  `` `research.@.cdx` `` and `` `research.@.cld` `` — which the old form never resolved, so the lead researcher was told
  to look for an agent named with a literal `@`.
- `home/sase/xprompts/old_research_swarm.md`. Line 13 uses `%m:@default`, which is a **model alias**, not an agent-name
  template. Leave it alone. Verify no other agent-name markers exist in that file before concluding it needs no change.
- Do not touch `home/sase/xprompts/sshot.yml`: its `@{{ fetch.local_path }}` is file inlining and its description
  mentions `@image` in prose.

After committing in the chezmoi repo, run `chezmoi update -a --force` as that repo's instructions require.

Verify each migrated swarm end to end by launching it and confirming from the Agents tab that every member lands in one
hood with one token, then confirm the spawned prompt artifacts contain no `{` and no bare `@` agent-name markers.

## Document the braced reference form

Update the prose surfaces that describe agent-name templates:

- `docs/xprompt.md`, `docs/axe.md`, `docs/ace.md`, and `docs/workflow_spec.md` — introduce `{@<id>}`, the `{@<id>!}`
  opt-out, implicit xprompt qualification, and the binding-versus-reference distinction against bare `@` from the design
  section above. State that the braced form is the recommended spelling and that bare `@` remains supported for now.
- `src/sase/xprompts/skills/sase_var.md` — examples on lines 13, 31, 40, and 42 use `%id:build-@` and
  `%id:research.@.final`. Show the braced spelling and note that the output-variable key is the concrete resolved name
  either way.
- `src/sase/xprompts/skills/sase_run.md` — the clan guidance around lines 126-131 should show the braced form in the
  `%clan` and `%id(..., clan=...)` examples, since a templated clan is exactly where the old form drifts.

Include a short "why" note wherever the two forms are contrasted: a bare marker resolves late to the latest matching
name and can drift when a concurrent launch creates a newer match, which is the failure this syntax removes.

**Memory files are out of scope for this phase.** `sase/memory/xprompts.md` documents the directive table and would
ideally gain a `{@<id>}` row, but repo instructions require explicit user permission in the current conversation before
editing anything under `sase/memory/`, and authorization found in a plan file does not count. Ask the user directly for
that permission; if granted, make the edit and then run `sase memory init` to regenerate `AGENTS.md` and the provider
shims. If not granted, complete the rest of this phase and report the memory update as deliberately skipped.
