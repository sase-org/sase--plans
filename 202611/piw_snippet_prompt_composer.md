---
tier: tale
title: Expand the piw snippet to "prompt composer"
goal:
  Typing `piw` in the sase TUI prompt composer and in Neovim expands to "prompt
  composer", and the change is applied to the home directory.
size: small
proposed_by: bbugyi200.athena.1fv.f0
proposed_with: claude/opus
create_time: 2026-11-25 07:59:57
status: wip
---

# Expand the `piw` snippet to "prompt composer"

## Goal

The sase repo now calls the TUI's prompt input widget the **prompt composer** (see the
new `Prompt Composer` glossary term, whose aliases include `prompt input widget`).
Bryan's personal `piw` snippet still expands to the old name. Change its expansion to
`prompt composer` everywhere the snippet is defined, then deploy the change to the home
directory.

## Where the snippet lives

All work happens in the **`chezmoi` linked repo**. Open it first with
`sase repo open chezmoi -r "<reason>"` and use only the path it prints. Read that repo's
`AGENTS.md` and `home/dot_config/sase/AGENTS.md` before editing.

A full search of the chezmoi repo, the live `~/.config/sase/` directory, and the sase
repo found exactly two definitions and no `#[piw]` splices in any other snippet:

1. `home/dot_config/sase/sase.yml`, under `ace.snippets` (inside a `# keep-sorted start`
   / `# keep-sorted end` block):

   ```yaml
   piw: "prompt input widget"
   ```

2. `home/dot_config/nvim/lua/bb_utils/_snip_utils.lua`, the LuaSnip mirror. The rule in
   `home/dot_config/sase/AGENTS.md` requires every `sase.yml` snippet to have a matching
   LuaSnip definition in this file:

   ```lua
   -- SNIPPET: piw
   s({ trig = "piw", desc = "prompt input widget" }, { t("prompt input widget") }),
   ```

## Changes

1. In `home/dot_config/sase/sase.yml`, change the value only:

   ```yaml
   piw: "prompt composer"
   ```

   Keep the `piw` trigger, its position in the keep-sorted block, the double-quoted
   style, and the lowercase wording (it is spliced mid-sentence, like `mrp` and `prp`
   next to it).

2. In `home/dot_config/nvim/lua/bb_utils/_snip_utils.lua`, update both the `desc` and
   the text node so the mirror matches exactly:

   ```lua
   -- SNIPPET: piw
   s({ trig = "piw", desc = "prompt composer" }, { t("prompt composer") }),
   ```

   Leave the `-- SNIPPET: piw` comment and the surrounding alphabetical order alone.

Make no other edits. In particular, do not rename the trigger or add a new one, and do
not touch the sase repo.

## Verification

Run these from the opened chezmoi checkout:

1. `yq '.ace.snippets.piw' home/dot_config/sase/sase.yml` prints `prompt composer`.
2. `git grep -n -i "prompt input widget"` finds no text matches. A binary match in
   `memory-directory-map.png` is expected and unrelated.
3. `stylua --check home/dot_config/nvim/lua/bb_utils/_snip_utils.lua` and
   `luacheck --no-global home/dot_config/nvim/lua/bb_utils/_snip_utils.lua` both pass.
4. `just lint-keep-sorted` passes. If its setup step cannot fetch the `keep-sorted`
   binary, say so in your final response. Only a value changed, so the sort order is
   unaffected.
5. Optional read-only preview:
   `chezmoi --source "$PWD" diff ~/.config/sase/sase.yml ~/.config/nvim/lua/bb_utils/_snip_utils.lua`
   shows only the two expected line changes. Never run `chezmoi apply` with `--source`
   pointed at the opened checkout. Deployment happens only through the staged monitor
   below.

## Deploy after the host commits

The chezmoi repo's core memory requires `chezmoi update -a --force` after any commit to
that repo, including one landed by a SASE finalizer. Host completion commits and pushes
the opened chezmoi checkout only after this turn ends, so the command cannot run inside
the turn. Use `/sase_monitor` and stage it as the turn's last handoff before
`/sase_final`:

```bash
sase monitor start \
  -r "Apply the pushed piw snippet change to the home directory" \
  -s APPLYING -S APPLIED \
  -- chezmoi update -a --force
```

Do not pass `-n`/`--next`; nothing needs a follow-up agent. The monitor launches only
after host finalization, so it pulls the pushed commit into the chezmoi source and
applies it to `~/.config/sase/sase.yml` and the Neovim config.

Then finish through `/sase_final` with a `commit` decision for the chezmoi repo. The
primary sase checkout has no changes. Suggested commit subject:
`chore(sase): expand piw snippet to "prompt composer"`.

## Expected result

- In the sase TUI prompt composer, typing `piw` and pressing `Tab` inserts
  `prompt composer`. Running TUIs watch their prompt sources and rebuild the snippet
  catalog when the applied `~/.config/sase/sase.yml` changes, so no restart is needed.
- In Neovim, the `piw` LuaSnip trigger expands to `prompt composer` after Neovim
  restarts.

## Out of scope

- Other tailnet machines (apollo, mac) pick up the change on their next
  `chezmoi update`. Do not dispatch or SSH to them.
- Any further prompt-composer wording cleanup in the sase repo. Leftover old names in
  `docs/reference/config/ace-prompts.md` are tracked separately as task bead `sase-1v6`.
