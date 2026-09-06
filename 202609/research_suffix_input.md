---
tier: tale
title: "Add a suffix input to #research and use __a/__b suffixes in #research_swarm"
goal:
  "The two initial #research_swarm researchers write reports with explicit __a/__b
  filename suffixes via a new generic optional suffix input on the #research xprompt."
size: small
proposed_by: bbugyi200.athena.0gj
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0gj](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0gj.md)
- **COMMITS:**
  - [68bb0dd](https://github.com/sase-org/sase-research-artifacts/commit/68bb0dd3326adfeaf42637c330b757c6bdece13e)
    — feat: add research suffix input

# Add an optional `suffix` input to `#research` and use `__a`/`__b` suffixes in `#research_swarm`

## Goal

Make the two initial researchers in the `#research_swarm` xprompt swarm write reports
with explicit `__a` / `__b` filename suffixes (e.g. `202609/foo__a.md`), driven by a new
generic optional input on the `#research` xprompt that specifies the suffix identifier.

All file changes live in the **`sase-research-artifacts` linked repo**. Open it first
and use only the path it prints for reads and writes:

```bash
sase repo open sase-research-artifacts -r "Implement the #research suffix input plan"
```

## Background and rationale

- `#research` (`src/sase_research_artifacts/xprompts/research.md`) currently has one
  optional input, `report_target` (type `path`, default `null`). When set, the agent
  must write exactly `<research-repo>/<YYYYmm>/<report_target>`; when unset, the agent
  freely picks a filename under the month directory.
- `#research_swarm` (`src/sase_research_artifacts/xprompts/research_swarm.md`) is a
  four-segment swarm: researcher `cdx` (`@research_a`), researcher `cld`
  (`@research_b`), lead `final` (`@xlarge`), and `image`. Since commit `83f4c01` ("fix:
  make research report targets deterministic"), the two researchers are launched with
  `#research(report_target=research.{@1}.cdx.md)` and
  `#research(report_target=research.{@1}.cld.md)`, and the lead renames those two
  reports to `<name>__a.md` / `<name>__b.md` inside a `<name>/` directory it creates.
- This plan replaces the swarm's deterministic agent-id report targets with
  researcher-chosen descriptive stems plus mandatory `__a`/`__b` suffixes. Collision
  safety within a swarm is preserved (the two suffixes differ even if both researchers
  pick the same stem, so the original merge-conflict failure mode stays eliminated),
  while the reports carry descriptive names from birth and the lead can identify each
  researcher's report from its filename suffix instead of only from transcripts.
- The `report_target` input stays: it remains the fully explicit escape hatch and is
  unaffected by this change.

## Changes

### 1. `src/sase_research_artifacts/xprompts/research.md`

Add a second optional input after `report_target`:

```yaml
- name: suffix
  type: word
  default: null
  description:
    Optional filename suffix identifier (e.g. `a` or `b`). When set and `report_target`
    is not, the chosen report filename must end with `__<suffix>.md`. Ignored when
    `report_target` is provided, since that names the file exactly.
```

Turn the body's two-branch conditional into three branches. Keep the existing
`{% if report_target %}` and `{% else %}` branches byte-identical; insert a new
`{% elif suffix %}` branch between them with instructions along these lines:

```jinja
{% elif suffix %}
Write this research to a new markdown file under the $(sase repo path research --ensure)/$(date +%Y%m)/ directory.
Choose a descriptive filename stem yourself, but the filename MUST end with the
`__{{ suffix }}` suffix, i.e. `<stem>__{{ suffix }}.md` (double underscore before the
suffix). Create the report without overwrite: if the exact file already exists, pick a
different stem instead of replacing it.
```

### 2. `src/sase_research_artifacts/xprompts/research_swarm.md`

- In the `cdx` segment, replace `#research(report_target=research.{@1}.cdx.md)` with
  `#research(suffix=a)`.
- In the `cld` segment, replace `#research(report_target=research.{@1}.cld.md)` with
  `#research(suffix=b)`.
- Update the lead (`final`) segment's numbered steps to match the new reality:
  - Step 1 currently says to learn which report file each researcher wrote from the
    transcripts (`research.{@1}.cdx` -> `__a`, `research.{@1}.cld` -> `__b`) and to
    never assign `__a`/`__b` from filesystem order. Reword it: each researcher's report
    filename already ends with its suffix (`research.{@1}.cdx` wrote `__a.md`,
    `research.{@1}.cld` wrote `__b.md`); learn the exact paths from the transcripts and
    treat the `__a`/`__b` suffix already on each file as the authoritative authorship
    marker — still never reassign suffixes from filesystem order.
  - Step 3: when moving the two reports into `<month-dir>/<name>/` as `<name>__a.md` and
    `<name>__b.md`, preserve each report's existing `__a`/`__b` suffix.
- Leave the clan/id/wait/priority/model plumbing and the final-layout code block
  untouched.

### 3. `tests/test_xprompt_loading.py`

- `test_research_prompt_declares_typed_input`: the `research` xprompt now declares
  `[("report_target", "path"), ("suffix", "word")]`, both with `default is None`.
- `test_research_swarm_wait_argument_gates_researchers_only`: replace the two
  `report_target=...` containment assertions with
  `"some topic #research(suffix=a)" in cdx` and `#research(suffix=b)` in `cld`.
- `test_research_swarm_dispatches_distinct_deterministic_report_targets`: the
  deterministic-target rationale is gone, so rework and rename it (e.g. to
  `test_research_swarm_researchers_carry_distinct_suffixes`). Keep the two-dispatch
  clan-marker distinctness assertions (`%id:research.<marker>.cdx`,
  `%id(cld, clan=research.<marker>)`, the final/image `%wait` assertions), and assert
  that every `cdx` segment contains `#research(suffix=a)`, every `cld` segment contains
  `#research(suffix=b)`, and `report_target=` appears in none of the four researcher
  segments.
- Add a small direct test of the new `#research` branch using the already-imported
  `expand_single_xprompt` on the `research` xprompt:
  - with `{"suffix": "a"}`: the expansion mentions `__a` / `<stem>__a.md` and contains
    no leftover `{%` / `{{ suffix }}` Jinja artifacts;
  - with `{"report_target": "x.md", "suffix": "a"}`: the `report_target` branch renders
    (expansion contains `x.md` and does not instruct a `__a` stem), proving the
    documented precedence;
  - with no args: the free-choice `{% else %}` branch still renders with no `__` suffix
    requirement.

### 4. `docs/xprompts.md`

- `#research` section: add an Input table documenting both `report_target` and `suffix`
  (both optional), including the precedence rule (`report_target` wins; `suffix`
  constrains the filename to `<stem>__<suffix>.md` when set alone).
- `#research_swarm` section: update items 1–3 of the segment list — the two researchers
  now write suffix-tagged reports via `#research(suffix=a)` / `#research(suffix=b)` with
  self-chosen descriptive stems, and the lead moves them to a unified `<name>__a.md` /
  `<name>__b.md` under `<name>/`, preserving each report's suffix.

## Out of scope

- No changes to the sase primary repo, `default_config.yml`, the provider module,
  `#research/image`, `#research/more`, or `#research/prompt`.
- No feature flag: the `report_target` input and behavior remain fully supported; only
  the swarm's internal call sites change.
- No manual `CHANGELOG.md` edit — release-please derives it from the conventional commit
  message.

## Verification

From the `sase-research-artifacts` checkout printed by `sase repo open`:

```bash
just check   # ruff + mypy + pytest; its _setup target creates the .venv itself
```

`just check` in that repo is self-contained (it builds its own `.venv` against the local
sase and sase-core checkouts), so no separate install step is required. The sase primary
repo is untouched, so its `just check` lane is not needed.

## Success criteria

- `just check` passes in the plugin repo.
- Expanding `#research_swarm` yields `#research(suffix=a)` in the `cdx` segment,
  `#research(suffix=b)` in the `cld` segment, and no `report_target=` anywhere.
- `#research` with no arguments and with `report_target=...` behaves exactly as before.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact                              | Why                                                    | Uses |
| -------- | ------------------------------------- | ------------------------------------------------------ | ---: |
| cited-by | [agent:bbugyi200.athena.0gj--code][1] | prompt reference @plan:202609/research_suffix_input.md |    1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0gj.md

<!-- sase:referenced-by:end -->
