---
tier: tale
title: Add `--explain` to `sase plan validate` and auto-detect tier from frontmatter
goal: 'The /sase_plan skill no longer front-loads the plan-frontmatter schema; agents
  learn it on demand from `sase plan validate --explain`, which auto-detects the tier
  from the plan file''s own `tier:` property instead of a `--tier` flag.

  '
create_time: 2026-07-22 09:44:21
status: done
---

- **PROMPT:** [202607/prompts/validate_explain_option.md](prompts/validate_explain_option.md)

# Plan: `--explain` for `sase plan validate` + tier auto-detection

## Context & motivation

`sase plan validate` is run very frequently by agents driving the `/sase_plan` skill. Today the skill body carries the
entire plan-frontmatter schema (the tale and epic YAML shapes, the phase rules, the size/model guidance). That is a lot
of always-loaded context for a skill that fires on nearly every planning turn.

The goal is to move that schema guidance **out of the skill and into the `sase plan validate` command**, so agents only
pay for it when they actually run validation. Two coupled CLI changes make this possible:

1. A new `--explain` option that prints a tier-specific **preamble** (the schema guidance that used to live in the
   skill) before the normal validation results.
2. Removing the required `--tier` option: the command now determines the tier from the plan file's own `tier:`
   frontmatter property, and prints a clear, actionable error when that property is missing or invalid.

Once the command supports this, the `/sase_plan` skill source is updated to the slimmer form by **re-applying commit
`e0aa2327bb14`** (which was authored for exactly this feature and then immediately reverted by `8634bbc6670d` because
`--explain` did not yet exist). See the "Finalizer / commit steps" section — the coder must NOT hand-edit the skill; the
skill only changes via that git revert.

## Key facts discovered during exploration (so the coder does not re-derive them)

- The command lives in **`src/sase/main/plan_validate_handler.py`** (`handle_plan_validate_command`), parsed in
  **`src/sase/main/parser_plan.py`** (the `validate` subparser, ~lines 356-397), and rendered by
  **`src/sase/main/plan_validate_render.py`**.
- Tier detection already exists in Python: **`sase.sdd.plan_tiers.read_plan_tier(path)`** and
  **`read_plan_tier_from_content(content)`** return a normalized `"tale"`/`"epic"` or `None` (for missing/invalid tier,
  unreadable file, or bad YAML). Reuse these; do not reimplement frontmatter parsing.
- **No Rust / `sase-core` change is required.** The authoritative validator
  (`sase.sdd.plan_validate.validate_plan_file(path, tier, ...)`) still takes a tier argument, and it already emits a
  `required-missing` diagnostic on field `tier` when the property is absent and a `tier-invalid` diagnostic when the
  value is not `tale`/`epic`. This change only decides _which tier to pass_ and adds a CLI preamble — both
  presentation/routing concerns that already live in Python (matching the existing `read_plan_tier` pattern used by
  propose and the approval gate). Do not touch `../sase-core`.
- Removing `--tier` makes the CLI's `tier-mismatch` scenario **unreachable from `sase plan validate`** (the command
  always validates against the tier the file itself declares). `tier-mismatch` stays reachable via the approval
  cross-tier override path in `src/sase/_plan_approval_protocol.py` and its Rust tests — do not remove the mismatch
  logic; just stop exercising it from this CLI.
- `sase plan propose` (`plan_propose_handler.py`) already auto-detects the tier via `read_plan_tier` and defaults to
  `"tale"`. **Leave propose unchanged.** This plan only changes the `validate` subcommand.

## Behavior contract (what the coder must implement)

For `sase plan validate PLAN_FILE [flags]`:

1. **Tier is auto-detected from the frontmatter `tier:` property.** There is no `--tier`/`-t` option anymore.
   - Read the authored tier with `read_plan_tier(plan_path)`.
   - If it is a valid tier, validate the file against that exact tier via `validate_plan_file(path, tier)` (no
     `tier-mismatch` is possible).
