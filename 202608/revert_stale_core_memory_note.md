---
tier: tale
title: Revert the stale-core sentence from build_and_run.md
goal:
  Tier 1 memory stops permanently carrying a stale-core explanation that the build now
  prints on demand, with the derived instruction files regenerated to match.
proposed_by: bbugyi200.athena.v1.f1
create_time: 2026-08-07 17:38:31
status: done
---

# Revert the stale-core sentence from `build_and_run.md`

## Problem

Commit `5a039ef14` added five lines to `sase/memory/build_and_run.md` describing what
happens when the linked `sase-core` checkout falls behind the `sase-core-rs` floor. That
note is `type: short`, so it is always-loaded Tier 1 memory: it is rendered into
`AGENTS.md` and every provider shim (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`)
and enters the context of every agent in this repo, on every task, forever.

The same commit made the failure it describes impossible to miss. The note is now paying
a permanent context cost to pre-announce a message the build will print at the exact
moment it becomes relevant.

## Diagnosis

### What the note says versus what the guard says

The added text carries three claims:

1. the linked `sase-core` checkout is the thing that goes stale;
2. `just install`/`just check` now fail loudly instead of producing confusing test
   failures;
3. the remedy is `sase repo open sase-core`, then rerun `just install`.

The guard message added in the same commit (`Justfile:94` and `Justfile:701`) reads:

```
[setup] ERROR: the sase-core checkout is behind the sase-core-rs floor in
pyproject.toml; the extension built from it will not satisfy sase's tests.
In a SASE workspace run 'sase repo open sase-core'; otherwise update the checkout
directly. Then rerun 'just install'.
Set SASE_ALLOW_STALE_CORE=1 to proceed anyway (intentional bisects only).
```

Claim 1 is the message's first clause. Claim 3 is its second and third sentences,
verbatim and with a non-workspace fallback the note omits. Claim 2 is self-evident to
anyone reading it. The message is strictly more informative than the note — it also
names `SASE_ALLOW_STALE_CORE`, which the note never mentions.

### The guard actually fires on every path an agent takes

Verified by reading the recipe graph, not inferred:

| Command                          | Path to the guard                   | Result                                |
| -------------------------------- | ----------------------------------- | ------------------------------------- |
| `just install`                   | `install` → `rust-install`          | `ERROR` + exit 1 before the build     |
| `just test`                      | `test` → `_setup-visual` → `_setup` | `ERROR` + non-zero exit before pytest |
| `just check` / `just check-full` | `check`/`check-full` → `_setup`     | `ERROR` + non-zero exit               |
| `just lint`, `just fmt`, ...     | `_setup`                            | `ERROR` + non-zero exit               |

`rust-install` runs `validate_sase_core_rs_version` _before_ invoking `maturin develop`
(`Justfile:696`), so `just install` cannot even build a stale extension. `_setup`
handles bit `16` before every other branch (`Justfile:90-98`). There is no ordinary
agent entry point that reaches a stale-binding test failure without printing the remedy
first.

### There is no residual gap the note would cover

- **Ahead-direction skew** stays a warning by design; the note never addressed it.
- **Within-release checkout drift** (the blind spot documented in
  `tools/validate_sase_core_rs`' docstring: the Cargo workspace version does not move
  between releases) is not detectable by any version check, so the note could not help
  there either. It self-heals through `just install`, which unconditionally rebuilds
  from the checkout — and telling agents to run `just install` first is exactly what the
  _original_ sentence already says.
- **Non-dev paths** (`SASE_CORE_WHEEL`, no `Cargo.toml`, no `cargo` on `PATH`) resolve
  the published wheel, where `uv` enforces the floor normally.

So the surviving original sentence — run `just install` before other commands because
workspaces are ephemeral — remains the correct always-loaded advice, and everything the
addition contributed is now delivered just-in-time.

### Cost

`sase/memory/README.md` records the note growing from 59 lines / ~717 tokens to 64 lines
/ ~817 tokens, and the repo total from ~10191 to ~10291 approximate tokens. That is ~100
tokens of always-loaded Tier 1 budget, in every agent context in this repo, to duplicate
a message the failing command prints.

## Scope

Restore `sase/memory/build_and_run.md` to its pre-`5a039ef14` wording and regenerate the
derived instruction files. Change nothing in `Justfile`, `tools/`, or `tests/` — the
guard and its tests are correct and are what makes this revert safe.

## Decision: full revert, not a trimmed version

The alternative considered was keeping a one-clause pointer (e.g. "if the linked
`sase-core` checkout goes stale, `just install` will say so"). Rejected: a pointer to a
self-explanatory error message is still permanent context spend with no decision it
changes. An agent who never hits the failure gains nothing; an agent who hits it reads a
better message than the pointer. Tier 1 memory should carry what an agent must know
_before_ acting and that nothing will tell them at the point of need; this fails that
test in both halves.

## Steps

### 1. Revert the memory note

In `sase/memory/build_and_run.md`, restore the paragraph at lines 38-46 to its original
single sentence:

```markdown
**IMPORTANT**: One consequence of sase's ephemeral workspace directories (see the
sase.md file in this directory) is that you need to run `just install` before running
other commands like `just check` (since it is possible we haven't used this workspace
directory in a long time and package dependencies may have changed).
```

That is, drop everything from `The thing that` through `then rerun `just install`.` and
close the sentence after `may have changed).`.
`git show 5a039ef14 -- sase/memory/build_and_run.md` is the authoritative diff to
invert. Touch no other part of the file.

### 2. Regenerate the derived files

Run `sase memory init`. It rewrites `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, `QWEN.md`, and refreshes the asset-backed `sase/memory/README.md`
counters. Expect the README to return to 59 lines / ~717 tokens for this note and 789
lines / ~10191 tokens repo-wide, matching the pre-`5a039ef14` values exactly.

