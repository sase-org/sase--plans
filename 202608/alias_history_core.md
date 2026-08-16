---
tier: tale
title: Rust core alias-history projection and query
goal:
  Schema 22 indexes every recorded model-alias hop and exposes bounded, provenance-aware
  agent history through the Rust core and PyO3 binding.
size: medium
proposed_by: bbugyi200.athena.sase-n8.2
bead: sase-n8.2
create_time: 2026-08-16 11:46:02
status: done
---

- **PROMPT:**
  [prompts/202608/alias_history_core.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/alias_history_core.md)
- **PARENT:** [202608/launch_control_alias_history.md](launch_control_alias_history.md)
- **BEAD:**
  [sase-n8.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n8/sase-n8.2.md)

# Plan: Rust core alias-history projection and query

## Goal

Complete phase bead `sase-n8.2` in the linked `sase-core` repository by extending the
agent-scan wire with launch-time alias provenance, migrating the artifact index to
schema 22 with a durable per-alias projection, and exporting a bounded alias-history
query through PyO3. Preserve the epic design's exact request ordering, legacy first-hop
fallback, truncation accounting, freshness behavior, prompt-snippet normalization, and
soft handling of missing prompt files.

## Implementation

1. Extend the scanner wire and marker conversion paths.
   - Add `model_alias_trail: Vec<String>` and `model_alias_origin: Option<String>`
     beside `model_alias` on both `AgentMetaWire` and `PromptStepMarkerWire`, with serde
     defaults that keep legacy records compatible.
   - Parse both fields from `agent_meta.json` and prompt-step markers, and add scanner
     and wire round-trip coverage proving that valid values survive while absent values
     default to an empty trail and no origin.

2. Add the schema-22 normalized projection.
   - Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` to 22, add `model_alias_origin` to
     `agent_artifacts`, create
     `agent_artifact_model_aliases(artifact_dir, alias, position)` plus its lookup
     index, and expose its row count through `AgentArtifactIndexStatusWire`.
   - Derive a record's effective trail from `agent_meta.model_alias_trail`, falling back
     to the legacy one-element `agent_meta.model_alias` only when the trail is empty.
     Rebuild the child projection with delete-then-insert on every upsert and clear it
     on explicit artifact deletion.
   - Add a v21-to-v22 record-JSON re-projection migration that performs no filesystem
     reads, backfills every decodable historical record through the same projection
     helper, and skips malformed legacy payloads without preventing index open.

3. Implement the frontend-neutral alias-history query contract.
   - Add the query/result/group/limit/run wire types and schema-version constant from
     the epic design, including cached-by-default freshness, required non-empty aliases,
     per-alias zero-means-unlimited limits, exact ProjectSpec-key filtering, and all
     pinned run metadata.
   - For each requested alias, count all matching visible rows, select the newest
     bounded window through the normalized table, retain request order including empty
     groups, and report the matching trail position and exact truncation counts.
   - On `revalidate`, refresh only candidate artifact rows through the established
     marker-signature/scanner path before producing the final count/window. Decode
     canonical `record_json` for fields that are not denormalized while treating a bad
     row as non-fatal.
   - Read `raw_xprompt.md` only for returned rows and within the requested byte budget;
     strip leading blank/directive/xprompt lines, collapse whitespace, truncate only on
     UTF-8 character boundaries, add an ellipsis when content was truncated, return an
     empty string for directive-only prompts, and return `None` for unreadable files.

4. Export and bind the contract.
   - Re-export the new wires, constants, and query through `agent_scan` and the crate
     root.
   - Add the `query_agent_alias_history(index_path, query)` PyO3 function, register it
     in the extension module, document it with the nearby artifact-index bindings, and
     verify Python dictionaries deserialize and serialize through the exact wire shape.

## Verification

- Add focused Rust tests for scanner round-tripping; single and multi-alias request
  order (including an empty group); newest-first ordering and per-group truncation;
  non-entry trail membership and `alias_position`; hidden and project filters; legacy
  first-hop fallback; replace/delete projection lifecycle; pure v21-to-v22 migration;
  directive stripping, whitespace collapse, Unicode-safe truncation, directive-only and
  missing prompt cases; empty-alias rejection; cached/revalidate behavior; and the new
  status counter.
- Add a PyO3 binding test covering query deserialization and result serialization.
- Run `just check` from the linked `sase-core` repository root and resolve all failures
  before closing only `sase-n8.2` with a verification note.
