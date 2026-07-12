---
create_time: 2026-07-09 11:40:08
status: done
prompt: 202607/prompts/pyvision_memory.md
tier: tale
---
# Plan: Add a `memory/pyvision.md` long-term memory for fixing pyvision errors

## Goal

Create a new Tier‑2 (long-term) memory note, `memory/pyvision.md`, that codifies a durable, battle-tested set of rules
for diagnosing and fixing `pyvision` lint failures. The note's `description` must instruct agents to read it whenever
they set out to fix a pyvision error. Register the note so agents are actually pointed at it, and leave the repo's
memory/validation gates green.

## Context / Why

`pyvision` (`tools/pyvision-260708`, run by the `pyvision` stage of `just lint` / `just check`, or directly via
`just _lint-pyvision` / `just pyvision`) is the "unused/misused symbol" linter. It repeatedly trips up agents, and the
most recent CI failure (`ci_pyvision_stale_telegram_pragma`, commit `848aa07`) exposed a non-obvious failure mode:
**local `just _lint-pyvision` passed while CI failed**, because external URI pragmas resolve against a _stale local
checkout/cache_ whereas CI does a fresh clone of the current linked repo. Recording the resolution playbook as a
read-on-demand memory note will save future agents from re-deriving it.

### Evidence reviewed

- `tools/CLAUDE.md` — authoritative pyvision whitelisting guidance (pragmas, `--epic-symbol`, `--exclude-decorator`, URI
  pragmas, "do not edit vendored tool").
- `tools/pyvision-260708` — the tool itself; exact error strings and the test-support-path rule
  (`test`/`tests`/`testing` path components or `test_*.py` never keep a public symbol alive; `testing/` defs ignored).
- Chats: `ci_pyvision_stale_telegram_pragma` diagnosis (`sase-tmp_260709_104308-main`), the submitted plan
  (`...run-3g__plan`), and the implementation (`sase-gh-main-260709_105917`) that landed commit
  `eb2338f83 feat!: remove stale agent status grouping helper`.
- `memory/README.md` — memory frontmatter schema (`type: long`, `parent: AGENTS.md`, required `description`) and the
  `sase memory init` regeneration flow.

## Scope

**In scope**

1. Create `memory/pyvision.md` (a `type: long` note) with the rules below.
2. Regenerate the memory index/shims so the note is discoverable: `sase memory init`.
3. Verify no drift and keep gates green: `sase memory init --check` and `just check`.

**Out of scope**

- No production/source code changes; no changes to `tools/pyvision-260708` (it is vendored — see Rule 0 in the note).
- No changes to any _other_ memory note's substance beyond what `sase memory init` mechanically regenerates (`AGENTS.md`
  Tier‑2 index, provider shims `CLAUDE.md`/`GEMINI.md`/`OPENCODE.md`/`QWEN.md`, `memory/README.md`).

## Permission note (memory-file edits)

The repo rule requires explicit user permission to add/edit `memory/*.md`, `AGENTS.md`, and the provider shims. The user
explicitly requested this new note, which covers `memory/pyvision.md`. Registering it (step 2) is done the **sanctioned
way** — running `sase memory init`, not hand-editing — but that regeneration _will_ touch `AGENTS.md`, the four provider
shims, and `memory/README.md` (it adds the new Tier‑2 entry + stats). Confirming that regeneration is acceptable is part
of approving this plan. If the user prefers, step 2 can be skipped and the note left unregistered (agents would only
find it by directory listing).

## Design decisions

- **`type: long`, not `short`.** These are detailed troubleshooting rules read only when relevant, matching
  `tui_perf.md` / `cli_rules.md`, not always-loaded context.
- **`parent: AGENTS.md`** — top-level long note.
- **Description is action-triggered** so the Tier‑2 index line tells agents exactly when to read it (mirrors how
  `tui_perf.md` says "Read before changing anything that affects TUI performance").
- **Rules are ordered as a decision hierarchy** (delete > make private > legitimate non-test pragma > `--epic-symbol`),
  because the tool's whole purpose is to prevent dead code from accumulating; whitelisting is always the last resort.
- **The CI-vs-local URI-pragma trap gets its own rule**, since that is the specific, expensive lesson from the last
  incident and is not obvious from the tool's output.

