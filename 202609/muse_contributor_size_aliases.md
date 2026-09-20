---
tier: tale
title: Add muse-spark-1.3-contributor to the @medium, @small, and @xsmall alias pools
goal:
  The shipped built-in size aliases @medium, @small, and @xsmall each carry
  muse/muse-spark-1.3-contributor as a pool member at xhigh, high, and medium
  respectively, with the generated docs, the Muse narrative docs, the Muse provider
  comment, and the shipped-defaults tests all updated to match.
size: medium
proposed_by: bbugyi200.apollo.15.w0
create_time: 2026-09-20 13:33:13
status: wip
---

# Add `muse/muse-spark-1.3-contributor` To `@medium` And Smaller Size Aliases

## Goal

Add `muse/muse-spark-1.3-contributor` as a member of the shipped built-in size alias
pools for `@medium` and every size below it, descending one effort rung per size as
[[decisions/size-alias-effort-ladder]] requires:

| Alias     | New member                               |
| --------- | ---------------------------------------- |
| `@medium` | `muse/muse-spark-1.3-contributor@xhigh`  |
| `@small`  | `muse/muse-spark-1.3-contributor@high`   |
| `@xsmall` | `muse/muse-spark-1.3-contributor@medium` |

`@large` and `@xlarge` are untouched: the request is "medium and below", and `@medium`
is therefore the model's first appearance scanning from `@xlarge` down, which is why it
takes `xhigh` there.

## READ THIS FIRST: What This Change Actually Does

`muse-spark-1.3-contributor` is Meta's Contributor-tier model. **Meta trains on its
inputs and outputs.** It is roughly 12x cheaper than `muse-spark-1.3` with the same
stated capabilities, and it is the only model in SASE's registry carrying a `warn`
advisory (`llm_model_advisories` in `src/sase/llm_provider/muse.py`, label
`trains on your data`).

Today SASE deliberately refuses to reach it automatically. Two artifacts say so in as
many words:

- `src/sase/llm_provider/muse.py` pins both `_TIER_TO_MODEL` entries to the full-price
  `muse-spark-1.3`, with a comment explaining that Contributor models must never receive
  a user's proprietary source without being told to. A test,
  `tests/llm_provider/test_model_advisories.py::test_tier_mapping_never_routes_to_an_advisory_model`,
  pins that.
- `docs/llms.md` states: "`small` is what `@small` and `@xsmall` reach for
  automatically, so mapping it to the Contributor model would silently ship a user's
  proprietary source into Meta's training corpus. SASE does not make that decision on
  anyone's behalf. ... reaching one requires typing its name."

This plan does not break that test — the guard is on the _tier mapping_, and this change
goes through the _alias pool_, which is a second, unguarded route to the same place. The
practical effect: **on any machine with a `muse` binary on `PATH`, roughly one in four
`@xsmall`/`@small`/`@medium` agent launches will send that agent's prompt, repository
contents, and tool output to a model that trains on them** — with no `%model` directive
and no config change by the user.

That is the user's explicit, informed request, and this plan implements it in full. The
implementer's job is therefore _not_ to soften the change, but to make sure the repo
stops claiming the opposite: every comment and doc paragraph asserting that automatic
routing cannot reach a Contributor model must be corrected in the same change.

Opting back out is a one-line config change and stays supported — set
`llm_provider.model_aliases.builtin.<size>` to a pool without the Muse member.

No feature flag is required. Per `sase/memory/sase_flags.md`, a flag covers a temporary
route or a deprecated branch that must stay reachable; this is a permanent shipped
default with an existing config override, which is the "config field, not a flag" case.

## Implementation

### 1. Edit the shipped defaults

`src/sase/llm_provider/model_alias_defaults.yml` is the single edit point for shipped
alias targets. Append the Muse member to the end of each of the three pools — appending
keeps the existing round-robin order stable for the members already there.

Target the three entries so they read (exact member strings; keep lines inside the
file's existing width by folding the double-quoted scalar the way the `xsmall` entry
already does):

