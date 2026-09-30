---
tier: epic
title: Public-by-default bead attachments
goal: 'A bead note attachment is as visible as its bead unless SASE or its author
  marks it private. On a public project, non-sensitive attachments publish to a dedicated
  public `<project>--attachments` sidecar that anyone who can read the beads can fetch
  without credentials. SASE classifies every file mechanically and resolves uncertainty
  to private. Agents can only narrow an attachment''s audience; only humans widen
  it. Large files never go public, and `sase--beads` never holds bytes.

  '
phases:
- id: core_audience
  title: Core audience wire, decision table, and secret scanner (sase-core)
  depends_on: []
  size: large
  description: 'core_audience: add the optional descriptor visibility field (absent
    means private), the ordered attachment_audience_decision rule table with SASE
    zone and secret-file tables, the streaming secret scanner, the canonical MIME
    extension table and public object layout, and visibility on the roster and reference
    queries, plus Python bindings and tests that use only synthetic secrets.'
- id: push_protection
  title: Secret scanning on newly created public sidecars (sase-github)
  depends_on: []
  size: small
  description: 'push_protection: honor a new sdd_secret_scanning provider option by
    enabling GitHub secret scanning and push protection, best effort, right after
    sase-github creates a public repo, with tests and docs.'
- id: audience_cli
  title: Provenance facts, audience flags, and the beta flag
  depends_on:
  - core_audience
  size: large
  description: 'audience_cli: create the public_bead_attachments beta flag, gather
    provenance facts in Python (stat mode, git status, anonymous remote-visibility
    probe with cache, zones, run window, actor), scan ingested bytes, run the core
    decision, and add -K/--private, -W/--public, and -y/--yes to the five note-bearing
    verbs. Covers the agent-widening refusal, human confirmation, duplicate-digest
    intent, local-only reasons, the 🌐/🔒 write echo, the public_max_bytes config, and
    a count-only scanner run over the published transcripts.'
- id: public_store
  title: Public attachments sidecar, routing, and anonymous reads
  depends_on:
  - audience_cli
  size: large
  description: 'public_store: spike the page-embed path, then add the reserved hidden
    public attachments role (injected only for public beads, with PUBLIC default-yes
    consent and a beads-visibility guard). Covers the HTTPS-fetch/SSH-push bare clone
    with read-side auto-materialization, a role-parameterized GitAttachmentStore with
    extension-preserving layout, placement and outbox by audience with no cross-audience
    fallback, GH013 handling, fixes to the fetch-miss and lazy-discovery bugs, and
    purge and doctor coverage.'
- id: publish_lifecycle
  title: Publish, unpublish, and audience-aware doctor
  depends_on:
  - public_store
  size: medium
  description: 'publish_lifecycle: add the human-only (gate-compatible) sase bead
    attachment publish and the narrowing unpublish, both editing manifests through
    NoteEdited. Add doctor rescans keyed to the scanner rules version, store growth,
    push access, and a private-store-is-anonymously-readable finding.'
- id: presentation
  title: Audience badges, access states, and bead-page embeds
  depends_on:
  - public_store
  size: medium
  description: 'presentation: add 🌐/🔒 audience badges in show, read, history, and
    attachment list plus JSON visibility. Add the 🔒 no access, ⧉ on origin with dispatch
    hint, and ⛔ blocked states. Bead pages link public files, embed public images,
    render tokens as chips, and never show private digests or reasons.'
- id: tui
  title: TUI audience chips, add-note toggle, and queued uploads
  depends_on:
  - presentation
  size: medium
  description: 'tui: show 🌐/🔒 on beads-pane chips. In the add-note modal, run the
    audience decision off the event loop and add a toggle that narrows freely and
    widens only with confirmation. Queue TUI-authored attachments into the upload
    outbox instead of stranding them locally.'
- id: ga
  title: Remove the beta flag, finish docs, and agent guidance
  depends_on:
  - push_protection
  - publish_lifecycle
  - presentation
  - tui
  size: medium
  description: 'ga: remove public_bead_attachments by deleting its off branches and
    closing its flag bead. Rewrite the attachment docs around the audience model,
    put the agent visibility guidance into attach and note help, bead onboard, and
    the sase_new_task skill source, fix stale GA-era help text, and record the proposed
    follow-ups.'
proposed_by: bbugyi200.athena.0tz
create_time: 2026-09-30 01:57:05
status: wip
bead_id: sase-1d5
---

- **PROMPT:** [prompts/202609/public_bead_attachments.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/public_bead_attachments.md)
- **BEAD:** [sase-1d5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d5/README.md)

# Plan: Public-by-default bead attachments

## Background

Source research: `research:202609/bead_attachment_audience/bead_attachment_audience.md`.
Every phase worker reads it with `sase artifact read` before starting. §4 (deciding what
is sensitive), §5 (stores, wire, lifecycle), §6 (large files), §7 (requirement
adjustments), and §9 (recommended solution and acceptance) are the design inputs. This
plan implements §9's follow-up epic. Where it deviates from the report, it says so and
why (see "Settled design decisions").

Epic `sase-1ck` (bead note attachments) is **closed and GA**. Its design of record is
`plan:202609/bead_note_attachments.md`. The report's "do this now" item for `sase-1ck.5`
has landed: the private store's role is `attachments-private`, so the plain
`attachments` role name is free.

Facts verified while planning (paths are repo-relative; sase-core paths are relative to
the checkout that `sase repo open sase-core` prints):

- **Wire.** `BeadNoteAttachmentWire` in
  `crates/sase_core/src/note_attachment/manifest.rs` has
  `{name, sha256, size_bytes, mime_type, image?, origin?}` and no audience field. The
  bead and attachment wires have no `deny_unknown_fields`, so old readers ignore new
  optional fields (test `note_payloads_ignore_unknown_future_fields`).
  `BEAD_EVENT_SCHEMA_VERSION` is `1` (`bead/events/wire.rs`), and
  `event_schema_version_stays_one` pins it. `NoteEdited.attachments` is
  `Option<Vec<…>>`: `None` keeps the manifest, and `Some` replaces it.
