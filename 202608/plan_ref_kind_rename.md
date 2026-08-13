---
tier: epic
title: Rename the plans-sidecar artifact ref kind to plan
goal: The plans sidecar's document reference kind is spelled `plan` everywhere SASE
  writes it — `plan:<path>` in machine fields and `@plan:<path>` in prose — the `plans:`
  spelling is never emitted again, and every live `plans:` reference on this machine
  has been migrated.
phases:
- id: core
  title: Rename the SDD plan-reference grammar in sase-core
  depends_on: []
  size: small
  description: 'core: rename PLAN_REFERENCE_KIND/PREFIX to plan in crates/sase_core/src/plan/refs.rs,
    keep `plans:` as a read-only input alias that re-renders canonically, and update
    the Rust tests and the plan_reference_render kind guard.'
- id: python
  title: Switch every Python plan-reference literal to plan
  depends_on:
  - core
  size: medium
  description: 'python: replace the functional `plans:` literals in src/sase with
    a single shared constant set to `plan:`, update CLI help and docstrings, add the
    sidecar ref-kind naming regression test, and fix the 82 affected test files plus
    any visual snapshots.'
- id: beads
  title: Migrate bead design references
  depends_on:
  - python
  size: medium
  description: 'beads: add a prefix-only fast path to the design-ref repair planner
    so an alias-spelled reference is rewritten without needing its plan file on disk,
    then run the repair over this machine''s bead store and commit the result.'
- id: prose
  title: Rewrite prose and remaining stored references
  depends_on:
  - python
  size: medium
  description: 'prose: update the documentation that describes the grammar, then sweep
    prose `plans:<path>` citations to `@plan:<path>` across docs, the plans sidecar,
    ~/.sase/plans, and the two small stored-data sites, without touching immutable
    history.'
- id: land
  title: Verify and land the rename
  depends_on:
  - beads
  - prose
  size: small
  description: 'land: run the exhaustive verification lane over the combined tree,
    confirm no emitter can produce `plans:` again, and land the epic.'
proposed_by: bbugyi200.athena.zl.f1
create_time: 2026-08-13 12:21:26
status: wip
bead_id: sase-ky
---

- **PROMPT:** [prompts/202608/plan_ref_kind_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plan_ref_kind_rename.md)
- **BEAD:** [sase-ky](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ky/README.md)

# Plan: Rename the plans-sidecar artifact ref kind to plan

## Problem

The plans sidecar has **two** reference spellings, from two independent grammars:

1. **The typed artifact-ref grammar** (`crates/sase_core/src/artifact_ref/`) already
   calls this kind **`plan`**. `sase.sidecar_ref_config._BUILTIN_SIDECAR_REF_KIND` maps
   the `plans` sidecar role to ref kind `plan`, so `artifact_ref_context()` reports
   `ArtifactRefDocumentRoot(kind='plan', …)` today, and `@plan:<path>` resolves. `plans`
   is registered in `artifact_ref/kinds.rs` as a _deprecated alias_ of `plan` with the
   diagnostic `@plans: is deprecated; write @plan:<path>`.

2. **The SDD plan-reference grammar** (`crates/sase_core/src/plan/refs.rs`) still spells
   the same thing **`plans:`**. `PLAN_REFERENCE_KIND = "plans"` and
   `PLAN_REFERENCE_PREFIX = "plans:"`, and that is what every emitter writes: bead
   `design` fields, SDD association keys, plan-header canonical refs, agent
   prompt-archive plan labels, the axe SDD accept path, and the CLI's own help text.

