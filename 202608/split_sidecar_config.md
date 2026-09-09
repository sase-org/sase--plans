---
tier: epic
title: Split repos.sidecar into builtin and custom buckets
goal: "repos.sidecar is a two-bucket mapping (builtin for the reserved
  plans/beads/agents roles, custom for document sidecars such as research), keyed by
  role name, mirroring llm_provider.model_aliases; every enabled SASE project's config
  uses the new shape and the legacy list form is gone.

  "
phases:
  - id: dual_read
    title: Accept both shapes in the schema, parser, and doctor
    depends_on: []
    size: medium
    description: "dual_read: teach the JSON schema, the sidecar config parser, the
      memory-generation validator, and the CI bootstrap tool to read the new
      builtin/custom mapping while still accepting the legacy list, and add a doctor
      check that reports the migration and mis-bucketed roles.

      "
  - id: migrate_configs
    title: Write and migrate every enabled project to the new shape
    depends_on:
      - dual_read
    size: medium
    description: "migrate_configs: make sase repo init emit the two-bucket mapping,
      update the operator-facing guidance strings and docs examples, and migrate the
      sase, actstat, and bob-cli project configs to the new shape.

      "
  - id: drop_legacy
    title: Remove the legacy list form
    depends_on:
      - migrate_configs
    size: medium
    description:
      "drop_legacy: delete the legacy list branch from the schema, parser, memory
      validator, and CI bootstrap tool so repos.sidecar is a closed builtin/custom
      object, and land it as a breaking change."
proposed_by: bbugyi200.athena.um
status: done
bead_id: sase-gu
create_time: 2026-09-09 19:51:37
---

- **PROMPT:**
  [prompts/202608/split_sidecar_config.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/split_sidecar_config.md)
- **BEAD:**
  [sase-gu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gu/README.md)

# Plan: Split `repos.sidecar` into `builtin` and `custom`

## Problem

`repos.sidecar` is a flat list of entries, each carrying a `name` that doubles as the
sidecar role:

```yaml
repos:
  sidecar:
    - name: plans
      auto_clone: true
    - name: beads
      auto_clone: true
    - name: research
      description: Durable SASE research reports and generated media.
    - name: agents
      description:
        Hidden sidecar that stores commit-associated sase agent data for this project.
```

The list mixes two different kinds of entry. `plans`, `beads`, and `agents` are the
reserved roles (`RESERVED_SIDECAR_ROLES` in `src/sase/sdd/_store_types.py`); SASE
injects them for every managed project and a config entry only _overrides_ them. Every
other role — `research` here — is a user-declared document sidecar that exists only
because config declares it. Nothing in the config surface expresses that split, so a
typo'd `plas` role reads as a brand-new document sidecar rather than an error, and layer
merging needs bespoke identity matching (`_merged_sidecar_entries_cached` intersects
`{role, slug}` token sets) purely because config lists concatenate across layers.

`llm_provider.model_aliases` already solved exactly this problem: it splits into
`builtin` (overrides of SASE-owned names) and `custom` (user-defined names), both keyed
maps rather than lists. This plan applies that same shape to sidecars.

## Target shape

```yaml
repos:
  sidecar:
    builtin:
      plans:
        auto_clone: true
      beads:
        auto_clone: true
      agents:
        description:
          Hidden sidecar that stores commit-associated sase agent data for this project.
    custom:
      research:
        description: Durable SASE research reports and generated media.
```

Decisions, and why:

1. **Maps keyed by role, not lists.** This is the `model_aliases` shape. The map key
   replaces the `name:` field, so a duplicate role is unrepresentable, and layer merging
   becomes the ordinary `deep_merge` behavior in `src/sase/config/loading.py` — a
   project-local `custom.research.disabled: true` merges into the global
   `custom.research` entry per key. The bespoke `{role, slug}` token merge in
   `_merged_sidecar_entries_cached` exists only to emulate that over concatenated lists
   and is deleted in `drop_legacy`.
