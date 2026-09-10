---
tier: tale
title: Migrate sase bug-bash tasks into a new sase_bug_bash project note
goal:
  ~/bob/sase_bug_bash.md exists as a well-formed Bob project note parented to [[sase]],
  every open sase bug-bash task has been physically moved into it with
  status/metadata/logs intact, every inbound block link still resolves, and `bob
  projects sync` has registered it as a sub-project of sase.
size: medium
proposed_by: bbugyi200.athena.0e7
create_time: 2026-09-09 20:00:28
status: wip
---

# Plan: Migrate sase bug-bash tasks into `~/bob/sase_bug_bash.md`

Bryan is about to run a bug bash for `sase`. The bug-bash tasks he has already created
are scattered across `~/bob/sase.md` and `~/bob/sase_blog_0.md`. This plan collects them
into a single Bob **project note**, `~/bob/sase_bug_bash.md`, so the bug bash has one
home that Bob's project tooling understands.

This work happens entirely inside the Obsidian vault at `~/bob/`. **No bob-cli source
code changes are in scope.**

## Driving task

`~/bob/sase.md` carries the task that motivates this work:

```markdown
- [/] #task Create [[sase_bug_bash]], move tasks there, and bulk schedule!
  [created::2026-08-26] ^bug-bash-prj
```

Its wording — _move_ tasks, then _bulk schedule_ — sets the shape of this plan.

## Background: vault conventions you must respect

Read these before touching anything. They are the difference between a clean migration
and one that silently corrupts Bryan's task tooling.

### Project notes

A Bob project note has this frontmatter (see `~/bob/_templates/new_project.md`, and
`~/bob/sase_art.md` for a mature real-world example):

```yaml
---
parent: "[[<parent note>]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: <ISO-8601 with offset, e.g. 2026-08-26T08:06:23-04:00>
---
```

Body shape: a single completion-criteria task tagged `#task #prj ... #hide` anchored
`^prj`, then a `## Tasks` section holding ordinary `#task` lines.

Per the `obsidian` SASE reference memory, every new note under `~/bob/` must carry a
`parent` frontmatter field linking to another note in the vault.

### Task status symbols (Obsidian Tasks plugin)

| Symbol | Name        |
| ------ | ----------- |
| `[ ]`  | Todo        |
| `[/]`  | In Progress |
| `[*]`  | Next        |
| `[?]`  | Blocked     |
| `[x]`  | Done        |
| `[-]`  | Canceled    |

`[?]` Blocked is **derived**, not hand-set: `bob task-status-hooks` marks a task Blocked
when its `[scheduled:: ...]` date is later than the daily anchor, or when its
`[dependsOn:: ...]` names an open `[id:: ...]` task. Both tasks being moved are `[?]`
purely because they are scheduled for 2026-08-27 (tomorrow). **Copy statuses verbatim;
never "fix" them by hand.**

### What is machine-maintained — do NOT hand-edit

This is the most important section of this plan.

| Thing                                               | Owner                                                                                       | Rule                                                                                                                                                                                                                                         |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `task_count` / `open_task_count` frontmatter        | `bob-project-tasks` Obsidian plugin (`~/bob/.obsidian/plugins/bob-project-tasks/main.js`)   | Never author or adjust these. Omit them from the new note; the plugin adds them.                                                                                                                                                             |
| The `🧩 **Sub-projects:**` line under a `^prj` task | `bob projects sync` (`subproject_children_by_parent_link_name` in `src/native/projects.rs`) | **Never hand-edit `sase.md`'s Sub-projects line.** It is derived from the `parent:` frontmatter of child project notes. Setting `parent: "[[sase]]"` on the new note and running `bob projects sync` is the only correct way to register it. |
| `#hide` on `^prj` tasks                             | `bob projects sync`                                                                         | Author `#hide` on the new `^prj` line per the template, then let sync reconcile it.                                                                                                                                                          |
| `status:` frontmatter on project notes              | `bob projects sync` (derived from `^prj` task state)                                        | Author `status: wip` initially; sync maintains it thereafter.                                                                                                                                                                                |
| `[?]` Blocked status                                | `bob task-status-hooks`                                                                     | Preserve verbatim on move.                                                                                                                                                                                                                   |
| `done_tasks:` frontmatter                           | `bob move-done-tasks`                                                                       | Do not author it; it appears when tasks are completed.                                                                                                                                                                                       |

