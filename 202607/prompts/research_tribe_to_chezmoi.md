- **PLAN:** [../202607/research_tribe_to_chezmoi.md](../research_tribe_to_chezmoi.md)

 
%xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `q1`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `q1--plan`

- **Outcome:** `completed` · **Model:** `claude/opus` · **Launch:** `20260731071253`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-q1__plan-260731_071253.md`

**User:**

We already support built-in and user-defined configuration for agent
tribe panels that allows the user to customize the icon that is used and the
color of the tribe name. Can you now help me add support for a new `description`
field that describes the given agent tribe?

- If an agent tribe has any configuration at all then this field should be
required.
- We should display this description in the metadata panel when the
corresponding agent tribe panel is selected in the TUI.
- Make sure you add an excellent description to the built-in agent tribes that
we have configured and the tribes that I have configured in my chezmoi repo.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202607/tribe_description.md`

> # Plan: Required `description` For Configured Agent Tribes
> ## Goal
> Add a `description` field to `ace.tribes.<name>` config entries, make it **required whenever a tribe has any
> configuration at all**, ship excellent descriptions for every built-in tribe, and render the description in the ACE
> Agents-tab metadata panel whenever a tribe panel is selected.
> ## Current State
> Per-tribe display config already supports `icon`, `color`, and `initially_expanded`:
> - Schema: `src/sase/config/sase.schema.json:471-501` (`ace.tribes`, `propertyNames` pattern `^[A-Za-z0-9_.-]+$`,
>   `additionalProperties` = a closed object with the three fields).
> - Bundled defaults: `src/sase/default_config.yml:97-120` — six entries: `default`, `epic`, `research`, `chop`, `pinned`,

*See full plan file for details.*

### Member 2 of 2 — agent `q1--code`

