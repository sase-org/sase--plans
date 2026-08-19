---
tier: tale
title: Scope the `feature` task type to the sase project
goal:
  A machine-global `bead.task_types` policy can suppress a builtin task type for every
  project while one project re-enables it, and `feature` becomes agent-creatable only in
  the `sase` project — in the home agent instruction files, in every other project's
  generated instruction files, and at `sase bead create`.
size: medium
proposed_by: bbugyi200.athena.07u
create_time: 2026-08-19 11:05:00
status: wip
---

# Scope the `feature` task type to the sase project

## Problem

The reported symptom: the `bob-cli` project advertises and accepts the `feature` task
type even though only the `sase` project is supposed to use it.

### Root cause: `feature` is a builtin, not a sase-local type

`feature` is one of the five **builtin host task types** declared in
`src/sase/task_types/_builtin.py` (`_feature_spec()` at line 144, returned by
`_builtin_task_type_specs()` at line 269) and registered through `BuiltinTaskTypes` for
**every** project by `discover_task_type_specs()`
(`src/sase/task_types/_discovery.py:38-50`). `assemble_task_type_registry()`
(`src/sase/task_types/registry.py:48-63`) always starts from those five, then layers
plugin entry points, then `bead.task_types` project config.

The premise behind the report — "only this repo defines this type in its project-local
`sase/sase.yml`" — is not what the file says. `sase/sase.yml` declares exactly **one**
project-local task type, `flag`, and that type's `label:` is **`Feature flag`**
(`sase/sase.yml:291-303`). That label is the likely source of the misreading: `flag` is
genuinely sase-local and correctly absent from every other project; `feature` is
unrelated and has always been global.

Evidence gathered while diagnosing:

- `bob-cli`'s `sase/sase.yml` has no `bead:` key at all, so it adds no task types and
  overrides none.
- `bob-cli`'s committed `sase/task_types.json` holds exactly the five builtins, each
  with `"source": "builtin"`, `"package": "sase"`.
- `bob-cli`'s `AGENTS.md` / `CLAUDE.md` §"Task Bead Types" therefore documents `bug`,
  `ci`, `feature`, `flake`, and `memory`, and a `bob-cli` agent filed `bob-cli-z`
  ("Persist the Bob Mac Capture retained live draft across restarts") as a `feature`
  task bead on 2026-08-19. It is still `ready`.
- `sase`'s own snapshot additionally holds `flag` (`source: project`, not
  agent-creatable) and `github` (`source: plugin`, not agent-creatable).

So nothing is malfunctioning in the reported path: builtins are host-wide by design,
`sase memory init` snapshots the effective catalog per project, and the generated note
documents the agent-creatable subset. The gap is that **the intended policy has never
been expressed anywhere**, and the two config surfaces that could express it are both
broken for this shape of policy.

### Defect 1 — a second config layer cannot override a slug the first layer touched

`bead.task_types` already supports suppressing a builtin without inventing new syntax:
`use: builtin@feature` + `agent_creatable: false` deep-merges onto the builtin spec and
removes the type from `sase bead create` (`src/sase/task_types/fields.py:64-71`), from
the ACE create modal (`src/sase/ace/tui/modals/bead_create_modal.py:192`), from
`sase bead task-type list` (`src/sase/task_types/cli_list.py:54`), and from the
generated instruction-file Types section
(`src/sase/main/init_memory/root_rendering.py:258-262`) — while leaving existing beads
such as `bob-cli-z` fully readable, chipped, and templated. That is exactly the right
lever.

But the natural way to apply it once for all projects — disable in the machine-global
user layer (`~/.config/sase/sase.yml`), re-enable in `sase/sase.yml` — does not work.
`_apply_use_override()` rewrites the record's provenance to
`TaskTypeProvenance(source="project", package="project", builtin=False)`
(`src/sase/task_types/_project_config.py:143-152`), and the next layer's `use:` prefix
is matched against **that rewritten** provenance (`_project_config.py:113-131` →
`plugin_qualified_id_matches`). Verified against the live registry with the two layers
replayed in order:

```text
feature agent_creatable: False | provenance: TaskTypeProvenance(source='project',
    name='user', package='project', version='<unknown>', builtin=False)
DIAGNOSTICS:
  mismatched_use_prefix - task type 'feature' is provided by 'project', not 'builtin';
      use 'project@feature'
```

The re-enable is rejected and `feature` stays disabled **in the sase project too**. The
only prefix that works is `project@feature`, which is undiscoverable, is not what any
documentation describes, and silently becomes wrong again the moment the earlier layer's
entry is removed. A `use:` prefix must name the type's _provider_, and an override does
not change the provider.

### Defect 2 — the home instruction files ignore config entirely

`~/CLAUDE.md`, `~/AGENTS.md`, and `~/sase/memory/task_types.md` are machine-global and
are loaded by every agent in every project. Their Types section comes from
`_home_task_type_specs()`, which is a bare `tuple(BuiltinTaskTypes().task_type_specs())`
(`root_rendering.py:277-280`) and consults no config layer at all. Verified:

```text
home specs: ['bug', 'ci', 'feature', 'flake', 'memory']
```

So even a correct machine-global opt-out would leave every agent in every project still
reading "File one when you discovered a product or capability idea..." from the home
note, and only discovering the truth as a hard `sase bead create` error. The docstring's
stated intent — "so the home note never varies with plugins" — is about _plugins_;
`~/.config/sase/sase.yml` is a committed, machine-global file and is exactly as stable
as the builtin list.

## Design

I own the decisions below. Implement them as specified rather than re-deciding them.
Each records the alternative that was rejected, so a reasonable-looking "improvement"
during implementation does not silently undo a deliberate choice.

### D1. `agent_creatable: false`, not a new removal or `enabled:` mechanism

Suppress `feature` with the key that already exists. Do **not** add an `enabled:`
directive, a `bead.disabled_task_types` list, or any other way to delete a slug from the
catalog.

Rejected: a true removal. `bob-cli-z` is a live `ready` bead typed `feature`. Deleting
the slug would drop it to the unknown-type degradation path — raw key/value pairs under
a "not installed on this machine" header, the `untyped` triage fallback instead of the
type's own bar, and no chip. `agent_creatable: false` stops new beads while every
existing bead keeps rendering exactly as it does today. It is also already schema-valid
(`src/sase/config/sase.schema.json`, `taskTypeConfigEntry.agent_creatable`), already
validated by the Rust spec validator, and already proven in production by `flag`.

Rejected: moving `_feature_spec()` out of `_builtin.py` into `sase/sase.yml`. That would
match the original (mistaken) mental model, but it changes SASE's shipped defaults for
every user to encode one project policy, un-reserves the `feature` slug so a plugin can
claim it, forces any project that wants the type to copy a 25-line YAML block, and
orphans `bob-cli-z` on every machine.

### D2. The policy lives in the machine-global layer; `sase` opts back in

Write the disable once in the chezmoi-managed `~/.config/sase/sase.yml` (the `user`
config layer) and the re-enable once in `sase/sase.yml` (the `local` layer). Every
current and future non-sase project then inherits the policy with no per-project config,
which is what "no enabled sase project except sase" actually means.

Rejected: adding the disable to `bob-cli/sase/sase.yml` and `actstat/sase/sase.yml`. It
needs an edit in every project that exists now and every project created later, it
touches two repos this change otherwise does not need to touch, and it does nothing for
Defect 2 — the home note has no project layer to read, so it would keep advertising
`feature` everywhere regardless.

Keep both entries minimal — `use:` plus `agent_creatable:` and nothing else. Do **not**
also override `when_to_use` in either entry: the global entry's text would be inherited
by the sase project's re-enabled record, so any project-specific wording there would
either be wrong in `sase` or have to be duplicated in `sase/sase.yml`, where it would
drift from `_builtin.py`.

### D3. Fix 1: match `use:` against the provider, not against the previous override

