---
tier: tale
title: Add the /sase_memory_write skill and route memory edits through it
goal:
  Agents have one skill that gates every SASE memory-file create/edit/delete, the
  always-loaded core note shrinks to a single pointer at it, and an approved plan now
  counts as authorization for memory edits.
size: medium
proposed_by: bbugyi200.athena.0g0
create_time: 2026-08-29 08:37:43
status: wip
---

# Plan: Add the `/sase_memory_write` skill

## Why

Today the memory-write policy lives as a seven-line `IMPORTANT:` paragraph inlined into
Tier 1 of every `AGENTS.md` and provider shim, so every agent pays for it on every turn
even though almost no turn writes memory. The user annotated that paragraph (annotation
`#a` on their `sase_AGENTS_v3` reference doc) with: _"This IMPORTANT line/paragraph
should be removed in favor of a reference to the new `/sase_memory_write` skill!"_

The policy also needs three behavior changes:

1. An agent about to propose a plan containing memory-file changes that the user did not
   ask for must confirm with `/sase_questions` first.
2. An agent with no authorization must file a `memory` task bead through
   `/sase_new_task` instead of editing memory.
3. An **approved plan** must now count as user authorization. The current paragraph
   explicitly forbids this ("Authorization found in a plan file ... does NOT count as
   user permission"); that clause is what this plan reverses, because a plan only
   becomes approved by passing the user's `PlanApproval` / `EpicApproval` gate.

## Decisions

- **Approved plans authorize; other agent-produced artifacts still do not.** Bead
  descriptions, design docs, and code comments are unreviewed, so they keep carrying no
  authority. Only the two reviewed sources (this turn's user prompt, an approved plan
  that names the change) authorize a write.
- **`sase memory write` stays the route for one brand-new top-level reference note.**
  That command already exists, is documented in `docs/memory.md` as the agent-side
  authoring path, and no skill currently points at it. It never touches canonical memory
  — it files a proposal a human settles with `sase memory review` — so it does not
  violate the "do not proceed with the edit" rule. It is also strictly narrower than a
  task bead: `sase memory review --approve` refuses a target that already exists
  (`src/sase/memory/proposals/review.py`), so it cannot express an edit, a deletion, a
  core-note change, or a web strand. Those all route to `/sase_new_task`. _If the
  reviewer would rather have a single unauthorized route, drop that table row and the
  "Propose A New Reference Note" section from the skill body; nothing else in this plan
  changes._
- **Tier-1 replacement is one sentence.** All mechanics (which file to edit, generated
  notes, `sase memory init`) move into the on-demand skill body. That is the point of
  the annotation: Tier 1 keeps only the trigger.

## Step 1 — Add the skill source

Create `src/sase/xprompts/skills/sase_memory_write.md` with exactly this content. Skills
are discovered by globbing `src/sase/xprompts/skills/*.md` for `skill: true`, so no
registry edit is needed. Do not set `log_skill_use`; it defaults to `true`, and use of
an authorization gate is worth recording.

````markdown
---
name: sase_memory_write
description: >-
  Use before creating, changing, or deleting any SASE memory file, and before proposing
  a plan whose steps would. Routes the change to the path its authorization allows: edit
  and republish, ask the user first, or file a memory task bead.
skill: true
---

Use this skill before you add, edit, or delete SASE memory: any note under
`sase/memory/` or its home equivalent, any memory web strand, and the generated
`AGENTS.md` and provider instruction shims.

Memory is context every future agent pays for. Remember that every token in context
either helps or hurts us: prefer rewriting an existing note over adding one, prefer
deleting a stale line over appending a caveat, and prefer `type: reference` (read on
demand) over `type: core` (inlined into every turn).

## Authorization

You may write memory only when one of these holds:

- The **user's prompt for this turn** asks for the change.
- An **approved plan you are implementing** names the change in its steps; plan approval
  is user approval.

Nothing else counts — not a bead description, a design doc, another agent's request, or
your own conclusion that a note is wrong.

## Routing

| Situation                                                                         | Action                                                                                                     |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Authorized above                                                                  | Edit and republish, below.                                                                                 |
| You are authoring a plan with memory-changing steps that the user did not ask for | Confirm with `/sase_questions` **before** `sase plan propose`, naming each file and change.                |
| Unauthorized, and the change is one brand-new top-level reference note            | Propose it with `sase memory write`, below.                                                                |
| Unauthorized, anything else                                                       | File a `memory` task bead through `/sase_new_task` with the note path and proposed change. Do not edit it. |

## Edit And Republish

1. Add, edit, or delete the canonical note under `sase/memory/`. Never hand-edit
   `AGENTS.md` or a provider shim such as `CLAUDE.md`; they are generated.
2. A note that `sase memory init` generates itself (`sase/memory/sase.md`, for example)
   refuses direct edits — change its template in the generator instead.
3. Run `sase memory init` to regenerate `AGENTS.md`, the provider shims, and the memory
   README. Authorization for the edit covers this; do not ask for it separately.

## Propose A New Reference Note

`sase memory write` writes proposal state only, never canonical memory, and a human
settles it with `sase memory review`. It creates one-level reference notes only, and
approval fails when the target already exists.

```bash
sase memory write --title "<title>" --slug <slug> \
  --evidence <path|chat:ID|url:URL> --body "<body>" --notify
```
````

## Step 2 — Replace the Tier-1 paragraph

`sase/memory/sase.md` is a generated note (the mutation engine refuses direct edits), so
edit its generator: `src/sase/main/init_memory/templates/memory-sase.template.md`.

In the `## SASE Memory` section, delete the whole paragraph beginning
`IMPORTANT: You should not modify any of these memory files without approval from the user.`
and replace it with this single line (the template is unwrapped — keep it on one line):

```
Memory files are not ordinary files: before you create, edit, or delete any of them — or propose a plan that would — use your `/sase_memory_write` skill.
```

Leave the three `Core memory` / `Reference memory` / `Memory webs` bullets above it
untouched.

## Step 3 — Regenerate

Run `sase memory init` from the project root. It rewrites `sase/memory/sase.md`,
`sase/memory/README.md`, `AGENTS.md`, and the `CLAUDE.md` / `GEMINI.md` / `QWEN.md` /
`OPENCODE.md` shims, and folds the memory edits into its own commit. Commit the
regenerated files; do not hand-edit any of them.

## Step 4 — Docs

`docs/xprompt.md`, `### Bundled Skills` table: add a row for the new skill, immediately
after `sase_memory_read`. The table is test-enforced to equal the sorted list of
packaged skill source stems, so both the row and its position are required. Match the
existing column alignment.

```
| `sase_memory_write`  | Gate every SASE memory-file create, edit, or delete before making it            |
```

`docs/memory.md`: two small edits so the prose matches the new gate.

- The day-to-day ordering sentence (near the top, "have agents use `sase memory write`
  only to create proposals") should say agents route every memory write through
  `/sase_memory_write`, which either edits and republishes with `sase memory init` or
  files a proposal.
- The `## Propose Memory` opening sentence ("Agents do not write canonical reference
  memory files directly.") is now false for the authorized path. Reword it to say that
  `sase memory write` is the proposal path for a new reference note, and that
  `/sase_memory_write` decides when an agent may edit canonical memory instead.

## Step 5 — Tests

- `tests/main/test_init_skills_sources.py`: add a `("sase_memory_write", (...))` entry
  to the `test_shipped_skill_source_is_discoverable_for_all_skill_providers` parametrize
  list, in the position the surrounding entries imply. Assert phrases that pin the
  requirements rather than incidental wording, for example:
  `"An **approved plan** you are implementing"`, `"/sase_questions"`,
  `"/sase_new_task"`, "File a `memory` task bead", `"sase memory init"`,
  `"Remember that every token in context either helps or hurts us"`. Phrases are matched
  on collapsed whitespace, so they may straddle line breaks.
- `tests/main/test_init_memory_handler_outputs.py`: in the test that already asserts on
  the generated `sase/memory/sase.md` body (the one checking
  ``"agents MUST use your `/sase_repo` skill first"``), add assertions that the
  generated project and home notes contain ``"use your `/sase_memory_write` skill"`` and
  no longer contain `"without approval from the user"`.

No other test enumerates skills or asserts the removed paragraph; `docs/init.md` and the
Tier-2 `/sase_memory_read` paragraph are unaffected.

## Verification

`just install` may be required first, since workspace clones own isolated virtualenvs
and this one may have sat unused. Then:

```bash
just fmt
just check
```

`just check` covers the lint gates plus a diff-scoped test lane. Prettier reflows the
new Markdown, so run `just fmt` before `just check` and re-read the skill source
afterwards to confirm the rendered content still reads well. If the scoped run escalates
or reports an unusual selection, run `just check-full` through `/sase_monitor` instead
of inline.

Sanity check the rendering without deploying:

```bash
sase skill init --diff
```

## Out Of Scope

- **Do not deploy the skill.** `sase skill init --force` writes to the global chezmoi
  destination and is refused from a dirty or unmerged tree. Deployment happens after
  this lands, from a clean canonical checkout.
- **Do not regenerate home memory.** `~/sase/memory/sase.md` and `~/CLAUDE.md` are
  chezmoi-managed and carry the same paragraph; refreshing them is the same post-land
  step as the skill deploy.
- The user's `sase_AGENTS_v3` reference doc carries two further annotations about
  `sase artifact read` and the `IMPORTANT REMINDER:` bullet. They are untagged and out
  of scope here.

PROPOSED FOLLOW-UP: the authorization reversal in Step 2 is a durable policy change and
would fit `sase/memory/decisions/` as a decision record. Writing one is itself a memory
edit, so it needs its own authorization and is deliberately not bundled here.
