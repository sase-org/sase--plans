---
tier: epic
status: done
title: A feature flag is a task bead, not a bead type
goal: "SASE feature flags stop being a bespoke bead type and become ordinary task beads
  of a project-local `flag` task type whose required fields force every new flag to
  record what its two branches do and what has to be true before the losing branch is
  deleted. The `flag` issue type, `FlagRecord`, and `BeadFlagWire` are gone; only `beta`
  and `sunset` kinds survive; and the five live flag beads are migrated in place.

  "
phases:
  - id: slug
    title: Free the `flag` task-type slug
    depends_on: []
    size: small
    description:
      "slug: drop `flag` from the Rust core's reserved task-type slug list so a project
      may claim it, leaving the issue type untouched."
  - id: type
    title: Declare the `flag` task type in project config
    depends_on:
      - slug
    size: small
    description:
      "type: add the seven-field `flag` task-type spec to `bead.task_types` in this
      repo's project config, pin its glyph and accent, and refresh the committed catalog
      snapshot."
  - id: cli
    title: Two kinds, a derived default, and a rebuilt `sase flag new`
    depends_on:
      - type
    size: medium
    description:
      "cli: collapse the flag kinds to `beta` and `sunset`, derive the registry default
      and drop scope, and make `sase flag new` create the typed task bead from three
      required prose arguments."
  - id: reads
    title: Due-ness, identity, and integrity read task-type fields
    depends_on:
      - type
    size: medium
    description:
      "reads: repoint every flag-domain read path — due-state, the key and countdown
      chips, bead loading, registry integrity, the doctor, and the lint — from
      `issue.flag` to the typed task bead's field values."
  - id: gates
    title: FlagTriage is a task-bead gate
    depends_on:
      - reads
    size: medium
    description:
      "gates: make the reconciler and the FlagTriage contract select and describe flag
      beads by task type, and replace pending gates that still carry the old payload."
  - id: surfaces
    title: Every bead surface renders a flag as a typed task
    depends_on:
      - reads
    size: medium
    description:
      "surfaces: move the ACE beads pane, bead pages, CLI listings and detail, the
      mobile helper, the external mirror, and the stale-cleanup and duplicate-scan
      carve-outs onto the task type."
  - id: migrate
    title: Migrate the five live flag beads
    depends_on:
      - cli
      - gates
      - surfaces
    size: medium
    description:
      "migrate: rewrite the five existing flag bead event streams in place into typed
      task beads with hand-authored field values, then regenerate the projection, the
      mirror, and the pages."
  - id: retire
    title: Delete the `flag` issue type end to end
    depends_on:
      - migrate
    size: medium
    description:
      "retire: remove the flag issue type, `FlagRecord`, and `BeadFlagWire` from the
      Rust wire, the Python model, the storage codecs, the create grammar, and the
      compatibility mirror schema."
  - id: docs
    title: Memory notes, generated instructions, and documentation
    depends_on:
      - retire
    size: medium
    description:
      "docs: rewrite the two feature-flag memory notes, regenerate the instruction files
      and catalog snapshot, and update every doc page that still describes a flag bead
      type."
proposed_by: bbugyi200.athena.06a
bead_id: sase-pv
create_time: 2026-09-09 19:50:28
---

- **PROMPT:**
  [prompts/202608/flag_task_type.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/flag_task_type.md)
- **BEAD:**
  [sase-pv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-pv/README.md)

# Plan: A feature flag is a task bead, not a bead type

## The mistake being corrected

SASE feature flags were given their own bead **issue type** (`flag`), alongside `plan`,
`phase`, and `task`. That was wrong in three ways, and this epic fixes all three.

1. **A flag is not a fourth kind of thing.** It is a task: "retire this flag." Giving it
   its own issue type bought a bespoke storage field (`FlagRecord` on the bead wire, a
   `flag` column in the compatibility mirror, a `flag` block in the update event), a
   bespoke create grammar (`-T 'flag(<key>,<date>,<release>)'`), a bespoke presentation
   entry, and a bespoke row kind in the ACE pane — all to express something the
   task-type system, which did not exist when flags were designed, now expresses
   generically and better.
2. **Feature flags are a `sase`-project concern only.** The registry they join against
   lives in this repo's `src/sase/feature_flags/registry.py`, and `sase flag new`
   already refuses to run outside a checkout with `is_sase_managed: true`. Yet every
   SASE project on every machine carries the `flag` issue type in its bead schema.
   Declaring the type in **this project's** `sase/sase.yml` puts it exactly where it
   applies and nowhere else.
