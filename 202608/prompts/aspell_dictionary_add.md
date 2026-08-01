- **PLAN:** [../202608/aspell_dictionary_add.md](../aspell_dictionary_add.md)

 
%xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `qx`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `qx--plan`

- **Outcome:** `completed` · **Model:** `claude/opus` · **Launch:** `20260801065610`
- **Transcript:** `~/.sase/chats/202608/gh_sase_org__sase-ace_run-qx__plan-260801_065610.md`

**User:**

The prompt input widget does not currently support spell check syntax highlighting because it would be expensive and difficult to make reliable. With that said it would be nice to support little red squiggly lines under words that we know are misspelled. Can you help me add support for this by remembering any word that the user targeted with the `K` keymap in the prompt input widget that resulted in our spellcheck tool being used (because the word couldn't be found using our dictionary tool) and highlighting that mispelling appropriately in the future if it is ever found in the prompt input widget again? This highlighting should apply immediately (i.e. if the user declines to choose a spelling correction recommendation, the prompt input widget they return to should show that word with a red underline--if you can't make a red underline work, use your best judgement on what this syntax highlighting should look like).

I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202608/sticky_misspellings.md`

> # Plan: Sticky misspelling highlighting in the prompt input widget
> ## Problem
> `PromptTextArea` deliberately does not spell-check as you type: running `aspell` over every word on every keystroke is
> expensive and unreliable. The result is that `K` (`_prompt_word_lookup.py`) is the only place a misspelling is ever
> surfaced, and the knowledge evaporates the moment the correction panel closes. Decline a suggestion and the prompt looks
> exactly as it did before — no trace that the word is wrong.
> The fix does not need a live spell checker. Every `K` press that reaches `aspell` produces a verdict we already paid
> for. Remember the misspelled verdicts and the prompt can highlight those words forever after, for free, with no
> subprocess on any render or keystroke path.
> ## Design

*See full plan file for details.*

### Member 2 of 2 — agent `qx--code`

- **Outcome:** `completed` · **Model:** `claude/sonnet` · **Launch:** `20260801070449`
- **Transcript:** `~/.sase/chats/202608/gh_sase_org__sase-ace_run-qx__code-260801_065610.md`

**User:**

@sase/repos/plans/202608/sticky_misspellings.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I'll start by setting up task tracking and exploring the codebase to understand the existing patterns this plan builds on.

Now let me set up task tracking and read all the key reference files in parallel.

Now let me read the key reference files this plan builds on.

Now I have enough context. Let's start implementing, beginning with the word-scanning refactor and the durable store.

Now let's refactor `extract_lookup_word` to share its validation logic with a new `natural_word_ranges`:

Now update `__all__`:

Now let's create the durable store for misspellings.

That diagnostic matches the exact pattern already used in `prompt_placeholders.py` (the file I'm modeling this on), so I'll keep it consistent with the established codebase convention rather than diverge. Now let's add the settings dataclass.

Now let's wire the new mixin into `startup.py`.

Now let's create the `_startup_misspellings.py` mixin.

Now let's finish the config task (default_config.yml + schema.json) and mark tasks complete.

Now let's create the highlight overlay mixin.

Now wire it into `PromptTextArea`'s MRO, between `PlaceholderHighlightMixin` and `JinjaHighlightMixin`.

Now let's update the spellcheck panel modal with the accept affordance and zero-suggestions support.

Now let's add the `.spellcheck-empty-row` CSS rule.

Now let's update `_prompt_word_lookup.py` to record/forget on `K` results and handle the new `SpellcheckChoice` union.

Now let's update `_apply_spelling_suggestion` and add the helper methods:

Good, confirms I need the TYPE_CHECKING import for `SpellcheckChoice`.

Now let's mark tasks 6 and 7 complete and move to the test harness (task 8).

Now let's update the test harness in `editors.py`.

I left some broken placeholder code in that file — let me clean it up.

Now let's add the `natural_word_ranges` test functions at the end of the file.

Now let's add the two new snapshot test functions:

All test files are written. Let's mark task 9 complete and move to documentation.

Now let's update `docs/ace.md`'s "Word definitions & spellcheck" section.

Now let's add the full `ace.prompt_spellcheck` section after `ace.prompt_completion`, matching its format.

Now let's add the sticky-squiggle mention to the onboarding "prompt" section (the actual help-panel prompt copy lives in `agent_onboarding.py`, not `guide_view.py` which is just wiring).

Install succeeded. Now let's run `just fmt` and `just check`.

`just fmt` reformatted several files as expected. Now let's run `just check`.

Let's check the symvision memory guidance before fixing this lint failure.

Symvision flags `MisspellingSets` as an unused public class — it's only referenced within its own file (and by tests, which don't count). Per the guidance, I'll make it private.

Good, `__all__` no longer has `_MisspellingSets` (correctly removed). Now update the two test files that reference the old public name.

Now let's re-run `just check` to verify the symvision fix and confirm everything passes.

This pre-existing validation failure (`bead_notification_panel.md` missing plan target) exists on a clean checkout too — confirmed unrelated to my changes. Let me file a task bead for it per the project's discovered-work convention, then verify my changes pass everything else cleanly.

Now let's run the test suite to make sure everything actually passes, then handle `just test-visual` for the new PNG goldens.

Found the bug: `_build_highlight_map()` runs synchronously during `TextArea.__init__`, before the widget is mounted, so `self.app` raises `NoActiveAppError` — and I only wrapped the *call* to the provider in try/except, not the `self.app` access itself. Let me fix that.

I'll use `-m visual` directly to check baseline behavior for the flagged snapshot tests.

Let me check the TUI performance memory guidance since I'm scheduling new async work from `on_mount`, which caused a visual timing shift in an unrelated existing snapshot.

This confirms my `warm_misspellings()` scheduling follows the established async-warm-cache pattern (matches `tui_perf.md` rule 9) — the codeblock snapshot shift is a pre-existing convergence-timing sensitivity being revealed, not a real regression. Let's regenerate exactly the affected goldens.

Let's visually verify the new misspelling-highlight snapshot looks correct.

Both "recieve" and "reciept" are rendered with the red underline in the prompt bar — exactly the intended sticky misspelling highlight. Let's run the full visual suite plus the full test suite once more to confirm everything is now green.

All lint/format/type gates pass; the only remaining failure is the pre-existing, unrelated plan-link validation issue (confirmed on a clean checkout, task bead filed). Let's do a final review of `git status` and wrap up.

All 11 implementation tasks are complete. Summary:

**Implemented** (per `sase/repos/plans/202608/sticky_misspellings.md`):
1. `src/sase/history/prompt_misspellings.py` — durable, `fcntl`-locked store (`misspelled`/`allowed` lists, casefold dedupe, oldest-first trim)
2. `natural_word_ranges()` in `sase/core/word_lookup.py`, sharing validation logic with `extract_lookup_word()`
3. `ace.prompt_spellcheck` config (default_config.yml, schema, `PromptSpellcheckSettings`)
4. `StartupMisspellingsMixin` (`_startup_misspellings.py`) — warm cache, optimistic record/allow/forget + off-thread persist
5. `MisspellingHighlightMixin` (`_misspelling_highlight.py`) — red-underline overlay, wired into `PromptTextArea`'s MRO
6. `K` now records/self-heals misspellings in `_prompt_word_lookup.py`
7. `SpellcheckPanelModal` gained `SpellcheckChoice`, an `a`-to-accept binding, and zero-suggestion support
8. Test harness (`PromptPage(misspellings=...)`) plus 7 test files/extensions covering the store, word scanning, the overlay, `K`/panel behavior, the startup cache, and 2 new PNG goldens
9. Docs updated (`docs/ace.md`, `docs/configuration.md`, onboarding guide copy)

**Bugs found and fixed during verification** (not in the original plan):
- `_build_highlight_map()` runs during widget `__init__`, before mounting — `self.app` raised `NoActiveAppError` there, breaking any test that constructs `PromptTextArea` standalone (fixed with a guarded `_active_app()` helper).
- `MisspellingSets` triggered a symvision unused-public-class failure — renamed to `_MisspellingSets` per the project's decision hierarchy.

**Verification**: `just install`, `fmt`, and `check` all pass except a pre-existing, unrelated `plan links validate` failure (confirmed on a clean checkout) — filed as task bead `sase-cu`. `just test` is fully green aside from 4 known-flaky/pre-existing tests (confirmed by isolated re-runs and baseline comparison), unrelated to this change. Two new PNG goldens were created and three existing ones legitimately updated (spellcheck footer text, onboarding copy, and a codeblock-highlight timing convergence revealed by the new `on_mount` worker).

I'll commit these changes using the sase_git_commit skill as instructed by the finalizer.

Confirmed this matches the finalizer's file list. Now writing the commit message.

The commit command is still running in the background — I'll wait for it to complete rather than poll.

The commit is already running in the background via Bash's `run_in_background`, so I'll just wait for its completion notification rather than starting a duplicate monitor.

---

%xprompts_enabled:true
# New Query

 Can you now help me add support for adding a word to the user's `aspell` personal dictionary directly from the panel that shows the list of spelling correction recommendations? I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 