2. **`builtin` holds exactly `plans`, `beads`, `agents`.** These are
   `RESERVED_SIDECAR_ROLES`. `custom` holds every other role and must not name a
   reserved one. The schema can enforce this with `propertyNames`, which the model-alias
   split could not do (its builtin set is provider-dependent).
3. **Entry fields are unchanged** — `repo`, `description`, `auto_clone`, `visibility`,
   `disabled` — minus `name`, which becomes the key. A leftover `name:` inside an entry
   is rejected by `additionalProperties: false` and called out by the doctor check.
4. **`description` stays optional in the schema**, unlike `model_aliases.custom` where
   it is required. A later config layer must be able to write
   `custom: {research: {disabled: true}}` to opt out of an inherited sidecar, and
   requiring a description would break that minimal disable form. The existing
   description requirement for lazy enabled sidecars stays where it already lives
   (memory generation in `src/sase/main/init_memory/config.py`), and the new doctor
   check warns about custom entries that are enabled, lazy, and undescribed.
5. **Ordering is a rule, not authoring order across buckets.** Sidecar order is
   user-visible (repo inventory rows, generated agent instructions,
   `configured_sidecar_roles`). The parser emits `builtin` roles first in the canonical
   `plans, beads, agents` order, then `custom` roles in configured order. Pin this with
   a test; do not leave it to dict iteration accident.
6. **Expand, migrate, contract — in that order.** The installed `sase` reads _other_
   projects' configs (`actstat`, `bob-cli`). A single hard cut would silently drop those
   projects' `research` sidecars for the window between landing here and editing their
   repos. The three phases below make that window impossible.

## Scope facts established during research

- **No Rust core change is needed.** `crates/sase_core/src/config/merge.rs` merges
  generically (dicts recurse, lists concatenate/replace per layer strategy) and
  `crates/sase_core/src/config/schema.rs` builds the field model straight from the JSON
  schema: an object with named `properties` becomes a container, an open object becomes
  a `"map"` leaf. `repos.sidecar` therefore re-classifies from `"array"` to a container
  with two `"map"` children with no Rust edit. `sase_xprompt_lsp` has no `repos.sidecar`
  knowledge.
- **No plugin change is needed.** `sase-github`, `sase-telegram`, and `sase-nvim` never
  parse `repos.sidecar`.
- **The global `~/.config/sase/sase.yml` has no `repos.sidecar` block** — only
  `repos.linked`. Nothing to migrate in chezmoi. Do not confuse this with
  `file_hooks[].sidecars`, an unrelated list of role names that this plan must leave
  untouched.
- **Enabled projects are `sase`, `actstat`, and `bob-cli`**
  (`sase project list --json`). `actstat` and `bob-cli` both declare all four roles in
  the legacy list form.
- **Raw-config readers that bypass the parser** and must be updated independently:
  `src/sase/main/init_memory/config.py` (`_sidecar_repos_raw`),
  `tools/ci_bootstrap_sidecars` (`plan_sidecars`), and
  `src/sase/main/_repo_init_config.py` (`explicit_sidecar_config_update`, the
  ruamel-based writer). Everything else goes through
  `merged_sidecar_entries_from_config` / `configured_sidecar_roles` and needs no change.

## Non-goals

- Changing the internal normalized entry dict (`name`, `auto_clone`,
  `_sase_sidecar_role`, …) that `merged_sidecar_entries_from_config` returns. Keeping it
  identical is what confines this change to the parsing boundary; `repo_inventory.py`,
  `linked_repos.py`, `_linked_repo_paths.py`, `sdd/env.py`, `main/repo_handler_path.py`,
  and `_repo_init_config.configured_sidecar_specs` stay as they are.
- Changing repo-inventory source labels. `sase repo list` keeps reporting
  `repos.sidecar config`; a per-bucket label is not worth the test churn.
