---
tier: epic
title: Zero-friction model alias default edits
goal: 'Editing any value in src/sase/llm_provider/model_alias_defaults.yml requires
  no other change anywhere in the repo, and the full just check passes without the
  editor having to run it.

  '
phases:
- id: seam
  title: Frozen test defaults, re-pinned tests, hardened loader
  depends_on: []
  size: medium
  description: 'seam: split the defaults parser from the cached resource loader, add
    fallback-reference, selector-grammar, and fallback-cycle validation to the parser,
    add a test-owned frozen defaults map installed by an autouse conftest fixture,
    re-pin the 39 measured value-coupled assertions to named frozen constants, and
    rewrite the shipped-file test module as a value-agnostic contract suite with a
    shape-parity guard.'
- id: docs
  title: One generated table, zero literal values in prose
  depends_on: []
  size: medium
  description: 'docs: add a tools/ generator that rewrites a marked block in docs/llms.md
    from the shipped defaults, wire it into just fmt only, strip literal shipped values
    from prose across the six docs that restate them, and delete the docs-sync test
    without replacing it.'
- id: guidance
  title: De-hardcode product strings
  depends_on: []
  size: small
  description: 'guidance: interpolate the live medium_phase_worker default into the
    doctor message instead of hardcoding it, make the sase.schema.json and default_config.yml
    claims about shipped values value-free, and assert the doctor message value-agnostically.'
- id: verify
  title: Prove the acceptance criterion end to end
  depends_on:
  - seam
  - docs
  - guidance
  size: small
  description: 'verify: perturb every target and description in the shipped defaults
    YAML, prove the full check and visual suite pass with zero edits outside that
    file, confirm just fmt heals the generated docs block idempotently, exercise the
    hardened loader''s negative paths, then restore and report.'
proposed_by: bbugyi200.athena.sw.f1
create_time: 2026-08-03 14:46:44
status: wip
bead_id: sase-f1
---

- **PROMPT:** [prompts/202608/zero_friction_model_alias_defaults.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/zero_friction_model_alias_defaults.md)
- **BEAD:** [sase-f1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-f1/README.md)

# Epic: Make `model_alias_defaults.yml` a zero-friction edit point

## Goal

Editing any **value** in `src/sase/llm_provider/model_alias_defaults.yml` — a `target:`, a fallback's `@<effort>`
overlay, or a `description:` — must require **no other change anywhere in the repo**, and `just check` must still pass
without the editor running it.

Acceptance test (executed in the final phase): perturb every `target:` and every `description:` in the shipped YAML to a
different valid set, run the full `just check` (including the visual suite), and observe a clean pass with zero edits
outside that one file.

## Measured baseline (do not re-derive; verified on `f4acb7918`)

I ran the whole suite twice against a perturbed `model_alias_defaults.yml` to measure the real blast radius rather than
guess it.

**Perturbation used** (kept the alias _shape_ identical — same fallback graph, same which-alias-owns-a-target — and
changed only values and descriptions):

| alias                 | probe target                                     |
| --------------------- | ------------------------------------------------ |
| `coder`               | `codex/gpt-4.1`                                  |
| `medium_phase_worker` | `codex/o3@high`                                  |
| `smartest`            | `claude/sonnet@max`                              |
| `cheap`               | `claude/haiku@low \| codex/gpt-4.1@low`          |
| `cheaper`             | `claude/haiku@minimal \| codex/gpt-4.1-mini@low` |
| `cheapest`            | `claude/haiku@minimal \| codex/gpt-4o-mini`      |

Every `description:` was replaced too. This probe set parses cleanly under the current loader and selector grammar — it
produced only assertion mismatches, never a load error. **Reuse it as the frozen fixture values.**

