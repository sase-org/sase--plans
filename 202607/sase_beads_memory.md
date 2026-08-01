---
tier: epic
title: Migrate the sase_beads skill into generated Tier 2 memory
goal: '`sase bead` guidance lives in a concise, auto-generated `sase/memory/sase_beads.md`
  long-term memory note that every sase-managed project and the home root receive
  automatically, and no copy of the `/sase_beads` skill remains in the sase repo,
  the chezmoi repo, or the home directory.

  '
phases:
- id: generate
  title: Generated Tier 2 bead memory note
  depends_on: []
  size: medium
  description: 'generate: add the packaged bead-note asset plus the generated-long-note
    plumbing that writes `sase/memory/sase_beads.md` into every memory root and lists
    it in Tier 2 of every AGENTS.md.'
- id: retire
  title: Retire the sase_beads skill source
  depends_on:
  - generate
  size: small
  description: 'retire: delete the skill source and every in-repo reference to it,
    including the three bead CLI-contract tests that parsed the skill file.'
- id: cleanup
  title: Remove deployed skill copies and verify rollout
  depends_on:
  - retire
  size: small
  description: 'cleanup: git rm the six chezmoi skill copies, delete the seven home
    skill directories, and verify the new note reached home memory.'
proposed_by: bbugyi200.athena.qn
create_time: 2026-07-31 15:00:50
status: done
bead_id: sase-cp
---

- **PROMPT:** [prompts/202607/sase_beads_memory.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/sase_beads_memory.md)
- **BEAD:** [sase-cp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-cp/README.md)

# Plan: Migrate the sase_beads skill into generated Tier 2 memory

## Background

`src/sase/xprompts/skills/sase_beads.md` is a 385-line (~4.4k token) skill that is almost entirely _semantic_ memory: it
restates `sase bead` reference material rather than describing a procedure the agent must follow. The project owner
asked for it to become a long-term memory note instead, auto-added to every sase-managed project the same way
`sase/memory/sase.md` is, and made much more concise while keeping the non-obvious value.

Facts established while planning (do not re-derive them):

- `sase memory init` generates exactly one memory note today: `sase/memory/sase.md`, a **short** (Tier 1) note rendered
  from the packaged template `src/sase/main/init_memory/templates/memory-sase.template.md`.
- It runs for two roots per invocation — the project root (cwd) and the home root, which is the chezmoi source
  `~/.local/share/chezmoi/home` when chezmoi mode is on. The home root's writes are committed and applied to `~` by
  `_deploy_to_chezmoi` in `src/sase/main/init_memory_handler.py`, so a single `sase memory init` run rolls generated
  memory out to `~/sase/memory/` and to every provider instruction shim in `~`.
- `just check` runs `sase validate`, which includes `init memory --check` and `init skills --check`. Any change to
  generated memory therefore requires a `sase memory init` run in the same change, or `just check` fails.
- `sase skill init` never prunes: deleting a skill source leaves the generated `SKILL.md` copies behind in the chezmoi
  source and in `~`, and `chezmoi apply` does not delete them either. Removal is manual.

## Design decisions

1. **Tier 2, not Tier 1.** The note is `type: long`, reached through the AGENTS.md Tier 2 list and read with
   `sase memory read sase_beads.md`. Bead reference material must not sit in every agent's always-loaded context.
2. **Generated, not hand-written.** A hand-written note would only exist in this repo. Generating it gives every
   sase-managed project and the home root a copy on their next `sase memory init`, which is what "available to all
   sase-managed projects" requires.
3. **Static packaged asset.** The note has no per-root variation, so the packaged template takes no Jinja variables and
   gets **no** `sase.yml` override key (unlike `memory_sase_template` / `memory_readme_template`). Add an override key
   later only if a project actually needs to customize bead guidance.
4. **The note carries its own frontmatter.** Keeping `description:` next to the prose it describes is the maintainable
   arrangement; the generator re-normalizes it through `apply_memory_frontmatter` so the written file always has
   canonical frontmatter shape.
5. **Conciseness is the point.** The note documents only what `sase bead <cmd> -h` cannot tell an agent. Flag catalogs,
   per-command example galleries, JSON envelope field names, and output glyph legends are deliberately dropped.

## Generated Tier 2 bead memory note

Add the note asset and the plumbing that makes a _generated long note_ work end to end. The existing pipeline overlays
only generated **short** notes (`generated_short_notes` in `src/sase/amd/_memory.py`), so a generated long note needs an
equivalent overlay or the first `sase memory init` in a root would write the note without listing it in that root's
AGENTS.md Tier 2 section — and the very next `sase memory init --check` would report drift.

### Files