- Touching `repos.linked`, the deprecated `linked_repos`/`sibling_repos` aliases, or
  `file_hooks[].sidecars`.
- Editing any `sase/memory/*.md` note. The repository section of `AGENTS.md` is
  generated from config and regenerates itself; canonical memory notes need explicit
  user permission and are out of scope.

---

## Phase `dual_read`: Accept both shapes in the schema, parser, and doctor

Every reader learns the new shape while the legacy list keeps working exactly as it does
today.

### Schema — `src/sase/config/sase.schema.json`

- Add a `sidecarRepoEntry` definition: the current `sidecarRepo` definition minus
  `name`, still `additionalProperties: false`, with no required fields.
- Make `properties.repos.properties.sidecar` a `oneOf` of the legacy array
  (`items: {$ref: sidecarRepo}`) and the new object:
  - `additionalProperties: false`, properties `builtin` and `custom`.
  - `builtin`: object, `propertyNames: {enum: ["plans", "beads", "agents"]}`,
    `additionalProperties: {$ref: sidecarRepoEntry}`.
  - `custom`: object,
    `propertyNames: {minLength: 1, not: {enum: ["plans", "beads", "agents"]}}`,
    `additionalProperties: {$ref: sidecarRepoEntry}`.
  - Descriptions must state which roles belong in which bucket and that the key is the
    role name.
- Change the `repos.sidecar` default from `[]` to `{}`, and note the legacy array branch
  as deprecated in its description so editors surface it.
