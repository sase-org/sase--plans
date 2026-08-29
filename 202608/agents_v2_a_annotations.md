---
tier: tale
title: Apply the
goal:
  Core agent memory carries the eight `#a` corrections from the user's AGENTS v2
  annotation pass — a smaller always-loaded footprint, no Python-specific or
  skill-duplicating prose, and the audited-read commands named where agents need them.
size: medium
proposed_by: bbugyi200.athena.0fq
---

- **AGENTS:**
  - [bbugyi200.athena.0fq](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fq.md)
  - [bbugyi200.athena.0fs](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fs.md)
  - [bbugyi200.athena.sase-vd.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vd.1/README.md)
  - [bbugyi200.athena.sase-vd.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vd.2/README.md)
  - [bbugyi200.athena.sase-vd.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vd.3/README.md)
  - [bbugyi200.athena.sase-vd.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vd.4/README.md)
- **COMMITS:**
  - [affc43a](https://github.com/sase-org/sase/commit/affc43a6fef74e33c1c3edfb6cc51b5a978e20af)
    — docs(memory): apply AGENTS v2 \#a annotation trims
  - [2a4c075](https://github.com/sase-org/sase/commit/2a4c075375bc331320fd1b7ad596c5d612274a21)
    — docs(agents): align AGENTS v2 instruction annotations
  - [8426315](https://github.com/sase-org/sase/commit/84263159f6499bf922e33ae58c7b4ce193e6698f)
    — feat(git-setup): adopt the runner's numbered workspace claim
  - [0235ff0](https://github.com/sase-org/sase/commit/0235ff059ad3e5e87156508fd10bf43f7dbcade6)
    — feat(shells): pre-allocate VCS workspace on family follow-up launches
  - [b7fcee9](https://github.com/sase-org/sase/commit/b7fcee9db595cebb6b5fcbc474898fab8c6595e8)
    — feat(agent): rebind runner workspace identity
  - [1a14630](https://github.com/sase-org/sase/commit/1a1463028a7619fa7bcd6ad3331ee640ac5f69c5)
    — feat(workspace): skip VCS release on handoff and pid mismatch

# Plan: Apply the `#a` annotations from the sase AGENTS v2 review

The user annotated the rendered agent instruction file and asked that only the comments
tagged `#a` be implemented. The comments tagged `#task` are explicitly out of scope.

Everything here edits **generated agent memory**. The user's request for these edits is
the required approval, so `sase memory init` runs as part of the work (see "Regenerate",
below). Do not widen the change: this is always-loaded context, and every sentence added
or kept has to earn its per-turn cost.

## Sources Of Truth

`sase/memory/sase.md` and `sase/memory/task_types.md` are **generated**. Editing them
directly is wrong — they are overwritten on the next `sase memory init`. The canonical
sources are:

| Generated output                                      | Canonical source                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------------------ |
| `sase/memory/sase.md`, `AGENTS.md`, provider shims    | `src/sase/main/init_memory/templates/memory-sase.template.md`            |
| `sase/memory/task_types.md`                           | `src/sase/main/init_memory/templates/memory-sase-task-types.template.md` |
| `sase/memory/task_types/*.md`, `sase/task_types.json` | `src/sase/task_types/_builtin.py`                                        |
| deployed `sase_final` SKILL.md                        | `src/sase/xprompts/skills/sase_final.md`                                 |

The generator reflows prose, so source line wrapping does not have to match the rendered
output. Keep each template file's existing wrapping style.

## Step 1 — `memory-sase.template.md`

Six edits, all in `src/sase/main/init_memory/templates/memory-sase.template.md`.

### 1a. Name the strand-read command (annotation `h-ea87211d82ad`)

In the `**Memory webs**` bullet, replace the trailing clause:

- Old: `` — read strands by keyword (`glossary:stitch`) through the same skill.``
- New:
  `` — read strands through the same skill with `sase memory read <web>:<keyword>` (for example `glossary:stitch`).``

Use `<web>:<keyword>`, not `<web>:<slug>`. `sase memory read --help` and the
`/sase_memory_read` skill both document the selector as `web:keyword` (resolved by
canonical keyword, alias, or unambiguous prefix); the `<slug>.md` filename is the file
name, not the selector.

### 1b. Drop the Python-specific justification (annotation `h-c188ff34150c`)

In the `## Ephemeral` section:

- Old:
  `You need to be mindful not to run commands outside of these workspace directories, since they have their own isolated virtual environments.`
- New:
  `You need to be mindful not to run commands outside of these workspace directories.`

Delete the clause; do not substitute a language-neutral replacement. The template
renders for every SASE project, and the reason is not needed for the instruction to be
followed.

### 1c. Reword the repository-section lead-in (annotation `h-d16c732d854e`)

Both branches of the `{% if linked_repo_entries %}` block, so the two phrasings stay in
step:

- Old: `Configured linked and sidecar repositories for this context:`
- New: `Configured linked and sidecar repositories associated with this project:`
- Old: `No linked repositories are configured for this context.`
- New: `No linked repositories are associated with this project.`

The annotation targets only the first line (the rendered PDF had linked repos, so the
`else` branch never appeared). Changing both is a deliberate consistency extension.

### 1d. Drop the `gh:<owner>/<repo>` example (annotation `h-090e9a5c08b2`)

- Old:
  ``open it with `/sase_repo` (unlinked GitHub repos open as external repos, e.g. `gh:<owner>/<repo>`) and read``
- New:
  ``open it with `/sase_repo` (unlinked GitHub repos open as external repos) and read``

Safe to remove: the `/sase_repo` skill already documents `sase repo open gh:owner/repo`
and the bare `owner/repo` shorthand, and an agent reaches that skill before it needs the
argument form.

### 1e. Prefer audited reads over opening a repo (annotation `h-cdbd0fc93454`)

Insert one new paragraph **before** the `IMPORTANT REMINDER:` line, after the
`Web tools remain appropriate only for content a checkout does not contain…` paragraph.
Leave the `IMPORTANT REMINDER:` line last so it stays the section's closer.

New paragraph:

```
Prefer an audited read over opening a repo: read memory notes with `sase memory read`, and always read artifact files
stored in sidecar repos with `sase artifact read <ref> "<reason>"`.
```

See "Assumptions" for why the second half names `sase artifact read`.

### 1f. Move the finalizer mechanism into the skill (annotation `h-f4d7eab8ad2b`)

Replace the entire body of `## SASE Final Declaration` with:

```
Before any normal response that ends this SASE provider turn, use your `/sase_final` skill as the last action. This
includes a final answer, an incomplete-status response, an "I will wait" response, or any reply that intends to resume in
a later turn. Only a successfully executed plan, monitor, pipe, or questions handoff is exempt, because those commands
terminate the runner mechanically. Intending to resume later is not an exemption.
```

What is kept is the part an agent needs _before_ it invokes the skill: the trigger, the
response shapes that count, and the exemption. Everything removed is mechanism the skill
owns, and the annotation's "do not duplicate any instructions that already exist in that
skill" applies. Each removed clause already has a home:

| Removed from core memory                                                     | Already in `src/sase/xprompts/skills/sase_final.md` |
| ---------------------------------------------------------------------------- | --------------------------------------------------- |
| calls `sase final context`, submits one declaration with `sase final submit` | Steps 1–4                                           |
| declaration covers every repo changed, including `/sase_repo`-opened ones    | Rules, first bullet                                 |
| a scoped host prompt does not narrow the obligation                          | Rules, first bullet (extended in Step 2)            |
| no file/repo changes after a successful submit                               | Rules, "Do not mutate files or repositories after…" |
| repair the manifest and resubmit on validation errors                        | Rules, "If submit reports other validation errors…" |

## Step 2 — `sase_final.md` skill (the one genuine migration)

The core-memory sentence being removed says the obligation is not narrowed by a host
prompt scoped to one repository's **commit or conflict repair**. The skill's matching
rule covers only "commit". In `src/sase/xprompts/skills/sase_final.md`, first Rules
bullet:

- Old:
  `or being outside a host prompt scoped to one repository's commit is not a reason to leave your own work uncommitted.`
- New:
  `or being outside a host prompt scoped to one repository's commit or conflict repair is not a reason to leave your own work uncommitted.`

That is the only skill edit. Everything else in Step 1f is already present verbatim or
in substance; do not re-add it.

## Step 3 — `memory-sase-task-types.template.md`

Annotations `h-152b59510faf` and `h-f9be9e1d09b0` together strip the
`## File Discovered Work As Task Beads` section down to its two load-bearing sentences.
Replace the whole section body with a single paragraph:

```
Unless your prompt explicitly forbids creating beads (epic phase workers, for example, must record `PROPOSED FOLLOW-UP:`
notes on their own bead instead), you can and SHOULD capture discovered follow-up work as sase task beads. Before
creating any task bead, you MUST use `/sase_new_task`.
```

Removed, and why it is safe:

- `Pick the type above whose when_to_use matches what you found:` plus the three example
  bullets. The generated roster immediately above already lists every type with its
  one-line summary, and each strand's `when_to_use` is one
  `sase memory read task_types:<slug>` away. Step 4 restores the one nuance the bullets
  carried that the summaries did not.
- Everything after `you MUST use /sase_new_task.` — duplicate detection, epic causal
  link, `open` draft, `--size`, `-T "task(<slug>)"`, `-f/--field`. Verified present in
  `src/sase/xprompts/skills/sase_new_task.md` steps 4–7. The one clause not in the skill
  ("ready task beads are proposed to the project owner, who either launches an agent or
  closes them") is in `sase_beads.md`, which that skill reads at step 1. **No edit to
  `sase_new_task.md` is needed.**

Keep the first sentence. It is not part of either highlight, and it is the only place
that tells an agent it _should_ file discovered work at all — dropping it would change
behavior, not just token count.

## Step 4 — Two task-type summaries

The `#a` note asks that the roster descriptions absorb a little of what the deleted
bullets carried. Two targeted edits in `src/sase/task_types/_builtin.py`; leave `bug`,
`flake`, and `memory` alone (`flake`'s "on an unchanged tree" and `bug`'s "while doing
unrelated work" already imply "you did not cause it", and `memory`'s summary already
matches its deleted bullet word for word).

- `ci`: `A confirmed true test or lint failure, not a flake.` →
  `A confirmed true test or lint failure you did not cause, not a flake.`
- `feature`: `An out-of-scope product idea that should not become a wish list.` →
  `An out-of-scope product or tooling idea that should not become a wish list.`

Both new strings exceed the 88-column limit once indented, so write them as
parenthesized implicit concatenations in the style `_bug_spec()` already uses.
`tests/task_types/test_builtin.py` enforces single-line summaries of at most 120
characters; both new strings are well under that.

## Step 5 — Regenerate

Run from the repo root:

```bash
sase memory init -C
```

`-C/--no-commit` keeps the commit host-owned. Expect regenerated: `sase/memory/sase.md`,
`sase/memory/task_types.md`, `sase/memory/task_types/ci.md`,
`sase/memory/task_types/feature.md` (summary **and** digest), `sase/task_types.json`,
`sase/memory/README.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
`QWEN.md`.

Confirm with `git status` that no generated file was missed, then
`sase memory init --check` must report no drift.
`tests/main/test_init_memory_committed_drift.py` fails if the committed tree and the
generator disagree.

This machine has `use_chezmoi: true`, so the same run also refreshes home memory under
the chezmoi source root. That is operator machine state, not part of this change — the
precedent commits for this template (`50d9c3bc2`, `250c0121d`) are project-scoped only.
Do not hand-edit those outputs, and do not pull them into this change.

## Step 6 — Tests

Three assertions pin text this plan changes. Update them in the same commit.

`tests/main/test_init_memory_handler_outputs.py`

- `_FINAL_DECLARATION_MARKERS`: drop `"sase final context"`, `"sase final submit"`,
  `"do not make more file or repository changes"`, and `"repair the manifest"`. Keep
  `"/sase_final"`, `"Before any normal response that ends this SASE provider turn"`,
  `"incomplete-status response"`, `"I will wait"`,
  `"plan, monitor, pipe, or questions handoff"`, and
  `"Intending to resume later is not an exemption"` — those are exactly the contract
  core memory still owns.
- The `"No linked repositories are configured for this context."` assertion →
  `"No linked repositories are associated with this project."`.

`tests/main/test_init_memory_markdown_templates.py`

- `"Configured linked and sidecar repositories for this context:"` →
  `"Configured linked and sidecar repositories associated with this project:"`.

Nothing else needs touching: `tests/main/test_init_memory_handler_repositories.py`
asserts the repo-access sentences that survive unchanged,
`tests/main/test_init_skills_source_content.py` asserts `sase_final.md` strings that
this plan keeps, and no test asserts the deleted virtualenv clause, the
`gh:<owner>/<repo>` example, the `File Discovered Work` bullets, or the two builtin
summaries.

If a grep turns up an assertion this plan did not anticipate, fix the assertion to match
the new text rather than softening the intended edit.

## Step 7 — Verify

```bash
just install   # only if this checkout has sat unused
just fmt
just check
```

Hand `just check` to `/sase_monitor` if it runs long. The change touches memory
generation that many tests import, so the landing step should be `just check-full`
through `/sase_monitor` with the `TESTING` / `TESTED` status pair — not an inline run.

After this lands on the canonical branch, the deployed `sase_final` skill is stale until
someone runs `sase skill init --force` from a clean, merged tree (`generated_skills.md`
owns that rule). That deploy is **not** part of this change; call it out in the final
response instead.

## Assumptions

**The artifact half of annotation `h-cdbd0fc93454` names `sase artifact read`, not
`sase memory read`.** The annotation reads "the `sase memory read` command MUST always
be run when reading artifact files from sidecar repos", but `sase memory read` cannot
read artifacts — its selectors are memory notes, web names, and `web:keyword` strands
only (`sase memory read --help`). `sase artifact read <ref> "<reason>"` is the audited
artifact path documented in `sase_artifacts.md` and in the `/sase_repo` skill, and the
sidecar repos holding artifacts (`plans`, `research`) are exactly what the annotation
describes. Writing `sase memory read` there would put a command that cannot work into
always-loaded context. Step 1e therefore keeps `sase memory read` for the "prefer
audited reads over opening repos" half and uses `sase artifact read` for the artifact
half. **If this reading is wrong, say so at approval time — it is the only judgment call
in this plan that changes what agents are told to run.**

## Out Of Scope

The five `#task`-tagged annotations, including the plans sidecar / "Linked Repositories"
and "Sidecar Repositories" subsection restructuring, the `/sase_memory_write` skill
recommendation, the planner-agent `/sase_questions` memory approval rewrite, and the
README coverage badge. Do not touch them.