- **Core policy.** `attachment_placement(size, tiers, local_only)` picks the first
  opaque tier whose cap fits. `attachment_sensitive_path_reason` holds a path denylist
  (`~/.ssh/**`, `.env*`, `*.pem`, …) that misses SASE's own `telegram_bot_token`.
  Nothing scans content. `regex` is a direct dependency; `aho-corasick` is only a
  transitive one (through `regex`). The only MIME table is the one-way
  `extension_mime_for`, and there is no canonical MIME→extension function.
- **Queries.** `bead_attachment_roster` and `bead_attachment_references`
  (`bead/attachments.rs`) look only at notes. They skip +1 evidence attachments and
  notes embedded in `IssueCreated`.
- **Python stores.** `GitAttachmentStore(repo, label)`
  (`src/sase/bead/attachments/git_store/`) doesn't care about roles, but everything
  around it assumes one private git store:
  - `name` is always `"git"`;
  - `describe_label` appends `(private)`, and the echo appends it a second time
    (`upload.py`), so the echo reads `(private) (private)`;
  - `hidden_clone_path(project_key)` always returns the `attachments-private` clone;
  - `discover_stores` returns `{"git", "large"}`;
  - purge loops over `("git", "large")`;
  - the outbox dedups by digest only;
  - the sync-worker drain (`bead/_sync_worker_run.py`) and the doctor
    (`doctor/checks_attachment_store.py`) each build their own private store.
- **Layout.** Objects live at `files/objects/sha256/<xx>/<sha>` with no extension,
  through core's `artifact_object_relpath`. GitHub raw serves an extensionless object as
  `text/plain` with `nosniff` (report §1), so bead pages can't embed it.
- **Private store lifecycle.** `attachments-private` is a reserved, hidden sidecar role.
  Visibility is forced to `private` (`_linked_repo_config.py`), and only
  `sase repo init` creates it, after consent (`main/_repo_init_sidecars.py`,
  `sdd/_sidecar_init.py`, `sdd/_sidecar_bare.py`). The bare-clone helper refuses HTTP(S)
  remotes. `inject_default_linked_repos` already skips a hidden role whose role or slug
  collides with a configured sidecar. At write time a missing clone means `no_store`,
  and the object stays local.
- **Provider.** sase-github 0.2.18 honors `sdd_visibility` (`sdd_sidecar_visibility`,
  `create_github_sdd_repo(..., visibility=)`), so sase can already create either a
  public or a private sidecar.
- **Remote visibility.** Nothing in sase decides whether a git remote is public.
  Visibility comes only from config (`repos.sidecar.*.<role>.visibility`, default
  `public`). The pattern for reading one role's configured visibility is
  `_agents_sidecar_metadata` in `core/prompt_artifact_staging.py`.
- **Fetch bugs that a second store would expose** (`bead/attachments/fetch.py`):
  - `ensure_fetched` treats every non-transient error as corruption, and a plain miss in
    `GitAttachmentStore.get` or the rclone store is non-transient. Probing the public
    store first for a private-only object would therefore render `‼ digest mismatch`.
  - `_ensure_discovered` copies `store` and `outbox` but not `stores`, so lazy discovery
    never probes the large tier.
- **Pages.**
  - `bead_pages/rendering_identity.py` always renders `🔒 … (private attachment)`.
  - The `## Notes` section prints raw `@attachment:<name>` tokens.
  - +1 evidence on pages shows `sha256:<12>`.
  - `HostedLinkResolver` (`sdd/hosted_links.py`) already builds GitHub blob URLs for the
    plans, beads, and agents sidecars.
- **TUI.** The add-note path (`ace/tui/actions/_artifacts_beads_mutations.py`) never
  uploads or queues attachments.
- **Agent detection.** The `goals/cli.py` `_in_agent_run` pattern checks `SASE_AGENT`,
  `SASE_AGENT_NAME`, or a discovered identity, and fails closed.
- **Beta flag precedent.** Feature flags live in `src/sase/feature_flags/registry.py`
  and are checked with `current_flags().enabled(FeatureFlag.<key>)`. Flag beads are
  created only with `sase flag new`. `sase-1ck.4` created `bead_note_attachments` that
  way, and `sase-1ck.10` removed it.
- **Agent guidance.** No beads skill exists; beads guidance lives in reference memory.
  The `sase_new_task` skill source (`src/sase/xprompts/skills/sase_new_task.md`) already
  shows `sase bead +1 … -n "… @./repro.png"`.

## Human prerequisites (not agent work)

1. **Rotate the credentials the report found** (report TL;DR 4). No agent in this epic
   prints, greps for, quotes, or files beads about specific leaked values. The only
   corpus work is the count-only verification in `audience_cli`.
2. **Enabling secret scanning and push protection on the existing public sidecars** is
   the user's call. Beware: the agents-sidecar publisher has no GH013 handling yet, so
   push protection on `sase--agents` could stall transcript publication. This epic
   enables it only on the new attachments repo.
3. **After `ga` lands,** run `sase repo init` interactively on one machine to create the
   public `<owner>/sase--attachments` repo. Phase workers never create real GitHub
   repositories.

## Settled design decisions

1. **Visibility follows the bead.** `public` means readable by anyone who can read the
   bead store. The public role exists only when the configured beads sidecar visibility
   is `public`. In a project with private beads, every attachment is private (rule 0),
   and no `attachments` role is injected. That way `public` never secretly means
   "private".
2. **Bytes never enter `sase--beads`.** Public bytes live in a dedicated hidden sidecar,
   `<project>--attachments` (role `attachments`), held as a bare `--filter=blob:none`
   clone. Report §2.2 explains why.
3. **SASE decides in core, and uncertainty resolves to private.** The ordered rule table
   below is one pure core function (`rust_core_backend_boundary`). Python only gathers
   facts.
4. **Only humans widen.**
   - An agent's `--public` over a policy-private result is refused with the reason and
     the exact `sase bead attachment publish` command to offer through `/sase_gate`.
   - A human confirms on a TTY; `-y` skips the confirmation.
   - Three outcomes can never be widened: sensitive paths, known-secret-value hits, and
     over-cap sizes. A private bead store can never be widened either.