### The vault is live

Bryan is actively working in this vault right now. During research for this plan the
driving task changed status from `[*]` to `[/]` and picked up an active Pomodoro
reference in `~/bob/2026/20260826.md`. Consequences:

- **Line numbers in this plan are advisory only.** Re-locate every task by its text and
  block ID at implementation time.
- `git -C ~/bob status` is already dirty with unrelated daily-note edits. **Do not run
  `bob bulk-git-commit`** and do not commit the vault. Bryan's `bob nightly` job owns
  vault commits.
- Work quickly and in one pass; re-read a file immediately before editing it.

## Scope: which tasks move

### Discovery criteria

**In scope** — an open task whose _own text_ names the sase bug bash as the work to be
done.

**Out of scope**, explicitly:

1. **Non-sase bug bashes.** A naive `grep -i "bug.bash"` over the vault returns mostly
   Bryan's former-employer work notes: `prj_gbd.md`, `prj_mcr_cats.md`, `work_ref.md`,
   `old_ref.md`, `aut.md`, `launch.md`, `inbox.md`, `now_prjs.md`, `prj_pa_trouble.md`,
   `lit/bug_bashes`, and every `team_meet*.md` / `fscarpel_meet*.md` note. **None of
   these move.** Ignore them.
2. **Tasks that merely mention a bug bash in a SCHEDULE LOG.** Entries like
   `*2026-08-16* — Ready for bug bash?` and `*2026-08-18* — Bug bash today?` are
   scheduling nudges Bryan wrote on ordinary feature work. The affected tasks —
   `sase_art.md#^e2e-refs` ("E2E artifact refs") and `sase_art.md`'s "Make sure
   completion for artifact refs is perfect!" — are artifact-tab features, not bug-bash
   work. **They stay in `sase_art.md`.**
3. **Already-closed tasks.** `sase_blog_blockers.md` is a fully canceled project
   (`status: canceled`, all tasks `[-]`/`[x]`). Leave it alone.
4. **Generic E2E-test tasks** unconnected to the bug bash, e.g.
   `sase_agents_repo.md#^e2e-sase-8v`, `sase_update.md`'s E2E tasks,
   `sase_dyn_agent_fam_e2e.md`. These predate the bug bash and stay put.

There is **no `#bug_bash` tag** in the vault, and `[scheduled:: 2026-08-27]` is _not_ a
useful signal — dozens of unrelated tasks across `cash.md`, `job.md`, `bob.md`,
`gtd_daily.md` and others share that date.

### Seed set (verify before acting)

Research for this plan found exactly **two** open tasks that meet the criteria:

**1. From `~/bob/sase.md` (`## Tasks` section, ~line 321):**

```markdown
- [?] #task Write an E2E test playbook for sase and use it for bug bash!
  [created::2026-08-13] [scheduled:: 2026-08-27] ^e2e-playbook
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-25 → 2026-08-27_ — 🤷 no reason given ... (9 further SCHEDULE LOG entries
      down to _2026-08-16_ — Start bug bash today?)
```

**2. From `~/bob/sase_blog_0.md` (`## Tasks` section, ~line 39):**

```markdown
- [?] #task Design and implement a bug bash! [created::2026-08-07] [scheduled::
  2026-08-27]
  - RELATED: [[sase#^e2e-playbook]]
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-25 → 2026-08-27_ — Bug bash time?
```

Note this task has **no block ID** — you will mint one (step 3).

**Step 1 of implementation is to re-run discovery**, because the vault is live and Bryan
may have added tasks since this plan was authored. Do not assume the seed set is
complete.

### Decision: the driving task stays in `sase.md`