- **New** `src/sase/main/init_memory/templates/memory-sase-beads.template.md` — the note content in full (verbatim
  below). Name it `*.template.md` for consistency with its two siblings even though it has no Jinja variables; render it
  through `render_markdown_template(..., required_variables=frozenset(), context={})` with no override path.
- `src/sase/main/init_memory/root_rendering.py`
  - Add the template filename constant and a `_generated_beads_memory_relative_path()` helper returning
    `CANONICAL_MEMORY_RELATIVE_ROOT / "sase_beads.md"`.
  - Add a renderer that returns the finished note content: render the packaged template, run
    `format_generated_memory_markdown`, parse it with `parse_memory_note_text` to read its `description`, then re-apply
    canonical frontmatter with `apply_memory_frontmatter(..., note_type="long", parent=AGENTS_PARENT, description=...)`.
    Return a blocker string (same shape as the existing render errors) when the asset is missing or carries no
    description, so a broken asset fails initialization loudly instead of shipping a description-less Tier 2 entry.
  - Add a `generated_long_notes()` accessor mirroring `generated_short_notes()`, mapping the note's root-relative path
    to its description, for the AGENTS.md overlay.
  - In `render_expected_memory_files`, add the note to `note_overlay` (so the generated README counts and describes it)
    and append a `MemoryExpectedFile` for it, alongside the existing generated-`sase.md` entry.
- `src/sase/amd/_memory.py`
  - Thread a `generated_long_notes: Mapping[str, str]` (path → description) parameter through `plan_amd_memory_sync`
    into `_long_memory_descriptions`, `_long_memory_description_updates`, and `_render_managed_agents`.
  - `_long_memory_descriptions`: overlay the generated descriptions on top of the disk-discovered ones.
  - `_long_memory_description_updates`: **skip** generated paths. Their content is owned by the expected-files list;
    emitting a frontmatter "update" for the same path would produce two competing writes for one file.
  - `_render_managed_agents`: build `top_level_long_notes` from the disk-discovered notes unioned with a synthetic
    `MemoryNote` for each generated path not yet on disk (`type="long"`, `parent=AGENTS_PARENT`, description from the
    overlay, empty body), keeping the list sorted by `relative_path`. This is what makes the first init in a fresh root
    self-consistent.
- `src/sase/main/init_memory/root_planning.py` — render the note once in `memory_root_context`, pass its description
  overlay to `_amd_sync_plan` and its content to `render_expected_memory_files`, and surface render blockers the same
  way the `sase.md` render error is surfaced today.
- `src/sase/main/init_memory/project_deploy.py` — stage `memory_write_root(root) / "sase_beads.md"` next to the existing
  `sase.md` entry in the `stage_paths` tuple, so the generated note is committed with the rest of a project memory init.
- `src/sase/main/init_memory/templates/memory-README.template.md` — extend the "How Memory Files Are Used" bullet that
  names `sase/memory/sase.md` so it also names `sase/memory/sase_beads.md` as generated content.
- `docs/memory.md` and `docs/init.md` — mention the second generated note where the generated `sase/memory/sase.md` is
  described. No `docs/configuration.md` change: this template intentionally has no override key.

Watch `just _lint-toobig` (1000/850/700 line thresholds) if `root_rendering.py` or `_memory.py` grows; both have ample
headroom today.

### Note content

Write this file verbatim as `memory-sase-beads.template.md`. It is already wrapped for the repo's 120-column Markdown
style; run `just fmt` and let `format_generated_memory_markdown` and prettier agree before finishing (the generated
`sase/memory/sase_beads.md` in this repo is checked by `just fmt-md-check` too).

````markdown
---
type: long
parent: AGENTS.md
description:
  Read before creating, updating, closing, or querying sase beads — bead types and tiers, the status lifecycle agents
  must never hand-edit, task-bead triage, phase-bead description prefixes, and non-cascading close, resolution, and note
  semantics.
---

# SASE Beads

`sase bead <cmd> -h` documents every flag and `sase bead onboard` prints a quick start; this note covers only what that
help output cannot tell you. Always invoke `sase bead`, never `.venv/bin/sase bead`.

## Store And IDs

`sase bead` reads and writes the current effective SDD bead store, so never hand-build `sdd/...` paths. Launched agents
have `SASE_SDD_PLANS_DIR` (plan files) and `SASE_SDD_BEADS_DIR` (bead store); elsewhere resolve them from
`sase repo path plans`. Canonical state lives in `beads/events/**`; `issues.jsonl` is a generated projection. Sibling
workspaces and legacy stores are never merged in.

Any argument naming an existing bead accepts the full ID (`sase-a1`) or the suffix after the final dash (`a1`, `a1.2`).
Ambiguous shorthand fails and lists the candidates. Output and stored relationships always use full IDs.

## Types, Tiers, And Launching

