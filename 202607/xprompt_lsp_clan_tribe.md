---
tier: tale
title: Align xprompt LSP clan and tribe directives
goal: 'The xprompt language server recognizes, completes, documents, and snippets
  the current %clan/%c and %tribe/%t directives, including clan-level tribe syntax,
  without advertising the removed %family/%f or %group/%g spellings.

  '
create_time: 2026-07-19 11:39:49
status: done
prompt: 202607/prompts/xprompt_lsp_clan_tribe.md
---

# Plan: Align xprompt LSP clan and tribe directives

The runtime and ACE prompt input migrated from parallel-family/group terminology to clans and tribes, but the shared
Rust editor directive registry still exposes `%family`/`%f` and `%group`/`%g`. Because the xprompt LSP uses that
registry for canonicalization, diagnostics, completion, hover text, and snippets, current prompts such as
`%clan(research.@, tribe=research)` receive an incorrect `unknown_directive` diagnostic. The work belongs in the linked
`sase-core` repository; the Python host already parses and documents the current directives.

## Canonical editor directive contract

Update the shared editor registry in `crates/sase_core` to match the live prompt grammar:

- Replace the obsolete `family` entry and `%f` alias with `clan` and `%c`, using user-facing text that describes a named
  parallel agent clan.
- Replace the obsolete `group` entry and `%g` alias with `tribe` and `%t`, using user-facing text that describes the
  user-managed agent tribe.
- Preserve the existing single-value, required-argument metadata for both new entries. Keep `%family`, `%f`, `%group`,
  and `%g` unrecognized so the editor does not recommend launch syntax that the runtime has removed.
- Update any shared fan-out/editor tests whose alias expectations still assume `%t` is unused or that `%f`/`%g`
  canonicalize. Do not add runtime semantics to the LSP: the Python launch parser remains authoritative for full
  validation.

This registry change should automatically make canonical and alias forms valid to the editor analyzer, remove the false
`unknown_directive` diagnostics, expose the new names through normal completion, and supply accurate hover metadata.

## Clan-aware completion and snippets

Bring the Rust completion path up to the syntax already offered by ACE:

- Make directive-argument context detection understand the parenthesized clan form. In `%clan(research, tr)` and
  `%c(research, tr)`, classify only the active post-comma fragment as replaceable and offer the documented `tribe=`
  keyword; do not offer that keyword in the clan-name position or after a completed parenthesized expression.
- Attach concise documentation to the `tribe=` candidate and keep hover lookup tied to canonical `clan` metadata even if
  an internal completion sub-context is needed.
- Continue to offer simple colon snippets for `%clan:name` and `%tribe:name`. For snippet-capable clients, also expose
  the full `%clan(name, tribe=tribe)` skeleton so the supported combined form is discoverable without making `tribe=`
  mandatory for simple clan membership.
- Ensure alias-triggered completion inserts canonical spellings and uses the same replacement ranges as long-form
  completion.

Avoid introducing a second LSP-only directive list: name completion, canonicalization, diagnostics, hover, snippets, and
fan-out parsing should keep deriving from the shared editor registry, with only syntax-specific completion logic layered
on top.

## Regression coverage

Extend `sase_core` editor tests and `sase_xprompt_lsp` server tests to cover the behavior through both the domain API
and the public LSP surface:

- `%cla`/`%c` complete to `%clan`, and `%tr`/`%t` complete to `%tribe`, with the expected aliases and descriptions.
- `%family`, `%f`, `%group`, and `%g` no longer resolve or appear as completion candidates.
- Canonical and alias clan/tribe prompts produce no `unknown_directive` diagnostic, including the screenshot's
  parenthesized clan-level tribe form; a genuinely unknown directive still does.
- Hover content for clan and tribe arguments uses the new canonical metadata.
- Clan keyword completion replaces only the partial keyword with `tribe=`, and its long and short directive forms behave
  identically.
- Snippet-capable LSP clients receive both the simple clan form and the combined clan/tribe skeleton, while `%tribe`
  receives its normal named-value skeleton.

Run Rust formatting and the focused `sase_core` editor and `sase_xprompt_lsp` test suites, then run the full `sase-core`
workspace checks. From the SASE host checkout, rebuild/install the linked core as required by the development workflow
and smoke-test `sase lsp` against representative `%clan` and `%tribe` documents to confirm the installed binary—not only
unit-level helpers—returns clean diagnostics and current completions. No manual Cargo version edits are part of this
work; release-plz owns package versions.

## Documentation and delivery

Review the host xprompt/editor documentation after implementation. The xprompt reference already documents `%clan`,
`%tribe`, aliases, and the clan-level `tribe=` form, so only update prose if the implemented LSP completion behavior
needs an explicit discoverability note. Record the editor/LSP parity fix in the normal `sase-core` release notes or
commit metadata without manually changing release-plz-managed versions.
