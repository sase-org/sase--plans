---
tier: tale
title: Rename the research_a/research_b model aliases to sol_or_grok/opus_or_grok
goal:
  The research-swarm model aliases are named after the models they route to —
  `sol_or_grok` and `opus_or_grok` — in the sase-research-artifacts plugin default
  config, the `#research_swarm` xprompt, its tests and docs, and Bryan's chezmoi-managed
  global sase config, with no surviving `research_a` / `research_b` references.
size: small
proposed_by: bbugyi200.athena.02e
create_time: 2026-09-07 10:23:20
status: wip
---

# Plan: Rename the research model aliases

## Goal

Rename two custom model aliases:

| Old          | New            | Role                                 |
| ------------ | -------------- | ------------------------------------ |
| `research_a` | `sol_or_grok`  | Primary independent swarm researcher |
| `research_b` | `opus_or_grok` | Second-opinion swarm researcher      |

The alias `image` is **not** renamed. The `researchers` bucket, the `research` tribe,
the `cdx`/`cld` agent ids, and the "researcher A"/"researcher B" prompt text all stay as
they are — they describe swarm _roles_, and only the _model-target_ names change.

## Repositories

Two repos change. Open each with `/sase_repo` before touching it and use only the path
that command prints:

```bash
sase repo open sase-research-artifacts -r "Rename research_a/research_b model aliases"
sase repo open chezmoi -r "Rename research_a/research_b in the global sase model aliases"
```

The **sase repo itself does not change.** It has ~25 test files that mention
`research_a` / `research_b`, but every one of them is a synthetic alias-name fixture
(models panel rows, alias-history groups, config-schema samples). None loads this
plugin's `default_config.yml`, so they are decoupled from this rename. Leave them alone;
renaming them would be pure churn.

## Step 1 — sase-research-artifacts: default config

`src/sase_research_artifacts/default_config.yml`

Rename the two `llm_provider.model_aliases.custom` keys and add the grok last-resort
candidate that the new names promise:

```yaml
llm_provider:
  model_aliases:
    custom:
      sol_or_grok:
        model: codex/gpt-5.6-sol || grok/grok-4.6
        description: "Primary independent research-swarm researcher."
        bucket: researchers
      opus_or_grok:
        model: claude/opus || grok/grok-4.6
        description: "Assisting (second-opinion) research-swarm agents."
        bucket: researchers
```

Keep both `description` values verbatim. The new names say which models the alias picks
but no longer say which swarm _seat_ it fills, so the descriptions are now the only
place that role survives — they must not be dropped or shortened.

`||` is the ordered provider-availability fallback operator (`|` is load balancing; the
two cannot be mixed unparenthesized). Leave effort unpinned here so the alias keeps
inheriting `llm_provider.default_effort`, matching the current default's style.

**This is the one behavior change in the plan, and it is separable.** A default named
`sol_or_grok` that can only ever reach sol is a name that lies to every user who does
not override it, and the sibling `image` alias already ships a grok candidate in its
default. If the reviewer wants a strictly mechanical rename instead, strike only the two
`model:` lines above and keep `codex/gpt-5.6-sol` / `claude/opus` unchanged; every other
step in this plan is unaffected.

## Step 2 — sase-research-artifacts: the swarm xprompt

`src/sase_research_artifacts/xprompts/research_swarm.md`

- Line 26: `%model:@research_a` → `%model:@sol_or_grok`
- Line 47: `%m:@research_b` → `%m:@opus_or_grok`

Do not reflow the surrounding Jinja. Both directives sit on a line that continues into
`{% if wait %}`, and the segment bodies below them (the "You are researcher A/B"
instructions and the `#research(suffix=a|b)` calls) are unchanged.

## Step 3 — sase-research-artifacts: tests

`tests/test_default_config.py` — update the four assertions in
`test_default_config_loads_expected_model_aliases_and_bucket` to the new keys and, if
step 1's `||` change is kept, the new model values. Then add retired-name guards next to
the existing `research_lead` guard, using the same split-string-literal style that file
already uses so a grep for the retired name does not hit the guard itself:

```python
assert "research" "_lead" not in custom
assert "research" "_a" not in custom
assert "research" "_b" not in custom
```

`tests/test_xprompt_loading.py` — line 149 `"%model:@research_a" in cdx` →
`"%model:@sol_or_grok"`, line 154 `"%m:@research_b" in cld` → `"%m:@opus_or_grok"`.

## Step 4 — sase-research-artifacts: docs

Four prose references, all a plain name swap:

