---
status: done
tier: epic
title: Restore `feature` as an agent-creatable task type in every project
goal:
  "The builtin `feature` task type is agent-creatable everywhere by default: no
  machine-global `bead.task_types` entry suppresses it, the `sase` project no longer
  carries a re-enable override, and the shipped `create_refusal` copy no longer names
  one machine's policy."
phases:
  - id: global-default
    title: Drop the machine-global `feature` disable from chezmoi
    depends_on: []
    size: small
    description:
      "global-default: delete the `bead.task_types` block from the chezmoi-managed
      `home/dot_config/sase/sase.yml` so no machine-global layer turns `agent_creatable`
      off for `builtin@feature`. This phase's commit runs chezmoi's own `after` hook,
      which redeploys `~/.config/sase/sase.yml` and makes the change live for every
      project on the machine."
  - id: sase-cleanup
    title: Neutralize the builtin refusal copy and retire the sase re-enable override
    depends_on:
      - global-default
    size: small
    description:
      "sase-cleanup: in the sase repo, rewrite the `create_refusal` copy in
      `_feature_spec()` so it stops asserting that feature beads belong only to SASE,
      delete the now-dead `use: builtin@feature` override from `sase/sase.yml`, extend
      the builtin copy test, and regenerate `sase/task_types.json`. Gated on
      `global-default` because that regeneration reads the live machine-global config
      layer."
proposed_by: bbugyi200.athena.0dk
bead_id: sase-tn
create_time: 2026-09-09 19:50:17
---

- **PROMPT:**
  [prompts/202608/feature_task_type_default.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/feature_task_type_default.md)
- **BEAD:**
  [sase-tn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tn/README.md)

# Plan: restore `feature` as an agent-creatable task type in every project

## Goal

`feature` is one of SASE's five shipped builtin task types and it is agent-creatable by
default (`src/sase/task_types/_models.py:57-58` — `spec.get("agent_creatable", True)`).
Today that default is overridden off for every project on this machine, and the override
has leaked into SASE's shipped source. After this epic:

- no config layer disables `builtin@feature`;
- `sase/sase.yml` carries no `feature` entry at all;
- the builtin's `create_refusal` copy is policy-neutral, so a project that _does_
  disable the type gets a useful refusal instead of one machine's history.

## Problem

### What is configured today

Two layers conspire, and both were added by
`sase/repos/plans/202608/feature_task_type_scoping.md` (tier `tale`, status `wip`):

- `~/.config/sase/sase.yml` (chezmoi source: `home/dot_config/sase/sase.yml:3-7`)
  disables the type for every project:

  ```yaml
  bead:
    task_types:
      # Only the `sase` project files `feature` beads; sase/sase.yml re-enables it there.
      - use: builtin@feature
        agent_creatable: false
  ```

- `sase/sase.yml:62-67` re-enables it for the `sase` project only:

  ```yaml
  bead:
    task_types:
      # Re-enable the builtin `feature` type, which ~/.config/sase/sase.yml turns off for
      # every project. Product ideas discovered while working on SASE belong here.
      - use: builtin@feature
        agent_creatable: true
  ```

That plan's D2 chose the machine-global disable deliberately; D5 chose to leave the
other project repos alone. This epic reverses D2 and, as a consequence, D5's rationale.
Nothing else in that plan is reversed: D3 (`use:` prefixes match the type's original
provider) landed in `c624ce55f` and is load-bearing for this change too, because
`use: builtin@feature` in `sase/sase.yml` is only accepted after that fix. D4 (a
home-level task-type note) is moot: `sase memory init` no longer renders a home
task-types note at all — the catalog is a project-scoped memory web now
(`src/sase/main/init_memory/root_rendering_task_types.py:196-206`), and
`~/.config/sase/AGENTS.md` has no "Task Bead Types" section.

### Symptom 1 — every non-`sase` project rejects the type

Verified in the two other enabled projects' committed snapshots
(`sase/task_types.json`):

```text
sase    | source: project | package: project | creatable: True  | digest: 9445f9171caf31b5
actstat | source: project | package: project | creatable: False | digest: 25c171d60599d599
bob-cli | source: project | package: project | creatable: False | digest: 25c171d60599d599
```

Their generated `sase/memory/task_types.md` rosters list `bug`, `ci`, `flake`, `memory`
and omit `Feature`, so an agent in `actstat` or `bob-cli` that discovers an out-of-scope
product idea has no type to file it under and gets a hard `sase bead create` refusal if
it tries.

Note the `source: project` / `package: project` in all three rows:
`_apply_use_override()` rewrites provenance when any layer touches a slug
(`src/sase/task_types/_project_config.py:163-173`), so even the two projects that never
configured anything now describe `feature` as a project-owned type. It is a builtin.