## Proposed content of `memory/pyvision.md`

```markdown
---
type: long
parent: AGENTS.md
description:
  Read before fixing any pyvision lint failure (the `pyvision` stage of `just lint` / `just check`, or `just
  _lint-pyvision` / `just pyvision`) — unused public symbols, misused private symbols, and pragma errors.
keywords: pyvision, unused symbol, pragma, epic-symbol, external repo, lint
---

# Fixing pyvision Errors

`pyvision` (`tools/pyvision-260708`) is the unused/misused-symbol linter. It runs as the `pyvision` stage of `just lint`
and `just check`, and can be run alone with `just _lint-pyvision` or `just pyvision`. It scans `src/` and fails when:

- A **public** symbol (top-level function/class whose name does not start with `_`) has no non-test consumer.
- A **private** symbol (`_`-prefixed) is imported/used from another file.
- A **private** symbol is never used inside the file that defines it.
- A **pragma** (`# pyvision: ...`) or `--epic-symbol` entry is invalid or stale.

**Test references never count.** Any path with a `test` / `tests` / `testing` component, or a `test_*.py` file, only
satisfies the private-symbol "imported from non-test" guard — it can NOT keep a public symbol alive. Symbols defined
under `testing/` directories are ignored entirely.

## Rules

1. **Never edit the vendored tool.** `tools/pyvision-260708` (and any `tools/*-YYmmdd` file) is vendored from the
   dotfiles repo. Fix your _code_, not the linter. If the tool itself is genuinely wrong, change the source in dotfiles,
   commit it there with the commit skill, and re-vendor with `pyvendor` (see `tools/CLAUDE.md`). Never silence a finding
   by editing the script.

2. **Diagnose the exact error category before touching anything.** The message tells you the fix:
   - `Unused public functions/classes...` → a public symbol has no non-test consumer (Rule 3).
   - `Private functions/classes should not be imported...` → a `_name` is used across files (Rule 4).
   - `Private functions/classes must be used in the file where they are defined:` → a dead private symbol (Rule 4).
   - `Error: pyvision pragma in <file>:<line>: ...` → a pragma problem (Rule 5).
   - `Error: --epic-symbol '<...>': ...` → a stale/invalid epic whitelist (Rule 6).

3. **Fix an unused public symbol by the decision hierarchy — prefer deletion.** In order:
   1. **Delete it** if it is genuinely dead (no consumer anywhere, including linked repos). Delete its tests too.
   2. **Make it private** (`_`-prefix) if it is only used within its own file; update in-file callers.
   3. **Add a legitimate non-test pragma** only if a real consumer exists that pyvision cannot see — a non-Python file,
      config, or another repo (Rule 5).
   4. **Add `--epic-symbol`** only when the symbol is intentionally unused now but a _later phase of an in-progress
      epic_ will consume it: add `--epic-symbol <bead_id>(<symbol>)` to the pyvision invocation in the `Justfile`.
      Whitelisting is always the last resort. A public symbol exercised _only_ by tests is not "used" — delete it, make
      it private and call it from a non-test path, or give it a real non-test consumer.

4. **Fix private-symbol errors at the root, not by flipping visibility.**
   - `should not be imported`: stop importing the `_name` across files, OR — only if a real **non-test** file
     legitimately needs it — make it public (which then requires a real consumer per Rule 3). Tests importing it is not
     a reason to make it public.
   - `must be used in the file`: the private symbol is dead within its file — delete it, or wire up the intended in-file
     caller.

5. **Pragma rules.** A `# pyvision: <ref>` comment on the line directly above a public def marks a consumer pyvision
   can't discover.
   - Pragmas apply to **public** symbols only (`pragma cannot be applied to private symbol`).
   - `<ref>` must be a repo-root-relative path to the referencing file, or an external repo URI. It must **not** be a
     test/testing path (`referenced test-support path ... is forbidden`) or a markdown path
     (`referenced markdown path ... is forbidden` — docs are not valid consumers). A local path ref must exist and
     actually reference the symbol.
   - If pyvision says `symbol '<name>' is already imported by other Python files. Remove this unnecessary pragma`, the
     pragma is stale — delete it (a real Python consumer now exists).
   - For a cross-repo consumer use a URI pragma, e.g. `# pyvision: https://github.com/sase-org/sase-telegram.git`. Only
     for **real** external consumers — not as a broad whitelist.
   - `--exclude-decorator <name>` excludes every def carrying that decorator (matched by simple name) — use it when an
     entire decorator-marked family (e.g. `@hook`) should be out of scope, rather than per-symbol pragmas.

