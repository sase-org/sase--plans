---
tier: epic
title:
  Add an @image built-in model alias and route the research swarm image agent through it
goal: 'SASE ships a sixth built-in model alias, @image, defaulting to the round-robin
  pool "codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh", every surface that described the
  built-in set as five size aliases is correct, and the #research_swarm image segment
  launches on @image instead of a hardcoded Codex model.

  '
phases:
  - id: core_alias
    title: Core @image alias in the alias policy layer
    depends_on: []
    size: small
    description:
      "core_alias: add the image entry to the shipped model-alias defaults, name it in
      BUILTIN_MODEL_ALIAS_NAMES and the built-in display order, re-export the constant,
      then update the frozen defaults fixture and every test that hardcodes five
      built-in aliases, and add resolution, kind, override, completion, and doctor
      coverage for the new alias."
  - id: surfaces_and_docs
    title: Built-in alias surfaces, schema, and docs
    depends_on:
      - core_alias
    size: medium
    description:
      "surfaces_and_docs: rename the ACE Launch Control built-in section label and
      refresh its PNG goldens, correct the config schema and shipped default_config
      comments, regenerate the model-alias defaults table, and fix the doc prose that
      claims the built-in set is exactly five size aliases."
  - id: research_swarm_image
    title: Route the research swarm image segment through @image
    depends_on:
      - core_alias
    size: small
    description:
      "research_swarm_image: in the sase-research-artifacts linked repo, switch the
      #research_swarm image segment from a hardcoded Codex model to %model:@image, make
      the #research/image prompt provider-neutral, and update that repo's xprompt docs
      and loading test."
proposed_by: bbugyi200.athena.081
create_time: 2026-09-09 20:00:09
status: wip
---

- **PROMPT:**
  [prompts/202608/image_model_alias.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/image_model_alias.md)

# Add an `@image` Built-in Model Alias and Route the Research Swarm's Image Agent Through It

## Background

SASE ships a fixed set of built-in model aliases that always resolve, even when a user
has configured none. Today that set is exactly the five **size** aliases — `@xsmall`,
`@small`, `@medium`, `@large`, `@xlarge` — declared in
`src/sase/llm_provider/model_alias_defaults.yml` and named in
`BUILTIN_MODEL_ALIAS_NAMES` (`src/sase/llm_provider/model_alias_policy.py`).

Image generation is a capability, not a size. Until now only Codex could generate
images, so the `#research_swarm` xprompt swarm (in the `sase-research-artifacts` linked
repo) hardcoded `%model:codex/gpt-5.6-sol` on its final image segment. Grok can also
generate images, so that hardcoded provider should become a configurable alias.

This epic adds a sixth built-in alias, `@image`, with the shipped default:

```text
codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh
```

Note the single `|`: that is a **round-robin load-balanced pool**, so successive image
launches alternate between Codex and Grok (skipping any provider whose CLI is
unavailable). This is the value the project owner asked for; do not silently convert it
to a `||` ordered fallback.

`@image` is the first built-in alias that is not a size alias. That is the main source
of risk in this epic: several code comments, doc sentences, one ACE section label, and
several tests assert "exactly five built-in aliases" or enumerate the five size names.

### No feature flag

Per `sase/memory/sase_flags.md`, a feature flag is a _temporary_ route for behavior that
is not yet ready. `@image` is permanently-intended behavior that is already
user-overridable through `llm_provider.model_aliases.builtin.image`. Do **not** create a
feature flag for this work.

### No Rust core change

`crates/sase_core/src/model_route.rs` in the `sase-core` linked repo owns
`PUBLIC_SIZE_ALIASES`, which is tied to `PhaseSizeWire` (phase-size routing). `@image`
is not a size alias and is not reachable from size routing, so that constant must
**not** gain an `image` entry. The Rust xprompt LSP consumes the model-completion
catalog generically (`is_model_alias_kind` matches on `implicit_alias`/`user_alias`), so
it picks up `@image` with no change. Verified during planning; do not modify
`sase-core`.

### Verified facts that shape the work

- `_parse_model_alias_defaults()` requires the YAML alias key set to equal
  `BUILTIN_MODEL_ALIAS_NAMES` **exactly**. `tests/_model_alias_defaults_fixture.py`
  builds its own frozen YAML through that same parser, so adding `image` to the tuple
  without adding a frozen `image` entry makes every test that installs the frozen
  defaults blow up with a `RuntimeError`. Fix both together.