### Symptom 2 — the shipped builtin encodes one machine's policy

`src/sase/task_types/_builtin.py:158-163` ships this to every SASE installation:

> Agents never create this type with `sase bead create` or `/sase_new_task` when it is
> not agent-creatable. That is the machine-global override outside the SASE project.
> **File a feature bead only in SASE**, or use an in-scope type here.

`create_refusal` is the copy printed after "task type '<slug>' cannot be created by
agents" (`src/sase/task_types/fields.py:73-88`). Its job is to keep a refusal from
reading like the invitation in `when_to_use`. That job is legitimate and worth keeping;
naming one user's project and one user's `~/.config` in a shipped default is not. It
also travels: `snapshot.py:42-44` copies `create_refusal` into every project's committed
`sase/task_types.json`.

### Symptom 3 — the `sase` override is a no-op that documents a fiction

Once the machine-global entry is gone, `sase/sase.yml`'s `agent_creatable: true`
restates the builtin default, and its comment points at a config block that no longer
exists. Keeping it also keeps `feature` reported as `source: project` in
`sase/task_types.json`, which is wrong.

## Design

I own the decisions below. Implement them as specified rather than re-deciding them.

### D1. Delete both entries; do not set `agent_creatable: true` anywhere

The builtin default is already `True`. The machine-global `bead:` block and the
`sase/sase.yml` `use: builtin@feature` entry are both deleted outright.

Rejected: flipping the machine-global entry to `agent_creatable: true`. It reads as
policy, invites the next reader to wonder what it is answering, and keeps `feature`'s
provenance rewritten to `source: project` in every project's snapshot for no benefit.

Rejected: moving the disable down into `actstat/sase/sase.yml` and
`bob-cli/sase/sase.yml` to preserve today's behavior there. The user's instruction is
that the type should not be disabled for projects by default; that is the whole request.

### D2. `create_refusal` stays on the builtin, with policy-neutral copy

Replace the third and fourth sentences. The new copy must:

- keep the substring `never create` and one of `cannot be created` /
  `not agent-creatable` (both are asserted by
  `tests/task_types/test_builtin.py:110-119`, which stays valid);
- stay under 400 characters and be `strip()`-clean (same test);
- name no project, no config file, and no machine;
- tell the agent what to do instead.

Use exactly this text:

```python
        "create_refusal": (
            "Agents never create this type with `sase bead create` or "
            "`/sase_new_task` where it is not agent-creatable. Do not refile "
            "the idea under another type: record it as a `PROPOSED FOLLOW-UP:` "
            "note on the bead you are working, or raise it with the project "
            "owner."
        ),
```

Rejected: deleting the `create_refusal` key. `_agent_create_refusal_message()` would
fall back to `when_to_use`, and a refusal that opens "File one when you discovered a
product or capability idea..." reads as an invitation to do the thing that was just
refused. That regression is exactly what
`test_feature_create_refusal_is_not_the_when_to_use_invitation` was written to prevent.

Rejected: leaving the copy alone because no project disables the type after this change.
A shipped default is read by every SASE user, not only by this machine; and the copy is
embedded verbatim in every project's committed `task_types.json`.

### D3. Two phases, because the machine-global config only goes live after a commit

`~/.config/sase/sase.yml` is a chezmoi deployment target. `sase repo open chezmoi` hands
out a _workspace clone_, not `~/.local/share/chezmoi`, so an agent editing that clone
cannot apply the change in its own turn. What does apply it is chezmoi's own project
config (`<chezmoi>/sase/sase.yml`):

```yaml
commit_hooks:
  after: "chezmoi update -a --force"
```

`CommitWorkflow` runs `commit_hooks.after` after the push
(`src/sase/workflows/commit/workflow.py:271-290`,
`workflows/commit/command_hooks.py:44-49`), so the redeploy happens when the host
finalizer commits phase 1 — after that phase's turn ends.

This matters because `sase memory init` renders `sase/task_types.json` from the **live**
merged config (`root_rendering_task_types.py:103-118` → `get_task_type_registry()`),
which includes the `user` layer. If `sase-cleanup` deleted the `sase/sase.yml` override
while `~/.config/sase/sase.yml` still disabled the type, the regeneration would mark
`feature` `agent_creatable: false`, drop `sase/memory/task_types/feature.md`, and strip
`Feature` from `AGENTS.md` and all four provider shims — the exact opposite of the goal,
committed as a clean green build.

Rejected: one `tale`. It cannot be ordered correctly inside a single turn.

Rejected: hand-editing `~/.config/sase/sase.yml` directly so one turn can do both. It is
a chezmoi target; the next `chezmoi update` reverts it, and it is not a path this repo's
agents may write.

