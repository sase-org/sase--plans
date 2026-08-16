---
tier: tale
title: sase-core LSP agent-catalog documentation passthrough
goal:
  An agent-family completion in an external editor shows the plan-aware markdown
  documentation block the Python agent-catalog helper already emits, instead of an empty
  documentation popup.
size: small
proposed_by: bbugyi200.athena.sase-n9.land
bead: sase-n9
create_time: 2026-08-16 15:01:34
status: done
---

- **PARENT:**
  [202608/agent_family_completion_previews.md](agent_family_completion_previews.md)
- **BEAD:**
  [sase-n9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n9/README.md)

# Plan: sase-core LSP agent-catalog documentation passthrough

## Problem

Epic `sase-n9` (Plan-aware agent-family completion previews) shipped three of its four
phases. Its `lspdoc` phase — the Rust half in the sibling `sase-core` repo — was closed
as done but was never implemented.

Verified against `sase-core` master (`e55bd44`, v0.27.14), in both the workspace-17 and
workspace-21 linked checkouts, with `git fetch origin` and `gh pr list` showing no
`sase-n9` commit and no open PR:

- `crates/sase_core/src/editor/wire.rs:256-272` — `AgentCompletionEntry` has `name`,
  `status`, `project`, `kind`, `member_count`, `detail`. There is no `documentation`
  field.
- `crates/sase_core/src/editor/completion.rs:1367-1382` —
  `build_agent_completion_candidates` still pushes `documentation: None`
  unconditionally.
- `crates/sase_xprompt_lsp/src/server.rs:3627` —
  `wait_completion_uses_kind_aware_agent_catalog` has no documentation coverage.

The closing note for that phase describes `CompletionCandidate.documentation`
(`wire.rs:109`), a long-pre-existing field on the _outgoing_ candidate struct, and
mistook it for the new field on the _incoming_ catalog entry.

The Python side of the contract already landed in the `sase` repo (commit `15e1fda0c`):
`_family_entries` in `src/sase/integrations/_editor_helper_agents.py` emits a
`documentation` markdown block on family entries via
`agent_family_plan_preview_documentation`. Because `AgentCompletionEntry` does not use
`#[serde(deny_unknown_fields)]`, today's LSP binary silently discards that key. Nothing
is broken; the editor documentation popup for an agent-family completion is simply
always empty, which is exactly the gap the epic set out to close.

## Design

This is the epic's already-authored `lspdoc` design, unchanged. The wire contract is
fixed by the epic plan (`plan:202608/agent_family_completion_previews.md`) and must not
drift:

> `AgentCompletionEntry` gains an optional string field named `documentation`. The
> Python helper omits it or sends `""` when there is nothing to show. Rust deserializes
> it with `#[serde(default)]` and sets `CompletionCandidate.documentation` to
> `Some(value)` only when non-empty. `AGENT_CATALOG_SCHEMA_VERSION` stays at `1`:
> `AgentCompletionEntry` does not use `deny_unknown_fields`, so an older LSP binary
> silently ignores the new key and a newer binary tolerates an older helper.

`crates/sase_xprompt_lsp/src/lsp_convert.rs:completion_item` already converts a
`Some(...)` candidate documentation into a `CompletionItem.documentation` through its
existing `markdown_doc` helper, so no conversion change is needed — the value the Python
helper already emits is markdown.

## Work

Open the repo through the SASE repo skill and work only from the printed path:

```bash
sase repo open sase-core -r "<reason>"
```

### 1. Wire field

`crates/sase_core/src/editor/wire.rs` — add to `AgentCompletionEntry`, after `detail`:

```rust
/// Optional markdown block supplied by the Python editor helper, rendered
/// in the editor's documentation popup. Empty when the helper has nothing
/// to show.
#[serde(default)]
pub documentation: String,
```

Match the surrounding style: a plain `#[serde(default)] String`, like `detail` and
`status`, not an `Option<String>`. Leave `AGENT_CATALOG_SCHEMA_VERSION` at `1`.

### 2. Passthrough

`crates/sase_core/src/editor/completion.rs` — in `build_agent_completion_candidates`
(the `CompletionCandidate` literal around line 1371), replace the hardcoded
`documentation: None` with the entry's value only when non-empty:

```rust
documentation: (!entry.documentation.is_empty())
    .then(|| entry.documentation.clone()),
```

That is the same idiom already used elsewhere in this file (see the
`(!entry.description.is_empty()).then(...)` occurrences), so match it rather than
inventing a new shape.

### 3. Fixtures

Update any `AgentCompletionEntry` literal that constructs all fields positionally or
exhaustively (struct-update syntax and `serde_json::json!` literals that omit the field
need no change, since the field defaults). `cargo check` finds these; do not guess.

## Tests

- `crates/sase_core` unit tests next to the existing `build_agent_completion_candidates`
  coverage: one asserting a catalog entry carrying `documentation` yields a candidate
  with `Some(...)`, one asserting an entry without it (or with `""`) stays `None`.
- `crates/sase_xprompt_lsp/src/server.rs` — extend
  `wait_completion_uses_kind_aware_agent_catalog` so one family entry in the
  `agent_catalog_response` JSON carries a `documentation` markdown string, and assert
  the resulting `CompletionItem.documentation` renders it while an entry without the key
  stays `None`. The JSON fixture in that test is built with `serde_json::json!`, so
  adding the key there also proves `#[serde(default)]` tolerates entries that omit it.

## Verification

From the `sase repo open sase-core` path:

```bash
just check
```

`fmt`, `clippy`, and the full Rust test suite must pass.

No change is needed in the `sase` repo. Do **not** release sase-core and do **not**
touch the `sase-core-rs` pin in `pyproject.toml`: the epic's `editor` phase already
improved external editors through the `detail` field against the released binary, and
this field activates whenever the next sase-core release is consumed.

## Non-goals

- Releasing sase-core or bumping `sase-core-rs` in the `sase` repo.
- Any change to the Python editor helper — its half of the contract already landed.
- Any change to `AGENT_CATALOG_SCHEMA_VERSION`, or to how `detail` /
  `label_details.description` are built.