- `xsmall.target`:
  `claude/claude-haiku-4-5@xhigh | codex/gpt-5.6-luna@xhigh | agy/gemini-3.8-flash-high | muse/muse-spark-1.3-contributor@medium`
- `small.target`:
  `claude/sonnet@high | codex/gpt-5.6-terra@high | grok/grok-4.6@low | muse/muse-spark-1.3-contributor@high`
- `medium.target`:
  `claude/sonnet@xhigh | codex/gpt-5.6-terra@xhigh | grok/grok-4.6@medium | muse/muse-spark-1.3-contributor@xhigh`

Leave `large` and `xlarge` alone. Leave every `description` alone — the descriptions are
size descriptions, not member lists.

Do **not** touch the commented example block in `src/sase/default_config.yml` (around
the `model_aliases:` comment). Those lines illustrate how a user _overrides_ a builtin
alias; they are not a copy of the shipped defaults and are not stale after this change.

### 2. Regenerate the docs table

The shipped-default table in `docs/llms.md` sits between
`<!-- BEGIN GENERATED: model-alias-defaults -->` and its END marker and is rendered from
the YAML by `tools/render_model_alias_docs`. Do not hand-edit it. Run:

```bash
just fmt-docs   # renders the generated block
just fmt-md     # prettier reflows the table
```

`tests/test_render_model_alias_docs.py::test_renderer_table_matches_runtime_loaded_defaults`
fails if the block drifts from the YAML, so this step is load-bearing.

### 3. Correct the Muse narrative in `docs/llms.md`

Two passages become false and must be rewritten in this change:

1. **The Model Mapping paragraph** (immediately after the Muse model table, beginning
   "**Both tiers map to `muse-spark-1.3` on purpose.**"). The tier mapping is still
   deliberately pinned to the paid model, so keep that sentence and its reason. Delete
   or rewrite the claim that "reaching one requires typing its name": it no longer
   holds. State plainly that the shipped `@xsmall`, `@small`, and `@medium` pools now
   include `muse-spark-1.3-contributor`, that an agent launched at those sizes can be
   routed to it automatically whenever a `muse` binary is present, and name the
   `llm_provider.model_aliases.builtin.<size>` override as the way to opt out.
2. **The Muse "Selection" section**, which opens "Muse is **explicit-only**." That is
   now wrong for the three small sizes. Add the same carve-out sentence the Grok
   "Selection" section already carries — "Separately, the shipped ... load-balanced
   pools can select Muse whenever a `muse` executable is available" — adapted to Muse,
   and note that for Muse that path lands on the Contributor model.