Do not hand-edit any of those six files; they are generated.

This is user-authorized follow-through for a user-requested memory edit, so it needs no
separate confirmation.

### 3. Confirm nothing else depended on the removed text

`tests/test_justfile_lint.py:126` and `:160` assert on `"sase repo open sase-core"`, but
those assertions target the **Justfile guard's** stderr, not the memory note. They must
keep passing untouched. No other file outside the six generated ones and the note itself
references the removed wording.

## Verification

1. `just install` — workspaces are ephemeral and this one may be cold. Confirm it
   reports the core version it built.
2. `just check`. Markdown-only changes are not on the exceptions list, and `check`'s
   `sase validate` step is the gate that catches memory-init drift between the repo and
   the chezmoi home root, so it is the check that actually proves step 2 landed
   completely. `just fmt-md-check` runs within it; the restored text is the wording that
   was already Prettier-clean before `5a039ef14`, so no reformatting should be needed.
3. `sase memory init --check` should report no drift after step 2.
4. `git diff --stat` should show exactly seven files: `sase/memory/build_and_run.md`,
   `sase/memory/README.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
   `QWEN.md`. Any `Justfile`, `tools/`, or `tests/` entry means step 1 overreached.
5. `git diff 5a039ef14^ -- sase/memory/build_and_run.md` should be empty, proving the
   note is byte-identical to its pre-guard state.

## Out of scope

- **The guard itself.** `Justfile`, `tools/validate_sase_core_rs`,
  `tools/validate_sase_core_rs_version`, `tools/validate_test_environment`, and their
  tests all stay exactly as `5a039ef14` left them. This plan is only viable _because_
  they work.
- **Auditing the rest of Tier 1 for similar redundancy.** Other notes may also be paying
  always-loaded cost for just-in-time information, but that is a separate review with a
  separate approval, not a rider on this revert.
- **Re-recording the removed detail elsewhere.** It is already in the guard message, in
  `~/.sase/plans/202608/stale_core_binding_guard.md`, and in commit `5a039ef14`'s
  message. Nothing is lost.