5. **The wire is additive.** `visibility` is optional on the descriptor, and absent
   means private, so every `sase-1ck`-era attachment stays private unless someone
   explicitly publishes it. `BEAD_EVENT_SCHEMA_VERSION` stays `1`. Unknown future values
   deserialize as private.
6. **Visibility is intent, not a locator.** Writers route by it. Store order, object
   presence, dedup, and outbox retries never change it. Readers try every store.
7. **Reasons stay local.** They appear in the echo, local JSON, and local CAS audience
   metadata. They never appear in descriptors, pages, commit messages, or bead text.
8. **The public cap is 25 MiB** (`bead.attachments.public_max_bytes`, which equals
   `auto_fetch_max_bytes`). Anything larger stays on the private tiers. v1 adds no new
   large-file transport: origin-only objects show `⧉ on <origin>` and a `%dispatch`
   hint.
9. **Remote visibility comes from an anonymous probe**, not from `gh`. A remote is
   public iff an unauthenticated smart-HTTP
   `GET <https-url>/info/refs?service=git-upload-pack` succeeds. The probe runs with no
   credentials, no netrc, a bounded timeout, and a cache. That is exactly the definition
   of public, it works for any smart-HTTP host, and it needs no provider changes.
   Unmappable hosts (such as SSH aliases), network errors, and remotes that aren't
   GitHub-style all resolve to `unknown`, which means private.
10. **Deviation: private descriptor redaction (report §5.3) is deferred.** Old readers
    require `sha256`, so redaction needs a mixed-fleet reader gate and is orthogonal to
    public-by-default. `ga` proposes it as a follow-up. Until then, docs and `--help`
    say plainly that **private protects bytes, not names or prose**.
11. **Deviation: the private store keeps today's creation path** (`sase repo init` with
    consent). The report's "created lazily on first `--private`" is not adopted, because
    lazy creation would need consent in non-TTY agent runs.
12. **Deviation: unpublish removes the object from the public tree with a withdrawal
    commit, not a purge tombstone.** Today a tombstone in any store stops every fetch,
    so a public tombstone would hide the private copy.
13. **Deviation: publish and unpublish handle note attachments only.** +1 evidence
    manifests are immutable, and adding a new event operation would break old readers.
    Evidence gets its audience from the decision at `+1` time.
14. **Refinement of report rule 8a:** a tracked file counts as already public only when
    its bytes equal the blob at the checkout's remote-tracking default branch, not just
    `HEAD`. Unpushed commits are not public.
15. **Human invocations have no run window.** "Self-produced media" (rule 7/8) needs an
    agent run's start time. A human's screenshot therefore resolves private, and the
    human can widen it with `--public`.
16. **Push protection is a backstop, not the classifier.** sase passes
    `sdd_secret_scanning: true` only when it creates the `attachments` role. A GH013
    rejection is a permanent, visible failure.
17. **No memory files change in this epic.** Agent guidance goes into CLI help,
    `sase bead onboard`, and the `sase_new_task` skill source. `ga` proposes a memory
    task for a pointer in the beads memory template.
18. **Beta flag `public_bead_attachments`, created by `audience_cli` with
    `sase flag new`.** It gates only authoring-time decisions: the audience decision,
    `-W/--public`, and the TUI toggle. It also gates `sase repo init` offering to create
    the public repo. Rendering, discovery, reads, read-side clone materialization, and
    draining already-queued uploads are never gated. `ga` removes the flag.

## Shared specification (all phases conform)

### Terms

| Term              | Meaning                                                                                                                                                          |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| public            | Readable by anyone who can read the bead store; for `sase`, the world. Treat it as irreversible publication.                                                     |
| private           | Readable only by principals on `<project>--attachments-private`. Not encrypted, and not "author only".                                                           |
| local-only (`-L`) | An availability state: bytes stay on this machine. It implies audience `private`, and `sase bead attachment push` later promotes the bytes to the private store. |

### Wire (sase-core)

- Add `enum AttachmentVisibilityWire { Public, Private }`, serialized snake_case, with a
  `#[serde(other)]` unit fallback that callers treat as `Private`.
- Add
  `#[serde(default, skip_serializing_if = "Option::is_none")] visibility: Option<AttachmentVisibilityWire>`
  to `BeadNoteAttachmentWire`. Add a helper `effective_visibility()` that returns
  `Private` for `None` and for the fallback.
- Writers under the flag always write an explicit value. The flag-off branch writes
  none, exactly as today.
- Add `visibility` to `BeadAttachmentRosterEntryWire` and `BeadAttachmentReferenceWire`.
  Extend both queries to cover +1 evidence attachments and `IssueCreated` notes. Mark
  each reference with its source: note or +1.
- `validate()` and the manifest/token one-to-one rule stay unchanged. Add a test that
  shows an old-shaped descriptor (no field) reduces as private and that a
  `visibility`-bearing event still passes the unknown-field tolerance test.

### Decision table (core `attachment_audience_decision`)

Python supplies an `AttachmentAudienceFactsWire` with these fields:

- `bead_store_visibility`
- `requested`: `auto | public | private | local_only`
- `actor`: `human | agent`
- `confirmed`
- `allow_sensitive`
- `path`: the resolved source path, or `None` for stdin
- `home` and `sase_home`
- `extra_sensitive_patterns`
- `size_bytes` and `public_max_bytes`
- `class`: the `AttachmentClassWire`
- `scan`: an optional `AttachmentScanWire`
- `owner_only`
- `checkout`: optional
  `{root, remote_visibility: public|private|unknown, ignored, tracked_identical_to_remote}`
- `workspace_root`
- `scratch_roots`
- `produced_during_run`: `Option<bool>`, which is `None` without a run window

Core returns an `AttachmentAudienceDecisionWire` of
`{outcome: public|private|local_only|refuse|confirm, rule, reason, widenable}`. The
first matching rule wins:

| #   | `rule`               | Signal                                                                                                                                                                                                                                                                                                                               | Policy result                                                         | Human may widen           |
| --- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- | ------------------------- |
| 0   | `store_private`      | The bead store is private.                                                                                                                                                                                                                                                                                                           | private                                                               | no                        |
| 1   | `sensitive_path`     | Core denylist, **plus core's SASE secret-file table** (for example `<sase_home>/telegram_bot_token`, gateway and fleet credentials), plus `extra_sensitive_patterns`.                                                                                                                                                                | `refuse`; with `allow_sensitive`: private (or `local_only` with `-L`) | no                        |
| 2   | `explicit`           | `requested` is `private` or `local_only`.                                                                                                                                                                                                                                                                                            | private / local_only                                                  | —                         |
| 3   | `size_cap`           | `size_bytes > public_max_bytes`                                                                                                                                                                                                                                                                                                      | private                                                               | no                        |
| 4   | `scan_hit`           | Scan hit: `known_value`, `credential_pattern`, or `env_dump`.                                                                                                                                                                                                                                                                        | private                                                               | yes, except `known_value` |
| 5   | `private_provenance` | Any of: `owner_only`; git-ignored; in a checkout whose remote is private, unknown, or absent; in a personal/config zone (`~/.config`, `~/.local/share`, `~/Documents`, `~/Downloads`, `~/Desktop`, mail, chat, notes vaults); in a SASE personal-content zone (telegram, notifications, mobile gateway, prompt and command history). | private                                                               | yes                       |
| 6   | `opaque_type`        | Class `archive`, `binary`, `pdf`, or `audio`, or any content the scanner can't read.                                                                                                                                                                                                                                                 | private                                                               | yes                       |
| 7   | `unverified_media`   | Image or video that is not both (`produced_during_run == Some(true)` and under the workspace, a scratch root, or a SASE publishable zone) **and** not `tracked_identical_to_remote` in a public checkout.                                                                                                                            | private                                                               | yes                       |
| 8   | `public_evidence`    | Tracked and identical to the remote-tracking blob in a public checkout; or scan-clean text or self-produced media under the workspace, a scratch root, or a SASE publishable zone (logs, tool-run logs, perf traces, TUI screenshots).                                                                                               | public                                                                | —                         |
| 9   | `fail_private`       | Anything else, including stdin.                                                                                                                                                                                                                                                                                                      | private                                                               | yes                       |

Widening:

- `requested: public` over a policy-private result:
  - `refuse` when `widenable` is false;
  - otherwise `refuse` for `actor: agent`, with `reason` naming the publish command;
  - otherwise `confirm` for a human who hasn't confirmed, and `public` once `confirmed`.
- `allow_sensitive` together with `requested: public` is always `refuse`.
- `requested: public` together with `local_only` is always `refuse`.

Zone membership in rule 5 does not apply to a file inside a checkout whose remote is
public. For example, a public dotfiles checkout under `~/.local/share` is judged only by
its checkout facts. That is safe, because rule 8 publishes files from a non-workspace
checkout only when their bytes are already public (`tracked_identical_to_remote`).

Core owns every zone and secret-file table, stored relative to `home` and `sase_home`.
`core_audience` finds the real SASE paths by reading the sase code: the telegram token
file, gateway and fleet credential files, notification and mobile-gateway stores, prompt
and command history files, and the logs, tool-run log, perf trace, and screenshot output
directories. It records them in table tests. Python can skip the network probe when a
pre-decision with `remote_visibility: public` already comes out non-public: unknown can
only make the result more private, so the probe could not change the outcome.

### Scanner (core, streaming)

- **Signature:**
  `attachment_scan_file(path, max_bytes, env, home, sase_home) -> AttachmentScanWire`.
- **Result:**
  `{outcome: clean|hit|skipped, hit?: {kind: known_value|credential_pattern|env_dump, rule_id, line}, bytes_scanned, rules_version}`.
  `ATTACHMENT_SCANNER_RULES_VERSION` starts at `1` and gets a binding getter.
- **Scope:** text-class content, including SVG, up to `max_bytes`. Anything larger, or
  unreadable, returns `skipped`.
- **What it scans:** the ingested CAS object, never the mutable source path.
- **Detector 1, known values:**
  - Take the values of env vars whose names match (case-insensitively) `*TOKEN*`,
    `*SECRET*`, `*_KEY`, `*API*KEY*`, `*PASSWORD*`, or `*CREDENTIAL*`. Add the trimmed
    contents of the files in the SASE secret-file table.
  - Keep only values of 12 or more characters. Skip absolute paths and values that are
    purely numeric or boolean.
  - Match them with Aho-Corasick. Add `aho-corasick` as a direct dependency; it is
    already in the lockfile.
  - Values live in memory only. They are never returned, logged, or serialized, and a
    hit reports only its kind, rule id, and line.
- **Detector 2, credential patterns:** a high-precision subset of the MIT-licensed
  gitleaks rules:
  - GitHub (`ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`, `github_pat_`);
  - Anthropic and OpenAI `sk-…` families;
  - Google `AIza…`;
  - AWS `AKIA`/`ASIA`;
  - Slack `xox[abprs]-`;
  - Telegram bot tokens;
  - Stripe `sk_live_`/`rk_live_`;
  - PEM private-key headers;
  - JWTs;
  - `key|secret|token|password = <value>` where the value has 20 or more characters and
    high entropy.
  - `tool_run/triage/normalize.rs` has reusable regexes.
- **Detector 3, env dumps:** five or more `NAME=value` lines within any ten consecutive
  lines, or a JSON/dict object with five or more well-known env keys (`PATH`, `HOME`,
  `USER`, `SHELL`, `PWD`, `LANG`, `TERM`, …). An env dump is a hit even with no secret
  match.
- **Streaming:** handle matches that straddle chunk boundaries, and bound the memory
  used by very long lines. A test must find a secret split across a chunk boundary.

### Canonical extension and public layout (core)

- `attachment_canonical_extension(mime_type) -> Option<&str>` is a stable reverse table
  pinned by tests. Examples: png→`png`, jpeg→`jpg`, gif, webp, svg, mp4, webm,
  text/plain→`txt`, json, markdown→`md`, csv, pdf.
- `attachment_public_object_relpath(sha256, mime_type)` returns
  `files/objects/sha256/<xx>/<sha>.<ext>`, or the extensionless form when the MIME type
  has no canonical extension.
- `attachment_object_digest_from_relpath` accepts both forms.
- Tombstone paths don't change.

### Provenance facts (Python, `audience_cli`)