2. **Missing / invalid tier → clear actionable error, exit 1.** When the file is readable but has no valid `tier:`
   property, the command must fail with a dedicated, human-obvious message instructing the agent to set the property,
   e.g.: `Set a valid \`tier: tale\` or \`tier: epic\` property in the plan
   frontmatter.`Recommended implementation: fall back to validating against`"tale"`so the authoritative diagnostics (the`required-missing`/`tier-invalid`on field`tier`, plus the schema table) still render, AND additionally print the dedicated actionable hint. Gate the hint on "a diagnostic exists whose `field_path
   == "tier"`", which naturally excludes file/UTF-8 errors.
3. **Unreadable / non-UTF-8 files keep today's behavior**: they must still surface `file-unreadable` / `utf8-invalid`
   (not the tier hint). Falling back to `validate_plan_file(path, "tale")` for the undetermined-tier case preserves this
   automatically, because those file diagnostics have an empty `field_path`.
4. **`--explain` (add short alias `-e`) prints a tier-specific preamble first**, then the normal validation output — on
   both success and failure. The preamble text is the tale or epic block reproduced verbatim in the next section, chosen
   by the detected tier. If the tier cannot be determined, do not guess a preamble; just emit the missing-tier error
   from (2).
5. **`--json` mode**: keep the existing stable envelope keys
   (`schema_version, ok, tier, path, diagnostics, expected_schema`). When `--explain` is passed and a valid tier is
   detected, add one extra top-level field `"explanation"` holding the preamble string. Absent `--explain`, the JSON is
   byte-for-byte as today.
6. **`--quiet`** keeps suppressing the success summary in human mode; `--explain` still prints its preamble (an explicit
   request for information overrides quiet).

This matches the workflow the re-applied skill will describe: run `sase plan validate <file> --explain` to learn the
schema + see diagnostics, edit, then rerun `sase plan validate <file>` (no flag) until it passes.

## The `--explain` preamble content (embed verbatim)

Store these two strings in the code (a small new module such as `src/sase/main/plan_explain.py` exporting the tale/epic
constants plus a `plan_explanation(tier: str) -> str` accessor is the recommended home; the render / handler layer
imports it). This prose is intentionally identical to the schema guidance that commit `e0aa2327bb14` removes from the
skill, so `--explain` becomes the single source of that guidance. Reproduce each block character-for-character.

### Tale preamble (printed when the detected tier is `tale`)

````text
A tale requires this frontmatter shape:

```yaml
---
tier: tale
title: Focused capability rollout
goal: Describe the outcome this plan will achieve.
---
# Plan: Descriptive title

Describe the implementation.
```
````

### Epic preamble (printed when the detected tier is `epic`)

````text
An epic requires a title and a non-empty ordered phase list:

```yaml
---
tier: epic
title: Workspace GC rewrite
goal: >
  Stale workspace checkouts are garbage-collected safely, and reclaim progress is visible.
phases:
  - id: core
    title: GC planner and safety checks
    depends_on: []
    size: medium
    description: "'GC planner and safety checks' section: implement workspace selection and safety guards."
  - id: cli
    title: sase workspace gc command
    depends_on: [core]
    size: small
    description: "'sase workspace gc command' section: add the CLI flow and progress reporting."
  - id: smoke
    title: End-to-end GC smoke exercises
    depends_on: [cli]
    size: small
    description: "'End-to-end GC smoke exercises' section: exercise successful and guarded cleanup."
    model: haiku
---
# Plan: Descriptive title