- `BUILTIN_MODEL_ALIAS_BUCKET_NAMES` is an empty frozenset (there are no built-in Launch
  Control buckets), so `@image` needs no bucket work.
- The ACE Models-panel PNG snapshot fixtures
  (`tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py`) build explicit
  `AliasView` lists by hand; they do **not** derive rows from
  `BUILTIN_MODEL_ALIAS_NAMES`. So adding `@image` alone changes no PNG. Renaming the
  panel's built-in section label does, because that string is rendered in ~50
  `models_panel_*` goldens.
- No `image` alias exists in the project owner's merged config today, so there is no
  builtin-vs-custom collision to migrate.
- `tools/render_model_alias_docs` regenerates the `model-alias-defaults` block in
  `docs/llms.md`. It runs from `just fmt-docs` (and therefore `just fmt`), and **no lint
  gate enforces it** — it must be run by hand.

### Discovered, deliberately out of scope

`sase-research-artifacts/src/sase_research_artifacts/default_config.yml` ships
`research_lead.model: "@smartest"`, and `@smartest` is a **retired** alias that no
longer resolves (`REMOVED_IMPLICIT_ALIAS_GUIDANCE` in
`src/sase/doctor/checks_config_common.py`). It is masked for the project owner because
their chezmoi config overrides all three `research_*` aliases. Do **not** fix it in this
epic. The `core_alias` phase records it as a `PROPOSED FOLLOW-UP:` note instead.

---

## Core `@image` alias in the alias policy layer

**Phase id:** `core_alias` · **Size:** small · **Depends on:** nothing

Add the `@image` built-in alias to the alias policy layer and make the whole existing
test suite green with six built-in aliases instead of five. This phase changes behavior;
the other two phases only change surfaces that describe it.

### Changes

1. `src/sase/llm_provider/model_alias_defaults.yml`
   - Add a sixth entry, last in the mapping (after `xlarge`):

     ```yaml
     image:
       target: "codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh"
       description: "Image-generation launch alias for agents that must produce images."
     ```

   - Update the file's header comment: it currently says this file is "the single edit
     point for changing what @xsmall, @small, @medium, @large, and @xlarge resolve to"
     and that user config "can override only these built-in names". Reword so it covers
     the five size aliases **plus** `@image`.

2. `src/sase/llm_provider/model_alias_policy.py`
   - Add `IMAGE_MODEL_ALIAS_NAME = "image"` alongside the public size-alias name
     constants, with a short comment noting it is a capability alias, not a size alias.
   - Append `IMAGE_MODEL_ALIAS_NAME` to `BUILTIN_MODEL_ALIAS_NAMES` (last — YAML-entry
     order is the contract this tuple documents).
   - Update the module comment that reads "SASE ships exactly five implicit built-in
     aliases: `xsmall`, `small`, `medium`, `large`, and `xlarge`."

3. `src/sase/llm_provider/alias_view.py`
   - Append `IMAGE_MODEL_ALIAS_NAME` to `_ROLE_ALIAS_ORDER` so `@image` sorts last
     within the built-in block deterministically rather than falling through the
     `except ValueError` branch.
   - Fix the docstrings that say built-in rows are "the five built-in size aliases"
     (`_sort_key`, `build_alias_views`) and the `_ROLE_ALIAS_ORDER` comment.

4. `src/sase/llm_provider/config.py`
   - Re-export `IMAGE_MODEL_ALIAS_NAME as IMAGE_MODEL_ALIAS_NAME` in the existing
     `model_alias_policy` re-export block, keeping its alphabetical position.

### Test updates (all of these currently hardcode five)

- `tests/_model_alias_defaults_fixture.py` — **mandatory**. Add a frozen `image` entry
  to `_FROZEN_ALIASES` (import `IMAGE_MODEL_ALIAS_NAME`). Follow the file's own
  convention: the frozen target is deliberately _different_ from the shipped one, e.g.
  `"codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh"` may be simplified, with description
  `"Frozen test description for image."`. Keep it a `target` (not a `fallback`) so the
  frozen alias-graph shape matches the shipped shape.
- `tests/llm_provider/test_model_alias_defaults.py` — extend
  `_DECLARED_ROLE_ALIAS_NAMES`.