- **`owner_only`:** `st_mode & 0o044 == 0` on the resolved source.
- **`checkout`:**
  - `root` is from `git rev-parse --show-toplevel`.
  - `ignored` is from `git check-ignore -q`.
  - `tracked_identical_to_remote` is true only when `git hash-object <file>` equals
    `git rev-parse <remote-tracking default branch>:<relpath>`.
  - Any git failure yields `false` or `unknown`.
- **`remote_visibility`:**
  - Take the origin URL and derive its HTTPS form through
    `sase._git_remote.parse_hosted_git_remote`.
  - Probe with an unauthenticated urllib GET: no auth handler, no netrc, 3 s timeout. A
    `200` whose content type is `application/x-git-upload-pack-advertisement` means
    public. `401`, `403`, or `404` means private. Anything else means unknown.
  - Cache results under the SASE home: public for 24 h, private for 1 h, and unknown not
    at all.
  - Tests inject the prober; no test touches the network.
- **`workspace_root`:** the checkout root of the invocation cwd.
- **`scratch_roots`:** SASE managed-tmp roots, plus the run's managed `TMPDIR` when SASE
  set it.
- **`produced_during_run`:** in an agent run, source mtime ≥ the run's start time (find
  the start time in agent run metadata); `None` otherwise.
- **`actor`:** one shared helper modeled on `goals/cli.py` `_in_agent_run`, failing
  closed to `agent`.
- **`bead_store_visibility`:** the configured beads sidecar visibility (merged sidecar
  entries, default `public`). Non-sidecar (legacy) bead storage counts as private.
- **Cost:** none of this runs unless the invocation actually attaches files. Import it
  lazily so beads without attachments do no new work.

### Stores and routing (`public_store`)

| Audience   | Placement tiers (core `attachment_placement`) | Store name / outbox `store` |
| ---------- | --------------------------------------------- | --------------------------- |
| public     | `[public ≤ public_max_bytes]`                 | `public`                    |
| private    | `[git ≤ git_max_bytes, large]` (unchanged)    | `git`, `large`              |
| local_only | none (local CAS only)                         | —                           |

- Placement never falls back across audiences.
- **Missing public store:** the public item is queued in the outbox with
  `⇡ pending upload` and a hint to run `sase repo init`. It drains automatically once
  the store exists.
- **Missing private store:** unchanged (`stayed local on this machine`).
- **Outbox entries** are keyed by `(digest, store)`. Old entries stay valid.

### CLI surface (`audience_cli`)

The five note-bearing verbs are `note`, `close -n`, `update -n`, `+1 -n`, and `attach`.
Each gains:

- `-K/--private` — force private.
- `-W/--public` — request public.
- `-y/--yes` — confirm a human widening without a prompt.

All three letters are free on all five verbs. `-K` and `-W` are mutually exclusive. `-W`
is also an error together with `-L` or `-S`, while `-K` with `-L` is fine. Sort help
alphabetically (`cli_rules`). The Rust fast path (`main/bead_fast_path.py`) must fall
through to Python whenever any of these flags appears. When a flag is given but nothing
is attached, print a dim warning and ignore it. The flags apply to every attachment in
the invocation; for per-file control, attach files in separate commands.

Echo row (stderr), replacing today's `(private) (private)`:

```text
attached build.log · text/plain · 88 KiB · 🌐 public → sase-org/sase--attachments · 0.4s
attached env.txt · text/plain · 2 KiB · 🔒 private (env dump) → sase-org/sase--attachments-private
attached shot.png · image/png · 1280x720 · 210 KiB · 🔒 private (image not produced in this run) · stayed local on this machine
```

Agent refusal (exit 1, nothing written):

```text
refusing --public for shot.png: SASE classified it private (image not produced in this run).
Re-run without --public; it will be stored privately. A human can publish it later:
  sase bead attachment publish sase-ab shot.png
Offer that command to the user with /sase_gate.
```

### Local audience metadata

Each ingested object that was decided under the flag gets a local JSON file,
`<cas>/audience/sha256/<xx>/<sha>.json`:

```json
{
  "rule": "…",
  "reason": "…",
  "explicit": false,
  "scanner_rules_version": 1,
  "decided_at": "…"
}
```

It is never published. `publish` reads it to refuse `sensitive_path`/`known_value`, and
doctor rescans read it. **Duplicate intent:** when any existing descriptor in the bead
store with the same `sha256` is private (checked with the extended references query), an
`auto` decision stays private and warns.

### Testing rules (every phase)

- **Synthetic secrets only.** Build token-shaped strings at runtime by concatenation, so
  no committed file contains a literal that our scanner, gitleaks, or GitHub push
  protection would flag. Pass explicit env maps; never read the real environment in
  tests.
- **No real GitHub repositories and no network.** Two-home tests use local bare remotes
  (`file://`), fake providers, and an injected visibility prober.
- **The flag.** Every flag-dependent path has tests with the flag both on and off.

## Phase 1: core_audience

Open the checkout with `sase repo open sase-core -r "<why>"` and follow its `AGENTS.md`:

- free functions over `*Wire` structs, with `thiserror` errors;
- no `macro_rules!`;
- `mod.rs` files are facades, and new files stay at or under 1,500 lines;
- import by module path;
- never edit versions or changelogs;
- Conventional Commits.

Implement, in `crates/sase_core/src/note_attachment/` (new files such as `audience.rs`,
`scanner.rs`, and `zones.rs`, plus tests under `tests/`), everything in the Wire,
Decision table, Scanner, and Canonical extension sections above:

- Extend `bead/attachments.rs` for the query changes.
- Add bindings in `crates/sase_core_py/src/note_attachment/mod.rs`:
  `attachment_audience_decision`, `attachment_scan_file`,
  `attachment_scanner_rules_version`, `attachment_canonical_extension`,
  `attachment_public_object_relpath`, and `attachment_object_digest_from_relpath`.
  Register each in `register_note_attachment`, with round-trip tests in the domain
  `tests.rs`.
- The roster and references bindings in `sase_core_py/src/beads/` must pass the new
  fields through.
- Decision-table tests cover every rule and every widening outcome, using synthetic
  facts.