So the deprecation from `202608/artifact_ref_contract.md` (§1043: "`@plans:<path>` →
`@plan:<path>`, deprecated alias for one release") was only ever half-completed. Grammar
2 keeps minting new `plans:` strings, which is why the spelling has not gone away.

This epic finishes the rename in grammar 2 and migrates the live references this machine
has already accumulated.

### Already supported — do not build it

The user's prompt anticipated needing "support to sidecar repo refs for defining a ref
name that is different from their repo name". **That support already exists**, in two
layers, and was verified by reading the code:

- `sase/sidecar_ref_config.py::_BUILTIN_SIDECAR_REF_KIND` maps the builtin roles
  `plans → plan`, `beads → bead`, `agents → agent`; `_sidecar_role_ref_kind()` falls
  back to the role name only for roles it does not know.
- Any sidecar can override its kind declaratively. `REF_KIND_CONFIG_KEY` is in
  `_KNOWN_REF_CONFIG_KEYS`, `_ref_override()` copies it into the spec, and
  `artifact_providers/_validation.py::validate_ref_providers` treats `ref.kind` as a
  free-form string keyed independently of the role — it only rejects a _duplicate_ kind
  across providers. `docs/artifact_references.md` already shows the inline form
  (`role: design` with `ref: {kind: design}`), and `use:` + `kind:` deep-merges the same
  way.

The `python` phase therefore adds a regression test that pins this behavior instead of
adding a mechanism, and the `prose` phase documents the rule explicitly.

## The two canonical spellings (apply this rule consistently)

| Context                                                                                                                                                                        | Form           | Why                                                                                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Machine fields and CLI arguments — bead `design`, Patch `REFS:` entries, `SASE_PLAN=` commit tags, SDD association keys, plan-header canonical refs, `sase plan show <TARGET>` | `plan:<path>`  | This is what `parse_artifact_ref` renders. It explicitly **rejects** the `@` sigil (`artifact reference must not include the prompt '@' sigil`), and the sibling document kind `research:` already persists bare — `REFS:\n  research:202607/report.md`. |
| Markdown prose — docs, plan bodies, bead notes and descriptions, skill text, source comments                                                                                   | `@plan:<path>` | `@` is the prompt sigil; a citation in prose is what the user asked for, and it is the form that expands when the prose is fed to an agent.                                                                                                              |

Both spellings say `plan`, never `plans`. Where this plan says "the canonical spelling",
it means `plan:` — the machine form.

## One deliberate deviation from "no backward compatibility required"

The prompt says no backward compatibility is needed. That holds for everything SASE can
rewrite. It cannot hold for **immutable history that already exists on this machine**:

- `SASE_PLAN=plans:<path>` trailers are in thousands of commits across the primary repo
  and its sidecars. `docs/commit_workflows.md` documents the commit modal loading the
  last `SASE_PLAN` footer tag, and `sase plan links refresh` re-derives association
  sections from those footers.
- Bead **event streams** are append-only (571 `plans:` occurrences under
  `sase/repos/beads/events/streams/`). Rewriting them would corrupt the canonical log
  that `issues.jsonl` is projected from.

So `parse_plan_reference` must keep **reading** `plans:` forever. It just stops
**writing** it: the parse normalizes the alias and re-renders as `plan:<path>`, so every
call site that round-trips through `rendered` migrates for free. This costs about five
lines in `refs.rs` and is what makes the `beads` phase's migration safe and complete.

If the reviewer would rather have `plans:` hard-fail, say so at approval and the `core`
phase can drop the alias — but then `sase plan links refresh`, bead history replay, and
the commit modal's plan pickup will each start missing older records, and that breakage
is not otherwise repairable.

## Non-goals — do not change these

- **The sidecar role name stays `plans`.** `PLANS_SIDECAR_ROLE = "plans"`, the sidecar
  repo `<project>--plans`, `store.kind_root("plans")`, `sase repo path plans`,
  `sase repo open plans`, and every `role == "plans"` test in the source tree are
  identity and storage. This includes `artifact_ref_context.py:73`, `sdd/_paths.py:66`,
  `sdd/_sidecar_init.py:305`, and `agents/cli_prompts.py:233`.
- **The ACE Artifacts sub-tab id stays `"plans"`.** `artifact_tabs.py:51`
  (`"plans": "ref:plan"`), `ARTIFACTS_ACCENTS["plans"]`, and every `subtab == "plans"`
  branch in `ace/tui/actions/clipboard/` and
  `widgets/artifacts/plans_data_sources.py:210` are sub-tab identity, not ref kinds.
- **`_LEGACY_PLAN_PREFIXES` and `LEGACY_PLAN_MARKERS`** — `plans/`, `sdd/plans/`,
  `.sase/sdd/plans/`, `sase/repos/plans/` are _directory_ path markers, not the ref
  prefix. They stay exactly as they are.
- **The `plans` entry in `artifact_ref/kinds.rs`.** It is already deprecated, already
  carries the right diagnostic, and already canonicalizes to `plan`. Leave it.
- **Immutable history.** Do not rewrite git history or commit trailers, bead event
  streams, agent prompt archives (`repos/agents/prompts/**`), hood `snapshot.json`
  files, `~/.sase/chats/`, `~/.sase/workflows/`, `~/.sase/dismissed_bundles/`,
  `~/.sase/interaction_requests/`, or `~/.sase/projects/*/artifacts/**`. Together these
  hold well over 90% of the machine's ~12,000 `plans:`-bearing files, and every one of
  them is a record of what was true at the time.
- **`CHANGELOG.md`** — its `plans:` occurrences are release-note scopes
  (`* **plans:** add filter query …`), not references.
- **`sase/memory/sase_sizes.md:11`** — "task beads, and tale plans: `xsmall`, …" is
  English, not a reference. No memory file needs to change in this epic; do not edit
  `sase/memory/`, `AGENTS.md`, or the generated provider shims.

---

## Phase `core`: Rename the SDD plan-reference grammar in sase-core

All of this phase lands in the sibling Rust repo. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "<reason>"`) and use only the path it prints.

### The change — `crates/sase_core/src/plan/refs.rs`

```rust
const PLAN_REFERENCE_KIND: &str = "plan";
const PLAN_REFERENCE_PREFIX: &str = "plan:";
/// Read-only: persisted history (commit trailers, bead event streams) still
/// carries this spelling. Never emitted.
const LEGACY_PLAN_REFERENCE_PREFIX: &str = "plans:";
```

`parse_plan_reference` strips `PLAN_REFERENCE_PREFIX` first, then
`LEGACY_PLAN_REFERENCE_PREFIX`. **Both** produce `legacy: false`, `kind: "plan"`, and
`rendered: format!("plan:{path}")`. That last detail is the whole migration mechanism:
`rendered` is what `sdd/associations/_normalization.py`, `sdd/plan_header_writes.py`,
and the `beads` phase's repair planner persist, so an alias-spelled input silently
becomes canonical on its next write. Path validation is identical for both prefixes.

`render_plan_reference(kind, path)` accepts `"plan"`. Keep accepting `"plans"` as an
input _kind_ argument and return the canonical `plan:` string — the only Python caller
passes a kind it read back out of a parse, so tolerating it costs nothing.

`canonicalize_plan_reference` needs no change beyond the constants; it already renders
through `render_plan_reference`.

`legacy_payload_and_candidates` and `LEGACY_PLAN_MARKERS` are untouched — those are
directory markers (see Non-goals).

### Wire versions

**No wire schema version moves.** `PlanReferenceResolutionWire` keeps its shape and
`PLAN_REFERENCE_RESOLUTION_WIRE_SCHEMA_VERSION` stays `1`; `ParsedPlanReference`'s
fields are unchanged. There is nothing for `tools/validate_sase_core_rs` to bump, and no
coordinated five-file edit in the sase repo. Confirm this by grepping for
`PLAN_REFERENCE_RESOLUTION_WIRE_SCHEMA_VERSION` in both trees before declaring the phase
done — if a version _is_ required, stop and record a `PROPOSED FOLLOW-UP:` note rather
than inventing a bump.

### Rust tests

Update `refs.rs`'s test module in place:

- `typed_reference_round_trips` — expect `plan:202607/durable.md` and `kind: "plan"`.
- `typed_reference_rejects_unsafe_payloads_and_unknown_kinds` — switch the five rejected
  payloads to the `plan:` prefix and keep the five stable validation messages
  byte-identical (`plan reference path must not be empty`, etc. — the messages name
  "plan reference", not the prefix, so they do not change). Keep
  `render_plan_reference("research", …)` rejecting.
- `unregistered_schemes_and_paths_remain_legacy` — `"research:202607/plan.md"`,
  `"sdd/plans/202607/plan.md"`, `r"old\plan.md"` all still parse as legacy. Note that
  `plans:202607/plan.md` is now explicitly **not** in this set.
- `canonicalize_uses_first_matching_root`, `exact_resolution_uses_first_root`,
  `unique_month_drift_resolves`, `multiple_month_drift_matches_are_ambiguous`,
  `missing_keeps_ordered_best_candidates` — swap the prefix in inputs and expectations.
- `every_legacy_form_resolves_to_the_same_file` — keep every existing value and **add**
  `"plans:202607/plan.md"`, proving the read alias resolves to the same file.
- Add `alias_prefix_reparses_as_canonical`: `parse_plan_reference("plans:202607/x.md")`
  returns `legacy: false`, `kind == "plan"`, `rendered == "plan:202607/x.md"`.

Then grep the rest of the crate and `crates/sase_core_py` for `plans:` string literals
(`plan/read.rs`, `plan/search.rs`, `bead/read.rs`, `bead/cli.rs`, `agent_stats/`, and
the `tests/*_parity.rs` files all mention it) and fix any that assert on the grammar.
The `artifact_ref/mod.rs` occurrences are the _other_ grammar's alias tests — leave
them.

### Done when

`cargo test` is green in `sase-core`, `parse_plan_reference` renders only `plan:`, and
`plans:` still resolves to the same file it did before.

---

## Phase `python`: Switch every Python plan-reference literal to plan

`just install` first — the workspace virtualenv rebuilds `sase_core_rs` from the
`sase-core` checkout, so this phase picks up `core` automatically.

### 1. One shared constant

Today the prefix is retyped as a bare literal in nine modules, and the one named
constant that exists — `LOGICAL_PLAN_REFERENCE_PREFIX` in
`ace/tui/widgets/prompt_panel/_file_path_hints.py:14` — is not importable from anywhere
sensible. Promote it: export `PLAN_REFERENCE_KIND = "plan"` and
`PLAN_REFERENCE_PREFIX = "plan:"` from `sase/sdd/plan_refs.py` (it already owns the
grammar's Python surface), re-export the old name from `_file_path_hints.py` if the TUI
module wants to keep its local alias, and import the constant at every site below rather
than retyping `"plan:"`.

### 2. Functional sites — verified line numbers at this revision

| File                                               | Lines              | Change                                                                                                                                               |
| -------------------------------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `artifact_ref_entries.py`                          | 116, 128           | `reference.startswith("plans:")` → the canonical prefix                                                                                              |
| `artifact_ref_entries.py`                          | 145                | the archive-role fallback `else "plans"` → `"plan"`; it is interpolated straight into `parse_artifact_ref(f"{role}:{relpath}")`, so it is a ref kind |
| `artifact_ref_entries.py`                          | 168                | `parsed.kind == "plans"` → `"plan"`                                                                                                                  |
| `bead_pages/rendering_identity.py`                 | 286                | `plan_ref.removeprefix("plans:")` → canonical prefix                                                                                                 |
| `bead_pages/rendering_identity.py`                 | 313                | `parsed.kind == "plans"` → `"plan"`                                                                                                                  |
| `ace/tui/widgets/prompt_panel/_file_path_hints.py` | 14                 | `LOGICAL_PLAN_REFERENCE_PREFIX = "plans:"` → `"plan:"`                                                                                               |
| `bead/design_ref_repair.py`                        | 200                | the `removeprefix` fallback — accept **both** spellings here, since it parses stored history                                                         |
| `sdd/associations/_normalization.py`               | 25, 47, 68, 90     | docstring, `parsed.kind == "plans"`, and both `f"plans:{…}"` emitters                                                                                |
| `sdd/plan_header_writes.py`                        | 321, 324, 330, 334 | the kind check, the label strip, and both `f"plans:{label}"` emitters                                                                                |
| `agents_sync/prompt_archive/preparation.py`        | 473, 480           | the `startswith`/`f"plans:{raw}"` normalizer and the label strip                                                                                     |
| `agents_sync/prompt_archive/validation.py`         | 305                | `.removeprefix("plans:")` — keep the adjacent `.removeprefix("plans/")`, that is the directory marker                                                |
| `axe/run_agent_exec_plan_accept_sdd.py`            | 50                 | `plan_ref=f"plans:{yyyymm}/{plan_name}.md"` → canonical                                                                                              |
| `plan_show/resolve.py`                             | 4, 293             | docstring and the `removeprefix`                                                                                                                     |

Where a site strips a prefix off _stored_ input (`design_ref_repair.py:200`,
`plan_show/resolve.py:293`, `agents_sync/prompt_archive/validation.py:305`,
`prompt_archive/preparation.py:480`), strip the canonical prefix **and** the legacy one,
for the same immutable-history reason as the `core` phase.

### 3. Docstrings and CLI help

- `sdd/plan_refs.py:184`, `bead/cli_common.py:345`, `core/artifact_file_helpers.py:210`
  — docstrings that name the `plans:` prefix.
- `main/parser_plan.py:148, 400, 415, 419` — `sase plan links refresh --plan …` and the
  three `sase plan show` examples.
- `main/parser_bead_lifecycle.py:149` — `sase bead create -T plan(plans:…)`.
- `main/parser_artifact.py:41` — `sase artifact path plans:…`.
- `bead/cli_admin.py:426` — the `--fix-design-refs` help line, if it names the prefix.

CLI examples are arguments, so they use the bare `plan:` form.

### 4. The sidecar ref-kind naming regression test

Add `tests/test_sidecar_ref_kind_naming.py` (or extend the nearest existing
`sidecar_ref_config` test module) pinning the behavior described under "Already
supported":

- `effective_sidecar_ref_policies` for the `plans` role yields
  `SidecarRefPolicy.ref_kind == "plan"` with no config at all;
- a custom sidecar role `notebooks` with `ref: {kind: note, icon: "◆"}` yields
  `ref_kind == "note"` — a ref name that differs from the repo/role name, configured,
  not hardcoded;
- the same override applied on top of `ref: {use: <installed provider>}` still wins;
- an unknown role with no `ref:` config falls back to its own role name as the kind.

### 5. Tests

82 test files hold 314 `plans:`-shaped occurrences. Most are one-line fixture strings.
Work file by file and classify each: a **ref** becomes `plan:`; a **role**, **sub-tab
id**, or **directory path** stays. The high-signal ones:

- `tests/sdd_store/test_plan_refs.py`, `tests/sdd_store/test_plan_ref_display.py`,
  `tests/test_plan_reference_resolver_integration.py`, `tests/test_plan_documents.py`,
  `tests/test_plan_display.py` — the grammar's own tests. Add a case to
  `test_plan_refs.py` asserting the read alias: parsing `plans:202607/x.md` yields
  `kind == "plan"` and `rendered == "plan:202607/x.md"`.
- `tests/test_changespec_refs_persistence.py`, `tests/test_changespec_refs_cli.py`,
  `tests/doctor/test_checks_changespec_refs.py` — Patch `REFS:` entries, bare form.
- `tests/test_commit_publication_inline.py`, `tests/test_commit_workflow_publication.py`
  — `SASE_PLAN=` trailers. Change the _expected new_ trailer to `plan:`; keep at least
  one fixture asserting an old `plans:` trailer still reads.
- `tests/test_file_path_hints.py` (121, 127, 309) — the TUI hint regex.
- `tests/ace/tui/**` (~25 files) — Artifacts plan rows, copy-as palette contexts, and
  artifact-ref completion.
- `tests/test_plan_validate.py:178, 185` — a `parent:` frontmatter value; note that
  `parent:` frontmatter is deprecated in favor of the `PARENT` bullet, so only the
  spelling changes.

### 6. Visual snapshots

`tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py` and
`tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py` both carry `plans:` fixture
strings, so the rendered reference text in those PNGs changes and gets one character
shorter. Regenerate and then verify clean:

```bash
just test-visual --sase-update-visual-snapshots
just test-visual
```

Eyeball at least one regenerated clan-panel PNG to confirm only the reference text
moved. The suite is slow; run it in the background and check back, and note that
`/sase_monitor` is currently unreliable.

### Done when

`just check` passes, `just test-visual` is clean, and no _functional_ `plans:` literal
remains in `src/sase/`. Two greps, because the path form and the bare-prefix form need
different patterns:

```bash
grep -rn 'plans:[0-9A-Za-z_./~-]' src/sase/   # plan-file citations
grep -rn "plans:\`\|\"plans:\"\|'plans:'\|plans:{" src/sase/   # prefix literals
```

The first should return only the four CLI-help/skill examples and the two module
docstrings the `prose` phase owns. The second should return only the read-side aliases
listed in §2, each carrying a comment naming the immutable-history reason.

---

## Phase `beads`: Migrate bead design references

This machine's bead store holds **454** `design` fields spelled `plans:<path>`, plus 4
entries in `refs[]` and about 115 prose mentions in `notes`/`description`. Verified with
a JSON walk over `sase/repos/beads/issues.jsonl`; open the store with
`sase repo open beads -r "<reason>"`.

### 1. Teach the repair planner about alias-only references

`sase bead doctor --fix-design-refs` (`bead/design_ref_repair.py`,
`bead/cli_admin.py:42`) is the right vehicle: it is event-backed, it previews before it
writes, and it already commits its own change. But as written it will **skip** all 454
rows, because after the `core` phase `plans:202607/x.md` parses as `legacy: false` and
resolves, so `_resolved_canonical_reference()` returns `True` and the loop `continue`s.

Fix that with a prefix-only fast path in `plan_design_ref_repairs`, placed **before**
the existing candidate search:

```python
parsed = _parsed_or_none(old_reference)
if parsed is not None and not parsed.legacy and parsed.rendered != old_reference:
    repairs.append(_DesignRefRepair(issue.id, old_reference, parsed.rendered))
    continue
```

This is strictly better than routing 454 rows through `_select_candidate`:

- it needs no filesystem lookup, so beads whose plan file has since been deleted or
  renamed still migrate (the candidate search would report those `no candidate` and
  leave them spelled `plans:`);
- it cannot pick the wrong file — the path payload is preserved byte for byte;
- it leaves the existing genuinely-broken-reference repair path untouched.

`_resolved_canonical_reference` keeps its current job of short-circuiting references
that are _already_ canonical; make sure the new branch runs first so it is not
swallowed.

Add unit tests: an alias-only reference produces exactly one repair with the same path
and no filesystem access; an already-canonical reference produces none; a genuinely
legacy path still goes through candidate selection.

### 2. Run the migration

```bash
sase bead doctor            # confirm the legacy-design-refs check now reports 454
sase bead doctor --fix-design-refs      # review the preview
sase bead doctor --fix-design-refs -y   # apply
sase bead doctor            # clean
```

Review the preview before applying — 454 rows is enough that a systematic error would be
expensive. Confirm afterwards that
`grep -c '"design":"plans:' sase/repos/beads/issues.jsonl` is `0` and that the same
count of `"design":"plan:` rows appeared. The repair commits and pushes through the
normal bead-store path; do not hand-edit `issues.jsonl`.

### 3. The other bead fields

- `refs[]` (4 entries) — migrate with `sase bead ref` (`sase bead ref list -j` to find
  them, then remove/add), not by editing the projection.
- `notes` / `description` prose (~115 mentions) — these are prose citations, so they
  take the `@plan:` form and belong to the `prose` phase's rule. They are individually
  low-value and each edit costs a bead event. **Recommendation: leave them.** Record the
  decision explicitly in the phase notes rather than silently skipping; if the reviewer
  wants them rewritten, that is a follow-up task bead, not a fifth stage here.
- Event streams stay untouched (Non-goals).

### Done when

`sase bead doctor` reports no legacy design references, `issues.jsonl` has zero
`"design":"plans:` rows, and the bead store is committed and pushed.

---

## Phase `prose`: Rewrite prose and remaining stored references

### 1. Documentation that describes the grammar

These sentences describe the reference _format_, so they must state the new rule, not
just swap a character:

- `docs/artifact_references.md:52` — the compatibility paragraph. Rewrite so it covers
  both grammars: `@plans:` and `plans:` are read-only compatibility spellings of `plan`,
  retained because commit trailers and bead event streams are immutable, and never
  emitted.
- `docs/artifact_references.md` "Provider Specs" — add a short paragraph stating the
  naming rule the `python` phase pinned: **a sidecar's ref kind is independent of its
  role name**, set with `ref.kind` (or inherited from `ref.use`), and the builtin roles
  `plans`/`beads`/`agents` ship as kinds `plan`/`bead`/`agent`. This is the part of the
  user's request that turns out to be documentation rather than code.
- `docs/sdd.md:174`, `docs/cli.md:151, 231, 237` — `sase plan show` target forms.
- `docs/change_spec.md:267` — Patch `REFS:` compatibility.
- `docs/editor.md:97` — the historical-alias list in completion.
- `docs/ace.md:990` — the logical-reference marker in the TUI.
- `docs/configuration.md:4367` — the historical-alias list.

### 2. Mechanical prose sweep

Everything else is a citation of a specific plan file and becomes `@plan:<path>`.

Match narrowly — `plans:` immediately followed by a path character
(`plans:[0-9A-Za-z_./~-]`) — so that YAML keys (`plans:` alone at end of line), the
`* **plans:**` changelog scopes, and English like "tale plans: `xsmall`" are never
touched. Verify the match set before rewriting, and rewrite in place with a scripted
substitution rather than by hand.

Three trees, in this order:

1. **This repo.** Source comments and module docstrings that cite a plan file —
   `artifact_ref_prompt_context.py:6-7`, `artifact_providers/builtin_entries.py:7`,
   `tools/validate_sase_core_rs_version:255` — plus
   `src/sase/xprompts/skills/sase_artifact_file.md:192, 201`. That skill is a
   **generated skill source**: after editing it, follow
   `sase/memory/generated_skills.md` (read it with `/sase_memory_read`) to redeploy, and
   do **not** edit the deployed copies directly.
2. **The plans sidecar** — 251 occurrences across 95 files (`sase repo open plans`).
   Pure prose; plan header blocks and `PARENT` bullets already carry bare relative
   paths, so nothing structural is at risk. Commit through the normal sidecar path.
3. **`~/.sase/plans/`** — 83 files of machine-local plans. Same rule. **Exclude this
   epic's own archived plan file**, whose examples deliberately show both spellings.

### 3. The two small stored-data sites

- `~/.sase/file_reference_history.json` — one `"@plans:202607/…"` entry → `"@plan:…"`.
  Machine-local completion history; a plain JSON edit is fine.
- `~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase-archive.sase` — 4 occurrences in
  archived Patch records. Bare `plan:` form (machine field, not prose).

### Done when

- `grep -rn 'plans:[0-9A-Za-z_./~-]' docs/ src/ tools/` returns nothing. That pattern
  has exactly **10** hits at this revision — the three module docstrings, the four
  CLI-help examples, the generated-skill example, and the
  `validate_sase_core_rs_version` comment — so the sweep is verifiable against a known
  number.
- `grep -rn 'plans:' docs/` returns only the deliberate compatibility sentences from §1;
  the bare-prefix mentions (`` `plans:` `` with no path) do **not** match the pattern
  above and must be reviewed by reading, not by grep.
- The plans sidecar and `~/.sase/plans` are clean under the path-form grep, and both
  sidecar trees are committed.
- `just check` still passes — the sweep touches docstrings, so lint and the docs gates
  both need re-running.

---

## Phase `land`: Verify and land the rename

1. `just install`, then `just check-full` over the combined tree. It routinely outruns a
   single turn — run it in the background and check back. `/sase_monitor` is currently
   unreliable, so do not rely on it.
2. `just test-visual` clean, with no pending snapshot diffs.
3. **Emitter sweep** — the actual acceptance test for this epic. In `sase-core` and this
   repo, confirm no code path can produce a `plans:` string:
   `rg 'plans:' crates/sase_core/src/plan/ src/sase/` should show only read-side
   aliases, each with a comment naming the immutable-history reason.
4. Manual smoke, end to end:
   - `sase plan show plan:<some/plan.md>` resolves, and `sase plan show plans:<same>`
     still resolves (the read alias).
   - `sase bead show <id>` for a migrated bead displays `plan:<path>`.
   - Open `sase ace`, select an agent with an associated plan, and confirm the metadata
     panel shows `plan:<path>` and resolves it — that panel is exactly where the
     `plans:202608/bead_close_publication_loss.md` "(missing)" bug surfaced before, so
     it is the right place to check for a regression.
   - Author `@plan:<path>` in a prompt and confirm it expands.
5. Land the epic through the normal close path.

## Risks

- **A `role == "plans"` site rewritten as a ref kind.** This is the main way to break
  the epic: the role name and the ref kind are now different strings that used to be the
  same. Every site in the Non-goals list is a real trap. Classify each hit before
  changing it, and prefer importing the shared constant over typing a literal, so that
  role comparisons stay visibly distinct from ref comparisons.
- **The read alias gets dropped somewhere.** If `parse_plan_reference` stops accepting
  `plans:`, thousands of commit trailers and every historical bead event silently stop
  resolving, and there is no way to repair them. The `land` phase's emitter sweep must
  confirm the alias is still _readable_, not just unemitted.
- **The prose sweep over-matches.** A greedy `plans:` substitution would corrupt YAML
  keys, changelog scopes, and ordinary English. Verify the match set first; the narrow
  character class above was checked against the known false positives
  (`sase/memory/sase_sizes.md:11`, `CHANGELOG.md`, `sase/sase.yml:173`).
- **Visual snapshot churn hides a regression.** Review a regenerated PNG by eye rather
  than trusting the bulk update.
- **Bead repair scope creep.** The prefix-only fast path must not change how genuinely
  broken references are repaired; those are a separate, riskier code path that this epic
  has no reason to touch.