- **Outcome:** `completed` · **Model:** `claude/sonnet` · **Launch:** `20260731073439`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-q1__code-260731_071253.md`

**User:**

@sase/repos/plans/202607/tribe_description.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I'll work through this plan step by step. Let me start by reading the relevant files to understand the current state.

Now let's start implementing. Step 1: schema changes.

Now step 2: bundled defaults in default_config.yml.

Now update `_tribe_displays_for_token`, `tribe_display_for`, `tribe_identity_colors`, and add `tribe_config_key`:

Now update `__all__`:

Let me review the full updated file to double check correctness.

Now populate the fields in `build_agent_tribe_summary_snapshot`:

Step 4 done. Now step 5: the renderer.

Those diagnostics are pre-existing and unrelated to my edits. Now let's update the renderer file.

Now insert the description line in `_append_header`, right after the `Name:` line:

Step 5 done. Now step 6: doctor check.

Let me verify the final file looks right.

Good, that matches the existing pattern (these aliases exist purely for symvision satisfaction). Doctor check is registered correctly.

Now step 7: docs updates.

Now update the `ace.tribes` field table and its explanatory paragraph:

Now update `docs/ace.md` tribe-panel prose:

Step 7 done. Now step 8: re-verify chezmoi user tribes.

No matches — confirmed there are still no user-defined `ace.tribes` entries in chezmoi. No chezmoi edit is needed.

Now let's look at the existing test files to write the required test updates.

Now update the schema test file:

Now let's look at the agent_tribe_summary tests (both files) and the renderer test:

Now let's look at the widget test for `_agent_display_tribe.py`:

Now let's add the new focused tests for description styling, missing hint, default mapping, and markup safety, right after `test_tribe_header_colors_only_the_structured_name_identity`:

Now let's look at an existing doctor test to model the new one on:

Now let's run `just install` and then the targeted tests to validate everything works.

Now let's run the targeted tests.

Let's rerun the targeted tests.

All 65 targeted tests pass. Now let's mark the testing task complete and move to visual snapshots.

Let's run the visual test suite now to see what changed.

Confirmed: two of the seven failures (`models_panel_alias_picker_reordered*`) are pre-existing flakiness unrelated to this change — they fail identically on unmodified master. The other five are caused by my change and need golden regeneration.

5 tests updated their goldens. Let's check which PNG golden files changed and eyeball them.

Let's eyeball a couple of the regenerated PNGs to confirm they show exactly the expected new description line.

This looks exactly right — the "Epic phase-worker clans from sase bead work, one member per phase of an approved plan." description renders in italic gray beneath `Name: ▲ @epic`. Let's spot-check one more, the collapsed panel showing the missing-description hint scenario isn't applicable here since chop has a description too, but let's check the clan-collapse one since it wasn't in the plan's known list.

This confirms the change is correctly propagating and rendering exactly one extra description line for `@chop`. Now let's run the full visual suite once more to confirm everything passes (excluding the two pre-existing flaky models-panel tests).

Only the two pre-existing, unrelated failures remain (confirmed to fail identically on unmodified master). All tribe-description-related visual tests now pass. Per CLAUDE.md guidance, I'll file a task bead for that pre-existing flake since it's discovered work I didn't cause.

Now let's finish visual testing task and move to the final `just check` run.

Just a formatting fix needed. Let's run `just fmt` to auto-format.

Formatting fixed automatically. Now let's re-run `just check`.

Now let's run the remaining checks (`validate-committed-plans` and `test`) manually to confirm the rest of `just check` passes despite the pre-existing validation blocker.

That passes. Now the full test suite.

This matches the plan's implementation and testing scope exactly. Let's mark the final task complete.

## Summary

Implemented the plan in full: `ace.tribes.<name>.description` is now required by schema, all six bundled tribes ship excellent descriptions, and the ACE Agents-tab metadata panel renders the description (or a `no description - set ace.tribes.<name>.description` hint) directly under `Name:` in the `TRIBE` header.

**Changes:**
- `sase.schema.json` — `description` required (1-160 chars) for any configured tribe
- `default_config.yml` — descriptions for `default`, `epic`, `research`, `chop`, `pinned`, `review`
- `tribe_display.py` — `_sanitize_description`, `MAX_TRIBE_DESCRIPTION_CHARS`, `configured` flag, `tribe_config_key()` helper
- `agent_tribe_summary.py` — `description`/`description_missing` snapshot fields
- `_agent_display_tribe.py` — renders the description line (plain text, never markup) in the cheap header path
- New `checks_config_tribes.py` doctor check (`sase doctor -C config.tribes`), registered in `checks_config.py`
- `docs/configuration.md` and `docs/ace.md` updated
- Confirmed via `/sase_repo chezmoi` that no user-defined tribes exist there, so no chezmoi edit was needed

**Testing:** added/extended unit tests across schema, display resolution, snapshot, renderer, and doctor-check layers; regenerated the 5 visual PNG goldens genuinely affected by the new header line (verified by comparing against unmodified master) and eyeballed the results.

`just check` passes except for three **pre-existing** failures on master, unrelated to this work (verified via `git stash`): two flaky `models_panel_alias_picker_reordered` PNG snapshots, a broken plan-link (`conventional_commit_subject_gate`), and a provider-casing assertion in `test_xprompt_model_completion.py`. Filed task beads `sase-bh`, `sase-bi`, `sase-bj` for these.

I'll commit the uncommitted changes from this session using the sase_git_commit skill, as directed by the finalizer.

The commit is running in the background (the `sase commit` workflow can take a while due to rebase/push). I'll wait for it to complete.

It's still running in the background; I'll wait for the notification rather than poll further.

The commit completed successfully according to the log. Let me verify the working tree is clean and pushed.

The commit succeeded and pushed cleanly. I'll clean up the leftover scratch commit message file I created.

Committed and pushed as `ba611aa48` — `feat(ace-tribes): require a description for configured agent tribes`, covering all 24 files from the finalizer's list (schema, defaults, display/snapshot/renderer code, doctor check, docs, tests, and regenerated PNG goldens). Working tree is clean and in sync with `origin/master`.

---

%xprompts_enabled:true
# New Query

 Are the `@chop` and `@research` agent tribes configured in this repo or in my chezmoi repo? They are supposed to be configured in my chezmoi repo. If not, use your /sase_plan skill to plan the appropriate changes.
 

%xprompts_enabled:false
### Questions and Answers

#### Q1: Scope

> Both @chop and @research are configured in this repo (src/sase/default_config.yml), not chezmoi. @research is definitely yours — only your chezmoi xprompts (research_swarm.md, old_research_swarm.md) ever emit tribe=research. @chop is arguable: sase source hard-codes tribe=chop for every AXE chop proposal (chop_proposal_models.py, chop_proposal_planning.py), so any user writing an agent-proposing chop gets it for free — but no bundled chop actually proposes agents today, so in practice it only shows up because of chops you configured. Which scope should the plan cover?

- [x] **Move research only** — Relocate ace.tribes.research to chezmoi sase.yml; leave ace.tribes.chop bundled because sase source hard-codes the chop tribe. Smallest correct change; @chop keeps working out of the box for any future agent-proposing bundled chop.
- [ ] **Move both** — Relocate ace.tribes.research AND ace.tribes.chop to chezmoi sase.yml. Bundled defaults then only configure tribes that shipped sase code names as constants (default, epic, pinned, review). Cost: a fresh sase install shows @chop with no icon/color and expanded by default.
- [ ] **Move both + keep chop discoverable** — Relocate both, but add a docs/schema note (and possibly a doctor hint) telling users that agent-proposing chops land in @chop and they may want to configure ace.tribes.chop themselves.

%xprompts_enabled:true