- Scanner tests cover every detector with synthetic secrets, the chunk-straddling case,
  `skipped` for over-cap and binary content, and the guarantee that no hit ever carries
  matched text.
- Verify with `sase tool run check` inside the sase-core checkout. This phase changes no
  sase code. The CI pin moves in `audience_cli`, either through the host's
  `revision_pin` or `just ratchet-core-revision`.

## Phase 2: push_protection

Open the checkout with `sase repo open sase-github -r "<why>"` and follow its guide.

- Add `sdd_secret_scanning(options) -> bool`, next to `sdd_sidecar_visibility` in
  `src/sase_github/workspace/sdd_repo.py`. It returns true only for a literal `True`;
  anything else is false.
- After `create_github_sdd_repo` creates a **public** repo with the option set, run
  `gh api -X PATCH repos/<repo> --input -` with
  `{"security_and_analysis": {"secret_scanning": {"status": "enabled"}, "secret_scanning_push_protection": {"status": "enabled"}}}`.
- The step is best effort: on failure, warn on stderr with the manual command, and never
  fail creation.
- Private repos and adopted (`found`) repos are untouched.
- Document the option in sase-github's `docs/configuration.md`. The `public_store` phase
  documents it in sase's hookspec docstring
  (`src/sase/workspace_provider/_hookspec.py`).
- Tests:
  - the PATCH runs only for public-plus-option;
  - a failure warns and doesn't fail creation;
  - without the option, nothing changes.
- Verify with `sase tool run check` inside the sase-github checkout.

## Phase 3: audience_cli

Start by creating the beta flag. As in `sase-1ck.4`, this is the sanctioned flag-bead
path:

```bash
sase flag new public_bead_attachments -k beta \
  --when-enabled "Bead note attachments get a SASE audience decision; public ones route to the public attachments sidecar, and -W/--public is accepted." \
  --when-disabled "Attachments are written without a visibility field (private), exactly as before, and -W/--public is refused with an enable hint." \
  --remove-when "The public_bead_attachments epic's ga phase lands."
```

Paste the printed registry entry into the registry. Then:

- **Provenance facts.** Implement them as specified, in new modules under
  `src/sase/bead/attachments/`, for example `provenance.py` and `remote_visibility.py`.
- **Flow.** Wire the facts into `authoring.py` and the upload/placement path. The order
  is:
  1. resolve;
  2. apply the sensitive-path rule (the existing early refusal keeps working);
  3. ingest into the CAS;
  4. classify;
  5. scan text classes up to `public_max_bytes`;
  6. gather facts;
  7. run `attachment_audience_decision` (on `confirm`, prompt on a TTY; non-TTY humans
     need `-y`);
  8. set `visibility` on each descriptor;
  9. write the local audience metadata;
  10. place by audience.
- **Before `public_store` lands,** public items have no store and follow the queued
  "missing public store" rule.
- **Python model.** Carry `visibility` through the attachment model and codec (the
  Python attachment model and `note_codec.py`) so round-trips never drop it.
- **Flags.** Add `-K`, `-W`, and `-y` to the five parsers
  (`main/parser_bead_lifecycle_notes.py` and `parser_bead_lifecycle_state.py`) and
  handlers, and update the fast path. With the flag off, skip the decision entirely,
  write no `visibility`, accept `-K` as a no-op, and refuse `-W` with
  `sase flag enable public_bead_attachments`.
- **Config.** Add `bead.attachments.public_max_bytes`: default 25 MiB, and a value above
  95 MiB falls back to the default. Put it in `src/sase/default_config.yml`, the schema,
  and the `bead/config.py` getters. `-S` never yields public, and `-S` with `-W` is an
  error.
- **Echo and refusal.** Implement the echo rows and the agent refusal text shown above,
  fixing the doubled `(private)` label.
- **Count-only verification.** Open the agents sidecar with `sase repo open agents`. Run
  the scanner over its published transcripts and cited file objects and print **only**
  aggregate counts per hit kind: never values, matched text, or line contents. Confirm
  that the credential-pattern detector flags at least the report's 13 known files, and
  record the counts in a note on your phase bead. Commit a helper only if it prints
  nothing but counts.
- **Docs.** Add a short `#### Attachment visibility (beta)` subsection to
  `docs/beads.md`.
- **Tests:**
  - every decision rule reached from the CLI;
  - an agent-run `--public` refusal;
  - human TTY confirmation and `-y`;
  - duplicate intent;
  - the flag on and off;
  - the fast path falling through with `-K`/`-W`/`-y`;
  - the remote prober mapping 200/401/404/timeout, and cache TTLs.
- **Verify** with `sase tool run check`.

## Phase 4: public_store

**Spike first.** Everything in it is read-only; push nothing. Use existing public
objects: an extensionless object in `sase--agents/files/objects/…` and a PNG in
`sase--research`. Check the headers with `curl -I`. Render `![x](<blob URL>?raw=true)`
and the raw URL through `gh api markdown` and fetch the resulting camo URL. Decide which
URL form bead pages embed, and record the finding in a note on your phase bead. If
embedding can't work, pages link instead of embedding. Keep the extension layout either
way, because it also gives correct raw content types.

Then implement:

- **Role.**
  - Add `ATTACHMENTS_SIDECAR_ROLE = "attachments"` to `sdd/_store_types.py`. Make it
    reserved and hidden, exclude it from document roles, add it to the built-in role
    order and schema enum, add a default description, add it to `repo_inventory`, and
    add it to `_OPTIONAL_HIDDEN_SIDECAR_ROLES`.
  - Force its visibility to `public`.
  - Inject it through `inject_default_linked_repos` only when the configured beads
    visibility is `public`. The existing collision skip covers a user's custom
    `attachments` sidecar.
- **Creation (`sase repo init`, flag-gated during beta).**
  - The consent text says `Repository visibility: PUBLIC.` and explains who can read the
    repo. On a TTY the prompt defaults to yes (`[Y/n]`). Without a TTY, the role is
    skipped with a warning.
  - Guard: before creating, probe the beads remote anonymously. If it isn't public,
    refuse and name `repos.sidecar.builtin.beads.visibility: private`.
  - Pass `sdd_secret_scanning: true` in this role's provider options
    (`_sidecar_provider_options`).