Describe the implementation.
```

Phase IDs must be unique slugs. Dependencies may only name earlier-listed phases; do not use self, duplicate, unknown,
or forward references. Give every phase a `description` that names its section in the plan body and briefly summarizes
that section; do not reference the plan file itself because `sase bead show` already displays it. Every phase must
declare `size: small | medium | large`. Use `medium` when the phase is potentially a lot of work and justifies its own
plan file. Use `large` when you suspect that plan file would itself be large enough to merit an epic tier. Use `small`
otherwise. Small phase agents implement directly and do not create plans. Medium and large phase agents create plans
before implementation. By default, phase size also selects the model capability appropriate for the work, unless that
phase has an explicit `model` override.

A phase's `model` is optional. Only set it when the user's prompt requested a specific model, or when that phase's agent
does not do real consequential work (for example, a phase that exercises or tests the feature itself). An explicit phase
model is allowed for every size and always takes precedence over the size-derived default. The optional top-level
`model` selects the tale's coder follow-up or the epic's land agent.
````

(The coder should treat the exact bytes inside these outer fences as the preamble. If any drift is suspected,
cross-check against the `-`/`+` lines of commit `e0aa2327bb14` — the same prose is what that commit strips from the
skill.)

## Files to change (implementation commit)

- **`src/sase/main/parser_plan.py`** — remove the `-t/--tier` argument from the `validate` subparser; add `-e/--explain`
  (`action="store_true"`). Keep options sorted alphabetically (`--explain`, `--json`, `--quiet`) per the CLI rules
  memory. Update the subparser `help`/`description` (drop "against an explicit tier schema"; say the tier is read from
  the file's `tier:` property) and the `epilog` examples (replace the three `--tier`/`-t` examples with `--explain` and
  bare-run examples). Also fix the plan-group-level example (~line 31)
  `sase plan validate sase_plan_feature.md --tier tale`.
- **`src/sase/main/plan_validate_handler.py`** — replace `tier = str(args.tier)` with tier auto-detection (via
  `read_plan_tier`), implement the missing/invalid tier error path (contract items 2-3), and thread `args.explain`
  through to rendering. `read_and_validate_plan_file(...)` (used by propose with an explicit `tier=`) keeps its current
  signature — do not break it.
- **New `src/sase/main/plan_explain.py`** (recommended) — the two verbatim preamble constants + `plan_explanation(tier)`
  accessor.
- **`src/sase/main/plan_validate_render.py`** — add rendering of the preamble in human mode and the `"explanation"`
  field in JSON mode; keep the base JSON envelope unchanged when `--explain` is absent.
- **`src/sase/_plan_approval_protocol.py`** (~line 76) — the error hint builds
  `sase plan validate {path} --tier {tier}`; drop `--tier {tier}` (optionally use `--explain`) so the suggested command
  is still valid after this change.

## Tests to update / add (implementation commit)

Primary file: **`tests/test_plan_validate.py`**. It passes `-t tale`/`-t epic` everywhere and asserts `--tier` in help —
all of that must change:

- Drop every `-t <tier>` / `--tier <tier>` argument from `_invoke(...)` calls; the plan fixtures already declare their
  own `tier:` in the frontmatter, which now drives validation.
- Replace `test_parser_requires_explicit_tier` and `test_parser_accepts_all_validate_options`: `--tier` no longer
  exists; assert the parser accepts `--explain`/`-e` (and still `--json`/`--quiet`) and no longer accepts `-t`.
- Rework the tests that depended on a CLI-produced `tier-mismatch`
  (`test_failure_human_output_is_location_bearing_and_self_teaching`, `test_failure_json_envelope_has_core_parity`, and
  the `tier: epic` + `-t tale` and `tier: story` + `-t tale` rows of
  `test_cli_passes_through_every_core_diagnostic_family`): the CLI can no longer produce `tier-mismatch`. Use a genuine
  same-tier failure to exercise the location-bearing/self-teaching output and the JSON envelope instead.
- Keep `test_missing_and_non_utf8_files_are_validation_failures` green under auto-detection (missing file still yields
  `file-unreadable`; non-UTF-8 still yields `utf8-invalid`) — drop the `-t tale` args.
- **Add new coverage**: (a) `--explain` prints the tale preamble for a `tier: tale` file and the epic preamble for a
  `tier: epic` file, before the results, on both a passing and a failing plan; (b) `--json --explain` adds the
  `"explanation"` field and omits it without `--explain`; (c) a readable file with no `tier:` (and one with
  `tier: story`) exits 1 with the actionable "set `tier: tale` or `tier: epic`" message.

Do NOT change `tests/test_plan_command_handler.py` (it covers `propose`, which is unchanged). The `--tier` hits in
`tests/main/test_bead_fast_path.py` and `tests/test_bead/*` are the unrelated `sase bead --tier` flag — leave them.

## Finalizer / commit steps — CRITICAL, read fully

Do these ONLY if/when a finalizer or commit gate asks you to commit. There are **two commits** in this repo, in order:

**Commit 1 — the implementation.** Commit everything under "Files to change" and "Tests to update / add" via the
sanctioned `/sase_git_commit` workflow. Do NOT hand-edit or include the `/sase_plan` skill source in this commit. Run
`just install` (ephemeral workspace) then `just check` and confirm green before committing.

**Commit 2 — re-apply the skill edit + keep its test in sync.**

1. Re-apply commit `e0aa2327bb14` by reverting its revert `8634bbc6670d`, but stage it into the working tree instead of
   creating a raw git commit (so the commit still goes through the sase workflow): `git revert --no-commit 8634bbc6670d`
   (then `git revert --quit` to clear any sequencer state). This restores the e0aa2327 edits to
   `src/sase/xprompts/skills/sase_plan.md`. **Do not hand-edit the skill** — the revert is the only permitted way its
   content changes.
2. In the same working tree, update **`tests/main/test_init_skills_sources.py`**: the `sase_plan` entry of
   `test_shipped_skill_source_is_discoverable_for_all_skill_providers` asserts phrases that the re-applied skill deletes
   (the `--tier` example lines and the long schema prose such as "Both tiers require a non-empty frontmatter `title`",
   "phase must declare `size: small | medium | large`", etc.). This test reads the live skill source, so it WILL fail
   after the revert. Replace those `expected_phrases` with phrases the new skill body actually contains — e.g.
   `single \`tier: <tier>\` property`, `sase plan validate sase*plan*<name>.md --explain`, `Validate (with
   \`--explain\`)`, `rerun validation without \`--explain\``, `unique slug ID` — keeping the still-present ones (`sase
   plan propose sase*plan*<name>.md`, `writes a handoff marker`, `do not poll response files yourself`). Verify against
   the re-applied source.
3. Run `just check` again and confirm green (this catches any missed phrase).
4. Commit the skill source + the test update together via `/sase_git_commit` (message along the lines of "feat: slim
   /sase_plan skill via `--explain`").
5. Run the skill init command: **`sase skill init --force`** (regenerates the deployed provider skill files from the
   updated source), then **`chezmoi apply`** to deploy them to their live locations (per the generated-skills workflow).
   These generated files live in the `chezmoi` linked repo, not this one; the user asked only to run init + deploy, then
   terminate — do not separately commit chezmoi.
6. Terminate.

## Non-goals / guardrails

- No changes to `../sase-core` / `sase_core_rs`; no changes to the Rust validator, the wire schema, or the
  `PLAN_WIRE_SCHEMA_VERSION`.
- No behavior change to `sase plan propose`, the approval gate's cross-tier override, or `sase bead --tier`.
- Do not modify the `/sase_plan` skill source by hand at any point (only via the Commit-2 revert).
- Preserve the stable `--json` envelope for the non-`--explain` case.

## Validation / done criteria

- `sase plan validate <tale-file>` and `<epic-file>` (no flag) pass/fail based on the file's own `tier:`; `-t/--tier` is
  gone.
- `sase plan validate <file> --explain` prints the correct tier preamble first, then results; `--json --explain` adds
  `"explanation"`.
- A readable file lacking a valid `tier:` exits 1 with the actionable message; an unreadable/non-UTF-8 file still
  reports `file-unreadable`/`utf8-invalid`.
- `just check` is green after Commit 1 and again after Commit 2.
- The `/sase_plan` skill matches commit `e0aa2327bb14`, regenerated via `sase skill init --force` + `chezmoi apply`.