Rejected: three phases (splitting the `_builtin.py` copy edit out so it can run in
parallel with `global-default`). Both sase-repo edits regenerate `sase/task_types.json`;
splitting them buys a few minutes of parallelism in exchange for a guaranteed conflict
in a generated file.

### D4. `actstat` and `bob-cli` are regenerated after the epic lands, not inside it

Their `sase/task_types.json` and `sase/memory/task_types.md` become stale the moment
`global-default` is deployed, and staler again when the new `create_refusal` digest
lands. Both are fixed by one `sase memory init` per repo — but that must happen after
this epic is on `master`, because the `sase` executable those repos use is a uv-tool
**editable install pointing at `/home/bryan/projects/github/sase-org/sase/src`**
(`~/.local/share/uv/tools/sase/lib/python3.14/site-packages/_editable_impl_sase.pth`).
Regenerating them from an epic phase would write a `create_refusal` and digest that the
installed `sase` does not yet produce, and they would drift right back.

The drift is self-announcing: `sase validate` runs `init memory --check`
(`src/sase/main/validate_handler.py:39`), which reports "`feature` spec digest changed
... run `sase memory init`". See [After the epic lands](#after-the-epic-lands) for the
two commands.

Rejected: a third phase that opens both repos and regenerates them. Those commits land
on their own default branches immediately, while this epic is still on a branch — so
they would advertise a spec that no installed `sase` produces, and they would be wrong
for as long as the epic is unlanded (or forever, if it is abandoned).

### D5. No documentation change

