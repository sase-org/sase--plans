---
tier: tale
title: Move the research tribe config from bundled defaults to chezmoi
goal:
  The bundled ace.tribes defaults configure only tribes that SASE source itself assigns, and the user-owned research
  tribe is configured in the chezmoi sase.yml overlay with its current icon, color, and a description.
create_time: 2026-07-31 08:37:51
status: done
---

- **PROMPT:** [prompts/202607/research_tribe_to_chezmoi.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/research_tribe_to_chezmoi.md)
- **AGENTS:**
  - [bbugyi200.athena.q1.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.q1.f0.md)
- **COMMITS:**
  - [f7239d4](https://github.com/bbugyi200/dotfiles/commit/f7239d409c3a4fc2819765ddbeb64a3ab3c441ab) — feat(sase): configure the research agent tribe display

# Plan: Relocate `ace.tribes.research` To The Chezmoi User Config

## Goal

`ace.tribes` in `src/sase/default_config.yml` should configure only tribes that **SASE's own source assigns**. The
`research` tribe is assigned exclusively by Bryan's personal xprompts in his chezmoi repo, so its display config belongs
in `~/.config/sase/sase.yml` (chezmoi source: `home/dot_config/sase/sase.yml`), not in the shipped defaults. Nothing
about how `@research` looks on Bryan's machine may change.

## Current State

### The six bundled tribes and who assigns them

`src/sase/default_config.yml:105-128` configures `default`, `epic`, `research`, `chop`, `pinned`, `review`. Of these:

- `default` — the reserved untagged identity; `RESERVED_DEFAULT_TRIBE` in `src/sase/core/agent_tribe.py`.
- `epic` — assigned by sase's epic phase-worker launch path.
- `chop` — hard-coded by sase source for every AXE chop proposal that proposes agents (`chop_proposal_models.py`,
  `chop_proposal_planning.py`).
- `pinned` — the value the ACE tribe modal seeds by default.
- `review` — assigned by AXE review runners.
- `research` — **not assigned anywhere in this repo.**

### `research` is user-owned, verified three ways

1. `grep -rn "tribe:research\|tribe=research\|%t:research" src/ docs/ sase/memory/ src/sase/xprompts/` returns nothing.
   Every hit under `tests/` is a directive-parsing fixture string that never touches display config.
2. The only real emitters live in the chezmoi checkout:
   - `home/sase/xprompts/research_swarm.md:11` — `%clan(research.{@1}, tribe=research, …)`
   - `home/sase/xprompts/old_research_swarm.md:9,13,17` — `%id(tribe=research)`
3. `grep -rn "research" src/sase/ace/ src/sase/core/agent_tribe.py src/sase/agents/ src/sase/doctor/` yields one
   unrelated comment in `config_edit_helpers.py`.

Note the distinction: `research` as a **document/sidecar role** (`sase repo path research`, `@research:` references,
`docs/configuration.md` sidecar sections) is genuinely a sase concept and is untouched by this plan. Only the **agent
tribe** of the same name moves.

### Chezmoi currently configures no tribes

`sase config layers` shows exactly two layers carrying an `ace` key: `default` (built-in) and `user`
(`~/.config/sase/sase.yml`). The user layer's `ace:` block contains only `snippets:`
(`home/dot_config/sase/sase.yml:24-130`). No plugin layer ships `ace.tribes` — `plugin:sase_github` contributes only
`xprompts`. So `@research` is styled today purely by the bundled default.

### What reads the bundled `research` entry

- `src/sase/ace/tui/models/tribe_display.py` — `_tribe_displays_for_token()` builds `_TribeDisplay` per configured key;
  unknown keys fall back to `DEFAULT_TRIBE_DISPLAY` (no icon, `configured=False`) and
  `TRIBE_IDENTITY_FALLBACK_COLOR = "#FFD75F"`.
- `docs/configuration.md:634` — prose naming "∴ in teal-green for `research`".
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py` — `test_tribe_panel_display_config_png_snapshot`
  renders one agent per bundled tribe and asserts each panel title's icon and color against the **real merged config**.
  Lines 160-168 assert the literal `"∴ @research"` border-title text; line 176 asserts
  `("research", "∴ ", "@research", "#5FD7AF")`. Golden:
  `tests/ace/tui/visual/snapshots/png/agents_tribe_panel_display_config_120x40.png`.

Nothing else in the repo depends on it. `tests/ace/tui/test_agent_panel_titles.py:98-113,244-257` also uses `∴` /
`#5FD7AF` / `@research`, but passes `icon=` and `color=` **explicitly** to `agent_panel_border_title`, so it is a pure
unit test of the title builder and needs no change. `tests/ace/tui/models/test_tribe_display.py` patches its own tribes
dicts. `tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py` uses `agent_clan="research"` but never
`tribe="research"`, so no other golden renders a `@research` panel.

### Tests are isolated from the user's real config

`tests/conftest.py:258-276` (`_isolate_sase_home`, autouse) monkeypatches `config_core.CONFIG_DIR` and
`config_targets.CONFIG_DIR` to a per-test tmpdir, so the suite sees bundled defaults only. Adding `research` to chezmoi
therefore cannot mask its removal from the defaults: tests will observe `research` as an **unconfigured** tribe both
locally and in CI. This is what makes the visual-test update below a real assertion rather than a host-dependent one.

## Design

### 1. The rule this change establishes

Bundled `ace.tribes` entries are the ones SASE source itself can assign. Any other tribe — one your own xprompts assign
with `%tribe:` / `tribe=` — is yours to configure in `~/.config/sase/sase.yml`. This keeps the shipped defaults honest
(a fresh install never styles a tribe it will never produce) and keeps `docs/configuration.md`'s bundled-tribe list
truthful.

`chop` stays bundled deliberately: sase source hard-codes `tribe=chop` for every agent-proposing AXE chop proposal, so
any user who writes one gets `@chop` without configuring anything. That decision is settled — do not move `chop`.

### 2. Zero visible change for the user

The chezmoi entry keeps the exact `icon: "∴"` and `color: "#5FD7AF"` and omits `initially_expanded` (defaulting to
`true`), which is precisely today's resolved state. After `chezmoi apply`, `@research` renders identically. The only
observable difference is a fresh, un-chezmoi'd sase install, where `@research` would render with the gold fallback and
no icon — which is correct, because such an install never produces the tribe.

### 3. The description is rewritten, not copied

The bundled string ("Research agents that answer open questions with durable reports instead of code changes.") was
written for a generic audience. In Bryan's config it should name the workflow that actually produces the tribe. Use:

```
Research swarm agents from #research_swarm that ship durable reports instead of code changes.
```

93 characters — under the 160-char schema cap and under the ~95-char soft limit where the metadata panel starts wrapping
in a narrow layout (see the Risks section of the prior tribe-description plan).

### 4. The visual test becomes the regression guard

Rather than deleting the `research` agent from `_tribe_display_agents()`, keep it and flip its assertions to the
unconfigured fallback. The test then demonstrates configured-vs-unconfigured tribe rendering side by side in one PNG,
and it fails loudly if `research` ever creeps back into the bundled defaults. Panel count (6), panel key order
(`[None, "epic", "pinned", "research", "review", "chop"]`), and the `agent_count` assertion are all unaffected: panel
keys come from the agents present (`_panel_keys_for` in `src/sase/ace/tui/models/agent_panels.py`), not from config, and
`chop` sorts last only because it is collapsed.

### 5. Non-goals

- Do not move `chop`, `pinned`, `review`, `epic`, or `default`.
- Do not touch the `research` **sidecar/document role** anywhere (`sase repo path research`, `@research:` references,
  `sdd` config, `docs/configuration.md` sidecar sections).
- Do not add a doctor check or schema rule that enumerates "sase-owned" tribes; the bundled-defaults regression test in
  step 5 is sufficient and cheaper.
- Do not edit `CHANGELOG.md` (release-please manages it).

## Implementation

Do step 1 before step 2 so the config exists in the chezmoi source before the default is withdrawn.

### Step 1 — Add the tribe to the chezmoi user config

Open the repo through the skill and use only the path it prints:

```bash
sase repo open chezmoi -r "Add the user-owned ace.tribes.research entry now that sase no longer bundles it"
```

In `home/dot_config/sase/sase.yml`, insert a `tribes:` block under the existing `ace:` key, **after** the `snippets:`
block ends (immediately after the `# keep-sorted end` line that closes `snippets`, before the blank line preceding
`axe:`). Keys under `ace:` stay alphabetical, so `tribes` follows `snippets`.

```yaml
# Tribes assigned by my own xprompts, not by sase itself. sase bundles
# display config only for the tribes its own source can assign (default,
# epic, chop, pinned, review); anything else is mine to configure here.
tribes:
  research:
    icon: "∴"
    color: "#5FD7AF"
    description: "Research swarm agents from #research_swarm that ship durable reports instead of code changes."
```

Match the file's existing two-space indentation exactly (`tribes:` at the same column as `snippets:`). Do **not** add a
`# keep-sorted` region — the surrounding `snippets` block owns that marker pair and a second region here would be new
convention, not existing convention.

Do not run `chezmoi apply` or `chezmoi update` mid-implementation. The chezmoi repo's own `CLAUDE.md` defines the
workflow: commit first, then `chezmoi update -a --force`. See step 8.

### Step 2 — Remove the bundled entry

In `src/sase/default_config.yml`, delete lines 114-117:

```yaml
research:
  icon: "∴"
  color: "#5FD7AF"
  description: "Research agents that answer open questions with durable reports instead of code changes."
```

Leave the explanatory comment block at lines 97-104 unchanged — every sentence in it still applies. The remaining five
entries keep their current order: `default`, `epic`, `chop`, `pinned`, `review`.

### Step 3 — Docs

In `docs/configuration.md`, the `#### ace.tribes` section:

- Line 634 currently reads: "The bundled defaults use ⌂ in sky blue for `default`, ▲ in lavender-purple for `epic`, ∴ in
  teal-green for `research`, and † in amber-orange for `chop`." Drop the `research` clause so it names only the four
  icon+color entries that remain (`default`, `epic`, `chop`) plus the existing following sentence about `pinned` and
  `review`. Re-flow the paragraph to the file's prevailing wrap width (the surrounding prose wraps at 120 columns).
- Add one sentence to the same section stating the ownership boundary: SASE bundles display config only for the tribes
  its own source assigns (`default`, `epic`, `chop`, `pinned`, `review`); a tribe your own xprompts assign with
  `%tribe:` has no bundled entry, renders with ACE's gold fallback and no icon until you configure it under
  `ace.tribes`, and — once configured — requires a `description` like any other entry.

No change is needed in `docs/ace.md`: it never names the bundled tribe set.

### Step 4 — Update the display-config visual test

In `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`, `test_tribe_panel_display_config_png_snapshot`:

- Keep `_tribe_display_agents()` exactly as is (six agents, one still `tribe="research"`).
- In the border-title label loop (currently the tuple containing `"∴ @research"`), change that entry to `"@research"`.
- In the `(key, icon, label, color)` table, remove the `("research", "∴ ", "@research", "#5FD7AF")` row and instead
  assert the fallback for that panel: the `@research` label renders in `TRIBE_IDENTITY_FALLBACK_COLOR` (`#FFD75F`,
  imported from `sase.ace.tui.models.tribe_display` rather than hard-coded) and the title contains no `∴`. Use the
  existing `_assert_title_identity_color` helper for the color assertion.
- Add a short comment on that assertion explaining **why** `@research` is unstyled here — it is a user-owned tribe
  configured in the operator's own `~/.config/sase/sase.yml`, and tests run against bundled defaults only — so the next
  reader does not "fix" it by re-adding the bundled entry.

### Step 5 — Bundled-defaults regression test

Add one test to `tests/test_config_schema_tribes.py` asserting the bundled tribe key set is exactly
`{"default", "epic", "chop", "pinned", "review"}`, with a docstring stating the rule from Design §1. Load the bundled
config directly via `from sase.config.core import _load_default_config` — that private accessor is already the
established test-side precedent (`tests/test_axe_lumberjack_config.py:249-251`), and it avoids any dependence on
plugin/user layers. Assert with `set(...) == {...}` so both an unexpected addition and an unexpected removal fail.

### Step 6 — Regenerate the affected PNG golden

Exactly one golden changes: `agents_tribe_panel_display_config_120x40`.

```bash
just test-visual
```

Confirm the only tribe-related failure is that snapshot and that the diff is limited to the `@research` panel border
title losing `∴ ` and turning gold. Then:

```bash
just test-visual -- --sase-update-visual-snapshots
```

Inspect the regenerated PNG (`Read` the golden file) and verify: the `@research` panel title reads `@research` in gold,
every other panel keeps its current icon and color, and no other row shifted. Two `models_panel_alias_picker_reordered*`
PNG tests are known-flaky on master (task beads `sase-bh`/`sase-bi`) — confirm any such failure reproduces on a clean
tree before dismissing it, and never regenerate those goldens as part of this change.

### Step 7 — Verify and check

```bash
just install
just check
```

`just install` first is mandatory in an ephemeral workspace. Report any pre-existing failure honestly rather than
folding it into this change; file a task bead if it is new.

Optionally, once the chezmoi change is live (step 8), `sase config layers` should show `ace` in the `user` layer and
`sase doctor -C config.tribes` should stay `OK`.

### Step 8 — Commit and apply

Two repos change, so there are two commits. Follow the normal sase commit workflow (the `sase_git_commit` skill) for
each when the user or the finalizer asks for a commit — do not commit unprompted.

For the chezmoi commit specifically, that repo's `CLAUDE.md` requires running `chezmoi update -a --force` **after** the
commit lands so the edit reaches `~/.config/sase/sase.yml`. That instruction is the repo's own standing convention;
follow it rather than asking again. Until it runs, `@research` renders with the gold fallback in a live ACE session —
mention this to the user so the transient appearance change is not a surprise.

## Testing

- `pytest tests/test_config_schema_tribes.py` — new bundled-key-set assertion passes; existing schema accept/reject
  cases still pass.
- `pytest tests/ace/tui/test_agent_panel_titles.py` — unchanged and still green (explicit icon/color args).
- `pytest tests/ace/tui/models/test_tribe_display.py tests/ace/tui/models/test_agent_tribe_summary.py` — unchanged and
  still green (they patch their own tribes dicts).
- `pytest tests/doctor/test_checks_config_tribes.py` — unchanged and still green.
- `just test-visual` — the display-config snapshot passes against the regenerated golden; every other tribe-panel golden
  (`agents_tribe_panel_level_1..4`, `_isolation_armed`, `_selected_expanded`) is untouched because those fixtures use
  only `tribe="epic"`.
- `just check` — full gate.
- Manual: after `chezmoi update -a --force`, `sase config show | grep -A4 'research:'` shows the user-layer entry, and
  an ACE session renders `∴ @research` in `#5FD7AF` with the new description in the metadata panel when the panel is
  selected.

## Risks

- **Window between the two commits.** If the sase-repo commit lands before chezmoi is applied, `@research` briefly
  renders unstyled. Harmless and self-correcting; step 8 tells the user.
- **Chezmoi source vs. live file.** Editing `home/dot_config/sase/sase.yml` does nothing until `chezmoi apply` /
  `chezmoi update -a --force`. Never hand-edit `~/.config/sase/sase.yml` directly — chezmoi would overwrite it.
- **Schema strictness.** `ace.tribes.<name>.description` is `required` (error severity), and any error-severity config
  diagnostic blocks _all_ ACE Config Center writes. The chezmoi entry must include its description in the same edit that
  introduces the `research` key — never commit an icon/color-only entry as an intermediate state.
- **YAML indentation in a 284-line hand-maintained file.** Insert `tribes:` at the same indentation as `snippets:` under
  `ace:`; a two-space slip would silently create a top-level or misparented key. Verify with `sase config layers` after
  applying, not by eye.
