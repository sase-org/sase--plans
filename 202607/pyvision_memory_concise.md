---
create_time: 2026-07-09 12:04:21
status: done
prompt: 202607/prompts/pyvision_memory_concise.md
tier: tale
---
# Plan: Make `memory/pyvision.md` more concise and instance-free

## Goal

Rewrite the body of the Tier‑2 long-term memory note `memory/pyvision.md` so it is meaningfully shorter and no longer
references specific past incidents of us fixing pyvision errors, while preserving every durable rule. Keep the note's
frontmatter (`type`, `parent`, `description`, `keywords`) byte-for-byte identical so the generated memory index/shims do
not change, and leave all memory/validation gates green.

## Context / Why

`memory/pyvision.md` (created in commit `e137094bc`) is the read-on-demand playbook for fixing `pyvision` lint failures.
It works, but it grew verbose (~95 lines) and bakes in two references to specific incidents that don't belong in a
durable rules doc:

- Rule 6 frames the URI-pragma CI trap as "the trap behind the last CI break."
- Rule 7 explains "don't over-delete" with a parenthetical about the `agent_status_groups` /`group_agent_statuses`
  Telegram fix.

The user wants the note tightened and those instance-specific mentions removed. The rules themselves are still correct
and worth keeping — this is an editorial/condensing pass, not a rules change.

### Redundancy to collapse

- The intro "It scans `src/` and fails when:" bullet list (4 items) restates the same categories as Rule 2's "Diagnose
  the exact error category" mapping. These should merge into one error→fix mapping.
- Rule 4 (private-symbol fixes) is the fix half of two of those categories and can fold into the same mapping.
- Rule 3.4 and Rule 8 both cover `--epic-symbol`; consolidate into one "Epic symbols" section.
- Rule 7 ("don't over-delete") shrinks to a one-line caveat inside the deletion hierarchy once the example is dropped.

## Scope

**In scope**

1. Rewrite the body of `memory/pyvision.md` to be more concise and free of specific-incident references, preserving all
   durable guidance and the exact pyvision error strings.
2. Keep frontmatter (`type: long`, `parent: AGENTS.md`, `description`, `keywords`) unchanged, byte-for-byte.
3. Normalize markdown wrapping with `just fmt-md` and confirm no memory drift and green gates.

**Out of scope**

- No production/source code changes; no change to `tools/pyvision-260708` (vendored).
- No substantive rule changes — nothing that would alter the _advice_ pyvision agents get, only its length and phrasing.
- No frontmatter/description change, so `sase memory init` should be a no-op for `AGENTS.md`, the provider shims
  (`CLAUDE.md`/`GEMINI.md`/`OPENCODE.md`/`QWEN.md`), and `memory/README.md`.

## Permission note (memory-file edits)

Editing `memory/*.md` requires explicit user permission. The user explicitly asked to make this note more concise, which
grants permission for `memory/pyvision.md`. Because the `description` is unchanged, running `sase memory init` for
verification should produce no changes to `AGENTS.md` or the shims — the diff is expected to be limited to
`memory/pyvision.md` alone. If regeneration unexpectedly wants to touch other files, stop and confirm.

## Design decisions

- **Body-only edit; frozen frontmatter.** The generated index only carries the `description`. Leaving it identical keeps
  regeneration a no-op and avoids the Prettier/YAML frontmatter-wrapping fights encountered when the note was created.
- **One error→fix mapping.** Merge the intro "fails when" list, Rule 2's category mapping, and Rule 4's private-symbol
  fixes into a single bulleted "Error → fix" section keyed by the actual error string.
- **Keep the decision hierarchy prominent.** The delete > make-private > pragma > `--epic-symbol` order is the core of
  the note; keep it as its own section, with "don't over-delete" as a trailing one-line caveat (no example).
- **Preserve every quoted error string.** The note's value is that it maps real pyvision output to a fix; keep the exact
  strings (`Unused public functions/classes...`, `pragma cannot be applied to private symbol`,
  `external repository '<url>' does not reference symbol '<name>'`, etc.).
- **Generic phrasing for the CI trap and over-delete rule.** State the mechanism and the fix, not the incident.

## Proposed rewritten `memory/pyvision.md`

Frontmatter is copied verbatim from the current file (unchanged). Body below:

