---
tier: tale
title: Land core attachment audience salvage
goal:
  Reapply the verified sase-core attachment audience, scanner, and public object layout
  onto current master, re-verify it, and close only phase bead sase-1d5.1.
size: medium
proposed_by: bbugyi200.athena.sase-1d5.1
bead: sase-1d5.1
create_time: 2026-09-30 07:50:27
status: wip
---

- **PARENT:**
  [202609/public_bead_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/public_bead_attachments.md)
- **BEAD:**
  [sase-1d5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d5/sase-1d5.1.md)

# Plan: Land core attachment audience salvage

## Outcome

Phase `sase-1d5.1` lands in the linked `sase-core` checkout: optional attachment
visibility (absent means private), the ordered audience decision, the streaming secret
scanner, the canonical MIME extension and public object layout, and roster/reference
visibility. Only that phase bead closes.

## Why this is a salvage

The phase was implemented and verified once, then the host commit failed
(`missing_bead_action`) before any commit reached `origin/master`. Bead note #2 holds
the unlanded patch. Note #3 reopened the bead after the host fix landed. Apply that
patch. Repair it only where current master disagrees. Leave the accepted design alone.

Design of record, already read this epic:

- `plan:202609/core_attachment_audience.md` — phase implementation plan.
- `plan:202609/public_bead_attachments.md` — Wire, Decision table, Scanner, Canonical
  extension, and Phase 1 (`core_audience`). Research input is
  `research:202609/bead_attachment_audience/bead_attachment_audience.md` §4.2–4.3. Read
  those with `sase artifact read` if a conflict forces a repair.

Patch of record:

- Bead attachment `sase-1d5.1-core.patch` (sha256 prefix `6dde48591102`).
- View path used at planning time:
  `/home/bryan/.sase/attachments/views/6dde485911020c95/sase-1d5.1-core.patch` (111224
  bytes). If that view is gone, take the attachment from `sase bead read sase-1d5.1`.
- Authored against sase-core `a354a8a96c06443fb2ed47b5699e9ae2142862d7`.
- Planning-time `sase-core` `HEAD` was `cfc6385b8a86e9e625c6e85fba37cb16fec86c8f`
  (`origin/master`, one later commit that does not touch the patched paths).
  `git apply --check --3way` exited 0 and left the tree clean. Re-check on the tree you
  actually have; master may have moved.

## Scope

Open the checkout with
`sase repo open sase-core -r "Apply the unlanded core attachment audience patch"` and
follow the `AGENTS.md` path it prints.

In scope, all inside `sase-core`:

- `crates/sase_core/src/note_attachment/` — `audience.rs`, `scanner.rs`, `zones.rs`,
  `public_objects.rs`, facade exports, and `tests/`.
- `crates/sase_core/src/bead/attachments.rs` plus the attachment, note, close, and +1
  tests the patch updates.
- `crates/sase_core/Cargo.toml` and `Cargo.lock`: direct `aho-corasick` dependency only,
  as the patch adds it.
- `crates/sase_core_py/src/note_attachment/` bindings and round-trip tests. Roster and
  reference bindings must serialize the new visibility and source fields.

Out of scope for this phase. Record anything discovered here as `PROPOSED FOLLOW-UP:` on
`sase-1d5.1`. Do not create beads, and do not edit these trees:

- The `sase` Python repository, its `sase-core-revision.txt` pin, memory files, feature
  flags, stores, CLI, pages, and TUI. Later phases own those. The pin moves in
  `audience_cli`.
- `sase-github` and every other linked repo.
- Versions and `CHANGELOG.md` (release-plz owns them).
- Parent epic `sase-1d5` and every ancestor plan bead. Closing this assigned phase is
  allowed. Closing an ancestor is not.

## Implementation

1. Read `AGENTS.md` in the opened checkout before editing. Free functions over `*Wire`
   structs, `thiserror` errors, no `macro_rules!`, `mod.rs` is a facade, new files stay
   at or under 1,500 lines, import by module path, never add root `pub use` names or
   `core_*` prelude aliases. Never run bare `cargo` or `just check-full`. Commit
   subjects are Conventional Commits. The intended subject is
   `feat(attachments): core attachment audience policy and scanner`. The host commits
   when the turn ends; do not commit by hand.

2. Confirm the sase-core worktree is clean and tracks `origin/master`. Apply:

   ```bash
   git apply --3way /home/bryan/.sase/attachments/views/6dde485911020c95/sase-1d5.1-core.patch
   ```

   Use the live attachment path if the view path above is missing.

3. When the apply is clean, keep the patch's choices. The stable +1 identifier is
   `plus-one:<reporter>` via `plus_one_note_key`, with
   `BeadAttachmentSourceWire::{Note, PlusOne}` serialized as `note` and `plus_one`.
   `BEAD_EVENT_SCHEMA_VERSION` stays `1`.

4. When the apply conflicts, resolve toward the contract below and the patch's behavior.
   Keep both sides' unrelated master edits. Re-run the decision, scanner, and binding
   tests that cover the conflicted code.