- `AGENTS.md:30` — the `research_a`/`research_b`/`image` architecture bullet.
- `README.md:108` — the "Defaults" paragraph.
- `docs/configuration.md:81` — the "Default Config" bullet.
- `docs/xprompts.md:60`, `:63`, `:91` — the `cdx` / `cld` segment descriptions and the
  closing "Depends on ..." line.

Do **not** hand-edit `CHANGELOG.md`; release-please generates it. Do not hand-edit the
`version` in `pyproject.toml` or `.release-please-manifest.json` either. This is a
breaking change to a published config contract, so the landing commit should be a
conventional `feat!` commit carrying a `BREAKING CHANGE:` footer naming both old and new
alias keys, which is what drives release-please's next version.

## Step 5 — chezmoi: the global user override

`home/dot_config/sase/sase.yml`, lines 241 and 245 — rename only the two keys under
`llm_provider.model_aliases.custom`:

```yaml
sol_or_grok:
  model: "codex/gpt-5.6-sol@xhigh || grok/grok-4.6@xhigh"
  description: "Primary independent research-swarm researcher."
  bucket: researchers
opus_or_grok:
  model: "claude/opus@xhigh || grok/grok-4.6@xhigh"
  description: "Assisting (second-opinion) research-swarm agents."
  bucket: researchers
```

Leave the `model`, `description`, and `bucket` values exactly as they are — these
already carry the `@xhigh` pins and the `|| grok/grok-4.6@xhigh` fallback that gave the
new names their meaning. This block is _above_ the `xprompts:` section's
`# keep-sorted start` marker, so no sort order is enforced on these keys and they may
stay in their current order.

`~/.config/sase/sase.yml` is a chezmoi-managed regular file, not a symlink, and it is
currently byte-identical to the source in this region. Editing the chezmoi source does
**not** update the live file. Do not run `chezmoi apply` from the agent's workspace
clone — that clone is not the chezmoi source directory, so an apply there would write
the _unrenamed_ content back. Applying is a post-merge action for the host; call it out
in the final report (see "Landing note").

## Verification

Run these in the opened `sase-research-artifacts` checkout:

```bash
just install   # the plugin's own .venv; ephemeral workspace clones start without one
just check     # ruff + mypy + pytest
```

`just check` resolves its `sase` and `sase-core` dependencies from the surrounding
linked workspace. If it reports a missing local sase-core checkout, open it first with
`sase repo open sase-core -r "sase-core source needed by the plugin's just check"` and
rerun. `just test-wheel` is not required — this change touches no entry point, no
packaged resource path, and no wheel contents beyond file bodies already covered by the
default lane.

Then confirm the rename is complete. In the plugin checkout:

```bash
grep -rn "research_a\|research_b" . --exclude-dir=.git --exclude-dir=.venv
```

The only surviving hits must be the intentional split-literal guards added in step 3. In
the chezmoi checkout the same grep must return nothing.

No `just check` is needed in the sase repo, because no file in it changes. If a step
unexpectedly requires one, run `just check` there too.

## Decisions worth recording

**No feature flag.** Feature flags are a sase-project concern only, and neither landing
order breaks the swarm anyway. The plugin's own default config defines the new names, so
`%model:@sol_or_grok` resolves the moment the plugin lands, whether or not chezmoi has
been applied yet; and a not-yet-renamed `research_a` sitting in the user config is
merely an unused custom alias, which is inert. The worst transient outcome is that the
swarm runs without the `@xhigh` pin and the grok last resort for one window.

**No entry in `REMOVED_IMPLICIT_ALIAS_GUIDANCE`.** That sase-repo registry
(`src/sase/doctor/checks_config_common.py`) exists for retired _built-in_ sase aliases;
`research_a`/`research_b` are plugin-provided custom aliases, so an entry there would be
a category error. It is also unnecessary: no personal xprompt under `~/sase/xprompts/`,
`~/.config/sase/xprompts/`, or the project's `sase/xprompts/` references either alias,
and `sase doctor`'s `check_config_model_xprompts` already reports any unresolved
`%model:@<token>` with generic guidance.

**Alias history is not migrated.** Per-alias run history is derived from the agent
artifact index, which records the alias each past run actually launched under. Runs from
before this rename stay recorded as `research_a` / `research_b` and stop being reachable
from the models panel, which keys its rows off configured aliases. That is a faithful
record of history, not drift; no backfill.

## Landing note

The user's live `sase` runs an **editable** install of this plugin pointed at the
primary `sase-research-artifacts` clone, so the rename reaches the running CLI as soon
as the change is merged and that clone is updated — no reinstall needed. The chezmoi
half needs `chezmoi apply` after its commit merges. Say both of these explicitly in the
final report.
