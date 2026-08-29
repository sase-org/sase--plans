---
tier: tale
title:
  Remove sase memory write/review and apply the /sase_memory_write review annotations
goal:
  The `/sase_memory_write` skill states three authorization cases in prose instead of a
  table and no longer mentions a proposal command, and `sase memory write` / `sase
  memory review` are gone from the CLI, the code, the docs, the notification actions,
  and the tests.
size: medium
proposed_by: bbugyi200.athena.0g5
---

- **AGENTS:**
  - [bbugyi200.athena.0g5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0g5.md)
  - [bbugyi200.athena.sase-vk.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vk.1/README.md)
- **COMMITS:**
  - [1be5429](https://github.com/sase-org/sase/commit/1be5429ea9812ff722c94cd2f1103ffc9b6142da)
    — feat(memory): make web descriptors tier-free
  - [1791874](https://github.com/sase-org/sase/commit/179187499eb9df7fca11551ffe66afc9b4496297)
    — feat(memory)\!: remove unused memory proposal path

# Plan: Remove the memory proposal path and apply the skill annotations

## Why

The user reviewed the newly added `/sase_memory_write` skill and left four annotations
on `~/bob/ref/docs/sase_memory_write.md` (one is a frontmatter note, three are
comments):

1. On "not a bead description": _"If an agent is asked to work a bead and the bead
   describes memory changes then a bead description should suffice. Add this as a third
   authorized case."_
2. On "Routing": _"Can you change the format of this to not use a table? Make sure this
   still conveys the same information."_
3. On "Propose A New Reference Note": _"The `sase memory write/review` commands are
   obsolete. I added these a long time ago and never used them. Remove this section and
   remove both of those commands and all references (be thorough)."_

Comment 3 is the large one. The plan that introduced the skill
(`sase/repos/plans/202608/memory_write_skill.md`) already anticipated it: _"If the
reviewer would rather have a single unauthorized route, drop that table row and the
'Propose A New Reference Note' section from the skill body; nothing else in this plan
changes."_ The reviewer went further and asked for the underlying commands to go too.

## Decisions

- **Hard removal, no `sunset` feature flag.** `sase/memory/sase_flags.md` requires a
  flag when a deprecated branch must stay reachable while callers migrate. There are no
  callers to migrate: the only proposal ledger on this host is an empty lock file
  (`~/.sase/projects/gh_sase-org__sase/memory_proposals.lock`) with no
  `memory_proposals.jsonl` beside it, and the author of the commands says they were
  never used. The user asked for removal, not deprecation.
- **Delete, do not deprecate, the whole proposal stack.** `sase memory write` and
  `sase memory review` are the only entry points into `sase/memory/proposals/` and
  `sase/memory/review_tui/`; with them gone, both packages are dead. Symvision would
  flag the survivors anyway.
- **The `memory_review` notification action goes with them.** Nothing else emits
  `action: memory_review`. Both the ACE notification modal and the shared pending-action
  store degrade gracefully on an unknown action (a "Unsupported notification action"
  warning toast, and a skipped pending entry), so any stale inbox row from an old run
  stays harmless.
- **`sase memory log --include glossary` survives; `--include proposals` does not.**
  `--include` keeps its `append` shape with a single remaining `glossary` choice rather
  than becoming a boolean flag, so the option's contract does not change for the value
  that still exists.
- **`memory_write_root()` is unrelated and stays.** `sase/memory/paths.py`,
  `sase/memory/mutation.py`, `sase/memory/web/mutation.py`,
  `src/sase/main/init_memory/*` and the ACE Memory panel's `_submit_memory_write` are
  about writing canonical notes, not proposals. None of them are touched.
- **The skill file itself stays.** Only its body changes; `name`, `skill: true`, and the
  `docs/xprompt.md` bundled-skill table row are unaffected.

## Step 1 — Rewrite the skill source

Edit `src/sase/xprompts/skills/sase_memory_write.md`. Keep the frontmatter exactly as it
is. Replace the body from `## Authorization` through the end of the file with:

```markdown
## Authorization

You may write memory only when one of these holds:

- The **user's prompt for this turn** asks for the change.
- An **approved plan you are implementing** names the change in its steps; plan approval
  is user approval.
- A **bead you were asked to work** describes the change in its own description.

Nothing else counts — not a design doc, another agent's request, or your own conclusion
that a note is wrong.

## Routing

**Authorized above?** Edit and republish, below.

**Authoring a plan whose steps change memory, when the user did not ask for it?**
Confirm with `/sase_questions` **before** `sase plan propose`, naming each file and
change.

**Unauthorized?** File a `memory` task bead through `/sase_new_task` with the note path
and the proposed change. Do not edit the note.

## Edit And Republish

1. Add, edit, or delete the canonical note under `sase/memory/`. Never hand-edit
   `AGENTS.md` or a provider shim such as `CLAUDE.md`; they are generated.
2. A note that `sase memory init` generates itself (`sase/memory/sase.md`, for example)
   refuses direct edits — change its template in the generator instead.
3. Run `sase memory init` to regenerate `AGENTS.md`, the provider shims, and the memory
   README. Authorization for the edit covers this; do not ask for it separately.
```

This is all three comments at once: the new third authorization bullet and the shortened
"Nothing else counts" line (comment 1), the table replaced by bolded question/answer
prose that keeps every row's meaning (comment 2), and the deleted "Propose A New
Reference Note" section plus its routing row (comment 3).

Leave the frontmatter `description` as written. It already names exactly the three
surviving routes ("edit and republish, ask the user first, or file a memory task bead")
and never mentioned the proposal path, so it stays accurate after this rewrite.

## Step 2 — Delete the proposal implementation

Delete these files and directories outright:

- `src/sase/memory/cli_write.py`
- `src/sase/memory/cli_review.py`
- `src/sase/memory/proposals/` (all of `__init__.py`, `identity.py`, `ledger.py`,
  `models.py`, `paths.py`, `review.py`, `validation.py`, `write.py`)
- `src/sase/memory/review_tui/` (all of `__init__.py`, `app.py`, `_callbacks.py`,
  `_modals.py`, `_models.py`, `_render.py`, `_styles.py`)

Do **not** delete `src/sase/memory/locks.py` or `src/sase/memory/atomic_write.py`; both
have many non-proposal consumers.

## Step 3 — Unwire the CLI

`src/sase/main/parser_memory.py`:

- Delete the `write_parser` block (its parser and all nine arguments) and the
  `review_parser` block (its positional, mutually exclusive action group, and remaining
  options).
- Remove the three `sase memory write` / `sase memory review` example lines and the
  `sase memory log --include proposals` line from the `memory` group epilog.
- Remove the `sase memory log --include proposals` line from the `log` subparser epilog.
- Narrow `--include` to `choices=("glossary",)` and reword its help to describe only the
  legacy glossary audit log.

`src/sase/main/memory_handler.py`:

- Delete the `if sub == "write"` and `if sub == "review"` branches.
- Update the fallback usage string to
  `Usage: sase memory {agent-docs,init,list,log,read,show,web}`.

## Step 4 — Unwire `sase memory log`

In `src/sase/memory/cli_log.py`, remove the proposal half of the command:

- Drop the `from sase.memory.proposals import (...)` block and the
  `_PROPOSAL_EVENT_TYPES` constant.
- In `handle_memory_log_command`, drop `include_proposals`, the `proposal_events`
  computation, the `payload.update(_build_memory_log_proposal_payload(...))` call, and
  the `proposal_events=` argument passed to `_render_memory_log_summary`.
- Delete `_build_memory_log_proposal_payload`, `_proposal_summary_panel`,
  `_proposal_events_panel`, `_filter_memory_proposal_events`,
  `_ordered_memory_proposal_events`, `_proposal_event_actor`, `_proposal_event_detail`,
  and `_include_proposals`.
- Remove the `proposal_events` parameters from `_render_memory_log_summary` and
  `_build_memory_log_summary_dashboard` and the branches inside them that render those
  panels.

Keep every glossary-side counterpart (`_include_glossary`,
`_build_memory_log_glossary_payload`, `_glossary_events_panel`) intact.

## Step 5 — Unwire the package exports

`src/sase/memory/__init__.py`: delete the entire
`from sase.memory.proposals import (...)` block and every proposal name from `__all__`
(`EvidenceRecord`, `MemoryProposalAuthorError`, `MemoryProposalBodyError`,
`MemoryProposalError`, `MemoryProposalEvent`, `MemoryProposalEvidenceError`,
`MemoryProposalState`, `MemoryProposalTargetError`, `MemoryProposalWriteResult`,
`ProposalAuthor`, `ProposalWarning`, `build_memory_proposal_warnings`,
`create_memory_proposal`, `generate_memory_proposal_id`,
`memory_proposal_event_to_dict`, `memory_proposal_ledger_path`,
`memory_proposal_lock_path`, `memory_proposal_state_to_dict`,
`parse_memory_proposal_evidence`, `proposal_author_from_agent`,
`read_memory_proposal_events`, `read_memory_proposals`, `reduce_memory_proposal_events`,
`require_proposal_author`, `validate_memory_proposal_target`).

Keep the `sase.memory.read_log` import block and its `__all__` entries — including
`AgentIdentity`, `AgentIdentityError`, and `require_agent_identity`, which the proposal
package only re-exported.

## Step 6 — Remove the `memory_review` notification action

- `src/sase/notifications/senders.py`: delete `notify_memory_proposed` and its private
  helper `_memory_proposal_evidence_files`. If `Any` is then unused, drop it from the
  `typing` import.
- `src/sase/notifications/pending_actions.py`: delete the
  `_ACTION_KIND_BY_NOTIFICATION_ACTION["memory_review"] = "memory_review"` line so the
  mapping is purely gate-adapter derived again.
- `src/sase/ace/tui/actions/agents/_notification_handlers.py`: delete
  `handle_memory_review` and the `from sase.memory.review_tui import MemoryReviewTuiApp`
  import.
- `src/sase/ace/tui/actions/agents/_notification_actions.py`: remove the
  `handle_memory_review as handle_memory_review` re-export and its `__all__` entry.
- `src/sase/ace/tui/actions/agents/_notification_modal_flow.py`: remove
  `handle_memory_review` from the local import list and delete the
  `elif result.action == "memory_review":` branch, so the action falls through to the
  existing "Unsupported notification action" warning.
- `src/sase/ace/tui/modals/notification_modal_constants.py`: remove the
  `"memory_review"` entries from both `ACTION_BADGES` and `ACTION_ICONS`.

## Step 7 — Update the tests

Delete these files entirely:

- `tests/main/test_memory_write.py`
- `tests/main/test_memory_review.py`
- `tests/main/test_memory_review_tui.py`
- `tests/test_memory_proposals.py`
- `tests/ace/tui/test_memory_review_notification_action.py`

Edit these:

- `tests/main/test_memory_log.py` — delete the proposal imports
  (`create_memory_proposal`, `reject_memory_proposal`, and anything else from
  `sase.memory.proposals`),
  `test_memory_log_include_proposals_json_adds_proposal_events`, and
  `test_memory_log_include_proposals_rich_output`. Keep every glossary and read-log
  test.
- `tests/main/test_memory_parser_handler.py` — delete
  `test_parser_requires_memory_write_target_or_slug`,
  `test_memory_write_dispatches_to_write_handler`, and
  `test_memory_review_dispatches_to_review_handler`, plus any now-unused helpers or
  imports they were the only users of.
- `tests/main/test_parser_command_help.py` — in
  `test_memory_help_marks_primary_command_and_init_alias`, drop the `memory_write_help`
  and `memory_review_help` locals and their six assertions, drop the
  `sase memory write --title`, `sase memory review --list`,
  `sase memory review mem-... --edit`, and `sase memory log --include proposals`
  assertions, and change the subcommand-list assertion to
  `{agent-docs,init,list,log,read,show,web}`.
- `tests/notification_store/test_senders.py` — delete
  `test_emits_memory_review_action_data` and any fixture or import that existed only for
  it.
- `tests/test_timezone_display_cli.py` — rename
  `test_memory_review_absolute_date_uses_configured_timezone`; it only asserts
  `format_time_or_age("2026-01-02T03:00:00Z") == "2026-01-01"` and has nothing to do
  with the removed command, so keep the assertion under a name that describes it (for
  example `test_absolute_date_rendering_uses_configured_timezone`).

Then regenerate the completion snapshot, which encodes both removed subparsers:

```bash
just sync-completion-spec
```

Leave `tests/shard_timings.json` alone. Its two stale rows for the deleted review tests
are pruned by the separate shard-timings ratchet workflow, not by `just check`.

## Step 8 — Update the docs

- `docs/memory.md` — delete the `## Propose Memory`, `## Review Proposals`, and
  `## Review TUI` sections. In the intro paragraph (around line 44), end the day-to-day
  sentence after `/sase_memory_write`, which now only edits and republishes with
  `sase memory init`. In the ACE Memory panel paragraph (around line 52), drop the
  clause saying the panel "does not replace the agent-facing `sase memory write` /
  `review` proposal path above". In `## Audited Reads` (around line 155), delete the
  `--include proposals` paragraph.
- `docs/cli.md` — delete the `sase memory write` and `sase memory review` table rows;
  they are also the only links to the two removed `memory.md` anchors. Reword the
  `sase memory log` row so it mentions only `--include glossary`.
- `docs/init.md` — delete the `sase memory review --list`,
  `sase memory log --include proposals`, and `sase memory write --title ...` lines from
  both example blocks (around lines 71/88 and 350/351); delete the `sase memory write`,
  `sase memory review`, and `sase memory log --include proposals` table rows (around
  lines 117-119); reword the "reviewed reference memory proposals" cross-reference
  (around line 311); and delete the two paragraphs describing `sase memory write` and
  `sase memory review` plus the `--include proposals` sentence (around lines 331-345).
  Fix the surviving `# read requires SASE agent identity; write requires ...` code
  comment, which no longer has a write to describe.
- `docs/notifications.md` — delete the `memory.proposed` row from the sender table
  (around line 325), the whole `### Memory Proposal Notification` section (around line
  567), and the `memory` tag paragraph (around line 682). If `memory` is left with no
  producer, make sure the surrounding tag prose still reads correctly.
- `docs/ace.md` — delete the `memory_review` row from the notification action table
  (around line 3673) and the two sentences about memory proposal notifications that
  follow it (around lines 3677-3681).

## Step 9 — Update the generated memory README

`sase/memory/README.md` is generated, so edit the template, not the output:

- `src/sase/main/init_memory/templates/memory-README.template.md` — delete the `- `sase
  memory write` proposes a new reference memory note for review.` and `- `sase memory
  review` reviews pending memory proposals.` bullets (lines 88-89).
- Then run `sase memory init` from the workspace to regenerate `sase/memory/README.md`,
  `AGENTS.md`, and the provider shims. The user's prompt for this turn authorizes this
  memory change, so `/sase_memory_write` routes it to edit-and-republish with no extra
  confirmation.

The home-scope README under chezmoi carries the same two bullets. It regenerates from
the same template on the next home-scope `sase memory init` / chezmoi deploy; do not
hand-edit the chezmoi copy.

## Verification

```bash
just install    # ephemeral workspace clones may have drifted deps
just check
```

`just check` covers ruff, mypy, symvision (which is what would catch any surviving
proposal symbol), keep-sorted (both `__all__` blocks edited here are keep-sorted
regions), markdown formatting for the docs and the skill source, `just validate`, and
the scoped test lane.

Because this change deletes packages and rewrites a keep-sorted `__all__`, hand
`just check-full` to `/sase_monitor` with the `TESTING` / `TESTED` status pair before
the turn ends; it is the landing gate and routinely outruns a single turn.

## Follow-ups (not part of this change)

- **Deploy the skill.** `src/sase/xprompts/skills/sase_memory_write.md` is a generated
  skill source. Per `sase memory read generated_skills.md`, the chezmoi deploy must run
  from a clean, merged tree: land this change first, then run `sase skill init --force`
  and `chezmoi apply`. The skill is not currently deployed at all — `/sase_memory_write`
  does not yet resolve for Claude — so that deploy is what actually makes it invocable.
- **Regenerate shell completions.** `~/.local/share/chezmoi/home/dot_zfunc/_sase` still
  contains `_sase_memory_write` / `_sase_memory_review` completers. It is generated by
  `sase completion install zsh`; rerun that after landing.
- **Breaking-change commit footer.** This removes two public CLI subcommands.
  `CHANGELOG.md` is generated by release-please from conventional commits, so the commit
  subject/body must carry the breaking marker (e.g. `feat(memory)!:` with a
  `BREAKING CHANGE:` footer naming `sase memory write`, `sase memory review`, and
  `sase memory log --include proposals`).
- **Stray local state.** `~/.sase/projects/gh_sase-org__sase/memory_proposals.lock` is a
  zero-value leftover lock in user state, not in the repo. It can be deleted by hand;
  nothing in this change reads it.