`sase.md#^bug-bash-prj` ("Create [[sase_bug_bash]], move tasks there, and bulk
schedule!") is **not** moved. It is the meta-task of performing _this migration_, not
bug-bash work, and it is currently `[/]` In Progress with a live Pomodoro reference in
today's daily note (`~/bob/2026/20260826.md` links it twice, once under a 🍅 marker).
Moving it would break both links and disturb an in-flight Pomodoro for no benefit. Leave
it in place for Bryan to complete himself.

### Decision: physical move, not transclusion

Bryan's `sase_blog_blockers.md` shows an alternative pattern — reference tasks that
transclude the original, e.g. `- [x] #task ![[sase_more_agent_clis#^meta-code]]`.
**Reject it here**, for two reasons:

1. Both the user's request and the driving task say _move_.
2. `bob projects sync` bulk-schedules "every open ordinary task" **in the note it is
   syncing**. Transcluded reference tasks would receive the schedule while the real
   tasks — still living in `sase.md` / `sase_blog_0.md` — would not. That defeats the
   "bulk schedule" half of the driving task.

## Implementation

### Step 1 — Re-run discovery

Confirm the seed set is still accurate and complete. From `~/bob`:

```bash
grep -rn "#task" --include="*.md" . | grep -iE "bug.?bash"
grep -rn "sase_bug_bash" --include="*.md" .
```

Apply the discovery criteria above to anything new. If you find a task that is a genuine
judgment call, include it and **call it out explicitly in your final report** rather
than silently dropping it.

### Step 2 — Capture inbound links before moving anything

Moving a task moves its block ID, which breaks every `[[note#^blockid]]` link pointing
at the old location. Enumerate them first:

```bash
cd ~/bob && grep -rn "e2e-playbook" --include="*.md" .
```

Known at authoring time:

- `~/bob/sase_blog_0.md` — `- RELATED: [[sase#^e2e-playbook]]` (a sub-bullet of the very
  task being moved, so it travels with it, but the **target** changes)

Re-run the grep for any block ID you end up moving. Also check for
`[id:: <note>__<slug>]` / `[dependsOn:: <note>__<slug>]` inline fields on moved tasks:
neither seed task has one, but if a newly discovered task does, its `id::` value embeds
the old note name and every `dependsOn::` referencing it must be kept in sync (see the
`sase_agents_repo.md#^e2e-sase-8v` pair for the pattern).

### Step 3 — Create `~/bob/sase_bug_bash.md`

Create the note with exactly this shape. Use the current local timestamp for `created`.
Omit `task_count`, `open_task_count`, and `done_tasks` — those are machine-maintained.

```markdown
---
parent: "[[sase]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: <NOW, e.g. 2026-08-26T08:06:23-04:00>
scheduled: 2026-08-27
---

- [ ] #task #prj Run the sase bug bash and file a bead for every bug it turns up!
      [created::2026-08-26] #hide ^prj

## Tasks

<moved tasks go here>
```

About `scheduled: 2026-08-27` — this is the **"bulk schedule"** limb of the driving
task. `bob projects sync` propagates a project note's `scheduled:` frontmatter to every
open ordinary task in that note that lacks an equal-or-later schedule. Today it is a
**no-op**, because both seed tasks already carry `[scheduled:: 2026-08-27]`; its real
value is governing tasks added later. It also keeps exactly one `#hide` on `^prj` while
the date is in the future. **If Bryan vetoes the date or the bulk-schedule behaviour in
approval feedback, drop this one line** — everything else in the plan stands unchanged.

Word the `^prj` completion criteria to match Bryan's voice (imperative, ends in `!`);
the line above is a reasonable default, not a mandate.

### Step 4 — Move the tasks

For each in-scope task, move it as one unit:

- The `- [status] #task ...` line **verbatim** — status symbol, all inline fields
  (`[created:: ...]`, `[scheduled:: ...]`, `[priority:: ...]`), and trailing
  ` ^block-id` unchanged.
- **Every** indented sub-bullet beneath it: notes, `RELATED:` lines, and the full
  `🗓️ **SCHEDULE LOG**` / `🛠️ **WORK LOG**` subtrees. Do not truncate the logs; they are
  Bryan's scheduling history.

Preserve the existing indentation characters exactly — the vault mixes tabs and spaces
between notes, so copy bytes rather than reformatting.

Specific actions:

1. Cut `^e2e-playbook` and its 10-entry SCHEDULE LOG out of `~/bob/sase.md`; append to
   `## Tasks` in `sase_bug_bash.md`.
2. Cut "Design and implement a bug bash!" and its sub-bullets out of
   `~/bob/sase_blog_0.md`; append to `## Tasks` in `sase_bug_bash.md`. **Mint a block ID
   for it** — `^design-bug-bash` — appended to the task line so it becomes linkable. Its
   `RELATED: [[sase#^e2e-playbook]]` sub-bullet must be retargeted to
   `RELATED: [[sase_bug_bash#^e2e-playbook]]`, since both tasks now live in the same
   note.
3. Verify no blank-line or heading damage in either source note: `## Tasks` and the
   following section heading must still be separated correctly, and no stray blank lines
   should be left where the tasks were.

### Step 5 — Retarget inbound links

Rewrite every link found in step 2 to point at `sase_bug_bash`:

- `[[sase#^e2e-playbook]]` → `[[sase_bug_bash#^e2e-playbook]]`

Then confirm zero stale references remain:

```bash
cd ~/bob && grep -rn "sase#\^e2e-playbook" --include="*.md" .   # must return nothing
```

Leave `[[sase#^bug-bash-prj]]` links in `~/bob/2026/20260826.md` **untouched** — that
task is not moving.

### Step 6 — Let the tooling reconcile

Dry-run first, read the diff, then apply:

```bash
bob projects sync --dry-run     # inspect: expect sase.md's Sub-projects line to gain [[sase_bug_bash]]
bob projects sync
```

This registers `sase_bug_bash` as a sub-project of `sase` (via the new note's `parent:`
frontmatter), reconciles `#hide`/`status:`, and applies the `scheduled:` propagation.

If the dry run proposes changes to notes unrelated to this migration, that is
pre-existing drift from Bryan's live editing — report it, do not fight it.

Then, **dry-run only**:

```bash
bob task-status-hooks --dry-run
```

Report what it would change but **do not apply it**. It rewrites task statuses across
the vault from the live Pomodoro ledger and will touch Bryan's in-flight Pomodoro state;
that is his call to make, not this migration's.

### Step 7 — Verify

1. `bob projects list` — `sase_bug_bash` appears with the expected open-task count and a
   valid `^prj` state.
2. `cd ~/bob && grep -n "Sub-projects" sase.md` — the line now contains
   `[[sase_bug_bash]]`.
3. `grep -c "#task" sase_bug_bash.md` — one `^prj` task plus each moved task.
4. Every moved task's SCHEDULE LOG is intact (compare entry counts against
   `git -C ~/bob diff` for the source notes).
