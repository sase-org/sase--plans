---
tier: tale
title: The `$` grammar and a jump that always lands
goal:
  Pressing `$` then one key follows any chip the link rail paints, across tabs and
  panes, and the jump reveals, expands, pages, or rescopes whatever it takes to land on
  the target.
size: medium
proposed_by: bbugyi200.athena.sase-ug.7
bead: sase-ug.7
---

- **PARENT:** [202608/link_rail_every_tab.md](link_rail_every_tab.md)
- **BEAD:**
  [sase-ug.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ug/sase-ug.7.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ug.7](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ug.7.md)
- **COMMITS:**
  - [d070280](https://github.com/sase-org/sase/commit/d07028050cb831849d1e666ab267a39223779f9b)
    — feat(tui): add link-follow key grammar

# The `$` grammar and a jump that always lands

## Objective

Complete phase `sase-ug.7` of the link-rail epic. Phase `sase-ug.6` mounted the
read-only rail above the footer on all three tabs and painted `$$`/`$1`-`$9`/`$0` chips,
but `$` is bound to nothing: the rail is currently a legend for a keymap that does not
exist.

This phase makes the legend true. `$` becomes a one-shot prefix — never a toggle, never
an armed mode the user cannot see, because the chips are painted before the prefix is
pressed. `$1`-`$9` follow a chip, `$$` follows the lead chip, `$0` opens the Links
panel. The whole grammar is gated on the same predicate the rail's visibility uses, so
on an entity with no links `$` is inert and passes through untouched.

The hard half is landing. 63% of link jumps cross panes, every pane has a filter query,
and `_navigate_to_relation_target` returns from the cross-pane branch before its reveal
path is ever reached — so a jump to a filtered-out target silently lands nowhere today.
A `$N` jump must switch tab and pane, resolve the target's owning project, widen the
pane's query, expand the fold that hides it, and raise the head-slice cap, toasting
exactly what it changed at each rung — and only then report an honest dead end naming
the ref it could not resolve.

The rail's painted chips are the keymap, so the key assignment and the render must come
from one shared ordering: `$4` works when chip 4 was truncated off the rail, because the
key map is complete even when the render is not.

Everything in this phase stays behind the existing `link_rail` beta flag
(`src/sase/ace/tui/link_rail_flag.py`), whose registry entry already covers "the ACE
link rail, link-following keys, and links panel". No new flag, and no flag removal —
phase `land` (`sase-ug.10`) owns that.

## Implementation

1. **Teach the keymap layer `$`.** Add `"dollar_sign": "$"` to `_KEY_DISPLAY` and
   `"$": "dollar_sign"` to `_KEY_ALIASES` in
   `src/sase/ace/tui/keymaps/key_validation.py`, beside the existing `"+"`/`"-"`
   friendly config spellings. Then wire one new configurable app action,
   `follow_artifact_link`, through every place an `AppKeymaps` field is required: the
   dataclass field in `keymaps/app_keymaps.py` (in the tree-navigation prefix block),
   `("follow_artifact_link", "Follow Link", False)` in `_BINDING_META`
   (`keymaps/metadata.py`), `follow_artifact_link: "dollar_sign"` under
   `ace.keymaps.app` in `src/sase/default_config.yml`, a
   `Binding("dollar_sign", "follow_artifact_link", "Follow Link", show=False)` in
   `bindings.py` beside the `<`/`>`/`~` block, and a matching `APP_COMMAND_META` row in
   `commands/_app_metadata.py` — `ensure_metadata_covers_app_keymaps()` raises at import
   time without one. `load_keymap_registry` likewise raises when `default_config.yml` is
   missing the entry, and `test_build_app_bindings_count` fails when `_BINDING_META` is.
   `$` is unclaimed across every configured key string today; keep it that way by not
   reusing it.

2. **Generalize the one-shot numbered-prefix dispatcher.**
   `src/sase/ace/tui/modals/numbered_link_keys.py` hard-codes the `.` prefix, the
   `_pending_numbered_link` attribute, and "a repeated prefix re-arms". Introduce a
   frozen `NumberedLinkPrefix(prefix_key, chip_prefix, state_attr)` descriptor and
   descriptor-taking `arm_link_prefix`, `clear_link_prefix`, and
   `handle_link_prefix_key(widget, event, prefix, *, follow, on_double=None, on_zero=None)`.
   Keep the module's existing `.`-flavoured constants and the `arm_numbered_link` /
   `clear_numbered_link_prefix` / `handle_numbered_link_key` functions as thin wrappers
   over the descriptor, so the Memory pane, its help modal, and
   `help_modal/binding_common.py` need no edits and
   `tests/ace/tui/modals/test_numbered_link_keys.py` keeps passing unchanged.

   Behaviour that must not drift: the dispatcher never fires while an `Input` has focus
   (and clears any armed prefix in that case), consumes the prefix key, consumes a
   decimal digit, and cancels on any other key _without_ consuming it so normal dispatch
   continues. The two new behaviours are reachable only through the optional callbacks —
   `on_double` for a repeated prefix key and `on_zero` for `0` — and when a callback is
   absent the current behaviour (re-arm, cancel) stands. Export a
   `LINK_FOLLOW_PREFIX = NumberedLinkPrefix("dollar_sign", "$", "_pending_link_prefix")`
   for the app to use.

3. **One ordering shared by the rail and the keys.** The rail's chip grouping is private
   to `widgets/link_rail.py` today (`_RailItem`, `_rail_items`, `_projected_group_key`),
   so a follow verb that re-derived it could disagree with what the user sees. Move it
   into a new pure module `src/sase/ace/tui/relations/link_keys.py` exporting
   `LinkRailItem` (chip, count, projected_group, neighbor_kind),
   `link_rail_items(chips)`, `MAX_DIRECT_LINK_KEYS = 9`, and
   `link_key_label(index, total_links)` returning `"$$"` for the single-link lead chip
   and `"$N"` otherwise. `link_rail.py` imports them and deletes its private copies; its
   rendering must not change, and every existing assertion in
   `tests/ace/tui/test_link_rail.py` stays green as written. The `$0` Links panel in
   phase `panel` will read the same module, which is what keeps the rail, the keys, and
   the panel from ever disagreeing.

   Keys are assigned from this order and never from what fits: the first
   `MAX_DIRECT_LINK_KEYS` items are addressable whether or not the width ladder rendered
   them, and items past that are reachable only through `$0`.

4. **The app-level follow verb.** Add `src/sase/ace/tui/actions/link_follow.py` with a
   `LinkFollowMixin`, exported through `actions/__init__.py` and `actions/__init__.pyi`
   and mixed into `AceApp` beside `LinkSubjectMixin` (both mixin lists in
   `src/sase/ace/tui/app.py`). Initialize `_pending_link_prefix = False` and the bounded
   `_link_trail` in `actions/_state_init_navigation.py` next to the existing
   `_link_index*` state, and declare `_pending_link_prefix` in `actions/_event_base.py`
   beside the other mode flags.
   - `action_follow_artifact_link()` arms the prefix, and only when the action's own
     availability predicate holds (`link_rail_enabled()` and a non-empty
     `link_edges_for_selection()`) and no `Input` has focus.
   - `_handle_link_prefix_key(event)` resolves an armed prefix through
     `handle_link_prefix_key(...)`: a digit `1`-`9` follows that item, a repeated `$`
     follows item 1, `0` opens the Links panel. Wire it into `on_key` in
     `actions/_event_keyboard.py` as its own branch in the elif chain, ahead of
     `_ancestor_mode_active`/`_leader_mode_active` and after the entry-jump branches, so
     an armed `$2` can never fall through to the bare-digit Artifacts sub-tab bindings
     nor `$0` to `start_saved_query_mode`.
   - `_follow_link_item(item)` applies the destination policy resolved in the snapshot
     at index-build time (`LinkChip.neighbor_target` plus `parse_link_ref`), never
     re-resolved on the key path and never through `_known_target_for_ref`'s O(n) scan:
     - a collapsed counted group (`item.count > 1`) opens the `$0` panel scoped to that
       group instead of jumping — fourteen destinations cannot be chosen between by one
       key;
     - `chop:` routes to AXE, matching `_axe_items` through `axe_item_key` and
       `ChopSnapshot.base_identity` (the same identity `relations/link_subject.py`
       already uses to build a chop ref);
     - `agent:` is the one intentional dynamic route: the Agents tab when the ref
       matches a loaded agent (compare `reference_for_agent_name(agent.name)` and the
       bare payload), Artifacts ▸ Agent otherwise;
     - every other kind routes to `neighbor_target.pane_id` on Artifacts;
     - a chip with no `neighbor_target` that is not a `chop:` is dangling: it stays
       visible and keyed, warns naming the ref, navigates nothing, and mutates no
       history. Hiding a dangling link would under-report the graph.
     - when the destination is already represented in the current pane, prefer same-pane
       selection over a gratuitous tab switch.
   - `_open_artifact_links_panel(scope=None)` is the single seam phase `panel`
     (`sase-ug.9`) fills in; until it lands, it warns that the panel is not yet
     available. `$0` and every counted chip must land there and nowhere else.

5. **A jump that always lands.** Order of attempts, each rung reversible, each toasting
   exactly what it changed:
   - `actions/navigation/_tree.py`: `_navigate_to_relation_target`'s cross-pane branch
     calls `_request_artifacts_entry(target)` and returns immediately, so the
     `reveal_entry_target` ladder below it is unreachable off-pane — the defect that
     makes a jump to a closed bead land nowhere while the Beads default query is
     `-status:closed`. Re-resolve the destination navigator after the request and, when
     the target still is not selected and that pane has finished loading, fall through
     to the same `reveal_entry_target` → "is not in the current results" ladder the
     same-pane branch already uses. Have `_request_artifacts_entry`
     (`actions/artifacts_navigation.py`) return whether the target is selected now, so
     callers escalate on fact rather than on a guess; existing callers ignoring the
     return value keep working.
   - Panes resolve their own filtered-out and folded targets. Beads and Plans already do
     (`_clear_filter_for_entry_jump`, `_notify_filter_cleared_for_entry_jump`,
     `_expand_parent_for_pending_target`). Give the Files pane
     (`widgets/artifacts/files_options.py`) and the Artifacts Agents pane
     (`widgets/artifacts/agents_options.py`) the same treatment _before_ their existing
     "is no longer visible" warning: clear the committed filter once, refresh, and toast
     what was cleared. For Files also expand the collapsed group that owns the pending
     target through `_group_fold_registry()` — the group-fold analogue of the Beads epic
     expansion — so a folded target is revealed by expanding only what hides it.
   - App-level ladder in `LinkFollowMixin`, above what a pane can do for itself:
     1. when the target names a project other than `artifacts_project_scope`, call
        `_set_artifacts_project_scope(project, picked=True)` first. Never report a
        target missing before resolving its owning project.
     2. `_save_current_tab_position()`, switch `current_tab`,
        `_request_artifacts_entry`.
     3. if it did not land and the pane is loaded, drop the bounded head slice through
        the pane-generic limit contract — `extract_limit(pane.host_limit_query())`, then
        `apply_host_limit_query` with the remainder plus `limit:all` — and request the
        target once more by its stable identity.
     4. only then warn, naming the ref that could not be resolved.
   - Record one hop per successful jump onto `_link_trail`, bounded at 32
     (`_MAX_LINK_TRAIL`), carrying the pre-jump tab, pane key, origin
     `ArtifactEntryTarget`, and the pane's pre-jump query source, so a later `Ctrl+O`
     can undo a widening. A dangling or failed jump leaves the trail untouched. This
     phase only _writes_ that record: `Ctrl+O` / `Ctrl+Shift+O` routing, fold-state
     capture, and the breadcrumb chip are phase `trail` (`sase-ug.8`). Keep the hop
     record private to `link_follow.py` for now so Symvision sees an in-file consumer;
     phase `trail` publishes it when it imports it.

6. **Suppress the surface when the selection has no links.** `check_app_action` already
   gates `follow_artifact_link` on the flag plus `link_edges_for_selection()`, and an
   unavailable action produces no binding and no footer chip. Extend the same gate to
   the command palette, which uses its own predicate layer: add a
   `link_edges_present: bool | None = None` field to `CommandContext`
   (`commands/types.py`), populate it defensively in `commands/context.py` from
   `link_edges_for_selection()` when the flag is on (leaving it `None` when the flag is
   off or the call raises), and hide `app.follow_artifact_link` in
   `is_command_available` (`commands/availability.py`) ahead of the per-tab dispatch
   when the flag is off or the field is `False`. Add `$$ / $1-$9 / $0` help rows to the
   Artifacts, Agents, and AXE help sections, gated on `link_rail_enabled()` so the beta
   surface stays out of help — and out of the help goldens — while the flag is off.

7. **Tests.**
   - Keymap: `$` is a valid key, displays as `$`, canonicalizes from the raw glyph to
     `dollar_sign`, binds `follow_artifact_link`, and appears once in the command
     catalog.
   - Prefix dispatcher: the `.` descriptor's behaviour is byte-for-byte what it was; a
     `$` descriptor arms and resolves independently of it; `on_double` and `on_zero`
     fire for `$$` and `$0`; a focused `Input` suppresses arming and resolution both.
   - Key-to-chip mapping: `$1`-`$9` follow `link_rail_items` order; a collapsed
     projected group takes exactly one key and does not renumber the chips after it; a
     chip truncated off the rail by width still answers its key; `$$` follows the lead
     chip at n=1; a key with no chip (`$7` on a three-link subject) is consumed and does
     nothing; a non-digit after `$` cancels without being consumed.
   - Destination routing, one case per ref kind in both endpoint positions:
     bead/patch/stitch/file and a provider document to their Artifacts pane, `chop:` to
     AXE, `agent:` to Agents when loaded and to Artifacts ▸ Agent when not.
   - Always lands, one case per pane: a filtered-out target, a folded target, a target
     outside the head slice, and a cross-project target each reveal, and the toast names
     what changed. A cross-project target is never reported missing before the scope
     change is attempted.
   - Dangling target: visible, keyed, warns, navigates nothing, leaves `_link_trail`
     empty. A failed jump leaves history untouched.
   - Invisibility: flag off, or a selection with no edges, leaves `$` inert — no
     binding, no footer chip, no palette entry, no help row — and `$0` unreachable.
   - Generation guard (`tui_perf` rule 4): a selection change during an async link-index
     refresh never follows or paints the prior row's chips; re-read the subject after
     every await before applying.

## Verification

- Run the focused suites while iterating: `tests/ace/tui/test_link_rail.py`,
  `tests/ace/tui/test_artifacts_relation_key_resolution.py`,
  `tests/ace/tui/test_artifacts_relation_navigation.py`,
  `tests/ace/tui/modals/test_numbered_link_keys.py`, `tests/test_keymaps_*.py`,
  `tests/test_command_catalog*.py`, plus the new link-follow tests.
- Run `just install` first if this workspace has drifted, then `just check` after all
  repository changes; hand it to `/sase_monitor` if it runs long. The rail render and
  the flag-off help sections are unchanged, so no PNG goldens should move — if any do,
  run `just test-visual` and only accept a change you can explain.
- Out of scope, do not fix here: the stale `sase/artifact_relations.json` /
  `sase/memory/*` generated files noted on the parent epic bead. Regenerating SASE
  memory needs the user's explicit approval, which this phase does not carry.
- Run `sase bead epic-symbols sase-ug.7`, and resolve or re-key every remaining
  `--epic-symbol` entry before closing. Prefer a real in-repo consumer; a symbol only a
  later phase will consume gets an `--epic-symbol` keyed to the parent epic `sase-ug` or
  to the consuming phase, never to `sase-ug.7`, which is about to close.
- Close only `sase-ug.7`, with a note naming the checks actually run. Do not close
  `sase-ug` or any ancestor. Record discovered follow-up work as a `PROPOSED FOLLOW-UP:`
  note on `sase-ug.7` rather than creating beads.
