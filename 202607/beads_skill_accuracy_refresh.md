---
tier: tale
title: Refresh the sase_beads skill for accuracy, practice fit, and conciseness
goal: The sase_beads skill matches the current CLI contract, teaches the commands
  agents actually use in practice (including close/open), and spends fewer tokens
  doing it; contract tests and deployed skill files stay in sync.
create_time: 2026-07-27 09:39:12
status: done
---

- **PROMPT:** [202607/prompts/beads_skill_accuracy_refresh.md](prompts/beads_skill_accuracy_refresh.md)

# Plan: Refresh the sase_beads skill for accuracy, practice fit, and conciseness

Revise the `/sase_beads` xprompt skill source (`src/sase/xprompts/skills/sase_beads.md`) so it matches the current
`sase bead` CLI contract, teaches the commands agents actually use in practice, and spends fewer tokens doing it. Update
the two skill-contract tests that pin its example lists, then regenerate and deploy the skill files.

## Verified findings driving the changes

All findings below were verified against the repo at HEAD (`src/sase/main/parser_bead.py` defines the CLI contract) and
against real usage (runtime prompts in `src/sase/default_config.yml`, the commit finalizer in
`src/sase/commit_instructions.py`, and the live bead store contents).

1. **`close` and `open` are missing.** The runtime's own instructions use `sase bead close`: the `bd/land_epic` xprompt
   says "Close the epic with `sase bead close <id>`" (`src/sase/default_config.yml`), and the post-completion finalizer
   says "run `sase bead close <bead-id>`" (`src/sase/commit_instructions.py:118`). The skill only documents
   `sase bead update <id> --status closed`, so agents that learned closing from this skill diverge from every other
   instruction they receive. `close` also accepts multiple IDs and `-r/--reason`, and `sase bead open <id>` reopens
   (doctor advisories reference it: `src/sase/doctor/checks_beads.py:158`).
2. **Unquoted `--type phase(...)` examples contradict the skill's own quoting rule.** The intro says "Quote `--type`
   values so shell expansion works reliably," yet four examples use bare `--type phase(<plan-bead-id>)`, which fails in
   zsh (parentheses are glob characters). All `--type` example values must be quoted.
