---
tier: epic
title: Integrate the sase-90 Chats sub-tab with the sase-91 publication repair
goal: "The Chats sub-tab reports sync provenance truthfully against the real sidecar:
  agents published under the v1 spelling read as shared rather than local, no historical
  agent name can raise out of the provenance catalog, quarantined publication is never
  presented as queued, and both epic plans plus their bead dependencies describe the
  shared state accurately.

  "
phases:
  - id: published-index
    title: Shared dual-spelling published-agent index
    depends_on: []
    size: medium
    description:
      "'Phase 1: Shared dual-spelling published-agent index' section: add
      `src/sase/agents_sync/published_index.py`, a headless reader that resolves both
      the v1 and v2 sidecar spellings for a local agent name, lists the published set
      per project target with the files each entry carries, caches by sidecar HEAD, and
      never raises on a historical agent name."
  - id: plan-alignment
    title: Correct both epic plans and wire cross-epic bead dependencies
    depends_on: []
    size: small
    description:
      "'Phase 2: Correct both epic plans and wire cross-epic bead dependencies' section:
      fix the false v1/v2 spelling claim and the outbox assumptions in the sase-90 plan,
      add the consumer contract to the sase-91 plan, and add the sase-90.6 to sase-91.3
      and sase-90.8 to sase-91.6 bead dependency edges."
  - id: catalog-integration
    title: Route the chat provenance catalog through the shared index
    depends_on:
      - published-index
      - plan-alignment
    size: small
    description:
      "'Phase 3: Route the chat provenance catalog through the shared index' section:
      make whatever chat-catalog provenance code sase-90.3 landed consume
      `published_index`, so `shared` is detected under both spellings and an
      unresolvable name yields `unknown` with a diagnostic instead of an exception."
create_time: 2026-09-09 19:53:06
status: wip
---

- **PROMPT:**
  [prompts/202607/chats_publication_epic_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/chats_publication_epic_integration.md)

# Plan: Integrate the sase-90 Chats sub-tab with the sase-91 publication repair

## Why

`sase-90` (Artifacts → Chats sub-tab with sync provenance) and `sase-91` (Repair
agents-sidecar publication) both read the same three pieces of state: the agents
sidecar's `agents/<name>/` tree, the durable publication outbox, and the agent-identity
naming helpers. Neither plan mentions the other, no bead in either epic depends on a
bead in the other, and `sase-90.3` — the phase that builds the provenance catalog on top
of all three — is in progress right now.

Five defects were verified against the live system on 2026-07-24. Each one makes
`sase-90` ship a wrong answer, and each one is caused by an assumption that `sase-91`
either contradicts or is about to change.

### Defect 1 — the v1/v2 sidecar spelling claim is false

`artifacts_chats_subtab.md` states, in the "Sidecar published set" bullet:

> Both the v1 layout (top-level `manifest.json` + `agents/`) and the v2 layout
> (`users/<username>/machines/<machine>/…`
>
> - `agents/`) expose the same `agents/<global-name>/chat.md`, so keying off that
>   directory works for both.

They do not use the same spelling. v1 publishes under
`machine_qualify_v1_transport_agent_name(name, machine)`; v2 publishes under
`globalize_agent_name(name, owner)`:

| Layout | Helper                                    | Directory under `agents/`   |
| ------ | ----------------------------------------- | --------------------------- |
| v1     | `machine_qualify_v1_transport_agent_name` | `athena.0a--code`           |
| v2     | `globalize_agent_name`                    | `bbugyi200.athena.0a--code` |

