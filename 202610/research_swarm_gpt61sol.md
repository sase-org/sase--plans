---
tier: tale
title: Switch research swarm Codex default to GPT-6.1 Sol
goal: Make
size: small
proposed_by: bbugyi200.athena.0ux
create_time: 2026-10-01 11:44:13
status: wip
---

# Switch research swarm Codex default to GPT-6.1 Sol

## Outcome

`#research_swarm`'s default `codex_model` becomes `codex/gpt-6.1-sol@xhigh`, and the
Codex member of the plugin's `@image` alias pool becomes the same model. Effort stays
`xhigh`. The pool stays an equal-weight round-robin (`|`), in the same order, with the
Grok and Antigravity members unchanged.

This is one `small` tale. The two settings have one source each, the call sites that
quote them are a closed list, and one agent can edit and verify them directly. No phase
needs its own plan.

## Where the settings live

Open the linked `sase-research-artifacts` repo with `sase repo open` and use only the
path that command prints. Read that checkout's `AGENTS.md` before editing. Do not name
an ephemeral workspace directory in new files.

Both requested values are packaged by that plugin:

- `src/sase_research_artifacts/xprompts/research_swarm.md` declares `codex_model` with
  default `codex/gpt-5.6-sol@xhigh`. The Codex segment renders `%m:{{ codex_model }}`.
  It does not hardcode a model id in the body.
- `src/sase_research_artifacts/default_config.yml` contributes the custom alias `image`
  through the `sase_config` entry point:

  `codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh | agy/gemini-3.8-flash-high`

  The image segment renders `%m:{{ image_model }}`, and `image_model` defaults to
  `@image`. Changing the pool is what changes the image agent's Codex candidate. Do not
  replace `@image` with a concrete model.

`gpt-6.1-sol` is already a Codex catalog model. Shipped `@large` and `@xlarge` selectors
already use `codex/gpt-6.1-sol@xhigh`, so this effort suffix is valid. Do not change the
catalog, short alias `gpt61sol`, or any other provider's model.

A fresh search of the plugin finds exactly twelve `gpt-5.6-sol` occurrences, and every
one is this default or a quote of it:

- `src/sase_research_artifacts/xprompts/research_swarm.md` (the input default)
- `src/sase_research_artifacts/default_config.yml` (the `@image` pool)
- `README.md` (the `codex_model` default in the xprompt summary)
- `docs/xprompts.md` (the input table and the `<clan>.cdx` description)
- `docs/configuration.md` (the pool prose)
- `tests/test_default_config.py` (the pool equality)
- `tests/test_xprompt_loading.py` (the input default, two expanded `%m:` assertions, the
  leak guard on the Claude and lead segments, and the authored image-segment guard)

Replace the token `gpt-5.6-sol` with `gpt-6.1-sol` at all twelve sites. Keep every
surrounding character, including `@xhigh` where it is already present and its absence on
the negative image-segment assertion (`%model:codex/gpt-5.6-sol`). That assertion checks
the unexpanded image segment, which must keep using `%m:{{ image_model }}` and must not
grow a hardcoded Codex model. The leak guard (`codex/gpt-5.6-sol@xhigh` not in the
Claude segment plus the lead) must move to the new id; leaving the old string would
still pass and would stop detecting a leak of the current default. `xp.inputs[9]`
remains the `codex_model` input. In `docs/xprompts.md`, shorten the padding after the
new default by one space so the table columns stay aligned. The new id is one character
longer.

After the edit, a search of the plugin repo for `gpt-5.6-sol` returns nothing, and a
search for `gpt-6.1-sol` returns only those twelve sites.

## Leave these alone

User config does not override either setting, so do not edit chezmoi:

- `home/dot_config/sase/sase.yml` defines custom aliases `sol_or_grok` and
  `opus_or_grok` only. It does not define `image`. Nested config merge keeps the plugin
  `image` key, so the packaged pool is the live `@image` alias.
- The same file's `rs` and `rsa` snippets invoke `#research_swarm` without
  `codex_model=`. No user or project xprompt shadows `research_swarm`, so the plugin
  input default is what those snippets launch.
- `sol_or_grok` still names `codex/gpt-5.6-sol@xhigh`. The current swarm does not select
  it. Plugin tests already assert `@sol_or_grok` is absent from expanded segments. Leave
  that alias on the old model.
- The `#codex` snippet (`gpt-5.6-sol`), `%model:#codex`, and
  `home/dot_codex/config.toml` (`model = "gpt-5.6-sol"`) are the Codex shorthand and the
  Codex CLI default. They are not `codex_model` and they are not the `@image` pool.

Also leave unchanged:

- Claude, Grok, Muse, Gemini, lead, and linker defaults, and `image_model`'s default
  `@image`.
- The Grok and Antigravity pool members, the single `|` operator, member order, and the
  Antigravity member's lack of an `@effort` suffix.
- Queue weights, `%q(1.5x, w=0.25)`, provider gating, and segment structure.
- `CHANGELOG.md`. release-please generates it from conventional commits at release time.
  There is no hand-maintained Unreleased section.
- `AGENTS.md`. It does not name the model.
- The wheel-contract test and `.github/workflows/publish.yml`. They count queue
  directives and do not pin this model.
- The host `sase` repo, `sase-core`, and every other linked repo. Their `gpt-5.6-sol`
  strings are unrelated fixtures and catalog tests.

## Verify

From the opened plugin checkout, run `sase tool run check`. That is the guarded
`just check` (ruff, mypy, and pytest, excluding the slow wheel-contract test). Do not
invoke bare `just check`. Do not run the wheel-contract test for this string change.

The check is enough when:

- `tests/test_default_config.py` still accepts the pool against the config schema, still
  sees no `@effort` on the `agy/` member, and still sees bucket `researchers`.
- `tests/test_xprompt_loading.py` still expands an omitted `codex_model` to
  `%m:codex/gpt-6.1-sol@xhigh` only on the Codex researcher, still routes a custom
  `codex_model` only to that role, and still expands `image=true` to `%m:@image`.
- The repo search described above is clean.

Do not edit host SASE tests to follow the plugin. None of them assert this default.