- `src/sase/default_config.yml`: `sidecar: []` becomes
  `sidecar: {builtin: {}, custom: {}}` (with a comment matching the file's style).
  Cross-layer safety: when a project still uses a list, `deep_merge` sees dict base vs
  list override and the override wins wholesale, so the legacy path is unaffected.

### Parser — `src/sase/_linked_repo_config.py`

- Add a private `_sidecar_config_entries(config)` that returns entry mappings with
  `name` injected:
  - mapping input: `builtin` roles in canonical `plans, beads, agents` order, then
    `custom` roles in configured order; skip non-mapping values defensively (runtime
    parsing stays forgiving, exactly like `get_custom_model_aliases`); a role appearing
    in both buckets resolves to the `custom` entry, matching
    `model_alias_config_source`'s "custom wins" rule, and is reported by doctor.
  - list input: today's behavior, unchanged.
- `_merged_sidecar_entries_cached` calls it instead of
  `_entries_for_repos_key(config, REPOS_SIDECAR_CONFIG_KEY)`. Keep the `{role, slug}`
  token merge for now; it is a no-op for mapping input (keys already deduplicate) and is
  still required for the legacy list.
- Everything downstream — normalization, `configured_sidecar_roles`,
  `inject_default_linked_repos` — is untouched.

### Other raw readers

- `src/sase/main/init_memory/config.py`: `_sidecar_repos_raw` should yield
  `(label, role, entry)` triples so the mapping form reports
  `repos.sidecar.custom['research']` and the legacy form keeps reporting
  `repos.sidecar[0] ('research')`. The role comes from the map key instead of
  `entry["name"]`; every existing validation rule (`auto_clone`/`disabled` must be
  boolean, enabled lazy entries require a description, `repo` must be a usable string)
  is preserved verbatim.
- `tools/ci_bootstrap_sidecars`: `plan_sidecars` accepts a mapping (iterate `builtin`
  then `custom`, role from the key) as well as the current sequence.

### Doctor — new `src/sase/doctor/checks_config_repos.py`

Model it on `src/sase/doctor/checks_config_model_aliases.py`: read raw config, collect
`problems` with `key`/`message` entries, cap details with `MAX_DETAIL_ROWS` from
`checks_config_common`. Report:

- `repos.sidecar` is a list — name each entry's target bucket
  (`repos.sidecar.builtin.<role>` for reserved roles, `repos.sidecar.custom.<role>`
  otherwise) so the message is directly actionable.
- a reserved role under `custom`, or a non-reserved role under `builtin`.
- a role defined in both buckets (state that `custom` wins).
- an entry that still carries a `name:` key.
- `builtin`/`custom` that is not a mapping, or an entry value that is not a mapping.
- an enabled, non-`auto_clone` `custom` entry with no usable `description` (it will fail
  `sase init memory`).

Register it in `src/sase/doctor/checks_config.py` as
`CheckSpec(id="config.repos", group="config", title="Sidecar repo config")`, following
the module's existing ordering and its `_check_<name> = check_<name>` alias convention.

### Tests

- `tests/test_config_schema.py`: the mapping form validates; a reserved role under
  `custom` and a `name:` key inside an entry both fail; the legacy list still validates.
- Sidecar parser tests (extend the existing modules that cover
  `merged_sidecar_entries_from_config` and `configured_sidecar_roles`): both shapes
  produce identical normalized entries; the emitted role order is
  `plans, beads, agents`, then custom roles in configured order; a project-local
  `disabled: true` under `custom.<role>` still suppresses an inherited entry.
- New doctor test module mirroring `tests/doctor/test_checks_config.py`'s style, one
  case per problem class.
- `tests/main/test_init_memory_handler_repositories.py`: keep the legacy-label case and
  add its mapping equivalent asserting the `repos.sidecar.custom['research']` label.
- CI bootstrap tool tests, if the tool has them; otherwise cover `plan_sidecars`
  directly.

### Verification

`just install`, then `just check`. Run `just check-full` if the scoped lane escalates or
reports an unusual selection.

---

## Phase `migrate_configs`: Write and migrate every enabled project to the new shape

The canonical shape becomes what SASE writes, documents, and what every enabled
project's config uses.

### Writer — `src/sase/main/_repo_init_config.py`

`explicit_sidecar_config_update` currently loads `repos.sidecar` as a `MutableSequence`
and appends `CommentedMap` entries carrying `name`. Rewrite it to build the mapping
form:

- `plans` (`auto_clone: true`), `beads` (`auto_clone: true`), and `agents`
  (`description: DEFAULT_AGENTS_DESCRIPTION`) under `builtin`; `research`
  (`description: DEFAULT_RESEARCH_DESCRIPTION`) under `custom`.
- Existing-role detection reads both buckets, so an already-declared role — including
  one with `disabled: true` — is preserved verbatim, which is the current contract.
- If `repos.sidecar` is still a list, return a `ConfigUpdate` whose `error` says to
  migrate to the builtin/custom mapping (pointing at `sase doctor`) rather than mixing
  shapes in one file. Do not attempt an automatic comment-preserving list-to-map
  rewrite.
- Keep using `set_key` for the absent case and `dump_yaml` for the in-place case;
  `added_roles` and `sidecar_config_action_detail` reporting are unchanged.

### Operator-facing strings

`src/sase/main/_repo_init_sidecars.py:188` must say
`repos.sidecar.builtin.agents.visibility: private`.

### Docs

- `docs/configuration.md`: rewrite the `repos.sidecar` prose and YAML example for the
  two buckets; replace the six `repos.sidecar[].*` table rows with
  `repos.sidecar.builtin.<role>.*` / `repos.sidecar.custom.<role>.*` rows, dropping the
  `name` row and explaining that the key is the role. State that the legacy list is
  accepted only until the migration completes.
- `docs/init.md`: update the `sase repo init` YAML block to the new shape.
- Grep the rest of `docs/` for `sidecar:` YAML examples (`docs/sdd_storage.md`,
  `docs/ace.md`, `docs/xprompt.md` are candidates) and update any that show
  `repos.sidecar` config. Leave `file_hooks[].sidecars` examples alone.

### Config migration

- `sase/sase.yml` in this repo: `plans`, `beads`, `agents` under `builtin`; `research`
  under `custom`, descriptions preserved character for character.
- `actstat` and `bob-cli`: open each with `/sase_repo`
  (`sase repo open actstat -r "..."`), edit `<printed path>/sase/sase.yml` into the same
  shape preserving each project's descriptions, and commit in that checkout with the
  `/sase_git_commit` skill. Both currently declare `plans`, `research`, `agents`,
  `beads` in the legacy list. These are separate repositories, so they are separate
  commits — never edit them through any path other than the one `sase repo open` prints.
- Re-check that `~/.config/sase/sase.yml` still has no `repos.sidecar` block before
  finishing; if one has appeared, migrate it in the chezmoi source
  (`sase repo open chezmoi`), not in the deployed file.

### Verification

- `just install`, then `just check`.
- `sase doctor` reports no `config.repos` problems for this project.
- `sase repo list` shows the same sidecar rows, in the same order, as before the
  migration.
- Regenerating agent instructions leaves `AGENTS.md` and its provider shims unchanged —
  descriptions are carried over verbatim, so a diff here means the migration lost data.
- In each migrated sibling project, `sase repo list` still shows all four sidecars.

---

## Phase `drop_legacy`: Remove the legacy list form

A breaking change, landed only once every enabled project reads clean.

- `src/sase/config/sase.schema.json`: delete the `oneOf` array branch, making
  `repos.sidecar` a plain closed object with `builtin` and `custom`. Fold
  `sidecarRepoEntry` back into the `sidecarRepo` name if nothing else references it, and
  delete the now-unused definition. Drop the deprecation wording.
- `src/sase/_linked_repo_config.py`: delete the list branch from
  `_sidecar_config_entries`, and delete the `{role, slug}` token-intersection merge in
  `_merged_sidecar_entries_cached` — mapping keys already deduplicate, so the function
  reduces to normalization. Keep `resolve_sidecar_repo_identity` and the normalized
  entry shape exactly as they are.
- `src/sase/main/init_memory/config.py` and `tools/ci_bootstrap_sidecars`: a list-valued
  `repos.sidecar` becomes an explicit error naming the required mapping shape, not a
  silently ignored value.
- `src/sase/doctor/checks_config_repos.py`: keep the legacy-list problem — it is now the
  migration diagnostic for stale configs — and reword it as a required migration.
- `docs/configuration.md` and `docs/init.md`: remove the compatibility-window wording.
- Tests: delete the legacy-shape cases added in `dual_read`, and add a case asserting
  the list form is now rejected by both the schema and the memory validator.
- Commit as `feat(repos)!:` with a `BREAKING CHANGE:` footer stating that
  `repos.sidecar` is no longer a list and that reserved roles go under
  `repos.sidecar.builtin` while document sidecars go under `repos.sidecar.custom`.

### Verification

`just install`, then `just check-full` — this phase touches the schema and the config
parser, which is squarely in the broadening set. Also confirm `sase doctor` is clean and
`sase repo list` output is unchanged for this project.

## Risks

- **A missed raw reader silently drops a sidecar.** The three known bypass sites are
  listed above; before finishing `dual_read`, re-grep for `REPOS_SIDECAR_CONFIG_KEY`,
  `"sidecar"`, and `repos.sidecar` across `src/` and `tools/` to confirm nothing else
  reads the raw value.
- **`propertyNames` enforcement.** The bundled JSON schema is consumed both by
  yaml-language-server in editors and by the Rust validator; `propertyNames` support in
  the latter is unverified. The doctor check is the real enforcement, which is why it is
  not optional.
- **Ordering regressions.** Sidecar order is visible in generated instructions. The
  order test in `dual_read` is what keeps the bucket split from reshuffling agent-facing
  output.
- **Sibling repos are read by the shared installed `sase`.** This is why `drop_legacy`
  depends on `migrate_configs` and must not land first.