Observed in `~/.sase/projects/gh_sase-org__sase/repos/agents`: 338 directories under
`agents/`, **all** v1-spelled, **zero** v2-spelled, and all 338 contain `chat.md`. So a
catalog that looks up only the v2 global name classifies every one of the 338 published
agents as `local`. `sase-91` explicitly keeps both layouts ("Do not delete or rewrite
existing v1 sidecar payload. V1 and v2 coexist, as the original epic settled"), so this
is permanent, not transitional.

Also observed: a v1 entry contains exactly `chat.md`, `commits.json`, `meta.json` — **no
`README.md`**. That is why the `SASE_AGENT` footer links, which target
`agents/<global-name>/README.md`, 404 even for agents that _are_ present in the sidecar.
"Has `chat.md`" (what `sase-90` asks) and "has `README.md`" (what `sase-91` phase 6
verifies) are different questions about the same directory, and one index should answer
both.

### Defect 2 — `globalize_agent_name` raises on this project's legacy names

`validate_semantic_name` (`crates/sase_core/src/agent_identity/identity.rs:521`) calls
`parse_normalized_family_name_unchecked`, which returns `InvalidFamilyName` when a name
has more than one `--` or a role part containing `.`. `globalize_agent_name` calls
`validate_semantic_name`. Verified against the live binding:

```text
v2 '0a--code'          -> 'bbugyi200.athena.0a--code'
v2 '4x--epic.f-0'      -> RAISED ValueError: invalid family name '4x--epic.f-0'
v2 'fi--code.f0'       -> RAISED ValueError: invalid family name 'fi--code.f0'
v2 'fi--code.f0--plan' -> RAISED ValueError: invalid family name 'fi--code.f0--plan'
v2 'fi--code.f0--code' -> RAISED ValueError: invalid family name 'fi--code.f0--code'
v1 '4x--epic.f-0'      -> 'athena.4x--epic.f-0'          # the v1 helper never raises
```

Those are exactly the five malformed records `sase-91` documents in the `sase` project's
artifact store. The `sase-90` catalog resolves an owning agent and computes a global
name for **every** transcript, so it walks straight into the same defect that broke
publication — but in a TUI worker thread, where an uncaught exception fails the whole
pane load rather than one hood. `sase-91` phase 1 (`sase-91.1`) is the fix; until it
lands and the binding is rebuilt, `sase-90` must degrade to `unknown` rather than raise.

### Defect 3 — `sase-91` phase 3 changes outbox semantics `sase-90` phase 6 renders

`sase-90` phase 6 specifies the detail-panel copy
`"Queued to publish — 28 attempts, last error: …"`. `sase-91` phase 3 adds a quarantined
state for items past a retry threshold — items that will _never_ be retried. `sase-90`
has no representation for that, so it would report "queued to publish" for a permanently
dead item. That is precisely the failure `sase-90`'s own design section forbids:
"`unknown` must never be silently collapsed into `local` … conflating them is exactly
how a browsing UI becomes untrustworthy."

The same phase also fixes the error broadcast in `_record_failure`
(`src/sase/agents_sync/commit_publication.py:251-262`), which today writes one error to
every queued item. Verified in the live outbox: all 30 items carry
`last_error: "agents sync lock is busy"` with an identical `updated_at`. Until
`sase-91.3` lands, `last_error` is not attributable to the item it is stored on, and
`sase-90` must not present it as that agent's error.

### Defect 4 — outbox items are per-agent, not per-hood

`sase-90` describes the outbox as one that "lists agent hoods queued for publication".
`AgentPublicationOutboxItem` (`src/sase/agents_sync/publication_outbox.py:22-35`) is
per-agent: `local_agent`, `global_agent`, and a `local_hood` field. Joining a chat to
the outbox on `local_hood` would mark every chat in a hood as pending when one agent in
it is queued. `sase-91` phase 4 dedupes _publication_ by hood, which does not change the
per-agent identity of a request.

Related: `sase-90` plans to read the JSON file directly.
`list_agent_publications(project_key)` is already the public, lock-taking reader and is
what `sase-91` phases 3 and 4 evolve. A direct file read would silently desynchronize
from the schema-version bump quarantine requires.

### Defect 5 — v1 entries have no username

`sase-90` phase 3 specifies the sidecar index as
`{global_name: (machine, username, has_chat_md)}`. A v1 `meta.json` carries `name`,
`machine`, `schema_version: 1`, and timestamps — there is no `username` field, and the
v1 directory spelling does not encode one. The index must model username as optional and
the UI must not print a username it does not have.

## Approach

One shared reader, owned by neither epic, that answers "is this agent published, under
which layout, with which files" for both spellings and never raises on a historical
name. Then correct both plan files so the remaining phases are built on true statements,
and add the two cross-epic bead edges that enforce ordering.

Deliberately **not** in scope: changing `sase-91`'s design, re-litigating `sase-90`'s
provenance taxonomy, or touching `sase-core`. Defect 2's real fix is `sase-91.1`; this
plan only makes `sase-90` survive until it lands and benefit automatically once it does.

## Phase 1: Shared dual-spelling published-agent index

**ID:** `published-index` **Depends on:** none **Size:** medium

New module `src/sase/agents_sync/published_index.py`. It is headless (no Textual
import), safe to call from a TUI worker thread, and never takes the agents sync lock.

### Public surface

```python
SidecarLayout = Literal["v1", "v2"]


@dataclass(frozen=True, slots=True)
class PublishedAgentEntry:
    sidecar_name: str          # directory name under agents/, exactly as spelled on disk
    layout: SidecarLayout
    machine: str | None
    username: str | None       # always None for v1 — the spelling does not encode one
    files: frozenset[str]      # e.g. {"chat.md", "commits.json", "meta.json"}
    relpath: str               # "agents/<sidecar_name>"

    @property
    def has_chat(self) -> bool: ...     # "chat.md" in files  — what sase-90 asks
    @property
    def has_readme(self) -> bool: ...   # "README.md" in files — what SASE_AGENT footers need


@dataclass(frozen=True, slots=True)
class PublishedAgentIndex:
    project_key: str
    sidecar_path: str | None
    head_sha: str | None
    available: bool                             # False when configured but unreadable
    entries: Mapping[str, PublishedAgentEntry]  # keyed by sidecar_name
    diagnostics: tuple[str, ...]

    def lookup(self, local_name: str, owner: AgentOwnerIdentity) -> PublishedLookup: ...


@dataclass(frozen=True, slots=True)
class PublishedLookup:
    entry: PublishedAgentEntry | None
    resolvable: bool           # False when no candidate spelling could be computed
    candidates: tuple[str, ...]


def sidecar_name_candidates(
    local_name: str, owner: AgentOwnerIdentity
) -> tuple[str, ...]: ...


def load_published_agent_index(
    target: ProjectTarget, *, force: bool = False
) -> PublishedAgentIndex: ...
```

### Behavior

- `sidecar_name_candidates` returns the v2 spelling from
  `globalize_agent_name(local_name, owner)` followed by the v1 spelling from
  `machine_qualify_v1_transport_agent_name(local_name, owner.machine_name)`,
  deduplicated and order-preserving. **Each helper is called inside its own `try`**:
  `globalize_agent_name` raises `ValueError` for the legacy names in this artifact store
  (defect 2), and the v1 helper is the one that still succeeds. An empty result means
  the name is unresolvable, never an exception.
  - Once `sase-91.1` lands and the binding is rebuilt, the v2 candidate starts resolving
    for those names with no change here. Write the tests so they assert the _contract_
    (never raises; returns at least the v1 candidate for a legacy name) rather than
    pinning the current tuple length.
- `lookup` prefers a v2 match over a v1 match when both directories exist, so an agent
  republished under v2 reports the authoritative layout. `resolvable=False` with
  `entry=None` is materially different from `resolvable=True` with `entry=None`: the
  first means "could not check", the second means "checked, not published". Callers
  depend on that distinction to choose `unknown` versus `local`.
- `load_published_agent_index` lists `<target.sidecar_path>/agents/*/`, records each
  entry's directory contents, and classifies layout by spelling: a name whose leading
  segment equals `owner.machine_name` is v1; a name prefixed with
  `<username>.<machine>.` is v2. Read `meta.json` only for `machine`/`username` when
  cheap; never read `chat.md`.
- Cache keyed by `(sidecar_path, head_sha)` where `head_sha` comes from
  `git rev-parse HEAD` through `sase.agents_sync.git.run_git` — bounded,
  non-interactive, never prompting. A missing checkout, an unreadable `agents/` tree, or
  an unresolvable HEAD yields `available=False` plus a diagnostic naming the project and
  the reason, and an empty `entries` mapping. That is the input that makes `sase-90`'s
  `unknown` state explicable.
- A concurrent `sase-91` drain can be rewriting the checkout mid-read. Any `OSError`
  while listing degrades to a diagnostic and `available=False` for that project; it must
  never propagate.

### Tests

New `tests/agents_sync/test_published_index.py` against a `tmp_path` fake sidecar:

- a v1-spelled entry with `chat.md` but no `README.md` is found by `lookup` for the
  corresponding local name, reports `layout="v1"`, `username=None`, `has_chat=True`,
  `has_readme=False`;
- a v2-spelled entry reports `layout="v2"` with username and machine populated;
- an agent published under **both** spellings resolves to the v2 entry;
- an agent published under neither resolves to
  `PublishedLookup(entry=None, resolvable=True)`;
- each of `4x--epic.f-0`, `fi--code.f0`, `fi--code.f0--plan`, `fi--code.f0--code`
  produces a non-empty candidate tuple and never raises;
- a missing sidecar directory yields `available=False`, one diagnostic, and no
  exception;
- warm lookups at an unchanged HEAD perform no directory listing (assert with a listing
  counter).

Run `just install` then `just check`.

## Phase 2: Correct both epic plans and wire cross-epic bead dependencies

**ID:** `plan-alignment` **Depends on:** none **Size:** small

Edit the two epic plan files in the plans sidecar. Neither file's YAML frontmatter phase
list changes — no phase is added, removed, renumbered, or resized, so the existing beads
stay valid. Only prose in the body changes.

### `202607/artifacts_chats_subtab.md`

1. **"Data sources" → "Sidecar published set" bullet.** Delete the sentence claiming
   both layouts expose the same `agents/<global-name>/chat.md`. Replace with the two
   spellings, the helper each comes from, the observation that the live sidecar holds
   338 v1-spelled directories and zero v2-spelled ones, that `sase-91` keeps both
   permanently, and that lookups go through `sase.agents_sync.published_index` rather
   than being open-coded.
2. **"Classification rules" → rule 2.** "**Shared** if _either_ the v1 machine-qualified
   or the v2 owner-qualified spelling of the owning agent's name has a directory in some
   agents sidecar checkout that contains `chat.md`."
3. **"Classification rules" → rule 3.** Add: a name for which no candidate spelling can
   be computed is `unknown`, not `local` — `PublishedLookup.resolvable=False` and
   "checked, not published" are different claims.
4. **"Classification rules" → new note after rule 4.** Record defect 2 verbatim:
   `globalize_agent_name` raises `ValueError` for the five legacy names in this
   project's artifact store until `sase-91.1` lands and the binding is rebuilt. The
   catalog must not call `agent_local_hood`, `agent_name_in_hood`,
   `agent_name_ancestors`, or `agent_link_target` — all four begin with
   `parse_agent_family_name` and raise on the same names. Any classification exception
   degrades that one transcript to `unknown` plus a diagnostic; it must never fail a
   pane load.
5. **Phase 3 → "Three indexes" item 3.** Replace the sidecar-index description with
   "consume `published_index.load_published_agent_index(target)`; do not re-implement
   directory listing, spelling resolution, or HEAD-keyed caching." Change the index
   shape from `{global_name: (machine, username, has_chat_md)}` to the
   `PublishedAgentEntry` surface, and state that `username` is `None` for v1 entries.
6. **Phase 3 → `ChatCatalogEntry`.** Add `sidecar_layout: SidecarLayout | None` and
   `publication_quarantined: bool` beside the existing `sidecar_repo` /
   `sidecar_relpath` / `publication_pending` fields.
7. **Phase 3 → "Publication backlog" data source and phase 6 detail copy.** State that
   the outbox is read through `list_agent_publications(project_key)`, not by parsing the
   JSON, and that items are matched to a chat on `global_agent`/`local_agent` — **not**
   `local_hood`, which would mark every chat in a hood as pending.
8. **Phase 6 → PROVENANCE section, `local` bullet.** Split the single "Queued to
   publish" line into two distinct states:
   - queued → "Queued to publish — N attempts, last error: …"
   - quarantined → copy that says publication has stopped retrying, gives the attempt
     count and last error, and names the operation that clears it. The word "queued"
     must not appear for a quarantined item.

   Add: until `sase-91.3` lands, `last_error` is broadcast across every queued item and
   is not attributable, so it must not be rendered as this agent's error before then.

9. **Phase 8 bullet.** Note that verifying provenance against real data is only
   meaningful after `sase-91.6` drains the backlog; before that the live `shared` set is
   entirely v1-spelled and `users/` is empty.

### `202607/agents_sidecar_publication_recovery.md`

1. **Phase 3, quarantine bullet.** Quarantine must be durable and readable: a field on
   `AgentPublicationOutboxItem` surfaced through `list_agent_publications()`, with
   `PUBLICATION_OUTBOX_SCHEMA_VERSION` bumped and reads tolerant of items written by the
   older schema. Name `sase-90` phase 6 as the downstream consumer so the state is not
   left internal to the drain.
2. **Phase 3, `_record_failure` bullet.** Add that `last_error` is rendered verbatim in
   the Chats detail panel, so attributability is a user-visible correctness property,
   not only a diagnostic nicety.
3. **Phase 6, link-verification bullet.** Add that v1 entries carry `chat.md`,
   `commits.json`, and `meta.json` but no `README.md`, which is why footer links
   pointing at `agents/<global-name>/README.md` 404 even for agents present in the
   sidecar; verify link resolution through `published_index` and assert the v1 entries
   are untouched alongside the new v2 ones.

### Bead dependency edges

```bash
sase bead dep add sase-90.6 sase-91.3   # detail panel needs the quarantine field
sase bead dep add sase-90.8 sase-91.6   # real-data provenance verification needs the drain
```

Do **not** add a dependency from `sase-90.3` to `sase-91.1`. `sase-90.3` is in progress,
blocking it would strand a live agent, and the correction in edit 4 above plus the phase
1 module make it safe without one: the v1 spelling resolves for every legacy name today,
and the v2 spelling starts resolving on its own once `sase-91.1` lands.

Verify with `sase bead show sase-90.6` and `sase bead show sase-90.8` that both edges
are recorded. Plan-file edits do not require `just check` (see the exceptions in
`CLAUDE.md`), but confirm both files still parse with `sase plan validate`.

## Phase 3: Route the chat provenance catalog through the shared index

**ID:** `catalog-integration` **Depends on:** `published-index`, `plan-alignment`
**Size:** small

`sase-90.3` is in progress as this plan is written, so its output cannot be assumed.
Start by inspecting what actually exists.

**If a chat provenance catalog module exists** (under `src/sase/history/`, per `sase-90`
phase 3):

- Replace its sidecar lookup with `published_index.load_published_agent_index` and
  `PublishedAgentIndex.lookup`, deleting any open-coded directory listing, spelling
  computation, or HEAD caching it grew.
- Map the lookup result onto the taxonomy: an entry with `has_chat` → `shared` (carrying
  `sidecar_layout`); `resolvable=True` with no entry and an `available` index → `local`;
  `resolvable=False` or `available=False` → `unknown` with the index's diagnostic
  attached. Never collapse `unknown` into `local`.
- Wrap every identity-helper call so a `ValueError` on a legacy name yields `unknown`
  plus a diagnostic for that one transcript.
- Switch any direct outbox JSON read to `list_agent_publications(project_key)`, matched
  on `global_agent`/`local_agent`.

**If it does not exist yet**, make no code change beyond confirming `published_index` is
importable and exported, and close the phase noting that `sase-90.3` will consume the
reader directly under the corrected plan text. Do not pre-build the catalog — that is
`sase-90.3`'s work and duplicating it would create a merge conflict with a live agent.

### Tests

Extend the `sase-90` catalog tests (or add
`tests/history/test_chat_catalog_provenance_spellings.py` if none exist yet) with a
fixture sidecar containing one v1-spelled and one v2-spelled published agent:

- the transcript of the v1-published agent classifies as `shared`, not `local` — this is
  the regression that motivates the whole plan;
- the transcript of the v2-published agent classifies as `shared` with
  `sidecar_layout="v2"`;
- a transcript whose owning agent carries a legacy name (`fi--code.f0--code`) classifies
  without raising;
- a missing sidecar yields `unknown` with a diagnostic, and a machine with no sidecar
  configured at all yields `local` with no diagnostic, preserving `sase-90`'s stated
  degradation contract.

Run `just install` then `just check`.

## Acceptance criteria

- A chat whose agent is published only under the v1 spelling classifies as `shared`. The
  338 currently-published agents in `sase--agents` are not reported as `local`.
- No agent name already on disk can raise out of the provenance catalog. Legacy names
  classify as `unknown` with a diagnostic before `sase-91.1`, and resolve normally after
  it, with no further change to this code.
- The Chats detail panel distinguishes queued from quarantined publication, reads the
  outbox through `list_agent_publications`, and matches items per-agent rather than
  per-hood.
- Both epic plan files describe the sidecar layout, the identity helpers' failure modes,
  and the outbox API accurately, and neither file's phase frontmatter changed.
- `sase-90.6` depends on `sase-91.3` and `sase-90.8` depends on `sase-91.6`; `sase-90.3`
  is left unblocked.
- One shared module answers both "does this agent have `chat.md` in the sidecar"
  (sase-90) and "does it have `README.md`" (sase-91 footer links), so the two epics
  cannot drift apart again.
- No SASE memory file, `AGENTS.md`, or generated provider shim is edited; no historical
  artifact directory, chat file, or sidecar payload is rewritten.