Inside a single `apply_project_task_type_config()` pass, remember each slug's provenance
**as it stood before any `bead.task_types` entry touched it**, and match every `use:`
prefix — and phrase every `mismatched_use_prefix` message — against that remembered
origin.

Concretely: seed an `origin_by_slug: dict[str, TaskTypeProvenance]` from `records` at
the top of `apply_project_task_type_config()`, thread it into `_apply_use_override()`
and `_apply_new_project_type()`, use `origin_by_slug.get(slug, base.provenance)` for the
`plugin_qualified_id_matches()` call and for `_use_prefix_for()` in the error text, and
have `_apply_new_project_type()` record the provenance of a slug it newly defines so a
later layer can override that slug consistently too.

After the fix, `use: builtin@feature` is correct in every layer, in any order, whether
or not an earlier layer already overrode the slug — which is the only rule a user can be
expected to remember.

Rejected: adding `origin_*` fields to `TaskTypeProvenance`. The chain is entirely
contained in one function call (every layer's entries are replayed in one pass by
`_effective_raw_task_type_entries()`), so a local dict is enough and nothing outside
this module has to learn a new concept.

Rejected: changing what `_apply_use_override()` writes into `provenance`. An overridden
record legitimately reports `source: project` — that is what `sase bead task-type show`
and `sase/task_types.json` should say about a spec the project modified, and
`committed_task_type_records()` relies on `source in {builtin, project}` to decide
snapshot membership. Only the `use:`-matching input changes.

### D4. Fix 2: the home note reads builtins plus the machine-global layers

Replace `_home_task_type_specs()`'s bare builtin list with a new public helper —
`machine_global_builtin_task_type_specs()` in `src/sase/task_types/registry.py`,
exported from `src/sase/task_types/__init__.py` — that:

1. validates the `BuiltinTaskTypes()` specs into records (no entry-point discovery, so
   the home note still never varies with installed plugins),
2. applies `bead.task_types` from the machine-global layers only, with the `local` layer
   excluded, and
3. returns the specs of the resulting records whose slug is still one of the builtin
   slugs.

Excluding the `local` layer is what keeps project-local types (`flag`) and project-local
overrides out of a file shared by every project on the machine. Step 3 keeps a
machine-global entry that _defines a new slug_ out of the home note — the home note
documents the builtin baseline, not the union of everything anyone configured.

Thread the layer restriction as an explicit keyword on the existing replay —
`apply_project_task_type_config(records, diagnostics, *, include_local_layer: bool = True)`
forwarded to `_effective_raw_task_type_entries()`, which skips layers named `local` when
it is false. Do **not** reach for `set_include_local_config()`: it is a process-global
toggle that invalidates the shared config caches for every subsequent read in the
process, and `sase memory init` renders the project note in the same run.

Diagnostics from this restricted assembly are intentionally discarded. A machine-global
entry referencing a plugin-provided slug (say `use: sase-github@github`) resolves to
nothing here and would raise `unknown_task_type_use` against a builtin-only base; that
is expected, not a user error, and the live registry behind `sase config doctor`'s
`beads.task_types` check (`src/sase/doctor/checks_beads.py:64`) remains the surface that
reports real config mistakes.

### D5. Leave `bob-cli-z` and the two other project repos alone

`bob-cli` and `actstat` need **no** repository change. Their generated instruction files
and `sase/task_types.json` are regenerated by their own `sase init` post-commit hook, at
which point `feature` drops out of their Types section and their snapshot entry for it
becomes `agent_creatable: false`. Do not edit those repos, do not touch `bob-cli-z`, and
do not run `sase memory init` inside them from this change.

### D6. Scope the `sase` re-enable to the project layer, not to the builtin spec

`_builtin.py` is not modified by this change. `feature` remains a shipped builtin with
its current spec, glyph `✦`, accent `#5FD75F`, fields, body template, and digest; only
this machine's config decides who may file one.

## Implementation steps