5. Tests use synthetic secrets assembled at runtime by concatenation, and injected env
   maps. A committed file must not contain a realistic secret literal. The scanner never
   returns, logs, or serializes a matched value or the surrounding line. No test uses
   the network.

## Contract the applied tree must satisfy

### Wire and queries

- `AttachmentVisibilityWire { Public, Private }`, snake_case serde, with a
  `#[serde(other)]` fallback callers treat as private.
- `BeadNoteAttachmentWire.visibility` is `Option<AttachmentVisibilityWire>`, defaulted
  and omitted when `None`.
- `effective_visibility()` returns private for `None` and for the fallback.
- `BeadAttachmentRosterEntryWire` and `BeadAttachmentReferenceWire` carry effective
  visibility and a note-versus-+1 source.
- `bead_attachment_roster` and `bead_attachment_references` include initial
  `IssueCreated` notes, appended and edited notes, and `TaskPlusOneRecorded` evidence.
  Current-versus-historical pinning stays as it is today. +1 evidence has no note id;
  use `plus-one:<reporter>` rather than inventing a mutable note.
- `validate()` and manifest/token matching stay unchanged. An old descriptor with no
  visibility field, and an unknown visibility value, both reduce to private. A
  visibility-bearing event still passes the unknown-field tolerance test.

### Audience decision

`attachment_audience_decision` is a pure first-match over `AttachmentAudienceFactsWire`.
Python gathers facts later; this phase only implements the core function and the SASE
zone and secret-file tables, keyed off `home` and `sase_home`.

Facts include `bead_store_visibility`, `requested`
(`auto | public | private | local_only`), `actor` (`human | agent`), `confirmed`,
`allow_sensitive`, `path` (`None` for stdin), `home`, `sase_home`,
`extra_sensitive_patterns`, `size_bytes`, `public_max_bytes`, `class`, optional `scan`,
`owner_only`, optional `checkout` (`root`, `remote_visibility` of
`public|private|unknown`, `ignored`, `tracked_identical_to_remote`), `workspace_root`,
`scratch_roots`, and `produced_during_run` (`None` when there is no run window).

The result is `{outcome, rule, reason, widenable}` with outcome
`public | private | local_only | refuse | confirm`.

| #   | rule                                                                                                                                                                                                                                                                                                               | Policy result                                                             | Human may widen           |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- | ------------------------- |
| 0   | `store_private` — bead store is private                                                                                                                                                                                                                                                                            | private                                                                   | no                        |
| 1   | `sensitive_path` — core denylist, SASE secret-file table (`<sase_home>/telegram_bot_token`, gateway and fleet credentials, and the real paths found in sase), plus `extra_sensitive_patterns`                                                                                                                      | `refuse`; with `allow_sensitive`, private, or `local_only` when requested | no                        |
| 2   | `explicit` — requested `private` or `local_only`                                                                                                                                                                                                                                                                   | private / local_only                                                      | —                         |
| 3   | `size_cap` — `size_bytes > public_max_bytes`                                                                                                                                                                                                                                                                       | private                                                                   | no                        |
| 4   | `scan_hit` — `known_value`, `credential_pattern`, or `env_dump`                                                                                                                                                                                                                                                    | private                                                                   | yes, except `known_value` |
| 5   | `private_provenance` — `owner_only`; git-ignored; checkout remote private, unknown, or absent; personal/config zone (`~/.config`, `~/.local/share`, `~/Documents`, `~/Downloads`, `~/Desktop`, mail, chat, notes vaults); SASE personal zone (telegram, notifications, mobile gateway, prompt and command history) | private                                                                   | yes                       |
| 6   | `opaque_type` — `archive`, `binary`, `pdf`, `audio`, or content the scanner cannot read                                                                                                                                                                                                                            | private                                                                   | yes                       |
| 7   | `unverified_media` — image or video that is not (`produced_during_run == Some(true)` and under the workspace, a scratch root, or a SASE publishable zone) and is not `tracked_identical_to_remote` in a public checkout                                                                                            | private                                                                   | yes                       |
| 8   | `public_evidence` — tracked and byte-identical to the remote-tracking default branch in a public checkout; or scan-clean text or self-produced media under the workspace, a scratch root, or a SASE publishable zone (logs, tool-run logs, perf traces, TUI screenshots)                                           | public                                                                    | —                         |
| 9   | `fail_private` — anything else, including stdin                                                                                                                                                                                                                                                                    | private                                                                   | yes                       |

Widening runs after the policy result:

- `requested: public` over a policy-private result: `refuse` when `widenable` is false;
  `refuse` for `actor: agent`, with `reason` naming `sase bead attachment publish`;
  `confirm` for an unconfirmed human; `public` once `confirmed`.
- `allow_sensitive` together with `requested: public` is `refuse`.
- `requested: public` together with `local_only` is `refuse`.

A checkout whose remote is public is judged by its checkout facts, including when the
path also sits in a personal zone. Files outside the workspace become public under rule
8 only when `tracked_identical_to_remote` is set. Missing or unknown remote facts and a
missing run window stay private. Reasons stay generic and contain no secret value.

Record the real SASE secret-file and publishable-zone paths in table tests: telegram
token, gateway and fleet credentials, notification and mobile-gateway stores, prompt and
command history, logs, tool-run logs, perf traces, and TUI screenshot directories.