3. **Four kinds were two kinds and two mistakes.** `wip` ("shields a partially landed
   path that must not be the default yet") is just a `sunset` flag seen from the other
   side — the old path is the one that must stay reachable. `ops` ("a permanent
   operational switch") is, by the memory note's own admission, _not a feature flag_: it
   has no removal bead and no deadline, which is the definition of a config field. Both
   are deleted.

And the flag beads that exist today have descriptions that tell a future removal agent
almost nothing. Compare what is recorded against what is needed:

> `sase-nw`: "Opt-in beta: the follow-up coder inherits the planner's chat via #fork
> instead of starting from the approved plan file alone."

That is a decent sentence about the flag being _on_. It does not say what has to be true
before the flag can go away, and it does not say which branch dies when it does. Ninety
days later, the agent holding the `FlagTriage` gate has to reverse-engineer both. The
new task type makes both mandatory at create time.

## Design decisions

### D1 — The `flag` issue type is deleted, not kept alongside

`RESERVED_TASK_TYPE_SLUGS` in `sase-core` reserves `plan`, `phase`, `task`, and `flag`
precisely because they are issue types, so a task type named `flag` and an issue type
named `flag` cannot coexist permanently. That is the right constraint, and it forces the
correct answer: the issue type goes away, and `flag` becomes available as a task-type
slug. Freeing the slug (phase `slug`) and deleting the wire (phase `retire`) are
separate phases at opposite ends of the epic, so there is a deliberate window in the
middle where both exist. That window is what lets the store migration read old rows and
write new ones.

### D2 — Two kinds, and the default is derived from the kind

- **`beta`** — default **off**. The behavior is unproven; a user opts in.
- **`sunset`** — default **on**. The behavior is already the default; the flag exists
  only to keep the old, deprecated, or backward-compatible branch reachable while
  callers migrate.

Both kinds share one removal rule, and it is the reason two kinds are enough:

> **Removing a flag deletes the disabled branch and makes the enabled branch
> unconditional.**

Check it against all five live flags. `coder_inherits_planner_chat`, `epic_resume_gate`,
and `completion_refresh_on_update` are betas whose removal makes the new path
unconditional and deletes the old one. `prettier_enabled` and
`commit_finalizer_shared_clone_exempt` are sunsets whose removal deletes the escape
hatch (`SASE_DISABLE_PRETTIER`; strict single-owner classification) and makes the
already- default path unconditional. The only outcome that kills the _enabled_ branch is
abandoning a beta, which is the `FlagTriage` gate's **Close** option — an abandonment,
not a removal.

Two consequences follow, and both delete state that could drift:

- `FeatureFlagDefinition.default` is **derived**: `default = (kind == "sunset")`. The
  field stops being independently settable. `_default_for_kind` in `cli_new.py` already
  computes exactly this for the scaffold; it becomes the definition.
- `FlagScope` is deleted. Its only consumer is the resolver's `scope_violation`
  diagnostic, which rejects a `local`-config-layer override of a `global`-scoped flag —
  and every flag is `global`. The guard survives as an unconditional rule ("a feature
  flag may not be set from the `local` config layer"); the knob does not.

`kind` is stored in **both** the registry and the bead, and `tools/check_feature_flags`
lints that they agree. That is not new duplication: the registry `key` and the bead's
flag key are already joined and lint-enforced the same way. The registry needs `kind` at
import time to derive `default` without touching the bead store; the bead needs it so
every human-facing surface — the typed body block, the gate preview, the notification —
can say "beta" or "sunset" without importing the registry.

### D3 — The seven fields, and why each one is required

| field               | type                                              | role           | supplied by                 |
| ------------------- | ------------------------------------------------- | -------------- | --------------------------- |
| `key`               | `string`, pattern `^[a-z][a-z0-9]*(_[a-z0-9]+)*$` | data, template | `sase flag new <key>`       |
| `kind`              | `enum` of `beta`, `sunset`                        | data, template | `-k/--kind`, default `beta` |
| `when_enabled`      | `string`                                          | template       | **authored**                |
| `when_disabled`     | `string`                                          | template       | **authored**                |
| `remove_when`       | `string`                                          | template       | **authored**                |
| `remove_by_date`    | `date`                                            | data           | computed: today + 90d       |
| `remove_by_release` | `string`, pattern `^[0-9]+\.[0-9]+\.[0-9]+$`      | data           | computed: minor + 2         |

Seven required fields sounds heavy until you notice that `sase flag new` supplies four
of them. The author writes exactly three sentences, and they are the three a removal
agent cannot reconstruct:

- **`when_enabled`** — what the code does with the flag on.
- **`when_disabled`** — what the code does with the flag off. This is _the branch that
  gets deleted_, named explicitly, which is the single largest thing missing from every
  existing flag bead.
- **`remove_when`** — the qualitative gate. `remove_by_date` and `remove_by_release` say
  _when to ask_; `remove_when` says _how to answer_. A beta records the evidence that
  would promote it; a sunset records who still needs the old branch and what has to stop
  being true.

`when_enabled` and `when_disabled` are also, exactly, the two states the flag CLI's
"both-states test checklist" already demands tests for. Requiring both fields and both
tests is one idea, not two.

Deliberately **not** required:

- **Call sites.** `sase flag show` already finds them by walking the AST of the
  installed package. A hand-written location field would go stale; a derived one cannot.
- **`rationale`.** It existed only for `ops`, which is gone.
- **An `impact` or `owner` field.** Neither changes what the removal agent does.

`remove_by_date` and `remove_by_release` carry the `data` role only, so they are barred
from the body template by the Rust validator. That is the right split: the dates are
machine data that drive the countdown chip, and the prose is human data that drives the
body block. Nothing renders a raw threshold string twice.

### D4 — The body block

```
## Feature flag `{{ key }}` · {{ kind }}

- **On:** {{ when_enabled }}
- **Off:** {{ when_disabled }}

**Remove when:** {{ remove_when }}

Removal deletes the **Off** branch and makes the **On** branch unconditional.
```

The closing line is literal template text, not a field, because D2 makes it true of
every flag. A reader who has never seen this design learns the removal rule from the
bead.

### D5 — `agent_creatable: false`

`sase flag new` stays the only door. It is the only path that also mints the registry
scaffold, and hand-adding a registry member has always been forbidden. Setting
`agent_creatable: false` (as the `github` type already does) makes
`sase bead create -T 'task(flag)'` refuse, drops `flag` from the ACE create modal's
picker, and keeps `flag` out of the agent-creatable list in the generated
`task_types.md` memory note — which is correct, because an agent reaching for a flag
should be reading the short `feature_flags.md` note, not `/sase_new_task`.

One rough edge to fix while doing it: `resolve_created_task_type` currently rejects a
non-agent-creatable type with "it is reserved for the providing plugin", which is wrong
for a project-local type. It should surface the type's own `when_to_use` text instead,
so `flag` answers with "run `sase flag new <key>`".

### D6 — A flag task bead lives at `open`, and due-ness stays derived

Unchanged from today, on purpose. A task bead of type `flag` is gateable while its
status is `open` **and** `flag_removal_due(...) == "due"`; every other task bead is
gateable while `ready` or `snoozed`. Due-ness is never persisted, because both
thresholds are compared against a clock and a release string the caller owns.

The tempting alternative — create the bead `snoozed` until `remove_by_date`, so
"deferred, not actionable yet, will come back" is expressed by the status that already
means that — was considered and **rejected**. `snooze` knows about a wake time and a
`+1` target, not a release, so the release threshold would either have to be dropped or
smuggled in as an automatic re-snooze on wake. Both are worse than one explicit,
one-line selection rule. This epic changes the carrier, not the lifecycle.

### D7 — Presentation is inherited, not reinvented

The `flag` task type declares `glyph: "⚑"` and `accent_color: "#FF875F"` — the exact
glyph and accent the retired issue type used. A flag bead therefore looks identical
before and after the migration; only its storage changed. `#FF875F` stays in
`_RESERVED_ACCENT_COLORS` but moves from "one of the four issue-type accents" to "one of
the pinned task-type accents".

`bead_flag_presentation` survives as a first-party, `sase`-specific module. It keeps
both chips — the `⚑ <key>` identity cell and the urgency-graded `⧗ 88d · v0.18.0` /
`DUE ⧗ +3d` meter — and reads the two thresholds from `task_type_fields` instead of
`issue.flag`. Generalizing "render a custom chip from field values" into a task-type
hook was considered and rejected: one type needs it, and the hook would have to carry a
clock and a release string through the generic path to be correct.

The ACE artifacts pane keeps its dedicated **flags** group and its `(N due)` urgency
count. Membership is re-derived by one rule: a task bead whose `task_type` is `flag`
belongs to the flags group and _not_ to the tasks group. The group is worth keeping —
flag beads have a genuinely different lifecycle and their own urgency — and
`sase bead list -T flag` already exists as the generic equivalent on the CLI.

### D8 — Coordination with epic `sase-pq`

`sase-pq` ("A task bead's type is legible on every gate notification surface") is in
flight with phases `.4` through `.7` open. Once flags are task beads, `FlagTriage` is a
task-bead gate and should carry the same frozen `presentation.chip` that epic is
building.

Coordination notes have already been left on `sase-pq`, `sase-pq.5`, and `sase-pq.7`
asking that the task-bead gate kinds not be hard-coded to `{task_triage, bead_snooze}`,
and naming the overlapping files. **`sase-pq` lands first; this epic rebases onto it.**
The files both epics touch are `src/sase/scripts/_bead_task_triage_gates.py`,
`src/sase/scripts/_bead_task_triage_state.py`,
`src/sase/notification_gates/kind_validation/flag_triage.py`,
`src/sase/task_types/_validation.py`, `docs/notifications.md`, and `docs/axe.md`. Any
phase agent here that finds `sase-pq` still open on one of those files should rebase
rather than fight it.

### D9 — Cross-repo work

Phases `slug` and `retire` change the Rust core at `../sase-core`. An agent working them
MUST open that repo through the `/sase_repo` skill first and use the path it prints.
Both phases must keep the Python binding surface, the contract manifest, and the parity
tests in step; `just check-full` (through `/sase_monitor`, never inline) is the gate.

---

## `slug`: Free the `flag` task-type slug

Open `../sase-core` through `/sase_repo`. In `crates/sase_core/src/task_type/spec.rs`,
remove `"flag"` from `RESERVED_TASK_TYPE_SLUGS`, leaving `plan`, `phase`, `task`,
`untyped`, `unknown`, `all`, and `none`. Update the doc comment so it says the list
reserves the three bead issue types plus the four filter sentinels.

The `flag` **issue type** is untouched by this phase. This is the smallest change that
unblocks everything else, and it is independently safe: nothing in the catalog claims
the slug yet.

Adjust any Rust unit test that asserts `flag` is rejected as a task-type slug, and add
one that asserts a spec claiming `flag` now validates. On the Python side, check
`tests/` for an assertion on the reserved-slug set and update it.

Verify with `just check-full` through `/sase_monitor`, since a core change invalidates
the built extension and touches the contract manifest and core-floor probe.

## `type`: Declare the `flag` task type in project config

Add one entry to `bead.task_types` in this repo's `sase/sase.yml`. It is a brand-new
project-local type, so it carries a full spec and no `use:` key:

```yaml
bead:
  task_types:
    - schema_version: 1
      task_type: flag
      label: Feature flag
      summary: >-
        A SASE feature flag's removal bead: the dossier for one temporary boolean route.
      when_to_use: >-
        Agents never create this type with `sase bead create` or `/sase_new_task`. Run
        `sase flag new <key>`, which creates this bead, validates its fields, and prints
        the registry entry to paste. One bead per registered flag; close it only in the
        change that deletes the flag's disabled branch and its registry entry.
      glyph: "⚑"
      accent_color: "#FF875F"
      agent_creatable: false
      fields:
        - name: key
          label: Key
          type: string
          required: true
          role: [data, template]
          help: The snake_case registry key, e.g. prettier_enabled
          pattern: "^[a-z][a-z0-9]*(_[a-z0-9]+)*$"
        - name: kind
          label: Kind
          type: enum
          required: true
          role: [data, template]
          values: [beta, sunset]
          help:
            beta is off by default; sunset is on by default and guards a doomed branch
        - name: when_enabled
          label: On
          type: string
          required: true
          role: [template]
          help: What the code does with this flag enabled
        - name: when_disabled
          label: Off
          type: string
          required: true
          role: [template]
          help:
            What the code does with this flag disabled; this branch is deleted at
            removal
        - name: remove_when
          label: Remove when
          type: string
          required: true
          role: [template]
          help: What must be true before the disabled branch can be deleted
        - name: remove_by_date
          label: Remove by date
          type: date
          required: true
          role: [data]
          help: Calendar removal threshold, YYYY-MM-DD
        - name: remove_by_release
          label: Remove by release
          type: string
          required: true
          role: [data]
          help: Release removal threshold, e.g. 0.18.0
          pattern: '^[0-9]+\.[0-9]+\.[0-9]+$'
      body_template: |
        ## Feature flag `{{ key }}` · {{ kind }}

        - **On:** {{ when_enabled }}
        - **Off:** {{ when_disabled }}

        **Remove when:** {{ remove_when }}

        Removal deletes the **Off** branch and makes the **On** branch unconditional.
      triage:
        min_plus_ones: 0
```

`summary` must stay a single line under 120 characters and `when_to_use` under 400; the
Rust validator enforces both, and `sase bead task-type show flag` is the check that the
whole spec assembled.

In `src/sase/task_types/_validation.py`, update the `_RESERVED_ACCENT_COLORS` docstring:
`#FF875F` is about to stop being an issue-type accent and become a pinned task-type
accent. The frozenset contents do not change.

Fix the misleading rejection message for non-agent-creatable types in
`resolve_created_task_type` (`src/sase/task_types/fields.py`) per D5: surface the type's
`when_to_use` instead of asserting it belongs to a plugin.

Regenerate `sase/task_types.json` (`sase memory init` owns that snapshot) and commit it.
The `flag` entry lands with `source: project` and `package: sase`.

## `cli`: Two kinds, a derived default, and a rebuilt `sase flag new`

**Models and registry.** In `src/sase/feature_flags/models.py`, narrow `FlagKind` to
`Literal["beta", "sunset"]` and delete `FlagScope` and `FlagSource`'s scope-related
consumers. Remove `default`, `scope`, and `rationale` from `FeatureFlagDefinition`;
`default` becomes a property returning `self.kind == "sunset"`. `validate()` loses the
ops-rationale branch and now requires a bead for every flag, because every flag is
temporary. Update `src/sase/feature_flags/registry.py`'s five entries to the reduced
shape and its module docstring.

**Resolver.** In `src/sase/feature_flags/resolver.py`, replace the
`definition.scope == "global"` condition with an unconditional rule: a value from the
`local` layer is always a `scope_violation` (keep the diagnostic code; reword the
message to "a feature flag cannot be set from local config").
`src/sase/feature_flags/cli_show.py` has the same condition on its provenance table and
gets the same treatment.

**Schema.** `src/sase/feature_flags/schema.py` no longer emits scope, and
`deprecated: true` now keys off `kind == "sunset"` exactly as before. Re-run
`just sync-feature-flags-schema` and commit the regenerated block in
`src/sase/config/sase.schema.json`.

**`sase flag new`.** This is the heart of the phase. `src/sase/main/parser_flag.py`:

- `-k/--kind` choices become `beta` and `sunset`; default stays `beta`.
- `--scope` is deleted.
- Three new required options: `--when-enabled`, `--when-disabled`, `--remove-when`. Each
  accepts `@<path>` file input the way `sase bead create -f` does, so a long rationale
  does not have to be a shell argument.
- `-d/--description` stays optional and free-form: it seeds the _registry_ entry's
  one-line config-schema help, defaulting to `--when-enabled` when omitted. It is not
  written to the bead, because the typed body block is the bead's dossier and repeating
  it would render the same sentence twice on `sase bead show`.
- `-z/--size` stays optional and now defaults to `small`. The task-type spec's
  `default_size` key is deliberately not used: it is carried on the wire and in the
  snapshot but no Python create path consumes it today, so setting it there would be
  inert and misleading.

`src/sase/feature_flags/cli_new.py` builds the field map, validates it through
`validate_task_type_field_values` before touching the store (so a bad `--remove-when` is
rejected without creating a bead), and hands it to a rewritten `create_flag_bead` in
`src/sase/feature_flags/beads.py` that creates `IssueType.TASK` with `task_type="flag"`
and `task_type_fields=<map>`. The duplicate-key guard
(`live flag bead <id> already owns key <key>`) is preserved.

The printed scaffold shrinks to the surviving registry fields:

```
    FeatureFlag.demo_key: FeatureFlagDefinition(
        key=FeatureFlag.demo_key,
        kind='beta',
        description='...',
        bead='sase-xy',
    ),
```

Keep the both-states checklist it prints, and reword its first two lines to name the
`when_enabled` and `when_disabled` branches, which are now recorded on the bead.

`src/sase/feature_flags/defaults.py` keeps the 90-day / minor-plus-two computation but
returns the two threshold strings rather than a `FlagRecord`.

## `reads`: Due-ness, identity, and integrity read task-type fields

Introduce one accessor — the single place that turns a task bead into flag data — and
route everything through it, so no module parses `task_type_fields` twice:

```python
def flag_fields(issue: Issue) -> FlagFields | None:
    """Return key, kind, and both thresholds for a `flag` task bead, else None."""
```

It returns `None` for any bead whose `task_type` is not `flag` or whose thresholds are
unparseable, which is the same tolerance `flag_from_dict` gives today.

Then repoint each consumer:

- `src/sase/bead/flag_due.py` — `flag_removal_due` takes the thresholds rather than a
  `FlagRecord`. The predicate itself, and the rule that "due" requires **both**
  thresholds, are unchanged; this is the one shared definition and stays that way.
- `src/sase/bead_flag_presentation.py` — both chips read the accessor. Behavior,
  wording, glyph, accent, and urgency colors are byte-identical to today.
- `src/sase/feature_flags/beads.py` — `load_flag_bead_snapshots` lists `IssueType.TASK`
  and filters on `task_type == "flag"`; `FlagBeadSnapshot` drops `issue_type` in favor
  of the task type and gains `kind`.
- `src/sase/feature_flags/integrity.py` — the `wrong_type` finding becomes "names bead
  `<id>`, which is not a `flag` task bead", and a new finding catches
  `registry.kind != bead.kind` and, by construction, a `default` that disagrees with the
  kind. Keep every existing finding's code so the doctor's output stays diffable.
- `src/sase/doctor/checks_flags.py` and `tools/check_feature_flags` — follow the
  accessor; the lint gate `just _lint-flags` must stay green.
- `src/sase/feature_flags/cli_list.py`, `cli_show.py`, `cli_views.py`, `cli_json.py` —
  drop the scope column, keep the kind column, and read thresholds through the accessor.
- `src/sase/bead/cli_crud_update.py` — `-b/--remove-by` currently rejects anything that
  is not `IssueType.FLAG` with a `flag` record. It now targets a `flag` task bead and
  writes the two threshold **field values**. `task_type` itself stays immutable; only
  these two `data`-role values are editable, and only through this option.

## `gates`: FlagTriage is a task-bead gate

**Selection.** In `src/sase/scripts/_bead_task_triage_state.py`, `gateable_beads` stops
listing `IssueType.FLAG` and applies D6's rule inside the task branch: a task bead of
type `flag` is gateable while `open` and due; every other task bead is gateable while
`ready` or `snoozed`. Drop `Status.OPEN` from `_GATEABLE_STORE_STATUSES` only if the
store read still returns the flag beads — it must not, so keep it.

**Kind.** `expected_gate_kind` in `src/sase/scripts/_bead_task_triage_gates.py` returns
`FLAG_TRIAGE_KIND` for `task_type == "flag"` before its snoozed/ready branch.

**Fingerprint.** The same module's `presentation_fingerprint` builds its `flag` block
from the accessor rather than `issue.flag`. Bump the presentation format version so
every pending `FlagTriage` gate carrying the old payload is cancelled and recreated —
the same mechanism `sase-pq.6` uses, and the reason that epic should land first.

**Contract.** `src/sase/bead/_flag_gate_spec.py`, `_flag_gate_preview.py`,
`_flag_gate_response.py`, `_flag_gate_actions.py`, and
`src/sase/notification_gates/kind_validation/flag_triage*.py` carry the thresholds and
now also the kind. The gate's four options — Remove, Extend, Keep, Close — and their
feedback requirements are unchanged, but their help text should be updated to D2's
vocabulary: **Remove** deletes the Off branch, **Extend** pushes both thresholds out,
**Keep** means the behavior is permanent and belongs in a config field, **Close**
abandons the removal. The preview gains the three prose fields, which is the whole point
of the epic: the human answering the gate finally sees what the flag switches and what
would settle it.

`extend_flag_triage` writes the new thresholds as field values through the `reads`
accessor's write counterpart.

If `sase-pq.5` has landed, declare the gate's `presentation.chip` from the frozen
task-type display block here rather than hand-building it.

## `surfaces`: Every bead surface renders a flag as a typed task

- `src/sase/ace/tui/widgets/artifacts/beads_data.py` — the flags group and the
  `flag_due` map key off `task_type == "flag"`; the tasks group excludes those beads
  (D7). The external-ref carve-out at `_local_external_refs` keys off the task type.
- `beads_filtering.py`, `beads_list.py`, `beads_options.py`, `beads_rendering.py`,
  `beads_detail*.py` — `BeadRowKind` keeps its `"flag"` member; only the predicate that
  assigns it changes. Flag rows keep the countdown cell and gain the standard task-type
  chip.
- `src/sase/bead_pages/rendering.py`, `rendering_identity.py`, `roster.py` — the page's
  flag identity block reads the task type.
- `src/sase/bead/cli_query.py`, `cli_query_render.py`, `cli_detail.py`,
  `cli_detail_json.py`, `bead_summary_presentation.py` — `sase bead show` keeps its
  `FLAG` block; the `Flags:` stat counts typed tasks. `--type flag` is removed from
  `sase bead list` and `sase bead search` (`src/sase/main/parser_bead_queries.py`);
  `-T flag` already does the job and the examples should say so.
- `src/sase/bead/cli_crud_lifecycle.py`, `cli_work_entry.py`, `cli_work_task.py` — the
  `(IssueType.TASK, IssueType.FLAG)` tuples collapse to `IssueType.TASK`.
- `src/sase/external_mirror/_issue_planning.py` — the "flag beads never cover external
  issues" carve-outs key off the task type.
- `src/sase/scripts/sase_chop_bead_stale_cleanup.py` — flag task beads must be exempt
  from stale-cleanup sweeps; they are `open` by design, not neglected.
- `src/sase/xprompts/skills/sase_new_task.md` — the duplicate scan may see flag beads,
  which is harmless, but the skill must state that a feature flag is created with
  `sase flag new`, never through this skill.
- `src/sase/integrations/_mobile_helper_beads.py` and the ACE prompt panel's
  `_agent_bead_section.py` — follow the same predicate change.

## `migrate`: Migrate the five live flag beads

Exactly five flag beads exist, all `open`, each in its own event stream with a single
`issue_created` event and no later events. That makes an in-place rewrite the right
migration: bead IDs survive, so the registry's `bead=` pointers and every note and doc
that names them stay valid, and no `flag`-typed row is left behind for the `retire`
phase to trip over.

Write a one-shot migration under `tools/` that, for each of the five streams:

1. Rewrites the `issue_created` payload: `issue_type: "flag"` → `"task"`, removes the
   `flag` object, adds `task_type: "flag"` and `task_type_fields` with all seven values.
   `id`, `title`, `status`, `owner`, `created_at`, `created_by`, and `size` are
   preserved verbatim.
2. Recomputes the event's content-hash `event_id` through the same core helper that
   minted it, so the stream stays self-consistent.
3. Regenerates the `issues.jsonl` projection, rebuilds the compatibility mirror, and
   refreshes the bead pages.

Closing the old beads and creating new ones was considered and rejected: it would break
the registry pointers, scatter the history, and — because a closed bead still lives in
the store — force the `flag` issue type to survive forever as a legacy-readable variant.

The field values below are drafts derived from the current registry descriptions. The
phase agent must **verify each against the code before writing it**, because the whole
point of this epic is that these sentences be true.

| bead      | key                                    | kind     |
| --------- | -------------------------------------- | -------- |
| `sase-nw` | `coder_inherits_planner_chat`          | `beta`   |
| `sase-nx` | `prettier_enabled`                     | `sunset` |
| `sase-om` | `completion_refresh_on_update`         | `beta`   |
| `sase-pa` | `epic_resume_gate`                     | `beta`   |
| `sase-pk` | `commit_finalizer_shared_clone_exempt` | `sunset` |

`remove_by_date` and `remove_by_release` are carried across unchanged from each bead's
existing `flag` record, so no flag's deadline moves as a side effect of the migration.

- **`sase-nw` / `coder_inherits_planner_chat`**
  - On: the follow-up coder launched after plan approval inherits the planner's chat
    through `#fork`, so it starts with the planner's full reasoning context.
  - Off: the follow-up coder starts from the approved plan file alone, with no planner
    chat context.
  - Remove when: forked coders have landed several epics with no plan-fidelity
    regression against the plan-file-only path, and the `#fork` context cost is
    acceptable at typical planner chat lengths.
- **`sase-nx` / `prettier_enabled`**
  - On: Markdown is formatted with prettier whenever prettier is installed.
  - Off: Markdown formatting skips prettier entirely; the deprecated
    `SASE_DISABLE_PRETTIER` environment variable is the alias that reaches this branch.
  - Remove when: no workflow still needs a prettier escape hatch and
    `SASE_DISABLE_PRETTIER` is no longer exported anywhere.
- **`sase-om` / `completion_refresh_on_update`**
  - On: after a successful `sase update`, installed shell completion scripts are
    regenerated, zcompiled, and restamped.
  - Off: `sase update` leaves installed completion scripts untouched; they refresh only
    on an explicit completion install.
  - Remove when: the generator has soaked through several `sase update` runs without
    producing a broken or stale completion script on any supported shell.
- **`sase-pa` / `epic_resume_gate`**
  - On: the `epic_resume` chop raises an `EpicResume` gate when a failed phase agent has
    stalled an epic.
  - Off: a stalled epic raises no gate; recovery is entirely manual through
    `sase bead work <epic-id>`.
  - Remove when: the chop has gated real stalls without false positives on handoff races
    or fast retries at the configured `bead.epic_resume.settle_seconds`.
- **`sase-pk` / `commit_finalizer_shared_clone_exempt`**
  - On: in machine-wide shared clones (opened-external and sdd-kind repos), the commit
    finalizer's dirty-work guard classifies foreign-agent commits and already-published
    or pending-publication transitions as races.
  - Off: the guard falls back to strict single-owner classification and reports those
    same transitions as discards.
  - Remove when: shared-clone race classification has run without a real discard being
    misreported as a race, so the strict single-owner branch can be deleted.

Acceptance for this phase: `sase bead doctor` is clean, `sase bead list -T flag` prints
the same five beads with the same IDs and countdowns as `sase bead list --type flag`
does today, `sase bead show sase-nw` renders the new body block, `just _lint-flags` is
green, and no row anywhere in the store has `issue_type: flag`.

## `retire`: Delete the `flag` issue type end to end

Only now, with nothing reading or writing it, delete the type.

**Rust core** (open `../sase-core` through `/sase_repo`): `IssueTypeWire::Flag`,
`BeadFlagWire`, the `flag` field on `IssueWire` and on `BeadIssueUpdateEventFieldsWire`
(and its `validate()` emptiness check), and every flag branch in `bead/wire.rs`,
`bead/cli.rs`, `bead/mutation.rs`, `bead/events.rs`, `bead/read.rs`, `bead/jsonl.rs`,
`bead/search.rs`, and `query/`. Add the compatibility- mirror migration that drops the
`flag` column, drops `'flag'` from the `issue_type` CHECK, relaxes the parent-id and
flag-presence CHECKs, and rebuilds `idx_issues_external_ref` without its
`issue_type != 'flag'` predicate. Model it on the existing
`bead_needs_flag_type_migration` / `bead_flag_type_migration_sql` pair, which added the
type — this is its mirror image, and it needs the same `needs_*` / `*_sql` binding pair.

**Python:** `IssueType.FLAG` and `FlagRecord` (`src/sase/bead/model.py`, including its
four flag validation branches), `src/sase/bead/flag_codec.py`, `src/sase/bead/jsonl.py`,
`src/sase/bead/_db_codec.py`, `_db_rows.py`, `_db_schema.py`, `_db_migrations.py`,
`src/sase/core/bead_wire.py`, `src/sase/core/bead_mutation_facade.py`, the `flag(...)`
branch of `parse_type_arg` in `src/sase/bead/cli_crud_create.py` (and its `-T` help
text), the `"flag"` entry in `src/sase/bead_type_presentation.py` and its
`BeadTypeValue` literal, and the `IssueType.FLAG` mapping in
`src/sase/ace/tui/relations/beads.py`.

`BEAD_TYPE_CHIP_WIDTH` is computed from the presentation table, so removing an entry may
narrow it — check the ACE PNG snapshot suite with `just test-visual` and accept any
intentional change with `--sase-update-visual-snapshots`.

Any symbol that becomes unused here should be deleted rather than whitelisted; if a
Symvision `--epic-symbol` entry is genuinely needed, read `sase/memory/symvision.md`
through `/sase_memory_read` first and remove the entry in the same change that closes
its phase.

## `docs`: Memory notes, generated instructions, and documentation

The user explicitly asked for the feature-flag memory notes to be updated as part of
this work, so the required permission is on record and `sase memory init` must be run to
regenerate the derived files. No further permission is needed.

**`sase/memory/sase_flags.md`** (long note) is rewritten around this epic's design: two
kinds and what each one's default is; the one removal rule; the seven fields and the
three an author actually writes; `sase flag new` as the only door; `-b/--remove-by` for
extension; and the four `FlagTriage` answers in D2's vocabulary. Delete the `wip` and
`ops` paragraphs and the "promote it to `ops`" advice under **Keep** — the answer there
is now "it was never a feature flag; make it a config field," which the note already
says one sentence later.

**`sase/memory/feature_flags.md`** (short note) keeps its shape and gains one sentence:
flags are a `sase`-project concern, and a flag bead is a task bead of type `flag`.

Run `sase memory init` to regenerate `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, `QWEN.md`, `sase/memory/README.md`, `sase/memory/task_types.md`, and
`sase/task_types.json`. The generated `task_types.md` will **not** list `flag`, because
it is `agent_creatable: false` — that is correct, and it is why the short
`feature_flags.md` note is the one that has to mention flags.

**Docs:**

- `docs/beads.md` — remove `Flag` from the bead-type table; rewrite the flag bead
  lifecycle section around the task type; drop `-T 'flag(...)'`; keep and update the
  `-b/--remove-by` row; fix the "task and flag beads are top-level" and "invalid for
  plan, phase, and flag beads" sentences.
- `docs/notifications.md` and `docs/axe.md` — a `FlagTriage` gate is raised for a task
  bead of type `flag`; the `bead_task_triage` chop's description covers three gate kinds
  over one bead type. Coordinate with whatever `sase-pq.7` landed on these pages.
- `docs/cli.md` — the `sase flag` row and the removed `--type flag` filter.
- `docs/configuration.md` — the `feature_flags` section loses scope and the four-kind
  list.
- `docs/plugins.md`, `docs/completion.md`, `docs/commit_workflows.md`, `docs/xprompt.md`
  — incidental prose that names a flag's kind; verify each still matches.

Close with `just check-full` through `/sase_monitor`, plus `just test-visual`, and
confirm `sase flag list`, `sase flag show <key>`, `sase bead show sase-nw`, and
`sase bead task-type show flag` all render correctly.