### 1. `src/sase/task_types/_project_config.py` — provider-stable `use:` matching (D3)

- In `apply_project_task_type_config()`, build
  `origin_by_slug = {record.task_type: record.provenance for record in records}` next to
  the existing `by_slug` / `order` locals and pass it to both `_apply_use_override()`
  and `_apply_new_project_type()`.
- In `_apply_use_override()`, resolve
  `origin = origin_by_slug.get(slug, base.provenance)` after the `base is None` check,
  and use `origin` for both the
  `plugin_qualified_id_matches(plugin, builtin=..., package=...)` call and the
  `_use_prefix_for(...)` value interpolated into the `mismatched_use_prefix` message.
  Everything else about the override — the deep merge, the digest revalidation, and the
  `source="project"` provenance it writes — is unchanged.
- In `_apply_new_project_type()`, after a successful add, record
  `origin_by_slug.setdefault(slug, <the provenance it just wrote>)` so a later layer
  that overrides a config-defined slug is matched against a stable value too.
- Update the module docstring's description of `use:` to state that the prefix names the
  type's original provider and stays correct across layers.

### 2. `src/sase/task_types/_project_config.py` — machine-global replay (D4)

- Add `include_local_layer: bool = True` as a keyword-only parameter on
  `apply_project_task_type_config()` and forward it to
  `_effective_raw_task_type_entries()`.
- In `_effective_raw_task_type_entries()`, `continue` past any layer whose `name` is
  `"local"` when the flag is false. Apply the skip **before** the `list_strategy` reset
  branch so a skipped layer cannot clear entries collected from earlier layers. (The
  local layer is `concatenate` today, so this is defensive, not load-bearing.)

### 3. `src/sase/task_types/registry.py` — `machine_global_builtin_task_type_specs()` (D4)

Add the helper described in D4 and list it in `registry.__all__`:

```python
def machine_global_builtin_task_type_specs() -> tuple[Mapping[str, Any], ...]:
    """Return the builtin specs after machine-global ``bead.task_types`` config."""
```

Implementation notes:

- Build candidates as
  `tuple((spec, provenance) for spec in BuiltinTaskTypes().task_type_specs())` with the
  same builtin `TaskTypeProvenance` `_discovery.py` uses (`source="builtin"`,
  `name="sase"`, `package="sase"`, `builtin=True`, version from
  `importlib.metadata.version("sase")`). Factor that provenance out of
  `discover_task_type_specs()` into a small shared helper in `_discovery.py` rather than
  duplicating the literal in two modules.
- Run it through `validate_task_type_candidates()`, then
  `apply_project_task_type_config(..., include_local_layer=False)`, both with a
  throwaway diagnostics list.
- Do **not** call `resolve_task_type_presentation()`; the note renders `when_to_use`,
  `label`, and field names only, and skipping it keeps glyph/accent assignment a
  single-owner concern of the live registry.
- Return
  `tuple(record.spec for record in records if record.task_type in builtin_slugs)`.
- This helper is deliberately **not** memoized. It runs twice per `sase memory init` at
  most.

Export it from `src/sase/task_types/__init__.py` (import block and `__all__`).

### 4. `src/sase/main/init_memory/root_rendering.py` — use it (D4)

- Change `_home_task_type_specs()` to `return machine_global_builtin_task_type_specs()`
  and rewrite its docstring: builtin baseline plus machine-global `bead.task_types`,
  with the project layer and plugin types excluded so a home note is identical for every
  project on the machine.
- Update the `from sase.task_types import (...)` block at line 32 — drop
  `BuiltinTaskTypes` if it becomes unused, add the new helper.

### 5. `sase/sase.yml` — keep `feature` in the sase project (D2)

Prepend to the existing `bead.task_types` list, above the `flag` entry, with a comment
that names the global entry it answers:

```yaml
bead:
  task_types:
    # Re-enable the builtin `feature` type, which ~/.config/sase/sase.yml turns off for
    # every project. Product ideas discovered while working on SASE belong here.
    - use: builtin@feature
      agent_creatable: true
    - schema_version: 1
      task_type: flag
      ...
```