### Scanner

- `attachment_scan_file(path, max_bytes, env, home, sase_home) -> AttachmentScanWire`.
- `ATTACHMENT_SCANNER_RULES_VERSION = 1`, exposed by a binding getter.
- Result:
  `{outcome: clean|hit|skipped, hit?: {kind, rule_id, line}, bytes_scanned, rules_version}`.
  Hit kind is `known_value | credential_pattern | env_dump`.
- Text classes, including SVG, up to `max_bytes`. Oversize, unreadable, and binary
  content return `skipped`. Callers scan the ingested CAS object. The function itself
  takes a path.
- Known values: env names matching `*TOKEN*`, `*SECRET*`, `*_KEY`, `*API*KEY*`,
  `*PASSWORD*`, or `*CREDENTIAL*` (case-insensitive), plus trimmed SASE secret-file
  contents. Keep values of at least 12 characters. Skip absolute paths and purely
  numeric or boolean values. Match with direct `aho-corasick`.
- Credential patterns, high precision: GitHub
  `ghp_`/`gho_`/`ghu_`/`ghs_`/`ghr_`/`github_pat_`; Anthropic and OpenAI `sk-` families;
  Google `AIza`; AWS `AKIA`/`ASIA`; Slack `xox[abprs]-`; Telegram bot tokens; Stripe
  `sk_live_`/`rk_live_`; PEM private-key headers; JWTs; and
  `key|secret|token|password = <value>` where the value has at least 20 characters and
  high entropy. Reuse regexes from `tool_run/triage/normalize.rs` where they fit.
- Env dumps: five or more `NAME=value` lines inside any ten consecutive lines, or a
  JSON/dict object with five or more well-known env keys (`PATH`, `HOME`, `USER`,
  `SHELL`, `PWD`, `LANG`, `TERM`, and the rest of that set). An env dump hits even when
  no secret matches.
- Streaming: detect a value split across read chunks, and bound memory for very long
  lines.

### Public object layout

- `attachment_canonical_extension(mime_type) -> Option<&str>`, pinned by tests.
  Examples: png→`png`, jpeg→`jpg`, gif, webp, svg, mp4, webm, text/plain→`txt`, json,
  markdown→`md`, csv, pdf.
- `attachment_public_object_relpath(sha256, mime_type)` returns
  `files/objects/sha256/<xx>/<sha>.<ext>`, or the extensionless form when the MIME type
  has no canonical extension.
- `attachment_object_digest_from_relpath` accepts the extensionless form and the
  canonical-extension form. Validate digest, shard, and shape. Tombstone paths and the
  private object layout stay as they are.

### Bindings

Register these in `register_note_attachment`, each with a round-trip test:

- `attachment_audience_decision`
- `attachment_scan_file`
- `attachment_scanner_rules_version`
- `attachment_canonical_extension`
- `attachment_public_object_relpath`
- `attachment_object_digest_from_relpath`

Roster and reference bindings pass `visibility` and `source` through.

## Verification

Cover every decision rule, rule precedence, each widening outcome, and SASE path and
zone membership, including a public checkout under a personal-zone directory, stdin, an
unknown remote, no run window, and skipped scans. Cover each scanner detector, a
chunk-straddling secret, long-line bounds, over-cap and binary skips, and the absence of
matched text in scanner output. Pin the extension table and both public and old path
forms.

Inner loop, from the sase-core checkout:

```bash
just test -p sase_core attachment
just test -p sase_core_py note_attachment
```

The previous attempt reported 74 Rust tests and 6 Python binding tests. Treat that as
history. Re-run both.

Then run the required gate in that checkout:

```bash
sase tool run check
```

`just check` takes about five minutes; give the wrapped gate at least ten. Read
`/sase_monitor` before starting it if the gate would outlive the turn, and use
`sase monitor start` rather than a built-in background runner. A targeted `just test`
does not replace the gate. A failure that passes when rerun alone is a load flake until
a `sase-core flake:` bead says otherwise. Do not weaken an assertion to go green.

When the gate fails, compare the failure to a clean `origin/master` tree that does not
contain this patch. A failure that reproduces identically there is pre-existing:
`sase bead note sase-1d5.1 'PROPOSED FOLLOW-UP: <summary — detail>'`, cite any task bead
that already tracks it, and close this phase anyway. Fix failures that exist only with
this patch.

Other out-of-scope findings use the same `PROPOSED FOLLOW-UP:` note. Do not create
beads.

## Closeout

Run `sase bead epic-symbols sase-1d5.1` before closing. Planning time found no
`--epic-symbol` entries. If any exist when you finish, resolve each symbol or re-key its
Justfile line to a still-open bead (parent epic `sase-1d5` or a later phase).
`sase bead close` refuses while leftovers remain.

Close only this phase:

```bash
sase bead close sase-1d5.1 --note "<commands run, pass counts, and the sase tool run id>"
```

The note must name what you re-verified on the tree you applied, including the sase-core
SHA. Leave `sase-1d5` open for its land agent. Do not set bead status by hand.