- `tests/llm_provider/test_config_role_aliases.py` — the
  `set(targets) == set(BUILTIN_MODEL_ALIAS_NAMES)` assertion and its loop pick the new
  name up automatically; confirm and adjust any literal five-name list.
- `tests/llm_provider/test_alias_view.py` — the loop over
  `("xsmall", "small", "medium", "large", "xlarge")` and the `names[:5] == [...]` prefix
  assertion both need `image` appended (`names[:6]`).
- `tests/llm_provider/test_config_alias_resolution.py` — the
  `_special_model_alias_names() == {"xsmall", ...}` set assertion needs `"image"`.

Run `rg -n 'xsmall.*small.*medium.*large.*xlarge' tests/ src/` before finishing to catch
any enumeration missed above; ignore hits that are about the **phase/task size scale**
(`tests/test_phase_size_presentation.py`, `tests/test_plan_validate.py`,
`tests/test_bead/test_db_migrations.py`, `tests/test_notification_toasts.py`) — those
are sizes, not aliases, and must not change.

### New tests

Add coverage that pins the new behavior (place them beside the files above rather than
inventing a new module unless one is clearly warranted):

- `@image` resolves through the normal alias path to a round-robin selector whose
  members are `codex/gpt-5.6-sol` and `grok/grok-4.6`, both at `xhigh` effort, and whose
  selector mode is round-robin (not ordered fallback). Drive this off the shipped
  `implicit_alias_targets()` value, not a duplicated literal, where the surrounding
  tests already do so.
- `model_alias_kind("image")` is `"role"` (built-in), not `"user"` — this is what keeps
  it out of the ACE "Your aliases" section and out of custom-alias doctor checks.
- `llm_provider.model_aliases.builtin.image` overrides it, and
  `model_alias_config_source("image")` reports `"builtin"` when configured there.
- `@image` appears in the `%model` completion catalog as an implicit alias.
- `sase doctor -C config.model_aliases` reports **no** problem for a config that sets
  `model_aliases.builtin.image`, and still reports the "is a builtin alias; move it"
  problem for `model_aliases.custom.image` (the existing
  `model_alias_kind(alias) != "user"` branch in
  `src/sase/doctor/checks_config_model_aliases.py` now fires for `image`). Add this to
  `tests/doctor/test_checks_config_model_aliases.py`.

### Verification

- `just install` first (workspace venvs go stale).
- `just check` inline. If it runs long, hand it to `/sase_monitor` with a `--next`
  action.
- Because this phase changes the built-in alias contract, also run `just check-full`
  through `/sase_monitor` (never inline) before reporting the phase complete.
- `just test-visual` is **not** needed in this phase — no rendered string changes here.

### Follow-up note (do not fix here)

Record on this phase's bead, verbatim as a note starting with `PROPOSED FOLLOW-UP:`:
that `sase-research-artifacts`'s shipped `default_config.yml` sets
`research_lead.model: "@smartest"`, a retired alias that no longer resolves, and that
`research_a`/`research_b` ship single-provider targets where the project owner's own
config uses `||` fallbacks. Do not create a bead and do not edit that file in this epic.

---

## Built-in alias surfaces, schema, and docs

**Phase id:** `surfaces_and_docs` · **Size:** medium · **Depends on:** `core_alias`

Make every user-visible surface that describes the built-in alias set correct now that
it contains a non-size alias. No behavior changes.

### ACE Launch Control

- `src/sase/ace/tui/modals/models_panel_rendering_layout.py`: rename
  `_BUILTIN_SECTION_LABEL` from `"Built-in size aliases"` to `"Built-in aliases"`. It is
  no longer a size-only section.
- Regenerate the affected PNG goldens:
  `just test-visual --sase-update-visual-snapshots`, then re-run `just test-visual`
  clean. Roughly 50 `models_panel_*_*.png` goldens carry that label. **Review the diff**
  before accepting: the only pixel change should be the section header text. If any
  other golden changes, stop and investigate rather than blanket-accepting. Failure
  artifacts land in `.pytest_cache/sase-visual/`.