- `plan` — `-T "plan(<plan_file>[,<parent_id>])"`, top level, `--tier plan|epic`.
- `phase` — `-T "phase(<parent_id>)"`, child of a plan bead.
- `task` — `-T task`, standalone discovered follow-up; no tier, optional `--size` for model routing.

`sase bead work <epic-id|plan.md|task-id>` launches an epic's phase and land agents or one task worker. Epic launches
normally come from plan approval and task launches from a `TaskTriage` gate, so hand-create beads only for tracker or
backlog work. `--model` on an epic plan bead selects its land agent's model; on a phase bead it selects that phase's
work model.

A phase bead's description comes from its entry in the epic plan's `phases:` frontmatter and starts with that phase's
slug ID and `: ` (for example `login: add the endpoint and its auth checks.`). Keep that prefix when you write or edit
one by hand. Plan and epic bead descriptions do not take it.

## Statuses

`open` (draft) · `claimed` (runtime reserved) · `ready` (task bead awaiting triage) · `in_progress` · `closed`.

Never set `claimed` by hand — the agent runner owns that transition. A bead you were launched to work on is already
`in_progress` before you read your prompt, because an epic launch preassigns every phase bead and the land bead. Rerun
`sase bead work <epic-id>` to recover a runner that died; it reassigns every non-closed bead and never touches closed
phases.

## Task Beads

Capture useful work that falls outside your current task as a task bead: create it as an `open` draft, refine its
description and dependencies, then mark it `ready`. Each ready bead raises one `TaskTriage` gate, from which the owner
either launches `sase bead work <id>` or closes it as `canceled`.

```bash
sase bead create -T task -t "<title>" -d "<what is wrong and how you found it>"
sase bead update <id> -s ready
```

Epic phase workers are the exception: they never create beads. They append `PROPOSED FOLLOW-UP: <summary — detail>` to
their own phase bead, and the epic's land agent decides which proposals become task beads.

## Closing

```bash
sase bead close <id> --note "<what you verified>"
```

That is the completion path; prefer it over `update -s closed`. Every close records a resolution (`done` by default,
else `canceled` or `superseded` through `-R`) plus free-text `--reason`.

- **Closing never cascades.** A bead with any unclosed descendant is rejected, the error names those beads, and nothing
  is written. Finish or reopen them instead.
- `--force` sweeps unfinished descendants closed; it requires `--reason` and a resolution other than `done`.
- Re-closing is a safe no-op (exit 0, no event, no commit), but a conflicting resolution or reason is refused. Record
  later evidence with `sase bead note`.
- `-p 1-3` on an epic bead closes only those phase beads. `sase bead open <id>` reopens a bead and every closed ancestor
  above it.
- Never close the parent epic bead; its land agent does that.

## Notes And History

`sase bead note <id> "<text>"` appends an attributed entry atomically, while `update --notes` replaces the whole field,
so use `note` for progress, verification, and handoff state. `sase bead history <id>` replays the event stream field by
field (`--format full` recovers a value a later write replaced), and `sase bead history --lost-notes [--restore]` finds
and re-appends notes text that went missing.

## Reading And Repairing

`list` (repeatable `--status`/`--type`/`--tier` filters; with no `--status` and nothing active it falls back to closed
beads and says so), `search` (case-insensitive substring), `ready` (unblocked ready task beads), `show`, `blocked`,
`stats`, `dep list|tree|add|rm`, `ref list|add|rm` (artifact references, stored without the prompt-time `@`), `rm` (a
bead and all its children), and `doctor` (bead-store, plan-link, and artifact-reference health, with confirmed `--fix-*`
repairs).
````

### Tests

- Rendering unit test: the generated note has `type: long`, `parent: AGENTS.md`, a non-empty description, and lands at
  `sase/memory/sase_beads.md`.
- `plan_memory_root` on a fresh temp root: the expected files include the note, the rendered AGENTS.md Tier 2 section
  lists `sase/memory/sase_beads.md` with the asset's description, and the generated README counts it.
- **Idempotence regression test** (the reason the overlay exists): initialize a fresh root, then assert a following
  `--check` pass reports no drift.
