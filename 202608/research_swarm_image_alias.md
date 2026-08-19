---
tier: tale
title: Route the research swarm image agent through @image
goal:
  Ship a plugin `@image` model alias as the Codex/Grok xhigh round-robin pool and point
  the `#research_swarm` image segment at it so infographic launches no longer hardcode
  Codex.
size: small
proposed_by: bbugyi200.athena.082
create_time: 2026-08-19 13:36:46
status: wip
---

# Plan: Route the research swarm image agent through @image

## Context

`#research_swarm` is a four-segment xprompt swarm shipped by the
`sase-research-artifacts` plugin (linked repo of the same name). The first three
segments already route through plugin-contributed custom aliases:

- `<clan>.cdx` uses `%model:@research_a`
- `<clan>.cld` uses `%m:@research_b`
- `<clan>.final` uses `%m:@research_lead`

The fourth segment, `<clan>.image`, still hardcodes a single concrete model:

```text
%id(image, clan=research.{@1}) %wait(priority=20) %model:codex/gpt-5.6-sol
%wait:research.{@1}.final #fork:research.{@1}.final #research/image
```

Those aliases live in the plugin's `default_config.yml` under
`llm_provider.model_aliases.custom`, in the `researchers` bucket. There is no `@image`
alias anywhere today.

`%model` cannot take a selector. SASE accepts `A | B` round-robin pools and `A || B`
ordered fallbacks only on alias (and scalar launch-default) targets, not inside a
`%model` / `%m` directive. The new pool therefore **must** be an alias; do not write
`%model:codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh` on the swarm segment.

`grok/grok-4.6` is a valid provider-qualified model. Grok is never auto-detected, but a
pool member with an explicit `grok/` prefix is selected when the Grok CLI is installed
and not temporarily disabled. `xhigh` is a legal effort for `grok-4.6` (`low` / `medium`
/ `high` / `xhigh` only). Round-robin `|` skips members whose provider CLI is
unavailable, so a machine without Grok still launches Codex.

`#research/image` currently says "use GPT image". That instruction is Codex-specific and
will mis-steer the Grok pool member. The swarm still pins the model on the image
_segment_ (same pattern as `#research`, which has no `%model` of its own); do not add
`%model:@image` to the `#research/image` xprompt.

## Scope

All file changes belong in the `sase-research-artifacts` linked repo. Open it with
`sase repo open sase-research-artifacts` and a non-empty `-r` reason, then use only the
path that command prints. Do not edit the sase primary repo, sase-core, or any SASE
memory file.

This is not a feature flag. Model aliases are config, and the swarm already has a
user-reaching image agent; this change retargets that existing launch.

## Implementation

### 1. Define `@image` next to the other research-swarm aliases

In `src/sase_research_artifacts/default_config.yml`, add a custom alias named `image`
(bare key; the `@` appears only at `%model` use sites):

```yaml
image:
  model: "codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh"
  description: "Research-swarm infographic and image-generation agent."
  bucket: researchers
```

Quote the `model` value. `|` is YAML-significant; the sase default-config comments quote
pool strings for the same reason. Keep `|` (equal-weight round-robin), not `||` (ordered
fallback), and do not add weights.

Keep `bucket: researchers`. Update that bucket's description so it names the infographic
role as well as the three researcher roles. Stay under the schema's 160-character
description cap (alias and bucket).

Do not add `@image` to sase's own `default_config.yml`. SASE ships only the five builtin
size aliases; this plugin is already how research-swarm custom aliases are distributed.
User/project config still overrides the plugin layer through normal merge precedence.

### 2. Point the swarm image segment at the alias

In `src/sase_research_artifacts/xprompts/research_swarm.md`, change only the image
segment's model directive from `%model:codex/gpt-5.6-sol` to `%model:@image`. Keep
`%model` (not `%m`) on that segment, matching the current spelling. Leave `%id`, waits,
`#fork:research.{@1}.final`, and `#research/image` unchanged. The explicit `%model`
continues to win over any model the fork would otherwise inherit from the lead.

### 3. Make `#research/image` provider-neutral

In `src/sase_research_artifacts/xprompts/research_image.md`, drop the "GPT image"
wording. Keep the same job: generate an infographic of the research markdown file's main
points and write it next to the source file. Do not name Codex, GPT, Grok, or Imagine;
each pool member should use its own image tool.

Do **not** add `%model:@image` to this xprompt. Standalone `#research/image` keeps
today's behavior (caller model or `llm_provider.default_model`).

### 4. Tests

Update `tests/test_default_config.py`:

- Assert `custom["image"]["model"]` is exactly
  `codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh`
- Assert `bucket == "researchers"`
- Assert a non-empty description
- Assert the `researchers` bucket description still exists (and, if you pin its text,
  that it mentions the infographic/image role)
- Existing schema-validation test should keep passing against the merged plugin config

Update `tests/test_xprompt_loading.py` so the image segment is pinned the same way the
researcher segments already are:

- In `test_research_swarm_dependency_graph_preserved`, assert `%model:@image` and refute
  `%model:codex/gpt-5.6-sol`
- In the wait-argument expansion tests, assert the image segment still has
  `%model:@image` and does not pick up the optional researcher `wait`

Do not add sase-primary-repo tests. Host fixtures that mention `research_a` /
`research_b` are unrelated.

### 5. Docs in the plugin repo

Bring every shipped mention of the three research aliases and the image segment up to
date:

- `docs/xprompts.md` — document that `<clan>.image` uses `@image`
- `docs/configuration.md` — list `image` with `research_a` / `research_b` /
  `research_lead`
- `README.md` Defaults section — same
- `AGENTS.md` architecture bullet for `default_config.yml` — same
- `docs/xprompts.md` `#research/image` blurb — stop saying "via GPT image" if that
  remains after the prompt edit

Do not hand-edit `CHANGELOG.md`; release-please owns it.

## Out of scope

- Changing `@research_a`, `@research_b`, or `@research_lead`
- Putting `@image` in sase builtins or sase's commented alias examples
- Pinning `%model:@image` on `#research/image` itself
- Weights, `||` fallback, extra pool members, or a new Launch Control bucket
- Feature flags, Rust core, or sase-primary-repo edits

## Verification

From the opened `sase-research-artifacts` checkout:

1. `just install`
2. `just check` (plugin lint + default pytest lane; the slow `just test-wheel` job is
   not required — the wheel contract only asserts that some `llm_provider` plugin config
   exists)
3. Confirm the swarm image segment expands to `%model:@image` and the packaged config
   still validates against sase's public schema

Do not run `just check` in the sase primary repo unless that tree was actually edited
(it should not be).

## Acceptance

- Plugin default config ships `image` as `codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh`
  in the `researchers` bucket, with a quoted YAML string and a description.
- `#research_swarm`'s image segment launches with `%model:@image` and no hardcoded
  `codex/gpt-5.6-sol`.
- `#research/image` no longer names GPT/Codex as the image backend.
- Plugin tests and docs name `@image` alongside the other research-swarm aliases.
- No sase primary-repo, sase-core, memory, or changelog edits.