6. **External URI pragmas: reproduce like CI, and verify against the _current_ linked repo.** This is the trap behind
   the last CI break. Local `just _lint-pyvision` can PASS while CI FAILS, because pyvision resolves a URI pragma
   against a stale local checkout/cache, while CI does a fresh clone of the linked repo's current `main`. The error is
   `external repository '<url>' does not reference symbol '<name>'`.
   - Reproduce CI locally: `PYVISION_EXTERNAL_REPO_PATHS=<current-linked-checkout> just _lint-pyvision`.
   - Open the linked repo the sanctioned way and use the printed path — never guess it:
     `sase workspace open -p <linked_repo> -r "<reason>" <workspace_num>`.
   - Then decide from what the _current_ linked repo actually references:
     - Still imports/references the symbol → keep the symbol and its pragma; the local failure was a stale cache and
       there may be nothing to change here.
     - No longer references it → the pragma is genuinely stale: remove the symbol **and** its now-dead private helpers
       **and** its tests **and** the pragma; or retarget the pragma if a _different_ symbol is the real consumer.

7. **Don't over-delete — remove only what actually died.** When deleting a stale-pragma symbol, keep sibling
   symbols/helpers in the same module that still have live consumers, and re-check each one independently. (In the
   `agent_status_groups` fix, `group_agent_statuses` was removed but `status_bucket_header` / glyph helpers were kept
   because the linked Telegram repo still imported them.)

8. **`--epic-symbol` entries are self-cleaning — remove them when they go stale.** pyvision emits a targeted error and
   tells you to drop the entry when: the bead is not found, the bead is closed, the symbol is now properly used, or the
   symbol no longer exists as a public def. When an epic phase lands and its symbol gains a real consumer (or the bead
   closes), delete the matching `--epic-symbol <bead_id>(<symbol>)` from the `Justfile`.

9. **Verify with the real failing command, in an installed workspace.** Ephemeral `sase_<N>` workspaces may have drifted
   deps, so run `just install` first (otherwise pytest/uv can fail on the lockfile before your fix is even reached).
   Re-run the exact failing path — `just _lint-pyvision`, plus the `PYVISION_EXTERNAL_REPO_PATHS=...` form for URI
   pragmas — and then run `just check` as the repo requires after any code change.
```

## Implementation steps

1. Write the file above verbatim to `memory/pyvision.md`.
2. Run `sase memory init` to regenerate `AGENTS.md`, the provider shims, and `memory/README.md` so the new Tier‑2 entry
   appears (see Permission note).
3. Run `sase memory init --check` to confirm no residual drift, and `sase memory list` to confirm `memory/pyvision.md`
   is listed as an available long note.
4. Run `just install` (ephemeral workspace) then `just check` to confirm the memory/validation freshness gate and all
   other gates are green.

## Verification / acceptance criteria

- `memory/pyvision.md` exists with valid frontmatter (`type: long`, `parent: AGENTS.md`, non-empty `description`).
- The Tier‑2 "long-term Memory" section of `AGENTS.md` (and the regenerated provider shims + `memory/README.md`) lists
  `memory/pyvision.md` with its description.
- `sase memory init --check` reports no drift; `just check` passes (its `sase validate` freshness gate included).
- The note's rules are consistent with `tools/CLAUDE.md` and the actual pyvision error strings — no contradictions.

## Risks / open questions

- **Scope of `sase memory init` regeneration.** It rewrites generated shims and the README. This is expected and
  sanctioned, but it means the diff is larger than a single new file. Flagged above; confirm on approval.
- **Whether to register at all.** If the user wants the note present but _not_ wired into the always-loaded index, skip
  steps 2–3; the note still works with an explicit `sase memory read pyvision.md`, but agents won't be automatically
  pointed to it.
- **Wording drift over time.** The rules quote current pyvision error strings; if the vendored tool is later re-vendored
  with different messages, the note's quoted phrases may need a refresh (low risk, easy fix).

```

```
