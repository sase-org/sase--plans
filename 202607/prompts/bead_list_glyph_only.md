- **PLAN:** [../202607/bead_list_glyph_only.md](../bead_list_glyph_only.md)

 
%xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `qg`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `qg--plan`

- **Outcome:** `completed` · **Model:** `claude/opus` · **Launch:** `20260731104622`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-qg__plan-260731_104622.md`

**User:**

Can you help me make it so the `sase bead list` command shows a great visual indication of each bead's type? I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202607/bead_list_type_indicator.md`

> # Plan: Show each bead's type in `sase bead list`
> ## Problem
> `sase bead list` (default `--format compact`) renders every bead as an identical row shape and gives **zero** indication
> of the bead's type:
> ```
> ◐ sase-bv · Attribute beads to the agent that created them
> ◐ sase-bt · Fix xdist flake in artifact modal copy shortcut
> ```
> Those two rows look the same, but `sase-bv` is an **epic-tier plan** bead and `sase-bt` is a **task** bead. Type is not
> recoverable from the row:

*See full plan file for details.*

### Member 2 of 2 — agent `qg--code`

- **Outcome:** `completed` · **Model:** `claude/sonnet` · **Launch:** `20260731105413`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-qg__code-260731_104622.md`

**User:**

@sase/repos/plans/202607/bead_list_type_indicator.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I'll start by reading the key files this plan touches to understand current implementation before making changes.

Now let's look at the `sase bead list` doc section around line 655 and the `search` flag table for reference formatting:

Good, install succeeded. Now let's verify cell_len behavior for our glyphs.

Let's begin with task 1: updating `bead_type_presentation.py`.

Now update `BEAD_TYPE_CHIP_WIDTH` and add `bead_type_cli_cell`, plus `__all__`:

Now add `bead_type_cli_cell` after `bead_type_chip`, and update `__all__`:

These match the plan's expected xterm-256 codes exactly (220/117/177). Task 1 is done. Now let's move to task 2: updating `cli_query.py`.

Now update `handle_bead_list` and `handle_bead_show` to compute and pass `use_color`, and rewrite `_render_list_compact`:

Now rewrite `_render_list_compact`:

Good — confirmed pre-existing pattern unrelated to my change (same line offsets in the unmodified file). Now let's add the `-c/--color` parser args (task 3), then re-verify diagnostics after that's wired in.

Now add the same flag to `register_bead_show_parser`:

Now let's do a manual sanity check with the actual CLI before writing tests.

This looks correct and matches the plan's target output exactly. Let's also verify `show --format compact` byte-matches and check the phase type row and parent suffix.

Matches the plan's target output exactly for all three types with correct alignment. Now let's move to the tests (task 4). First let's mark task 3 complete.

Now let's add the compact-row type/color tests at the end of the file:

Now appending the new test suite covering the type/color grammar:

Now let's simplify the awkward `by_id` test:

Now let's run these new tests to verify they pass.

That's pre-existing (fixture used only for its side effect of initializing the store), unrelated to my change. Now let's check `test_bead_type_presentation.py` for the `cli_style` derivation test:

Now let's add the new tests, both the exact xterm-256 codes and the general derivation guarantee, plus tests for `bead_type_cli_cell`:

All pass. Now let's add show-side test coverage and extend the fast-path test docstring.

All pre-existing warnings unrelated to my edits (they're all in code before my inserted lines). Now let's extend `test_fast_path_defers_list_to_argparse`'s docstring/comment per the plan:

Pre-existing warnings, unrelated. Task 4 complete. Now on to task 5: documentation updates.

Now let's also document `-c/--color` in the `sase bead show` flag table:

Now let's run prettier to normalize the markdown table formatting we hand-wrote:

Good, prettier cleaned up the tables. Let's check the skill file diff too:

Good, the bead store resolves fine (likely a sidecar). Now let's file the two follow-up beads.

Now let's mark both ready for triage per the CLAUDE.md workflow.

Both follow-up beads filed and marked ready. Let's move to the final verification step.

Now let's run `just check` as required by the project's CLAUDE.md instructions.

Just formatting nits — let's run `just fmt` to auto-fix.

That was `just fmt` reformatting my file via ruff. Let's confirm everything is still correct and now run `just check` fully.

Now let's run the full `just check`.

Pre-existing warnings, unrelated. Now let's look at the golden contract test file.

This works. Now let's generate the rest of the affected golden fixtures.

Now let's re-run the golden contract tests plus the claimed-status test to verify everything passes.

All pass now. Let's re-run the full `just test` suite to confirm the whole 3 remaining failure categories (suite_gate flake, PNG snapshots pre-existing) and get a clean final signal.

Let's check whether a bead already tracks these pre-existing PNG snapshot failures before filing a new one.

Good — this is already tracked (`sase-c6` "Revisit recurring ACE PNG snapshot drift" and related beads), so no new bead needed. Now let's finish `just check` by handling the "committed plans" and skills-deployment steps.

Now let's run mypy on the touched files to be sure there are no type errors from my new code:

Now let's check the symvision (unused-symbol) lint, since the plan explicitly flagged this as a risk for new exported helpers.

All lints pass. Let's run `just fmt-md-check` and `_lint-toobig` for full confidence, then do a final `just check` pass to confirm the only remaining item is the expected skill-deployment check.

That's fine — no yaml files touched by this change anyway. All the individual check components (fmt, lint, mypy, symvision, toobig, keep-sorted, committed-plans, tests) pass except the expected skill-deployment gate. Let's do one final full `just check` run to confirm the only remaining failure is that expected one.

Everything is in order. Let's mark the final task complete.

The plan is fully implemented and verified. Summary:

**Implementation:**
- `src/sase/bead_type_presentation.py`: added a `cli_style` property (derived from `accent_color` via xterm-256 cube conversion — verified exact matches: plan→220, phase→117, task→177), a `bead_type_cli_cell()` helper, and switched `BEAD_TYPE_CHIP_WIDTH` to `rich.cells.cell_len`.
- `src/sase/bead/cli_query.py`: `_render_list_compact` now takes `use_color` and emits `{type_glyph} {type_word} {status_glyph} {id} · {title}{ ← parent_id}`, with the type column measured/padded once per call via `cell_len`. Both `handle_bead_list` and `handle_bead_show`'s compact branch resolve color via `resolve_color`.
- `src/sase/main/parser_bead_queries.py`: added `-c/--color {auto,always,never}` to `list` and `show`.
- Docs: `docs/beads.md` (new row grammar, type glyph table, `-c/--color` in both flag tables) and `src/sase/xprompts/skills/sase_beads.md` (updated grammar description, examples list untouched).
- Filed and marked ready two follow-up beads: `sase-cc` (port to Rust `handle_list`) and `sase-cd` (align `search --format compact`).

**Verification:**
- `just check` passes (fmt, ruff, mypy, pyscripts, changelog, symvision, toobig, committed-plans, full test suite) except the expected `init skills --check` gate, which per the generated-skills workflow is deliberately deferred to a post-land `sase skill init` step — not something to run in this change.
- Had to update pre-existing fixtures that pinned the old grammar: `tests/test_bead/test_claimed_status.py` and 9 golden `.stdout` files under `tests/test_bead/golden/cli/`, regenerated from actual CLI output against the golden stores.
- Confirmed the 5 remaining PNG snapshot failures and one flaky suite-gate test are pre-existing on `master` (reproduced identically before my changes) and already tracked by existing beads (`sase-c6`, `sase-c5`, `sase-bl`).
- Manually verified `sase bead list`, `--color always/never`, `NO_COLOR`, piped output, and `show --format compact` byte-identity, all matching the plan's target output exactly.

I'll verify the current repository state matches the work described, then commit using the sase_git_commit skill as instructed by the finalizer.

This matches exactly what the plan describes implementing, and the file list aligns with the "Work So Far" narrative. These are my changes — I'll commit them now via the `/sase_git_commit` skill.

This is a user-facing feature (new type indicator + `-c/--color` flag), so `feat` is the right tag. Let me write the commit message and run the commit.

It already exists from a prior pass — let me read it first as required, then overwrite.

That's a stale leftover from an unrelated prior commit — overwriting it now.

The commit command is running in the background; I'll wait for it to complete rather than poll.

The commit command is running in the background — I'll be notified automatically when it finishes, so I'll wait rather than poll.

---

%xprompts_enabled:true
# New Query

 Can we remove the word that describes the type (e.g. `task`, `plan`, etc...) from the output and just keep the colored symbol? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