While in `docs/llms.md`, check the surrounding size-alias prose (the "Implicit role
aliases" section and the short pool mentions elsewhere in the file) for any other
sentence that asserts advisory-flagged models are unreachable by automatic routing.

### 4. Correct the Muse provider comment

`src/sase/llm_provider/muse.py`, the comment directly above `_TIER_TO_MODEL`. Keep
`_TIER_TO_MODEL` itself unchanged — the tier guard and its test stay exactly as they
are. Fix the comment's claim about what the size aliases can reach, and say why the two
routes now differ: the tier mapping is SASE's own default choice, while the alias pool
membership is a configured shipped default the user can override.

That comment is also currently garbled — the sentence starting "`small` is what compact
size aliases can reach for automatically, and" runs straight into the next sentence with
no completion. Repair it while rewriting.

### 5. Update the shipped-defaults tests

`tests/llm_provider/test_load_balanced_alias_defaults.py`:

- `test_packaged_defaults_select_correct_effort_per_provider` — add a `"muse/"` entry to
  the `expectations` dict for `@xsmall`, `@small`, and `@medium`, expecting
  `("muse/muse-spark-1.3-contributor", "medium" | "high" | "xhigh")` respectively. Leave
  `@large` and `@xlarge` untouched.
- `test_shipped_size_aliases_follow_the_effort_ladder` — needs **no** edit. It derives
  the expected rungs from the YAML itself and will independently prove the
  `xhigh → high → medium` descent. If it fails, the YAML rungs are wrong, not the test.
- Add one new test that pins the intent, so a later cost or privacy cleanup cannot
  silently drop the member or re-rung it. Assert that each of the three pools contains
  exactly one `muse/` member, that it is `muse-spark-1.3-contributor`, that the rungs
  are `xhigh`/`high`/`medium` for medium/small/xsmall, and that `@large` and `@xlarge`
  contain no `muse/` member at all. Give it a docstring recording that this is a
  deliberate, user-requested opt-in to Meta's training terms.

Tests that need **no** change, confirmed by reading them:

- `tests/_model_alias_defaults_fixture.py` and everything built on it. The frozen
  fixture is intentionally decoupled from shipped values; only a _graph-shape_ change
  (target ↔ fallback) would touch it, and this is not one.
- `test_only_shipped_xsmall_has_antigravity_member`, the `@large`/`@xlarge` rotation
  tests, `tests/llm_provider/test_model_alias_defaults.py`, and the soft-disable and
  usage-limit tests that look up the `grok` member of `@small` by provider.
- `tests/llm_provider/test_model_advisories.py` — all of it, including the tier guard.

### 6. Check what `sase doctor` now reports

`sase doctor -C llm.model_advisory` collects routes from `build_alias_views`, which
reports each pool's _currently selected_ member. After this change, that check will
start emitting a `WARN` finding —
`@medium -> muse/muse-spark-1.3-contributor: Meta uses this model's inputs and outputs...`
— on a stock install with no user config, whenever the rotation cursor happens to sit on
the Muse member.

Run `sase doctor -C llm.model_advisory -v` after the edit and confirm the finding
appears and reads correctly. Do not suppress it: that warning is now the only consent
surface a user gets, and it should fire.

Do not change the doctor check in this plan (see Follow-ups).

## Verification

```bash
just install          # workspace venvs go stale; cheap insurance
just fix              # fmt-py, fmt-docs, fmt-md, keep-sorted — before any check
sase tool run check
```

`just check` is the whole recipe — do not run `just check-full`
([[decisions/check-full-is-explicit]]). Targeted runs while iterating:

```bash
.venv/bin/pytest tests/llm_provider/test_load_balanced_alias_defaults.py \
  tests/llm_provider/test_model_alias_defaults.py \
  tests/llm_provider/test_model_advisories.py \
  tests/test_render_model_alias_docs.py
```

Manual confirmation that routing actually reaches the new member:

```bash
sase doctor -C llm.model_advisory -v
```

## Done When

- `model_alias_defaults.yml` carries the Muse member in `xsmall`, `small`, and `medium`
  at `medium`, `high`, and `xhigh`, and nowhere else.
- The generated `docs/llms.md` table shows all three new members and the renderer parity
  test passes.
- No comment or doc sentence in the repo still claims automatic size-alias routing
  cannot reach a Contributor model.
- The new intent test and the existing ladder test both pass.
- `sase tool run check` is green.

## Follow-ups (file with `/sase_new_task`, do not implement here)

- **The advisory doctor check only sees the selected pool member.** `_resolved_routes`
  in `src/sase/doctor/checks_providers_advisory.py` reports one member per alias, so
  after this change `sase doctor` warns about the Contributor model only when the
  round-robin cursor happens to be parked on it. For a route that trains on user data,
  the warning should be deterministic: the check should scan every member of a pool, not
  just the selected one. This is a real gap that this change makes material, but
  widening the doctor check is beyond the requested scope.
- **The tier-mapping advisory guard has no alias-pool counterpart.**
  `test_tier_mapping_never_routes_to_an_advisory_model` guards one of the two automatic
  routes to an advisory model. Whatever the project decides the alias-pool rule should
  be after this change — an allowlist, a required acknowledgement, or nothing — it is
  currently unwritten, and the asymmetry should be recorded deliberately rather than
  left as an accident.