**Results.** 45 failures in the non-visual suite, of which **6 are pre-existing and unrelated** (see "Pre-existing
failures" below), leaving **39 real value-coupled failures** in 10 files:

| file                                                        | failures                |
| ----------------------------------------------------------- | ----------------------- |
| `tests/llm_provider/test_config_role_aliases.py`            | 15                      |
| `tests/llm_provider/test_load_balanced_aliases.py`          | 8                       |
| `tests/llm_provider/test_alias_override_resolution.py`      | 4                       |
| `tests/llm_provider/test_model_alias_defaults.py`           | 4                       |
| `tests/llm_provider/test_alias_view.py`                     | 2                       |
| `tests/llm_provider/test_config_alias_resolution.py`        | 2                       |
| `tests/llm_provider/test_config_aliases.py`                 | 1 (description-coupled) |
| `tests/llm_provider/test_model_alias_defaults_docs_sync.py` | 1                       |
| `tests/test_axe_run_agent_phases_tribes.py`                 | 1                       |
| `tests/test_bead/test_work_rendering_models.py`             | 1                       |

Every failure is the same defect: **the test restates a shipped value as a literal.** Example lines:
`assert ('codex', 'gpt-5.4') == ('codex', 'gpt-5.5')`, `assert ('claude/opus', 'high') == ('claude/opus', 'max')`.

**PNG snapshots: zero fallout.** The full visual suite (405 tests) was run against the same perturbation, including
changed descriptions, and only the 2 pre-existing config-center failures appeared. The Models-panel snapshot tests all
patch `models_panel.build_alias_views` with hand-built `AliasView` fixtures, so they never read the shipped YAML. **No
golden needs updating in this epic**, and no phase should regenerate one.

**Rust core: not involved.** `rg 'sase_core_rs' src/sase/llm_provider/` returns nothing for the alias-defaults path; the
loader, resolution, and policy modules are pure Python. No `../sase-core` change is in scope.

## Design

### 1. Tests read a frozen, test-owned defaults file — never the shipped one

All three public accessors (`role_alias_fallbacks`, `implicit_alias_targets`, `role_alias_descriptions` in
`src/sase/llm_provider/model_alias_policy.py`) funnel through one `@functools.cache`d private loader,
`_load_model_alias_defaults()`. Consumers in `config.py`, `model_alias_config.py`, and `model_alias_resolution.py`
import the _accessors_, so patching that single loader redirects every read process-wide.

An autouse fixture in `tests/conftest.py` installs a frozen `_ModelAliasDefaults` built from a **test-owned** alias map.
This is the same pattern the repo already uses for `_pin_configured_timezone`, `_isolate_default_llm_effort`, and
`_clear_config_caches`, which likewise reach into module privates from `conftest.py`.

Verified safe: nothing in `src/` or `tests/` calls `_load_model_alias_defaults.cache_clear()`, so replacing the cached
function with a plain callable breaks nothing.

**Why different values, not a copy of today's shipped values:** a copy would make the seam unobservable and would let a
test silently depend on the real file again. Distinct values also make the fixture _maximally discriminating_ — every
target-valued alias resolves to a unique `(provider, model, effort)` triple, so a mis-wired fallback chain can no longer
pass by coincidence. That is a genuine strengthening of these tests, not just churn.

### 2. Shape stays a code contract; values do not

The frozen fixture mirrors the shipped file's **shape**: for each alias, whether it declares a `target`, a `fallback`,
or neither, and for fallbacks, which alias name is referenced (effort overlay ignored). A shape change — such as the
recent `medium_phase_worker: fallback "@default@high"` → `target: "codex/gpt-5.5@xhigh"` — genuinely rewires the alias
graph and changes what the Models panel renders, so it _should_ require test attention.

A shape-parity test reads the real shipped file and asserts its shape signature equals the fixture's, failing with a
message that names the drifted alias and points at the fixture module. Pool-vs-concrete is **not** part of the
signature, so `cheap: "a | b"` → `cheap: "claude/opus"` stays free.

This is the epic's one explicit trade-off, and it must be stated in the fixture module's docstring: **value edits are
free; graph-shape edits are code changes.**

### 3. The loader validates what tests used to

Because the point is that nobody runs `just check` after a value edit, a bad edit must fail loudly and immediately on
the next `sase` invocation rather than silently rerouting a role. Move three checks that currently live only in
`tests/llm_provider/test_model_alias_defaults.py` into the parser, and add one that exists nowhere today:

- every `fallback` is `@<declared alias name>` with an optional valid effort overlay;
- every `target` parses under `parse_model_alias_selector` and, if it is a selector, has members;
- **new:** the fallback graph is acyclic and every chain terminates at `default` or at a target-valued alias.

`sase.xprompt.effort` is stdlib-only and `sase.llm_provider.load_balancing` does not reference `model_alias_policy`, so
importing both from the policy module is cycle-free (verified by importing the module standalone). If a cycle does
appear, use a function-local import inside the parser rather than restructuring.

### 4. Docs stop restating shipped values, and the one table that keeps them is generated

`docs/llms.md` and `docs/configuration.md` are enforced today by `test_model_alias_defaults_docs_sync.py`;
`docs/ace.md`, `docs/beads.md`, `docs/sdd.md`, and `docs/xprompt.md` restate values with no enforcement at all and can
already be stale.

Resolution: exactly **one** rendering of the shipped defaults survives, as a generated block in `docs/llms.md`, produced
by a `tools/` script wired into `just fmt` — **not** into `just check`. Prose everywhere else names aliases and links to
that block. The result is that a value edit needs nothing; the next `just fmt` anyone runs heals the table for free; and
`just check` can never fail because of a doc value.

The docs-sync test is deleted. Nothing may replace it — a freshness test would reintroduce exactly the failure this epic
removes.

### 5. Product strings interpolate instead of hardcoding

`src/sase/doctor/checks_config_model_aliases.py` hardcodes `"accept the shipped codex/gpt-5.5@xhigh default"`, and
`src/sase/config/sase.schema.json` plus `src/sase/default_config.yml` assert `"@coder ships as codex/gpt-5.5"` in
user-facing text. None is test-enforced today, so all three go stale silently. The doctor message interpolates the live
value; the schema and config-comment claims become value-free.

## Pre-existing failures — do NOT attribute these to this epic

Measured on a pristine `f4acb7918` worktree in this workspace, with the shipped YAML untouched:

- `tests/main/test_bead_fast_path_context.py::test_lightweight_context_uses_primary_vc_store_over_primary_non_vc_in_vc_mode`
- `tests/doctor/test_checks_beads.py::test_project_beads_skips_when_store_is_absent`
- `tests/test_bead/test_cli_golden.py::test_bead_cli_golden_contract[init]`
- `tests/test_bead/test_cli_resolution.py::test_workspace_context_rejects_primary_outside_pytest_sandbox`
- `tests/test_bead/test_cli_resolution.py::test_plain_checkout_non_sidecar_record_falls_back_to_legacy_resolution`
- `tests/test_bead/test_cli_changespec.py::test_create_plan_stores_sibling_workspace_plan_path_relative_to_primary`
- visual: `test_config_center_agent_clis_marked_png_snapshot`,
  `test_config_center_agent_clis_update_preview_png_snapshot`

These were measured **without** a preceding `just install`, so some may be stale-venv artifacts. Every phase runs
`just install` first; if any of these still fail afterwards on a clean tree, file a task bead via `/sase_new_task`
instead of fixing it inside this epic.

## Non-goals

- Changing any shipped default value. The YAML's committed content is identical before and after this epic.
- Regenerating any PNG golden (measured: none are affected).
- Adding a new env var, CLI flag, or config key for overriding shipped defaults.
- Any change under `../sase-core`.

---

# Phases

## Phase `seam` — Frozen test defaults, re-pinned tests, hardened loader

**Depends on:** nothing.

### Files

- `src/sase/llm_provider/model_alias_policy.py` (modify)
- `tests/_model_alias_defaults_fixture.py` (new)
- `tests/conftest.py` (modify)
- `tests/llm_provider/test_model_alias_defaults.py` (rewrite as the shipped-file contract suite)
- Re-pin: `tests/llm_provider/test_config_role_aliases.py`, `test_load_balanced_aliases.py`,
  `test_alias_override_resolution.py`, `test_alias_view.py`, `test_config_alias_resolution.py`,
  `test_config_aliases.py`; `tests/test_axe_run_agent_phases_tribes.py`; `tests/test_bead/test_work_rendering_models.py`

### Steps

1. **Split parse from load** in `model_alias_policy.py`. Extract the body of `_load_model_alias_defaults` after the
   resource read into `_parse_model_alias_defaults(text: str, *, source: object) -> _ModelAliasDefaults`. Keep
   `_load_model_alias_defaults` as the `@functools.cache`d resource reader that delegates to it. Both stay private and
   `_parse_model_alias_defaults` is called from its own file, so Symvision is satisfied without a pragma.

2. **Harden the parser** with the three checks from Design §3. Reuse `parse_model_alias_selector` /
   `ModelAliasSelectorError` from `sase.llm_provider.load_balancing` and `split_model_effort` / `EFFORT_LEVELS` from
   `sase.xprompt.effort`. Raise through the existing `_defaults_error` helper so messages keep naming the resource and
   the offending alias. Confirm no import cycle: `.venv/bin/python -c "import sase.llm_provider.model_alias_policy"` in
   a fresh process.

3. **Create `tests/_model_alias_defaults_fixture.py`.** Single source of truth, shaped as a Python mapping that is
   `yaml.safe_dump`-ed and fed through the _real_ parser, so the fixture data is validated by production code:

   ```python
   _FROZEN_ALIASES: dict[str, dict[str, str]] = {
       "default":             {"description": ...},
       "coder":               {"target": "codex/gpt-4.1", "description": ...},
       "epic_lander":         {"fallback": "@default", "description": ...},
       "big_epic_lander":     {"fallback": "@smartest", "description": ...},
       "xsmall_phase_worker": {"fallback": "@cheaper", "description": ...},
       "small_phase_worker":  {"fallback": "@cheap", "description": ...},
       "medium_phase_worker": {"target": "codex/o3@high", "description": ...},
       "large_phase_worker":  {"fallback": "@smart", "description": ...},
       "xlarge_phase_worker": {"fallback": "@smartest", "description": ...},
       "smart":               {"fallback": "@default", "description": ...},
       "smartest":            {"target": "claude/sonnet@max", "description": ...},
       "cheap":               {"target": "claude/haiku@low | codex/gpt-4.1@low", "description": ...},
       "cheaper":             {"target": "claude/haiku@minimal | codex/gpt-4.1-mini@low", "description": ...},
       "cheapest":            {"target": "claude/haiku@minimal | codex/gpt-4o-mini", "description": ...},
   }
   ```

   Export `FROZEN_TARGETS`, `FROZEN_FALLBACKS`, and `FROZEN_DESCRIPTIONS` mappings derived from it, plus
   `install_frozen_model_alias_defaults(monkeypatch)` which parses once (module-level `functools.cache`) and does
   `monkeypatch.setattr(model_alias_policy, "_load_model_alias_defaults", lambda: frozen)`.

   The module docstring must state the trade-off from Design §2 and instruct future editors: _shipped-value changes need
   nothing here; shipped-shape changes need this map updated to match._

   Descriptions must be distinct per alias and obviously synthetic (e.g. `"Frozen test description for cheap."`).

4. **Wire the autouse fixture** into `tests/conftest.py` next to `_pin_configured_timezone`, plus a non-autouse
   `real_model_alias_defaults` fixture that undoes the patch (and clears the loader cache) for the contract suite.
   `tests/conftest.py` is 568 lines against a 700-line `toobig` threshold — keep the body in
   `tests/_model_alias_defaults_fixture.py` and let conftest hold only the thin fixture wrappers.

5. **Re-pin the 39 assertions.** Run `just test` and fix each failure by importing the relevant `FROZEN_*` constant
   instead of retyping a literal. Do not weaken any assertion into a tautology
   (`assert x == implicit_alias_targets()[...]` proves nothing about wiring) — assert against the named constant for
   _that_ alias.

   Treat the measured list as a **floor, not a ceiling**: it was produced by one probe set. Discover the true set by
   running the suite with the fixture installed.

6. **Rewrite `tests/llm_provider/test_model_alias_defaults.py`** as the shipped-file contract suite. Every test in it
   takes the `real_model_alias_defaults` fixture. Keep the existing structural tests (name completeness, mutual
   exclusion, `default` declares neither, non-empty stripped descriptions, grammar, fallback resolution). **Delete** the
   four value-pinning tests (`test_smartest_ships_concrete_max_effort_target`,
   `test_coder_ships_common_concrete_target`, `test_cheap_family_ships_the_expected_members_and_efforts`,
   `test_medium_phase_worker_ships_concrete_xhigh_target`). Add:
   - the **shape-parity** test from Design §2;
   - negative tests that the hardened parser rejects an unknown `@ref`, an unparseable target, and a two-alias fallback
     cycle.

### Verification

`just install`, then `just fmt`, then `just check`. The alias suites must be green; only the pre-existing failures above
may remain.

---

## Phase `docs` — One generated table, zero literal values in prose

**Depends on:** nothing. (No file overlap with `seam` or `guidance`.)

### Files

- `tools/render_model_alias_docs` (new)
- `Justfile` (modify)
- `docs/llms.md`, `docs/configuration.md`, `docs/ace.md`, `docs/beads.md`, `docs/sdd.md`, `docs/xprompt.md` (modify)
- `tests/llm_provider/test_model_alias_defaults_docs_sync.py` (delete)

### Steps

1. **Write `tools/render_model_alias_docs`** — an extension-less executable Python script matching the existing `tools/`
   convention (`tools/validate_changelog`, `tools/postprocess_docs_pdf`). It reads `implicit_alias_targets()`,
   `role_alias_fallbacks()`, and `role_alias_descriptions()` and rewrites the content between
   `<!-- BEGIN GENERATED: model-alias-defaults -->` and `<!-- END GENERATED: model-alias-defaults -->` in `docs/llms.md`
   with a table of alias / description / shipped default. Constraints:
   - escape `|` as `\|` inside table cells, matching the current table's convention for pool values;
   - error clearly if either marker is missing;
   - be idempotent, and exit non-zero on no-op-vs-error ambiguity only, never on staleness.

2. **Wire into `just fmt` only.** Add a `fmt-docs` recipe and put it _before_ `fmt-md` in both `fmt:` and `fix:`, so
   prettier normalizes the generated table in the same invocation and `just fmt` stays idempotent as a whole. Do **not**
   add it to `fmt-check`, `lint`, or `check`. `_lint-pyscripts` requires the script be referenced from within the repo —
   the Justfile reference satisfies that.

3. **Replace the alias table in `docs/llms.md`** (currently around lines 703–718) with the marker block, and run the
   generator to populate it. This block is the only place in the docs tree that may contain a shipped value.

4. **Strip literal shipped values from prose** at the sites below. Prose names the alias and links to the generated
   block; it never states what the alias resolves to.
   - `docs/llms.md`: ~571–579, 728, 741–749, 951–954, 965, 980–989
   - `docs/configuration.md`: ~1017–1022, 1067–1070, 1083, 2350
   - `docs/ace.md`: ~2167, 2210–2219, 2939
   - `docs/beads.md`: ~1023–1027
   - `docs/sdd.md`: ~293–299
   - `docs/xprompt.md`: ~1269, 1624

   Example-config YAML blocks may keep concrete values, but the surrounding text must label them as _examples of user
   overrides_, and the values must be visibly not-the-shipped-defaults — follow the convention already used in
   `src/sase/default_config.yml` (`cheap: "claude/haiku | codex/gpt-4.1-mini"`).

5. **Delete `tests/llm_provider/test_model_alias_defaults_docs_sync.py`.** Do not add a replacement freshness test.

6. Sweep for stragglers: `rg -n 'gpt-5\.5|opus@max|sonnet@xhigh|gpt-5\.3-codex-spark' docs/` and confirm every remaining
   hit is either inside the generated block, a labelled example, or a provider/model-registry listing (the codex model
   list and short-name tables in `docs/llms.md` around lines 805 and 834 are registry facts, not alias defaults — leave
   them).

### Verification

`just install`, `just fmt` twice in a row (second run must produce no diff), then `just check`.

---

## Phase `guidance` — De-hardcode product strings

**Depends on:** nothing. (No file overlap with `seam` or `docs`.)

### Files

- `src/sase/doctor/checks_config_model_aliases.py` (modify)
- `src/sase/config/sase.schema.json` (modify)
- `src/sase/default_config.yml` (modify)
- `tests/doctor/test_checks_config_model_aliases.py` (modify)

### Steps

1. **`checks_config_model_aliases.py:192`** — replace the literal `"accept the shipped codex/gpt-5.5@xhigh default"`
   with the live value from `implicit_alias_targets()[MEDIUM_PHASE_WORKER_MODEL_ALIAS_NAME]`. Keep the substrings the
   existing test asserts (`"medium_phase_worker"`, `"remove it"`) intact.

2. **Extend `tests/doctor/test_checks_config_model_aliases.py`** to assert the message contains whatever
   `implicit_alias_targets()` currently returns for that alias — value-agnostic, so it passes both with and without the
   `seam` phase's frozen fixture installed. This is what keeps the two phases independent; do not reference any
   `FROZEN_*` constant here.

3. **`sase.schema.json:1552`** — drop `"The coder alias ships as codex/gpt-5.5, and"` from the `builtin` description,
   keeping the inheritance rule ("every registered `<provider>_coder` alias inherits `coder` unless explicitly
   overridden") and adding a pointer to `src/sase/llm_provider/model_alias_defaults.yml`. Confirm the file is
   hand-maintained (there is no generator; `src/sase/config/inventory.py` only resolves its path) and that
   `tests/test_config_schema_models.py` does not assert this description.

4. **`src/sase/default_config.yml`** — line ~782 states `@coder` ships as a specific model: make it value-free. Lines
   ~805 and ~816–819 are commented examples: reword the lead-in so they read as user-override examples rather than
   shipped values.

5. Sweep `rg -n 'ships as|shipped .*default' src/ --glob '!**/__pycache__/**'` for any remaining hardcoded claim.

### Verification

`just install`, `just fmt`, `just check`.

---

## Phase `verify` — Prove the acceptance criterion end to end

**Depends on:** `seam`, `docs`, `guidance`.

This phase produces no committed source change beyond, at most, a fix for whatever it finds. Its deliverable is the
demonstration.

### Steps

1. `just install`. Re-run the pre-existing-failure list on the untouched tree and record which of them still fail after
   a fresh install, so the perturbed run can be compared against a real baseline.

2. **Acceptance run.** Rewrite every `target:` and every `description:` in
   `src/sase/llm_provider/model_alias_defaults.yml` to a different valid set — use values distinct from both the shipped
   set and the frozen fixture set, keeping the alias shape unchanged. Run the full `just check` **and**
   `just test-visual`. Both must pass with **zero** edits outside that one file. Any failure is a defect in `seam`,
   `docs`, or `guidance` and must be fixed there.

3. **Fmt-heal check.** With the perturbation still applied, run `just fmt` and confirm the only diff outside the YAML is
   the generated block in `docs/llms.md`, and that a second `just fmt` produces no further diff.

4. **Negative path.** Confirm the hardened loader fails fast and legibly. One at a time, introduce into the YAML: an
   unknown fallback (`epic_lander: fallback: "@nope"`), a two-alias cycle (`smart: fallback: "@smartest"` +
   `smartest: fallback: "@smart"`), and a malformed pool (`cheap: target: "claude/opus || || codex/o3"`). After each,
   run `.venv/bin/sase doctor` and confirm a `RuntimeError` naming the resource and the offending alias. Revert each
   before the next.

5. **Restore.** `git checkout -- src/sase/llm_provider/model_alias_defaults.yml`, re-run `just fmt` to restore the
   generated block, and confirm `git status` is clean.

6. Report the acceptance result explicitly: the perturbation applied, the `just check` outcome, and the final list of
   pre-existing failures (if any) with evidence they also fail on a pristine tree.

### Verification

The acceptance run in step 2 is the verification. Do not report this epic complete on a partial or targeted test run.