`docs/beads.md:195-200` documents the suppression idiom generically ("Suppress a builtin
for a project or the whole machine with `use: <plugin>@<slug>` plus
`agent_creatable: false`... A machine-global entry in `~/.config/sase/sase.yml` applies
to every project on the machine"). That paragraph describes a mechanism this epic keeps
and does not name `feature` as its example. The only `feature` mentions in `docs/`
(`docs/plugins.md:677`, `docs/beads.md:214`) are neutral list entries. Do not edit
either file.

## Phase `global-default` — drop the machine-global disable

**Repo:** chezmoi (linked). **Depends on:** nothing.

### Steps

1. Open the repo and use only the printed path for every read and write:

   ```bash
   sase repo open chezmoi -r "Remove the machine-global bead.task_types entry that disables builtin@feature"
   ```

2. In `<chezmoi>/home/dot_config/sase/sase.yml`, delete the entire top-level `bead:` key
   — the comment line, `task_types:`, and the single `use: builtin@feature` /
   `agent_creatable: false` entry (lines 3-7, comment included). `bead:` has no other
   members in that file, so the key goes away with them. Leave `timezone`,
   `use_chezmoi`, and everything below `artifacts:` untouched.

3. Confirm nothing else in the chezmoi repo references the entry:

   ```bash
   grep -rn "builtin@feature" <chezmoi>
   ```

   Expect no matches.

### Verification

- `grep -c "^bead:" <chezmoi>/home/dot_config/sase/sase.yml` reports `0`.
- The file still parses:
  `python3 -c "import sys,yaml;yaml.safe_load(open(sys.argv[1]))" <chezmoi>/home/dot_config/sase/sase.yml`.

### Notes for the phase worker

- This is a linked repo opened through `/sase_repo`; it is part of your final
  declaration and needs its own `commit` decision.
- Do **not** run `chezmoi apply`, `chezmoi update`, or edit `~/.config/sase/sase.yml`.
  The redeploy is chezmoi's own `commit_hooks.after`, and it runs when the host commits
  this phase.
- Do not touch the `sase` repo in this phase.

## Phase `sase-cleanup` — neutral refusal copy, no project override

**Repo:** sase (primary). **Depends on:** `global-default`.

### Steps

1. `just install` first — this is an ephemeral workspace and the Rust binding may not be
   built yet.

2. **`src/sase/task_types/_builtin.py`** — in `_feature_spec()`, replace the
   `create_refusal` value with the exact string given in D2. Change nothing else about
   the spec: `label`, `summary`, `when_to_use`, `glyph`, `accent_color`, `fields`, and
   `body_template` all stay as they are, and do **not** add an explicit
   `agent_creatable` key (`tests/task_types/test_builtin.py:98-101` asserts builtins
   leave it unset).

3. **`sase/sase.yml`** — delete the first `bead.task_types` entry and its two comment
   lines (currently lines 64-67), so the list starts directly with the `flag` entry:

   ```yaml
   bead:
     task_types:
       - schema_version: 1
         task_type: flag
   ```

   Keep the `bead:` and `task_types:` keys — `flag` still needs them.

4. **Gate before regenerating.** Confirm the machine-global layer really is gone; if it
   is not, the regeneration in step 6 will silently delete `Feature` from the project's
   memory:

   ```bash
   .venv/bin/sase bead task-type show feature --json
   ```

   Require `"agent_creatable": true` **and** `"provenance": {"source": "builtin", ...}`
   — note `source` is nested under `provenance` in this payload
   (`src/sase/task_types/cli_show.py:104-110`), and it reads `"project"` today because
   the override deleted in step 3 rewrites it. If the command reports
   `agent_creatable: false`, or a `provenance.source` of `project`, then
   `global-default` has not been redeployed to `~/.config/sase/sase.yml` yet: **stop,
   change nothing further, and report the blocker.** Do not hand-edit the deployed
   config and do not run `chezmoi` yourself.

5. **`tests/task_types/test_builtin.py`** — extend
   `test_feature_create_refusal_is_not_the_when_to_use_invitation` (line 110) so the
   existing assertions stay and the copy is pinned as policy-neutral. Add, against the
   lowercased copy:

   ```python
   for banned in ("machine-global", "~/.config", "only in sase", "sase project"):
       assert banned not in lowered
   ```

   and assert the copy still tells the agent what to do instead, e.g.
   `assert "PROPOSED FOLLOW-UP:" in create_refusal`. Consider renaming the test only if
   the new name is clearly better; the existing name is still accurate.

6. **Regenerate the committed snapshot.** Preview first, then write:

   ```bash
   .venv/bin/sase memory init --check --diff
   .venv/bin/sase memory init --no-commit
   ```

   The only planned change must be `sase/task_types.json`. Its `feature` entry should
   move to `"source": "builtin"`, `"package": "sase"`, keep `"agent_creatable": true`,
   carry the new `create_refusal`, and get a new `digest`. No other entry changes.

   If `--check --diff` plans to touch `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
   `OPENCODE.md`, `QWEN.md`, or anything under `sase/memory/` — in particular if it
   plans to delete `sase/memory/task_types/feature.md` or drop the `Feature` row from
   `sase/memory/task_types.md` — the step-4 gate was wrong. Stop and report; do not
   write.

   Use `--no-commit`: completion is host-owned.

7. Verify and hand off:

   ```bash
   just check
   ```

   Run it through `/sase_monitor` if it is slow, with
   `--start-status TESTING --stop-status TESTED`.

### Verification

- `git diff --stat` shows exactly four paths: `src/sase/task_types/_builtin.py`,
  `sase/sase.yml`, `tests/task_types/test_builtin.py`, `sase/task_types.json`.
- `git diff sase/task_types.json` touches only the `feature` object.
- `.venv/bin/sase bead task-type list -a` still shows `feature` as agent-creatable, and
  `flag` and `github` still as not (plain `list` hides agent-uncreatable types, so pass
  `-a`).
- `.venv/bin/sase validate` passes (it is inside `just check`; `init memory --check` is
  the check that would fail on a stale snapshot).
- `just check` is green.

## After the epic lands

Not part of any phase — these two repos are on their own default branches and their
`sase` executable is the editable install at
`/home/bryan/projects/github/sase-org/sase/src`, so they must be regenerated only once
this epic is merged and that checkout is up to date (D4):

```bash
sase repo open actstat -r "Regenerate the task-type snapshot after feature became agent-creatable by default"
cd <printed path> && sase memory init          # expect: feature -> source builtin, agent_creatable true
```

and the same for `bob-cli`. Each run adds `Feature` to that project's
`sase/memory/task_types.md` roster and writes a new `sase/memory/task_types/feature.md`
strand, so each is a memory-file change in that repo and needs the project owner's
go-ahead in the turn that does it. Until then, `sase validate` in those repos reports
the drift and names `sase memory init` as the fix, so nothing fails silently.

## Risks

- **Ordering (highest).** Covered by the step-4 gate in `sase-cleanup`. The failure mode
  is quiet rather than loud: a regeneration run too early produces a green tree that
  removes `Feature` from SASE's own agent instructions.
- **Existing `feature` beads.** Unaffected in every direction. Nothing about this change
  touches bead storage, and `agent_creatable` only gates creation
  (`src/sase/task_types/fields.py:64-65`), so any already-filed `feature` bead —
  including the `bob-cli` one that motivated the original scoping plan — keeps its chip,
  template, and triage bar in every direction this policy moves.
- **Digest churn.** Every project that regenerates picks up a new `feature` digest
  because `create_refusal` is part of the validated spec
  (`tests/task_types/test_validation.py:56-59`). That is one `sase memory init` per
  project, and `sase validate` names it.
- **Rollback.** Restore the two YAML blocks and revert `_builtin.py`; nothing here is a
  schema or storage change.