### 6. chezmoi — the machine-global disable and the regenerated home memory (D2, D4)

Open the repo with `sase repo open chezmoi -r "<reason>"` and use only the printed path.

- In `home/dot_config/sase/sase.yml`, add a top-level `bead:` key (the file has none
  today):

  ```yaml
  bead:
    task_types:
      # Only the `sase` project files `feature` beads; sase/sase.yml re-enables it there.
      - use: builtin@feature
        agent_creatable: false
  ```

- Regenerate the home memory from the chezmoi `home/` directory so `home/AGENTS.md`,
  `home/CLAUDE.md`, `home/GEMINI.md`, `home/OPENCODE.md`, `home/QWEN.md`, and
  `home/sase/memory/task_types.md` lose their `feature` entry. Follow that repo's own
  `AGENTS.md` for the regeneration and deploy workflow; do not hand-edit the generated
  files.
- Commit the chezmoi change through `/sase_git_commit`, in its own commit, separate from
  the sase repo commit.

Note for the implementer: the generated files under `home/sase/memory/` are SASE memory
files. Approval of this plan is the explicit user authorization required to regenerate
them and the provider instruction shims they feed, in both repos. Do not ask again, and
do not hand-edit any generated file.

### 7. `docs/beads.md` — document the idiom (D1, D2, D3)

In the `### Task Types` section, after the paragraph beginning "The catalog is assembled
from three sources", add a short paragraph covering:

- suppressing a builtin for a project or a machine with `use: <plugin>@<slug>` +
  `agent_creatable: false`, and that existing beads of that type keep rendering
  normally;
- that the `use:` prefix always names the type's original provider (`builtin` for a
  builtin) no matter how many layers have already overridden the slug;
- that a machine-global entry in `~/.config/sase/sase.yml` applies to every project on
  the machine, including the home-level instruction files, and that a project's own
  `sase/sase.yml` entry wins over it.

### 8. Tests

`tests/task_types/test_project_config.py`:

- `test_use_override_prefix_names_the_original_provider`: two layers (a `replace` `user`
  layer and a `concatenate` `local` layer) both using `use: builtin@flake`; assert the
  final record carries the second layer's value and that **no** diagnostic is emitted.
  This test fails on `master` with `mismatched_use_prefix`.
- `test_use_override_still_rejects_a_wrong_provider_prefix_after_an_override`: same two
  layers, second one using a bogus `sase-github@flake`; assert `mismatched_use_prefix`
  and that the message names `builtin@flake` as the correct form.
- `test_machine_global_replay_skips_the_local_layer`: `user` and `local` layers both
  present, `include_local_layer=False`; assert only the user layer's change is applied.

New `tests/task_types/test_machine_global_specs.py` (or an added case in
`tests/task_types/test_builtin.py`):

- `machine_global_builtin_task_type_specs()` returns the five builtin slugs unchanged
  with no config;
- with a `user` layer disabling `feature`, the returned `feature` spec has
  `agent_creatable: False`;
- with a `local` layer disabling `bug`, `bug` is unaffected;
- a machine-global entry defining a brand-new slug does not appear in the result.

`tests/` coverage for the home note (extend whichever module already covers
`render_generated_task_types_memory_body`; find it with
`rg -l "render_generated_task_types_memory_body|_home_task_type_specs" tests/`):

- with a `user` layer disabling `feature`, the `include_project_memory=False` body has
  no `### \`feature\``heading, and still has`bug`, `ci`, `flake`, and `memory`.

`tests/task_types/test_flag_project_type.py` is the closest existing model for a
config-driven catalog test; mirror its fixture style.

Also check `tests/` for anything asserting the exact five-entry home note or the exact
`sase/task_types.json` bytes and update it for the new committed content.

### 9. Regenerate the sase project's memory