3. **`--size`/`-z` is undocumented.** `create` and `update` accept `--size {xsmall,small,medium,large,xlarge}` ("Phase
   size controlling plan-first prompting and default model routing", `src/sase/main/parser_bead.py`), and ~270 beads in
   the live store carry a size. Agents editing phase beads need to know the flag exists.
4. **Useful commands are undiscoverable.** `blocked`, `rm`, `stats`, `sync`, and `doctor` exist but are absent; `work`
   is mentioned only in passing. One compact list line each is enough — agents can run `--help` for details.
5. **The "Typical Workflow" does not match practice.** It teaches hand-assembling an epic (create epic bead, create
   phases, `dep add`), but in practice epic DAGs are created from an approved plan file: plan approval (or
   `sase bead work <plan.md>`) creates the plan bead, its phase beads, and their dependencies from the plan's `phases:`
   frontmatter (`src/sase/bead/epic_from_plan.py`). The workflow also closes via `update --status closed` (see finding
   1. and omits the one caution every runtime prompt repeats: never close the parent epic bead — the land agent does
      that.
6. **Redundant example bulk.** The `list` section spends 14 example lines where ~6 distinct behaviors exist, and states
   the closed-listing default limit twice; `search` has 5 examples where 3 suffice; `create` shows `--model` twice;
   `update` shows three `--status` lines; "Phase Bead Descriptions" demonstrates the same prefix in two commands.
7. **Frontmatter description omits `close`.** The description lists the command surface; add `close` so skill selection
   matches the refreshed content.

Sections verified accurate and kept (at most lightly tightened): SDD Path Convention (`sase repo path plans`,
`SASE_SDD_PLANS_DIR`/`SASE_SDD_BEADS_DIR` env vars per `src/sase/sdd/env.py`), Statuses (claim lifecycle and
`bead_claim_checks` reconciliation), Types, the slug-prefix phase-description convention (confirmed against live phase
beads, e.g. `merge: ...`, `encoding: ...`), the list output-format details (icons `○ ◎ ◐ ✓` per
`src/sase/bead_status_presentation.py`; JSON envelope keys `count`, `total`, `statuses`, `implied_status_closed`,
`results` per `src/sase/bead/cli_query.py`), and the closed-listing default of newest 20.

## Changes

### 1. Edit `src/sase/xprompts/skills/sase_beads.md`

Apply all of the following. Do not change the meaning of the kept sections; when trimming, preserve every behavioral
fact unless it is listed as a duplicate.

- **Frontmatter description**: change the command list to `(create, update, close, list, search, ready, show, dep)`.
- **Quoting**: quote every `--type` value in every example, e.g. `--type "phase(<plan-bead-id>)"`; keep the intro's
  quoting sentence.
- **create**: merge the two `--model` examples into one epic example whose comment states the semantics (epic plan bead
  → land-agent model; phase bead → per-phase work model). Add `--size` to the optional-fields example line and mention
  the choices `xsmall|small|medium|large|xlarge` and that size controls plan-first prompting and default model routing.
- **update**: keep one status example, `sase bead update <id> --status in_progress` (claiming ready work by hand), and
  point to `close`/`open` for completion and reopening. Keep the field-update examples (title, description, notes,
  assignee, design, model, clearing model, combined update) and add `--size`.
- **New `### close / open` section** (place it after `### update`):

  ```bash
  # Close finished work (the standard completion path; used by runtime prompts)
  sase bead close <id>
  sase bead close <id1> <id2> --reason "why"

  # Reopen a closed bead
  sase bead open <id>
  ```

  One sentence: `close` accepts multiple IDs and an optional `-r/--reason`; prefer it over `update --status closed`.

- **list**: keep the icons sentence, the JSON-envelope sentence, and the fallback-to-closed sentence. State the
  closed-listing limit exactly once: closed results default to the newest 20 unless `--limit`/`-n` is given (`0` =
  unlimited); the default open/claimed/in-progress listing is unlimited. Replace the 14 example lines with exactly these
  six (the contract test in change 2 pins them):

  ```bash
  sase bead list
  sase bead list --format json
  sase bead list --format full --limit 3
  sase bead list --status open --type phase
  sase bead list --tier epic
  sase bead list --status closed --limit 0
  ```

  Note that `--status`, `--type`, and `--tier` are repeatable.

- **search**: keep the explanatory paragraph (case-insensitive literal substring over human-readable fields; searches
  all four statuses by default; missing `--limit` or `--limit 0` = unlimited). Replace the five examples with exactly
  these three (pinned by the contract test in change 2):

  ```bash
  sase bead search auth
  sase bead search auth --format full --limit 3
  sase bead search auth --status open --type phase
  ```

- **Phase Bead Descriptions**: keep the convention statement, the `serialize:` example line, the resolution hint, and
  the plan/epic exclusion sentence; show only the `sase bead create` example (drop the duplicate `update` command,
  noting the same prefix applies when editing by hand).
- **New `### other commands` section** (after `### dep add`), one line per command:
  - `sase bead blocked` — list blocked beads with their blockers.
  - `sase bead rm <id> [<id2> ...]` — remove beads and all their children.
  - `sase bead stats` — project statistics. `sase bead sync` — stage bead state in git. `sase bead doctor` — health
    checks.
  - `sase bead work <epic-id|plan.md>` — launch an epic's phase and land agents (`--dry-run` previews). Normally driven
    by plan approval; do not run it casually from a working agent.
- **Typical Workflow**: replace the current 7 steps with two short parts:
  1. _Epics come from plans_: an approved epic plan file creates the plan bead, phase beads, and dependencies
     automatically (plan approval runs `sase bead work`); hand-create beads (`create` + `dep add`) only for standalone
     tracker/backlog work.
  2. _Working loop_: `sase bead ready` → `sase bead update <id> --status in_progress` → do the work →
     `sase bead close <id>`. Cautions: a bead you were launched to work is already `in_progress` (see Statuses), and
     never close the parent epic bead — the epic's land agent does that.

### 2. Update the skill-contract tests

- `tests/test_bead/test_cli_list.py::test_list_skill_examples_parse_against_cli_contract`: set the expected list to the
  six `list` examples above, in order.
- `tests/test_bead/test_cli_search.py::test_search_skill_examples_parse_against_cli_contract`: set the expected list to
  the three `search` examples above, in order. Note this test asserts `args.query == "auth"` for every example, which
  the three examples satisfy.

Both tests parse each example through `create_parser()`, so any malformed example fails loudly — keep the skill's
example lines and the tests' expected lists byte-identical.

### 3. Regenerate and deploy

Per the generated-skills workflow (`sase/memory/generated_skills.md`): after editing the source, run
`sase skill init --force`, then `chezmoi apply` to deploy the regenerated files to their live locations. Do not edit
deployed `SKILL.md` files directly.

## Validation

1. `just install` (fresh-workspace requirement), then `just check`.
2. `pytest tests/test_bead/test_cli_list.py tests/test_bead/test_cli_search.py tests/main/test_init_skills_sources.py`
   passes.
3. Every `sase bead ...` example line in the revised skill parses: the two contract tests cover `list`/`search`; eyeball
   the rest against `src/sase/main/parser_bead.py` (especially that all `--type` values are quoted and that
   `close`/`open`/`--size` usages match the parser).
4. `sase skill init --force && chezmoi apply` completes and the deployed `sase_beads/SKILL.md` reflects the new content.

## Out of scope

- Changing `sase bead` CLI behavior, help text, or `sase bead onboard` output.
- Documenting rarely used create flags (`--bug-id`, `--changespec`) or the admin commands beyond one-line mentions
  (`init`, `onboard`, `resolve-conflicts` stay undocumented; they are user-driven or self-explanatory via `--help`).
- Other skill files.