- New bead CLI-contract test replacing the three the next phase deletes: extract every line inside a
  ```bash fence of the packaged asset that starts with `sase bead
  `, substitute `<...>`placeholders, and assert each parses with`create_parser()`. This keeps the note's runnable
  examples honest, and it is why the note keeps its examples literal and inside fenced blocks — keep them that way.
- Update any existing `tests/main/test_init_memory_*.py` assertions that enumerate the expected generated-file set.

### Verification

`just install`, then `sase memory init` in this workspace (this writes `sase/memory/sase_beads.md`, refreshes
`AGENTS.md` and the provider shims, and deploys the note to the home root through chezmoi), then `just check`.
`sase memory read sase_beads.md --reason "..."` must return the new note.

## Retire the sase_beads skill source

Delete the skill and every in-repo reference to it.

- Delete `src/sase/xprompts/skills/sase_beads.md`.
- `docs/xprompt.md` — remove the `sase_beads` row from the skills table (~line 900).
- `tests/main/test_init_skills_sources.py` — remove the `"sase_beads"` parametrized entry and its expected-substring
  tuple (~line 180).
- Delete these three tests, which read the deleted skill file and assert exact example lists; the previous phase's
  packaged-asset contract test replaces their value:
  - `tests/test_bead/test_cli_show.py::test_show_skill_examples_parse_against_cli_contract`
  - `tests/test_bead/test_cli_list.py::test_list_skill_examples_parse_against_cli_contract`
  - `tests/test_bead/test_cli_search.py::test_search_skill_examples_parse_against_cli_contract`

  Remove the imports (`Path`, `shlex`) that become unused in each file; leave every other test in those files alone.

- `tests/main/test_init_skills_handler.py` — the docstring at ~line 141 names `sase_beads/SKILL.md` as an example; the
  test itself uses a stubbed source, so just reword it to a neutral skill name.
- Leave these alone: `.gitignore`, `tests/test_agent_bead_display.py`, `tests/test_bead/test_workspace_resolution.py`,
  and `tests/main/test_bead_fast_path_context.py` match `.sase_beads`, the unrelated legacy bead-store directory name;
  `tests/fixtures/qwen_stream/qwen-code-0.15.10-tools.jsonl` is a recorded provider transcript that must stay verbatim.

Verify with `just check` (`sase validate`'s `init skills --check` must stay green) and `sase skill list`, which should
no longer list `/sase_beads` among its sources.

## Remove deployed skill copies and verify rollout

`sase skill init` never prunes and `chezmoi apply` never deletes, so the generated copies must be removed by hand. Open
the chezmoi repo with the `/sase_repo` skill (`sase repo open chezmoi -r "<reason>"`) and use only the path it prints.

1. In the chezmoi source, `git rm -r` these six directories:
   - `home/dot_claude/skills/sase_beads`
   - `home/dot_codex/skills/sase_beads`
   - `home/dot_qwen/skills/sase_beads`
   - `home/dot_config/opencode/skills/sase_beads`
   - `home/dot_gemini/antigravity-cli/skills/sase_beads`
   - `home/dot_gemini/jetski/skills/sase_beads`
2. Commit that removal from the chezmoi checkout with the `/sase_git_commit` skill (`-t create_commit`; chezmoi is not a
   PR workflow). The chezmoi project's configured after-commit hook runs `chezmoi update -a --force`, which applies the
   change to `~`; confirm it ran.
3. Delete the deployed copies still present under the home directory, including one orphan that chezmoi does not manage:
   `~/.claude/skills/sase_beads`, `~/.codex/skills/sase_beads`, `~/.qwen/skills/sase_beads`,
   `~/.config/opencode/skills/sase_beads`, `~/.gemini/antigravity-cli/skills/sase_beads`,
   `~/.gemini/jetski/skills/sase_beads`, and the unmanaged `~/.gemini/skills/sase_beads`.
4. Verify:
   - `find ~ -maxdepth 6 -path '*skills*sase_beads*' -not -path '*/.local/state/sase/*'` prints nothing.
   - `sase skill list` shows no `/sase_beads` source and no `sase_beads` target.
   - `~/sase/memory/sase_beads.md` exists (the first phase's `sase memory init` deploys it through chezmoi) and the home
     `~/AGENTS.md` / `~/CLAUDE.md` Tier 2 section lists it. If it is missing, re-run `sase memory init` from the sase
     workspace and confirm the chezmoi deploy succeeded.
   - `sase memory read sase_beads.md --reason "verify bead memory rollout"` returns the note.

If a stale `sase skill init --force` run resurrects the copies before this epic lands, simply repeat steps 1–3; the land
agent should re-run the step 4 verification after landing.

## Permissions and follow-ups

- The project owner requested this migration directly and approves it through this epic's approval gate. That approval
  is the explicit user permission required by the `Memory File Edits Require Explicit User Permission` gotcha, and it
  covers exactly two things: adding the generated `sase/memory/sase_beads.md` (through its packaged asset) and the
  `sase memory init` runs that regenerate `AGENTS.md`, the provider shims, and the memory README. Do not edit any other
  memory note.
- Follow-up worth a task bead once this lands: `sase skill init` has no pruning path, so retiring any skill leaves
  orphan `SKILL.md` copies in the chezmoi source and in `~`. A `--prune` mode that deletes targets whose source no
  longer exists would make future retirements one command instead of a manual sweep.
