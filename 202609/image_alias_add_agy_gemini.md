---
tier: tale
title: Add agy/gemini-3.8-flash-high to the @image model alias pool
goal:
  The sase-research-artifacts @image alias round-robins across Codex, Grok, and
  agy/gemini-3.8-flash-high (no effort suffix on the agy member), with tests and docs
  updated and `sase tool run check` passing.
size: small
proposed_by: bbugyi200.athena.0qm
create_time: 2026-09-24 09:20:57
status: wip
---

# Add `agy/gemini-3.8-flash-high` to the `@image` model alias pool

## Goal

Add the Antigravity model `agy/gemini-3.8-flash-high` as a third equal-weight member of
the `@image` model alias pool that the `sase-research-artifacts` plugin ships. `@image`
is the model the opt-in `#research_swarm(image=true)` infographic segment launches with.

## Where the work happens

All changes are in the linked `sase-research-artifacts` repo, not in the sase repo. Open
it with `sase repo open sase-research-artifacts -r "<reason>"` and work only in the path
it prints. Read that repo's `AGENTS.md` before editing. Commit the repo through the
normal `/sase_final` repository obligation.

## Current state

`src/sase_research_artifacts/default_config.yml` defines the alias:

```yaml
llm_provider:
  model_aliases:
    custom:
      image:
        model: "codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh"
        description: "Research-swarm infographic and image-generation agent."
        bucket: researchers
```

`tests/test_default_config.py` asserts that exact `model` string.

No user or project config layer overrides `image` today (`sase config show` resolves it
to the plugin default), so the plugin default is what takes effect.

## Key constraint: no `@effort` suffix on the `agy` member

The `agy` (Antigravity) provider has no reasoning-effort setting. Effort is part of the
model name instead (`-high` / `-medium` / `-low`). An `@effort` suffix on an alias
member counts as an explicit effort, and `agy` raises `LLMInvocationError` for it when
the model is invoked. The sase decision record `decisions:size-alias-effort-ladder`
covers this, and this plugin's own `docs/xprompts.md` says the same about the
`gemini_model` default. So the new member must be exactly `agy/gemini-3.8-flash-high`
with no `@xhigh` or `@high`, even though the other two members carry `@xhigh`. This
matches the existing `gemini_model` default in `research_swarm.md`
(`agy/gemini-3.8-flash-high`).

## Changes

1. **`src/sase_research_artifacts/default_config.yml`**: change the `image` alias
   `model` to:

   ```yaml
   model: "codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh | agy/gemini-3.8-flash-high"
   ```

   Keep it a plain `|` pool with no weights. The pool stays equal-weight round-robin, so
   each of the three members now gets about one third of the image launches. Leave
   `description` and `bucket` as they are.

2. **`tests/test_default_config.py`**: update the `custom["image"]["model"]` assertion
   in `test_default_config_loads_expected_model_aliases_and_bucket` to the new
   three-member string. Also add an assertion that the pool contains the `agy` member
   and that the member has no `@` effort suffix. For example, split on `|`, strip each
   member, and assert `"agy/gemini-3.8-flash-high"` is one of the members. This keeps a
   later edit from "normalizing" it to `@xhigh` and breaking launches at invoke time.
   Keep the existing `jsonschema` validation test as is. It already covers schema
   validity of the changed file.

3. **Docs**: in `docs/configuration.md`'s "Default Config" section, add a short line to
   the `image` bullet that lists the pool's three members (Codex, Grok, and Antigravity
   Gemini, equal weight). Say that the `agy` member has no `@effort` suffix because
   Antigravity puts effort in the model name. Check `README.md`'s "Defaults" section and
   `AGENTS.md`. Update them only if they list pool members, which they do not today, so
   they should stay unchanged. Do **not** hand-edit `CHANGELOG.md`, because
   release-please manages it.

## Out of scope

- The `#research_swarm` xprompt (`research_swarm.md`). The image segment already
  launches `%model:@image`, so it picks up the new member automatically.
- The `gemini` researcher role, other aliases, and the `researchers` bucket.
- Any change in the sase repo or `sase-core`.

## Verification

In the opened `sase-research-artifacts` checkout:

1. Run `sase tool run check` (lint and tests). Do not run bare `just check`, because it
   is guarded. It must pass.
2. Optional sanity check: if the host's installed plugin is an editable install of this
   checkout, `sase config show | grep -A4 '^      image:'` should show the new
   three-member `model` string. If it is not editable, skip this step and do not
   reinstall anything.

## Commit

Use one commit in `sase-research-artifacts`, for example
`feat(config): add agy/gemini-3.8-flash-high to @image pool`.
