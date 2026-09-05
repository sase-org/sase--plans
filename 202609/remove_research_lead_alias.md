---
tier: tale
size: small
title: Remove the research_lead alias; lead uses @xlarge
goal:
  "The #research_swarm lead launches directly through the built-in @xlarge alias, and
  the now-unused research_lead custom model alias is removed from the plugin defaults
  and the chezmoi-managed user config."
proposed_by: bbugyi200.athena.0gg
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0gg](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0gg.md)
- **COMMITS:**
  - [6ed8763](https://github.com/sase-org/sase-research-artifacts/commit/6ed87637a300ea4befb9f994661ac5dda0ba79ef)
    — feat(research): use xlarge for swarm lead

# Remove the `research_lead` alias; the research-swarm lead uses `@xlarge`

## Goal

Have the `#research_swarm` xprompt's lead/consolidator segment launch directly through
the built-in `@xlarge` model alias, and remove the now-unused `research_lead` custom
model alias everywhere it is defined: the `sase-research-artifacts` packaged defaults
and the chezmoi-managed user config.

## Context

- The `#research_swarm` xprompt lives in the `sase-research-artifacts` linked repo at
  `src/sase_research_artifacts/xprompts/research_swarm.md`. Its third segment launches
  the lead with `%m:@research_lead`; the other segments use `@research_a`,
  `@research_b`, and `@image` and do not change.
- The `research_lead` alias is defined in two places, and both definitions must go:
  1. The plugin's packaged default in `src/sase_research_artifacts/default_config.yml`,
     currently `model: "@xlarge"`.
  2. A user override in the `chezmoi` linked repo at `home/dot_config/sase/sase.yml`
     (deployed as `~/.config/sase/sase.yml`), currently
     `claude/opus@max || codex/gpt-5.6-sol@max || grok/grok-4.6@max`.
- Intended behavior change: today the lead resolves through the user override chain
  above; after this change it resolves through `@xlarge`, whose shipped default is
  `(claude/claude-fable-5@xhigh | codex/gpt-6-astra@xhigh) || grok/grok-4.6@xhigh` and
  which has no user-level `builtin.xlarge` override.
- The `research_a` / `research_b` / `image` aliases and the `researchers` bucket stay.
  Both the packaged and the chezmoi `researchers` bucket descriptions enumerate a
  "lead/consolidator" role and need that wording trimmed.
- Open each repo with `sase repo open sase-research-artifacts` /
  `sase repo open chezmoi` (the `/sase_repo` skill) before touching its files.
- No feature flag is needed: this is a config-data removal whose single caller (the
  xprompt) migrates in the same change, not a code branch that must stay reachable.
- Historical plan archives in the plans sidecar mention `@research_lead`; they are
  immutable history and must not be edited. The sase repo itself has no operational
  `research_lead` references.

## Implementation

In the `sase-research-artifacts` repo:

1. In `src/sase_research_artifacts/xprompts/research_swarm.md`, change the lead
   segment's `%m:@research_lead` to `%m:@xlarge`. Touch nothing else in the xprompt.
2. In `src/sase_research_artifacts/default_config.yml`, delete the `research_lead` entry
   from `llm_provider.model_aliases.custom` and reword the `researchers` bucket
   description so it no longer lists the lead/consolidator role.
3. In `tests/test_default_config.py`, replace the two `research_lead` assertions with
   `assert "research_lead" not in custom`, keeping the existing `research_a` /
   `research_b` / `image` and bucket coverage (the bucket-description assertion checks
   only the "infographic" substring and still holds).
4. In `tests/test_xprompt_loading.py`, assert the rendered `final` segment contains
   `%m:@xlarge` and no `research_lead` spelling, mirroring the existing per-segment
   model assertions for `@research_a` / `@research_b` / `@image`.
5. Update the docs that enumerate the alias: the `## Defaults` section of `README.md`,
   the `default_config.yml` bullet in `AGENTS.md`, the `## Default Config` bullets in
   `docs/configuration.md`, and in `docs/xprompts.md` both the `<clan>.final` segment
   description and the trailing "Depends on ..." alias list. Describe the lead as
   launching through the built-in `@xlarge` alias and drop `research_lead` from the
   alias enumerations.

In the `chezmoi` repo:

6. In `home/dot_config/sase/sase.yml`, delete the `research_lead` block under
   `llm_provider.model_aliases.custom` and trim the "lead/consolidator" wording from the
   `researchers` bucket description. Keep the `research_a` and `research_b` overrides
   untouched. The deployed `~/.config/sase/sase.yml` picks the removal up on the user's
   next `chezmoi apply`; until then the deployed alias is an unreferenced orphan and
   harmless, so do not run `chezmoi apply` yourself.

## Verification

1. From the `sase-research-artifacts` repo: `just install`, then
   `just test tests/test_default_config.py tests/test_xprompt_loading.py`, then
   `just check`.
2. `grep -rn research_lead` (excluding `.git`) over both edited repos returns no
   matches.
3. Confirm `home/dot_config/sase/sase.yml` still parses as valid YAML, for example with
   `python -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))"`.

## Acceptance Criteria

- The `#research_swarm` lead segment launches with `%m:@xlarge`; the other three
  segments' model directives are unchanged.
- No `research_lead` alias definition remains in the plugin's packaged config or in the
  chezmoi-managed `sase.yml`, while `research_a`, `research_b`, `image`, and the
  `researchers` bucket remain intact.
- Plugin docs describe the lead as `@xlarge` and no longer enumerate `research_lead`.
- The focused tests and the plugin repo's `just check` gate pass.
