---
tier: tale
title: Drop the explicit-only qualifier from the README agent status cells
goal:
  The README "Works with your agents" table shows a plain **Supported** status for Muse
  Code and Grok Build, and README.md stays prettier-clean.
size: xsmall
proposed_by: bbugyi200.athena.05x
create_time: 2026-08-18 08:03:00
status: wip
---

# Drop `; explicit-only` from the Muse/Grok rows of the README agent table

## Goal

In the top-level `README.md`, the `## Works with your agents` table lists every
supported agent CLI with a **Status** cell. Five rows read `**Supported**`; the Muse
Code and Grok Build rows read `**Supported; explicit-only**`. Make those two rows read
`**Supported**` like the rest, so the Status column answers one question — is this agent
supported? — and the explicit-only nuance is carried only by the prose that already
explains it in the Quick start section.

This is a documentation-only change. No provider behavior changes: Muse and Grok remain
explicit-only at runtime (SASE still never auto-detects the generic `muse`/`grok`
executable names), and no Python, Rust, config, or test file is touched.

## Context

- `README.md` is the only place in the repo that renders this table. `docs/index.md`
  does not duplicate it, so no docs page needs a matching edit.
- `README.md` is also the package long description (`path = "README.md"` in
  `pyproject.toml`), so the edit reaches the next PyPI release page. That is expected
  and fine.
- Markdown in this repo is formatted by prettier (`just fmt-md`) and gated by
  `just fmt-md-check`, which runs inside both `just check` and `just check-full`.
  Prettier pads every table cell to the width of the widest cell in its column, so
  shrinking the two Status cells **also shrinks the Status column** on every one of the
  table's nine lines. An edit that touches only the two Muse/Grok lines will fail
  `fmt-md-check`. The exact post-format table is given below, so it can be written
  directly.

## Change to make

Edit `README.md`, `## Works with your agents` section (currently lines 98-108). Replace
the whole table with this prettier-clean form — the Status column narrows from 28 to 13
characters wide; the Agent column and the separator row are unchanged:

```markdown
| Agent                                                                           | Status        |
| ------------------------------------------------------------------------------- | ------------- |
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code)                   | **Supported** |
| [Antigravity CLI (`agy`)](https://antigravity.google/)                          | **Supported** |
| [Codex](https://github.com/openai/codex)                                        | **Supported** |
| [Qwen Code](https://github.com/QwenLM/qwen-code)                                | **Supported** |
| [OpenCode](https://opencode.ai/)                                                | **Supported** |
| [Muse Code](https://developer.meta.com/ai/resources/blog/build-with-muse-code/) | **Supported** |
| [Grok Build](https://docs.x.ai/build/overview)                                  | **Supported** |
```

Equivalent recipe if editing in place is preferred: replace both occurrences of
`**Supported; explicit-only**` with `**Supported**`, then run `just fmt-md` and commit
whatever prettier rewrites (it produces exactly the block above). Do not hand-align the
table without running prettier.

## Explicitly out of scope

Leave these alone — they remain accurate, because the explicit-only behavior itself is
not changing:

- `README.md` line ~71 (Quick start prose): "Muse Code and Grok Build are explicit-only:
  select either with a Muse/Grok model/provider directive because SASE does not
  auto-detect the generic `muse`/`grok` executable names." This sentence is where the
  nuance now lives; keep it.
- The `%model:muse/...` and `%model:grok/...` example commands in Quick start.
- Every `explicit-only` mention under `docs/` (`docs/agent_providers.md`,
  `docs/llms.md`, `docs/getting_started.md`, `docs/xprompt.md`, and the blog posts).
- All provider code and tests, including the auto-detect assertions in
  `tests/llm_provider/test_grok_provider_core.py`.

## Verification

1. `just install` first — workspaces are ephemeral and may have stale dependencies.
2. `node_modules/.bin/prettier --check README.md` (or the full `just fmt-md-check`) must
   pass, confirming the table alignment is prettier-clean.
3. `git diff README.md` should show exactly the table block: two Status cells losing
   `; explicit-only`, plus the column-width re-padding on the other six table lines. No
   other file appears in the diff.
4. `grep -n "explicit-only" README.md` should return only the Quick start prose line
   (~line 71) and nothing in the table.
5. Run `just check` before reporting done. A README-only change selects no scoped tests;
   the gates that matter are the markdown format check and `SASE validation`.
   `just check-full` is not required for a documentation-only edit.

## Done when

`README.md`'s agent table shows a plain `**Supported**` for all seven agents, the file
is prettier-clean, the Quick start explicit-only sentence is untouched, and `just check`
passes.