Run `sase memory init` in the sase repo. Expect `sase/memory/task_types.md`,
`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and
`sase/task_types.json` to change: `feature` stays in sase's Types section, and its
snapshot entry flips to `"source": "project"`, `"package": "project"` because the
project now overrides it. Commit the regenerated files with the change; do not hand-edit
them.

## Explicitly out of scope

- **Any new catalog-removal semantics.** No `enabled:` key, no
  `bead.disabled_task_types`, no way to delete a slug (D1).
- **Changing the builtin catalog.** `_feature_spec()` stays exactly as it is (D6).
- **`bob-cli`, `actstat`, and `bob-cli-z`.** No repo edits, no bead edits (D5).
- **The refusal message for a non-agent-creatable type.** `resolve_created_task_type()`
  appends the type's `when_to_use` after "cannot be created by agents", which reads as a
  contradiction for a type whose `when_to_use` was written as an invitation. `flag`'s
  `when_to_use` is deliberately written as the refusal explanation, so improving this
  means giving the spec a separate refusal field — a larger change than this plan, and
  only reachable by an agent that ignored its instruction note. File a follow-up task
  bead rather than fixing it here.
- **The `package` inconsistency between the two project-config paths.**
  `_apply_use_override()` writes `package="project"` while `_apply_new_project_type()`
  writes `package="sase"`, so a later-layer override of a config-_defined_ type must be
  spelled `sase@<slug>`. D3 makes that behavior stable and self-describing (the error
  message already names the right prefix), but does not unify the two values, because
  doing so would rewrite the committed `sase/task_types.json` entry for `flag`.
- **Task-type provenance in the generated notes.** Neither note says whether a type is
  builtin, plugin, or project — the ambiguity that produced this report. Making the note
  say so is a separate readability change with its own tradeoffs.

## Rust core backend boundary

No `../sase-core` work. The Rust core owns task-type spec validation
(`validate_task_type_spec`), the spec digest, and the snapshot codec; this change adds
no spec key, changes no wire shape, and produces no new digest input. Everything it
touches — replaying config layers, deciding which layers a given render consults, and
the provenance bookkeeping used to match a `use:` prefix — is registry assembly that
already lives in Python under `src/sase/task_types/`, and it is per-machine config
plumbing rather than shared domain behavior a web or editor frontend would need to
reimplement.

The one Rust-adjacent guard: `sase/task_types.json` is written through
`render_task_type_snapshot_json()` and parsed by the Rust codec, so step 9 must produce
that file with `sase memory init` rather than by hand.

## Verification

1. `just install`, then `just check`.
2. `just check-full` through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …` with a `--next` action), never
   inline.
3. `sase bead task-type list` in the sase repo still lists `feature` with
   `AGENTS = yes`; `sase bead task-type list -a` additionally lists `flag` and `github`.
4. `sase config doctor` (or `sase doctor`) reports no `beads.task_types` diagnostic — in
   particular no `mismatched_use_prefix` for `feature`. This is the direct regression
   check for Defect 1 against the real two-layer config.
5. `git diff` on the regenerated files shows `feature` still present in the sase repo's
   `AGENTS.md` Types section, and absent from the chezmoi `home/AGENTS.md` Types
   section.
6. `sase bead show bob-cli-z` is not part of this repo's verification, but confirm by
   inspection that nothing in the change removes a slug from any catalog.

## Definition of done

- A `use: builtin@<slug>` entry is accepted in any config layer regardless of whether an
  earlier layer already overrode that slug, with a regression test that fails on
  `master`.
- `apply_project_task_type_config()` can replay machine-global layers only, and
  `machine_global_builtin_task_type_specs()` is its one caller for the home note.
- `~/.config/sase/sase.yml` (via chezmoi) disables `feature` for every project;
  `sase/sase.yml` re-enables it for `sase`.
- The chezmoi home instruction files and `home/sase/memory/task_types.md` no longer
  document `feature`; the sase repo's still do.
- `docs/beads.md` documents the suppression idiom and the provider-prefix rule.
- `just check` passes, and a `just check-full` run started through `/sase_monitor`
  reports green.