5. `git -C ~/bob diff -- sase.md sase_blog_0.md sase_bug_bash.md` — the diff shows pure
   moves plus the retargeted link, with no incidental reformatting.
6. No stale `[[sase#^e2e-playbook]]` anywhere in the vault.

Use `bob query` (via the `/bob_query` skill) for any read-only Dataview verification.
That skill is read-only — do not use it to mutate the vault.

## Out of scope

- Committing or pushing the vault (`bob bulk-git-commit`, `git commit`).
- Applying `bob task-status-hooks` (dry-run and report only).
- Moving `sase.md#^bug-bash-prj`, or marking it done — Bryan closes it himself.
- Editing `sase_art.md`, `sase_blog_blockers.md`, `sase_agents_repo.md`, or any
  former-employer bug-bash note.
- Populating the bug bash with _new_ tasks (test scenarios, areas to probe). This plan
  migrates existing tasks only.
- Any change to bob-cli source, or to Obsidian plugins under `~/bob/.obsidian/`.

## Definition of done

`~/bob/sase_bug_bash.md` exists, is a valid Bob project note parented to `[[sase]]`,
holds every in-scope bug-bash task with status/metadata/logs byte- intact, `sase.md`
lists it as a sub-project via `bob projects sync`, every inbound block link resolves,
and the vault is left uncommitted with no unrelated edits.