- `src/sase/xprompt/model_completion.py`: fix the `_IMPLICIT_ALIASES` comment ("Built-in
  size aliases surfaced as `%model` completions") and the
  `build_model_completion_catalog` docstring sentence "The five implicit size aliases
  and user-configured aliases are inserted…". No logic change — `_IMPLICIT_ALIASES`
  already derives from `BUILTIN_MODEL_ALIAS_NAMES`.

### Config schema and shipped default config

- `src/sase/config/sase.schema.json`: update the `model_aliases` description ("builtin
  contains overrides for SASE's five size aliases") and the `model_aliases.builtin`
  description ("Overrides for SASE's built-in size aliases: xsmall, small, medium,
  large, and xlarge") to cover `image` too. Keep the rest of the grammar prose intact.
- `src/sase/default_config.yml`: fix the comment "SASE ships exactly five built-in
  aliases: @xsmall, @small, @medium, @large, and @xlarge" (it must now list `@image` and
  say the size aliases carry size routing while `@image` does not), and add an `image:`
  line to the commented `model_aliases.builtin:` grammar example. Leave the "Scalar
  launch defaults … may point at one of the five built-in size aliases" comment alone —
  that sentence is about size aliases and stays true.

### Docs

Regenerate the table first: `just fmt` (which runs `fmt-py`, then `fmt-docs` —
`tools/render_model_alias_docs` — then prettier `fmt-md`). Confirm the
`model-alias-defaults` block in `docs/llms.md` grew an `@image` row. No lint gate checks
this, so it is on you to run it.

Then correct the surrounding prose. Correct **only statements that became false**; leave
statements specifically about the five _size_ aliases (size→route tables, phase/tale
routing) intact.

- `docs/llms.md`
  - ~L1101: "Use `llm_provider.model_aliases.builtin` only to override the five size
    aliases" → the built-in aliases (five size aliases plus `@image`).
  - ~L1207-1210: "the compact five-size-alias contract" and "alongside the five size
    aliases and any custom aliases".
  - `#### Implicit role aliases` (~L1228): the fixed set is now six; add `@image` and
    one sentence explaining it is a capability alias for image-generating agents, not
    part of size routing. **Do not rename the heading** — `docs/configuration.md` links
    to its `#implicit-role-aliases` anchor.
  - ~L1286 source note: "shipped size-alias defaults" → shipped built-in alias defaults.
  - ~L1292: "A prompt can override the five size aliases".
  - ~L1657: "Overrides are independent per-alias for the five size aliases".
- `docs/ace.md`
  - ~L2725 and ~L2753: the section is now named **Built-in aliases**.
  - ~L2755: "**Built-in size aliases** always lists exactly five rows in size order —
    `@xsmall`, `@small`, `@medium`, `@large`, `@xlarge`" → six rows: the five size
    aliases in size order followed by `@image`.
  - ~L2799: the misplaced-alias sentence names the section by label.
- `docs/configuration.md`
  - ~L1330: the `%model:`/`%m:` value-menu sentence lists "the five built-in size
    aliases (`@xsmall`…)".
  - ~L1569 table row for `llm_provider.model_aliases.builtin`: "Overrides for the five
    built-in size aliases (`xsmall`, …)".
  - ~L1580 and the "SASE ships a fixed set of **built-in size aliases**" paragraph.
  - ~L1603 link text "[Built-in size aliases](llms.md#implicit-role-aliases)" — keep the
    anchor, adjust the text.
- `docs/xprompt.md` ~L1614 ("The size-specific worker routes use the five built-in size
  aliases directly") — **leave unchanged**; it is about size routing and is still true.
- `src/sase/doctor/checks_config_common.py` L9 comment ("the compact five-size-alias
  contract") — cosmetic; update it while you are here.

Do **not** touch `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, or `QWEN.md`. In particular
`src/sase/main/init_memory/templates/memory-sase-sizes.template.md` generates
`sase/memory/sase_sizes.md`; its line "Default model aliases are `@xsmall`, `@small`,
`@medium`, `@large`, and `@xlarge`" is arguably now incomplete, but editing it is a
memory change that requires explicit permission from the project owner in a live
conversation. Instead, record a `PROPOSED FOLLOW-UP:` note on this phase's bead
suggesting that sentence be scoped to "size model aliases".

### Verification

- `just install`, then `just fmt`, then `just check`.
- `just test-visual` clean after the snapshot refresh.
- `just check-full` through `/sase_monitor` before reporting complete.
- Sanity-check the rendered panel by eye if convenient (`/run`-style launch of ACE and
  `,m`), but the PNG goldens are the authority.

---

## Route the research swarm image segment through `@image`

**Phase id:** `research_swarm_image` · **Size:** small · **Depends on:** `core_alias`

Route the `#research_swarm` image agent through `@image` instead of a hardcoded Codex
model. This work is in the **`sase-research-artifacts` linked repo**, not in the sase
repo.

Open it first — this is mandatory, and the path it prints is the only path to read or
write:

```bash
sase repo open sase-research-artifacts -r "Route the #research_swarm image segment through the new @image built-in alias"
```

That repo's `Justfile` installs the **local** sase checkout into its own `.venv`
(`linked_workspace_sase_source` resolves to the sase workspace root), so `@image` from
the `core_alias` phase is visible to its tests without any extra coordination.

### Changes

1. `src/sase_research_artifacts/xprompts/research_swarm.md` — final segment only:

   ```text
   %id(image, clan=research.{@1}) %wait(priority=20) %model:codex/gpt-5.6-sol
   ```

   becomes

   ```text
   %id(image, clan=research.{@1}) %wait(priority=20) %model:@image
   ```

   Change nothing else in that segment (`%wait:research.{@1}.final`,
   `#fork:research.{@1}.final`, `#research/image` all stay).

2. `src/sase_research_artifacts/xprompts/research_image.md` — the prompt body currently
   says "Can you use **GPT image** to generate an infographic…". With `@image`
   round-robining onto Grok, that instruction is wrong half the time. Rewrite it to be
   provider-neutral — ask the agent to use its available image-generation tool — while
   keeping the rest (infographic illustrating the markdown file's main points, written
   to a new file in the same directory) identical. Keep the frontmatter `name` and
   `description` intact except for removing the GPT-specific wording from `description`.

3. `docs/xprompts.md` in that repo:
   - The `#research/image` section says "Generates an infographic (via GPT image)" —
     make it provider-neutral.
   - The `#research_swarm` segment-4 bullet should state that the image segment runs on
     the `@image` built-in alias.
   - The closing "Depends on the `research_a` / `research_b` / `research_lead` model
     aliases and the `researchers` bucket from this plugin's default config" line should
     also note the dependency on sase's built-in `@image` alias.

4. `tests/test_xprompt_loading.py` — that file already asserts
   `"%model:@research_a" in cdx`. Add the matching assertion that the image segment
   carries `%model:@image` (and no longer `%model:codex/gpt-5.6-sol`).

### Do not do

- Do not edit that repo's `AGENTS.md` or `CLAUDE.md`.
- Do not edit `src/sase_research_artifacts/default_config.yml` — `@image` is a sase
  built-in, so the plugin must not redefine it as a custom alias (that would trip the
  "is a builtin alias; move it" doctor check).
- Do not bump the `sase>=0.17.0` floor in `pyproject.toml`. That floor is already ahead
  of what is on PyPI and carries a comment explaining why; guessing a new number here
  would be worse than the existing note. Instead, record a `PROPOSED FOLLOW-UP:` note on
  this phase's bead: the plugin now depends on a sase release that ships `@image`, so
  the floor and its comment should be updated when the sase release containing `@image`
  is published, and the plugin release must not go out ahead of it.

### Verification

- `just install` then `just check` **inside the `sase-research-artifacts` checkout**
  (that repo's `check` is `lint` + `test`; its Justfile bootstraps its own venv against
  the local sase source).
- Also confirm the sase-side view: from the sase workspace,
  `sase xprompt show research_swarm` should render `%model:@image` on the last segment,
  and `sase xprompt explain` on a `#research_swarm(...)` invocation should resolve that
  alias to the codex/grok pool.

### Commits

This phase produces a commit in the linked repo, not in the sase repo. Run
`/sase_git_commit` from inside the `sase-research-artifacts` checkout so the stitch is
attributed to that repository.

---

## Cross-phase notes

- `core_alias` must land before `research_swarm_image` reaches users: an unresolvable
  `%model:@image` would break the swarm's image agent at launch. `surfaces_and_docs` and
  `research_swarm_image` are independent of each other and may run in parallel once
  `core_alias` is done.
- Every phase runs `just install` before anything else — SASE workspace directories are
  ephemeral clones with their own virtualenvs that may be stale.
- No phase may create git commits, branches, or PRs directly; use `/sase_git_commit`.