- **Bare clone.**
  - Generalize `ensure_attachments_private_bare_clone` so it is parameterized by role.
    The private role keeps refusing HTTP(S).
  - For the public role, set `remote.origin.url` to the anonymous HTTPS form when it can
    be derived, and `remote.origin.pushurl` to the configured remote. The root commit
    message is `chore(attachments): initialize public store`.
  - **Read-side auto-materialization (ungated):** when the public clone is missing but
    the remote exists and the anonymous probe says it is readable, create the clone,
    once per process, with a bounded timeout. This is what lets a fresh or
    zero-credential machine read public attachments.
- **Store.**
  - Parameterize `GitAttachmentStore` by name and layout. Construct the public instance
    with name `public`, label `<owner/repo> (public)`, and the core public relpath.
  - Look objects up by listing the `files/objects/sha256/<xx>/` tree, so both `<sha>`
    and `<sha>.<ext>` resolve.
  - `put` takes the MIME type.
  - Replace every hard-coded single-store assumption:
    `hidden_clone_path(project_key, role)`, `describe_label`, `discover_stores` (now
    `public`, `git`, `large`), `placement_tiers` by audience, `_store_identity`,
    `_store_for_pending_item`, the sync-worker drain, the doctor clone lookup, and the
    purge loop. Purge also covers the public store, and its scrub hint names the
    attachments repo, never beads.
- **Outbox.** Key entries by `(digest, store)`, as specified. Add a permanent `blocked`
  state that drains skip.
- **Fetch.**
  - Order stores by the descriptor's effective visibility: public goes
    `[public, git, large]`, and private goes `[git, large, public]`.
  - Split "not found" from digest mismatch, so a miss moves on to the next store.
  - Make `_ensure_discovered` copy `stores`.
- **GH013.** Detect `GH013`, "Push cannot contain secrets", or "push declined due to
  repository rule violations" in push stderr. Treat it as permanent:
  - reset the hidden clone's ref to the remote tip, so the rejected commit never rides a
    later push;
  - mark the outbox entry `blocked`;
  - store a private copy when a private store exists;
  - narrow every **note** descriptor that references the digest to `private`: find them
    with the references query, then append `NoteEdited(Some)` for each note, text
    unchanged, through the normal bead mutation path;
  - leave +1 evidence descriptors as they are (they are immutable), and let readers fall
    back;
  - echo `⛔ blocked by secret scanning — stored privately`.
- **Doctor.** Extend `project.attachment_store` to report the public store's
  reachability, tip, and outbox, including blocked entries. Retitle it so it no longer
  says only "Private".
- **Two-home tests on `file://` remotes:**
  - a public object written on home A is read on home B, which has only the public
    remote and no private clone;
  - private objects never reach the public remote;
  - nothing over the cap reaches the public remote;
  - outbox drains respect audiences;
  - a simulated GH013 goes through the whole path;
  - the fetch-miss and lazy-discovery regressions are fixed.
- **Verify** with `sase tool run check`.

## Phase 5: publish_lifecycle

- **`sase bead attachment publish <id> <name> [-y/--yes]`.**
  - It is human-only. Refuse inside an agent run (shared actor helper), **except** when
    it runs as an approved gate option. First work out how gate option commands execute
    (their environment and identity markers), then allow exactly that path, and test it.
  - Note attachments only; +1 evidence gets a clear refusal.
  - Steps:
    1. fetch the bytes;
    2. rescan with the current rules;
    3. refuse a private bead store, `size_cap`, and local metadata recording
       `sensitive_path` or `known_value`;
    4. preview exactly what becomes public (name, MIME type, size, target repo, bead
       page);
    5. warn that publication is irreversible and confirm (`-y` skips);
    6. upload to the public store;
    7. append `NoteEdited(Some(manifest))` with `visibility: public` and the note's text
       unchanged;
    8. publish.
- **`sase bead attachment unpublish <id> <name> [-y/--yes]`.**
  - Narrowing, so agents may run it too.
  - Steps:
    1. make sure a private copy exists (upload it to the private store, or keep it local
       and warn `⚠ only on <origin>`);
    2. remove the object from the public tree with a
       `chore(attachments): withdraw <sha>` commit (no tombstone);
    3. append `NoteEdited(Some)` with `visibility: private`;
    4. print the caveat that forks, clones, caches, and GitHub's retention of
       unreachable objects persist, so rotate first;
    5. print the `git filter-repo` runbook for the **attachments** repo only.
- **Doctor (`bead/attachment_doctor.py`, `doctor/checks_attachment_store.py`).**
  - Rescan cached public text objects whose recorded `scanner_rules_version` is older
    than the current one. Findings lead with "rotate the credential first", then give
    the `unpublish` command. Never unpublish automatically.
  - Report logical and physical bytes per store, with rotation guidance well before
    GitHub's 1–5 GB repo guidance.
  - Give writers without push access a finding.
  - If the private store is anonymously readable, report an **error** finding.
- **Help and docs.** Keep the `attachment` group's help and children alphabetical. Add
  `docs/beads.md` rows for both commands.
- **Tests:**
  - the agent refusal and the gate-path allowance;
  - the confirmation flow;
  - refusals for the non-widenable rules;
  - evidence refusal;
  - withdrawn objects remaining readable from the private store;
  - a rules-version bump triggering a rescan.
- **Verify** with `sase tool run check`.

## Phase 6: presentation

- **Audience badges.** Prefix every attachment line with `🌐` (public) or `🔒` (private
  or absent). This covers the `show`/`read` ATTACHMENTS blocks, +1 evidence,
  `history --format full`, and `attachment list`.
- **JSON.** `attachment list -j` and `read --format json` gain a materialized
  `visibility` (`public` or `private`), plus a local `audience_reason` when local
  metadata exists. The store codec never carries local keys.
- **New availability states** (`fetch.py` `attachment_state`/`attachment_badge`, and the
  pager copy in `pager/_resolve_artifact_refs.py`):
  - `no_access`: git fetch auth failures (`Repository not found`, `Permission denied`,
    `could not read Username`, HTTP 401/403/404) render `🔒 no access (<repo>)` instead
    of `✕ unavailable offline`.
  - `⧉ on <origin>`: a local-only object larger than `git_max_bytes`, plus a
    `%dispatch:<origin>` hint in `show`/`read` and `sase bead attachment path`. Read the
    `dispatch` reference memory for the exact syntax.
  - `⛔ blocked by secret scanning`, from local outbox state.