```markdown
---
type: long
parent: AGENTS.md
description:
  Read before fixing pyvision lint failures, including unused symbols, private misuse, pragmas, and epic whitelists.
keywords: pyvision, unused symbol, pragma, epic-symbol, external repo, lint
---

# Fixing pyvision Errors

`pyvision` (`tools/pyvision-260708`) is the unused/misused-symbol linter. It scans `src/` and runs as the `pyvision`
stage of `just lint` / `just check`, or alone via `just _lint-pyvision` / `just pyvision`.

**Test references never count.** A path with a `test` / `tests` / `testing` component (or a `test_*.py` file) only
satisfies the private-symbol "imported from a non-test file" guard — it can NOT keep a public symbol alive. Defs under
`testing/` are ignored entirely.

**Never edit the vendored tool.** `tools/pyvision-260708` (any `tools/*-YYmmdd` file) is vendored from dotfiles. Fix
your _code_, not the linter. If the tool is genuinely wrong, change the dotfiles source, commit it there with the commit
skill, and re-vendor with `pyvendor` (see `tools/CLAUDE.md`).

## Error → fix

- `Unused public functions/classes...` — a public symbol has no non-test consumer; fix by the hierarchy below.
- `Private functions/classes should not be imported...` — a `_name` is used across files. Stop importing it across
  files, or (only if a real non-test file needs it) make it public — which then needs a real consumer.
- `Private functions/classes must be used in the file where they are defined:` — a dead private symbol. Delete it, or
  wire up its intended in-file caller.
- `Error: pyvision pragma in <file>:<line>: ...` — a pragma problem (see Pragmas).
- `Error: --epic-symbol '<...>': ...` — a stale/invalid epic whitelist (see Epic symbols).

## Unused public symbol — decision hierarchy

Whitelisting is the last resort, and a symbol exercised _only_ by tests is not "used". In order:

1. **Delete it** if genuinely dead (no consumer anywhere, including linked repos) — and delete its tests.
2. **Make it private** (`_`-prefix) if only used within its own file; update in-file callers.
3. **Add a non-test pragma** only if a real consumer exists that pyvision can't see (non-Python file, config, another
   repo).
4. **Add `--epic-symbol`** only when a later phase of an in-progress epic will consume it.

When deleting, remove only what actually died: drop the symbol plus its now-dead private helpers and its tests, but keep
sibling symbols that still have live consumers and re-check each independently.

## Pragmas

A `# pyvision: <ref>` comment directly above a public def marks a consumer pyvision can't discover.

- Public symbols only (`pragma cannot be applied to private symbol`).
- `<ref>` is a repo-root-relative path to the referencing file, or an external repo URI. It must NOT be a test/testing
  path (`referenced test-support path ... is forbidden`) or a markdown path
  (`referenced markdown path ... is forbidden`). A local path must exist and actually reference the symbol.
- `symbol '<name>' is already imported by other Python files. Remove this unnecessary pragma` → the pragma is stale;
  delete it.
- Cross-repo consumer → URI pragma, e.g. `# pyvision: https://github.com/sase-org/sase-telegram.git`. Only for real
  external consumers, never as a broad whitelist.
- `--exclude-decorator <name>` excludes every def carrying that decorator (matched by simple name) — use it for a whole
  decorator-marked family (e.g. `@hook`) instead of per-symbol pragmas.

### URI pragmas fail differently in CI

Local `just _lint-pyvision` can PASS while CI FAILS (`external repository '<url>' does not reference symbol '<name>'`),
because pyvision resolves a URI pragma against a stale local checkout while CI clones the linked repo's current `main`.

- Reproduce CI: `PYVISION_EXTERNAL_REPO_PATHS=<current-linked-checkout> just _lint-pyvision`.
- Open the linked repo the sanctioned way and use the printed path — never guess it:
  `sase workspace open -p <linked_repo> -r "<reason>" <workspace_num>`.
- Decide from what the _current_ linked repo references: still references it → keep the symbol and its pragma; no longer
  references it → the pragma is stale, so remove the symbol, its dead helpers, its tests, and the pragma (or retarget
  the pragma to the real consumer).

## Epic symbols

`--epic-symbol <bead_id>(<symbol>)` lives in the pyvision invocation in the `Justfile`. Entries are self-cleaning:
pyvision tells you to drop one when the bead is missing/closed, the symbol is now properly used, or the symbol no longer
exists as a public def. Remove the matching entry once its epic phase lands and the symbol gains a real consumer (or the
bead closes).

## Verify

Ephemeral `sase_<N>` workspaces may have drifted deps, so run `just install` first. Re-run the exact failing path
(`just _lint-pyvision`, plus the `PYVISION_EXTERNAL_REPO_PATHS=...` form for URI pragmas), then `just check` as the repo
requires after any code change.
```

## Implementation steps

1. Overwrite `memory/pyvision.md` with the content above, keeping the frontmatter unchanged.
2. Run `just fmt-md` to normalize markdown wrapping to the repo's Prettier config.
3. Run `sase memory init` — expect it to report no changes to `AGENTS.md`/shims/`README.md` (description unchanged); if
   it wants to change those files, stop and reconcile before continuing.
4. Run `sase memory init --check` to confirm no drift.
5. Run `just install`, then `just check`, to confirm the `sase validate` freshness gate and all other gates are green.

## Verification / acceptance criteria

- `memory/pyvision.md` is materially shorter than the current version and contains no reference to specific past
  incidents (no "last CI break", no `agent_status_groups`/`group_agent_statuses`/Telegram example).
- All durable rules survive: never-edit-vendored-tool, test-refs-don't-count, the error→fix mapping, the deletion
  hierarchy, don't-over-delete, pragma rules, the URI/CI reproduction procedure, epic-symbol self-cleaning, and the
  install-then-verify step. Every quoted pyvision error string is preserved.
- Frontmatter (`type`, `parent`, `description`, `keywords`) is unchanged byte-for-byte.
- `sase memory init --check` reports no drift; `git status` shows `memory/pyvision.md` as the only changed file.
- `just check` passes.

## Risks / open questions

- **Regeneration touching other files.** Expected to be a no-op given the frozen description; if `sase memory init`
  changes `AGENTS.md`/shims, pause and confirm rather than committing a wider diff.
- **Prettier reflow of the body.** `just fmt-md` may re-wrap prose; that's fine and expected. The final committed body
  is whatever survives `sase memory init --check` + `just check`.
- **Judgment on what is "durable" vs "instance".** The plan treats the two named incidents as the only instance-specific
  content to strip; the generic mechanisms they illustrate are kept.

```

```