- **Bead pages** (`bead_pages/rendering_identity.py`, `attachment_presentation.py`,
  `sdd/hosted_links.py`):
  - Add an attachments-repo URL builder that uses the public relpath and the spike's
    chosen URL form.
  - `## Attachments` lists public files as `🌐 [name](url) · mime · size`, and private
    files as `🔒 name · mime · size (private attachment)`.
  - Embed public images inline, capped at 4 per note and at the auto-fetch size cap.
  - Note prose on pages renders `@attachment:<name>` as a link (public) or `🔒 name`
    (private), never as the raw token.
  - Private attachments never show digests; this fixes the +1 evidence `sha256:<12>`
    leak.
  - Pages never contain reasons.
- **Docs.** Update the troubleshooting-badges table in `docs/beads.md`.
- **Tests:** badge and JSON rendering; each new state; page output for mixed notes; the
  embed cap; no private digest or reason on any page.
- **Verify** with `sase tool run check`.

## Phase 7: tui

Read the `tui` reference memory and its `tui_perf.md` child before changing anything.

- **Chips.** Beads-pane detail chips (`ace/tui/widgets/artifacts/beads_detail_body.py`)
  show `🌐` or `🔒` from the descriptor. They never stat a store.
- **Add-note modal** (`ace/tui/modals/bead_note_modal.py` and the mutation action):
  - Run authoring and the audience decision in a worker, off the event loop.
  - Show each attachment's audience and its local reason.
  - Add one toggle key. It narrows to private freely. Widening from a policy-private
    result asks for confirmation and is refused for non-widenable rules.
  - Add the key to the keymap config in `src/sase/default_config.yml`.
  - With the flag off, the toggle is hidden and behavior is unchanged.
- **Queued uploads.** After the write, queue uploads into the outbox through the same
  helper the CLI uses (`post_write_queue`/outbox), so the bead sync worker drains them.
  Today they stay local.
- **Tests.** Add unit tests, then update the affected PNG goldens with a targeted
  `just fix-tui-screenshots` and inspect the report.
- **Verify** with `sase tool run check`.

## Phase 8: ga

- **Remove the flag** following the `sase_flags` removal rule: delete every off branch,
  make the on branches unconditional, remove the registry entry and schema block, and
  close the flag bead in the same change.
- **Docs.** Rewrite `docs/beads.md` `### Attachments` around the audience model:
  - the rule for visibility;
  - a summary of the decision table;
  - `-K`/`-W`/`-y`;
  - publish and unpublish;
  - the public store and `sase repo init`;
  - large files and `%dispatch`;
  - the badges;
  - mixed fleets (old readers treat attachments as private, which is safe);
  - **"private protects bytes, not names or prose"**.

  Also update `docs/configuration.md` (`public_max_bytes`, the `attachments` role),
  `docs/init.md` (public-store consent), `docs/sdd_storage.md`, and `docs/cli.md`.

- **Fix stale GA-era text:**
  - "Available with the flag off" in `parser_bead_lifecycle_notes.py`;
  - the nonexistent `resolve` child in the `attachment` group help;
  - the "(beta)" docstring in `bead/attachment_presentation.py`;
  - the missing `-L` in the `docs/beads.md` attach flag table.
- **Agent guidance.** Put the report's §4.4 guidance (visibility default; the five
  `--private` triggers; prefer generated excerpts; prose and filenames are public; never
  force public, offer a publish gate) into:
  - the `attach` and `note` help epilogs;
  - `sase bead onboard` (`bead/cli_admin.py` `handle_bead_onboard`);
  - a short pointer next to the `+1 … @./repro.png` example in
    `src/sase/xprompts/skills/sase_new_task.md`.

  Don't deploy skills (per `generated_skills`, they deploy from the landed tree). Don't
  edit memory files.

- **Record `PROPOSED FOLLOW-UP:` notes on the ga bead:**
  1. private descriptor metadata redaction (report §5.3) behind a mixed-fleet reader
     gate;
  2. reusing the core scanner in the agents-sidecar transcript and prompt-file
     publishers, then enabling push protection on the existing public sidecars;
  3. a `memory` task: a pointer to attachment visibility guidance in the beads memory
     template;
  4. a fleet streaming blob endpoint, only if origin-only usage justifies it.

  Never mention specific leaked credentials.

- **Verify** with `sase tool run check`.

## Acceptance (whole epic)

- **Clean public files flow end to end.**
  - A clean workspace log attached by an agent publishes.
  - A second SASE home with no private credentials fetches it.
  - It links on the bead page.
  - A TUI screenshot rendered during the run embeds inline (or links, per the spike).
- **Dirty or unverified files go private.**
  - The same log with an injected env dump, a secret env value, or a credential pattern
    goes private.
  - The reason appears only locally.
  - An agent's `--public` is refused with the publish-gate hint.
  - A human can widen the pattern and env-dump cases with confirmation, but not the
    known-value case.
- **Sensitive files never go public.** `-S` with `-W` fails. Neither
  `<sase home>/telegram_bot_token` nor an owner-only file ever goes public.
- **Nothing crosses audiences implicitly.** A `sase-1ck`-era descriptor never publishes
  implicitly. No fallback, retry, dedup, outbox drain, or store-order change moves bytes
  across audiences.
- **Large files stay off the public store.** Nothing over 25 MiB reaches it. Local-only
  objects above `git_max_bytes` show `⧉ on <origin>` and a `%dispatch` hint.
- **Access states are honest.** A reader without a grant sees `🔒 no access (<repo>)`,
  not "offline". A GH013 rejection is permanent, visible, and narrows note descriptors.
- **Private-bead projects stay private.** They inject no public role, and every
  attachment is private.
- **The hot path stays cheap.** `sase--beads` pack size doesn't grow with attachment
  bytes. Beads without attachments do no new work.
- **Hygiene.** No committed file contains a realistic secret literal. Every phase passes
  `sase tool run check` in each repository it touched.
