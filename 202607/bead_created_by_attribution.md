---
tier: epic
title: Attribute beads to the agent that created them
goal: Every newly created bead records the SASE agent responsible for it — epic and
  phase beads carry the agent that proposed the epic plan file, task beads carry the
  agent (or the human) that created them — and `sase bead show` plus the published
  bead page render that creator as a link to its `agents` sidecar page.
phases:
- id: core
  title: Record an explicit creator in sase-core bead creation and plan validation
  depends_on: []
  size: medium
  description: 'core: add an optional `created_by` to the Rust bead-create request
    with an explicit request/phase-parent/store-owner resolution order, and add the
    optional system-managed `proposed_by` plan frontmatter field to the plan validator,
    schema, and validated wire.'
- id: attribution
  title: Add the Python attribution resolver and creator plumbing
  depends_on:
  - core
  size: small
  description: 'attribution: add the `sase.bead.attribution` module that resolves
    the acting agent''s durable global name and a plan''s recorded proposer, thread
    `created_by` through the bead mutation facade and `BeadProject.create`, and surface
    `proposed_by` on the Python validated-plan dataclass.'
- id: wiring
  title: Record the creator on every bead creation path
  depends_on:
  - attribution
  size: medium
  description: 'wiring: stamp `proposed_by` into plan frontmatter at `sase plan propose`,
    and pass a resolved creator from `sase bead create` and from deterministic epic-from-plan
    creation so epics, phases, and tasks are attributed correctly.'
- id: show
  title: Render the creator and its agent link in bead detail output
  depends_on: []
  size: medium
  description: 'show: add the `CREATED BY` block to the shared bead detail renderer,
    resolve the hosted agents sidecar URL for `sase bead show`, add `created_by_url`
    to the detail JSON envelope, and refresh the affected golden fixtures and docs.'
- id: page
  title: Render the creator on published bead pages
  depends_on: []
  size: small
  description: 'page: add a linked `Created by` fact to the hosted bead page identity
    block using the page renderer''s existing agent-link resolver, and refresh the
    bead page goldens.'
create_time: 2026-07-31 09:12:26
status: done
bead_id: sase-bv
---

- **PROMPT:** [202607/prompts/bead_created_by_attribution.md](prompts/bead_created_by_attribution.md)
- **BEAD:** [sase-bv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bv/README.md)

# Plan: Attribute beads to the agent that created them

## Context

`created_by` is stored on every bead (`Issue.created_by`, `IssueWire.created_by`) and is already exported in bead JSONL,
in the `sase bead show --format json` envelope, and as the **actor** on the `issue_created` event in the bead event
stream (`sase bead history` renders it). Despite that, the field carries no information today.

`create_issue` in `crates/sase_core/src/bead/mutation.rs` sets it unconditionally from the bead store's configured
owner:

```rust
let owner = store.config.owner.clone();
// ...
    owner: owner.clone(),
    created_by: owner,
```

`owner` comes from `bead/config.json`, which `get_default_config` seeds from `git config user.email`. Every bead in this
project therefore reports `created_by: "bryanbugyi34@gmail.com"`, and every `issue_created` history entry is attributed
to that same address, regardless of which of the thousands of SASE agents actually created the bead. The value is a
duplicate of `owner`, which `sase bead show` already prints on its own line.

There are exactly three bead-creation call sites in this repository, which keeps the change tractable:

| Call site                                       | Creates                          |
| ----------------------------------------------- | -------------------------------- |
| `src/sase/bead/cli_crud.py::handle_bead_create` | any bead from `sase bead create` |
| `src/sase/bead/epic_from_plan.py` (epic)        | the epic plan bead               |
| `src/sase/bead/epic_from_plan.py` (phase loop)  | every phase bead of that epic    |

All three funnel through `BeadProject.create` → `sase.core.bead_mutation_facade.create` → the `bead_create` binding →
`create_issue`.

### What already exists and should be reused

- **`HostedLinkResolver.agent_url(agent_name)`** (`src/sase/sdd/hosted_links.py`) already turns an agent name into the
  exact `agents` sidecar URL this feature needs, memoized per process. Verified in this workspace:

  ```text
  'q8'                              -> .../sase--agents/blob/main/agents/bbugyi200.athena.q8/README.md
  'bbugyi200.athena.q8'             -> .../sase--agents/blob/main/agents/bbugyi200.athena.q8/README.md
  'bbugyi200.athena.sase-8v.1--code'-> .../sase--agents/blob/main/families/bbugyi200.athena.sase-8v.1.md#member-code
  'bryanbugyi34@gmail.com'          -> None
  ```

  It accepts both local and global spellings, routes family members to their family page anchor, and returns `None` for
  anything that is not a resolvable agent — including an email address. No new link plumbing is required, and no
  is-this-an-agent heuristic is required either.

- **`present_agent_name`** (`src/sase/core/agent_identity_facade.py`) hides the current owner's `<user>.<machine>.`
  prefix and leaves foreign names fully qualified. It is the correct display transform for a stored global name.

- **`globalize_owned_agent_name`** produces the durable spelling (`q8` → `bbugyi200.athena.q8`).

- **`discover_agent_identity`** (`src/sase/agent/identity.py`) discovers the running agent from `SASE_AGENT_NAME`,
  `SASE_AGENT`, or `SASE_ARTIFACTS_DIR/agent_meta.json`, and reports which source it used.

- **Plan frontmatter already records proposal provenance.** `bead:` is documented in the core schema as "bead id of the
  agent that proposed this plan, written by SASE", and `sase plan propose` already stamps managed frontmatter fields
  before archiving the plan. Adding a sibling field for the proposing agent's _name_ follows an established pattern.

- **`append_issue_note`** in the same Rust module already demonstrates the exact actor-resolution shape this plan needs:
  `author: Option<String>`, blank-filtered, falling back to `store.config.owner`.

### Boundary

Per the Rust core backend boundary, attribution _policy_ (what a creator resolves to, and how a phase inherits from its
parent) is core domain behavior and belongs in `sase-core`. _Discovering_ the running agent from process environment and
artifact metadata is a host concern and stays in Python.

## Design

### The recorded value

`created_by` stores the **durable global agent name** (`bbugyi200.athena.q8--plan`) for agent-created beads, and the
store owner's email for human-created ones.

Global rather than bare-local, even though `assignee` stores bare-local names, because:

- Bead records are published to the public `sase--beads` sidecar and are read outside the owning machine's context. The
  `agents` sidecar itself is owner-sharded and its pages are named globally.
- `assignee` is a transient runtime claim; `created_by` is permanent provenance. Provenance earns the unambiguous
  spelling. This matches the plan `AGENTS:` header and commit footer labels, which already use global names.
- Readability is recovered at render time by `present_agent_name`, which prints `q8--plan` for your own agents and the
  fully qualified name for anyone else's.

`created_by` is **write-once**. It is set at creation and is never accepted by `sase bead update`, and no `--created-by`
flag is added; the value is derived, not declared.

### Attribution policy

Resolution runs in one order, with each step falling through when it yields nothing:

1. **Explicit creator** supplied by the caller.
2. **Phase inheritance** — a `phase` bead with a resolvable parent takes the parent's `created_by`. This is what makes
   "phase beads carry the epic plan's proposer" true without the caller repeating itself for every phase, and it also
   makes a hand-run `sase bead create -T "phase(<epic>)"` correct for free.
3. **Store owner** — the configured email, exactly as today.

Steps 1–3 live in `create_issue` in the Rust core. What Python supplies for step 1:

| Bead being created                        | Explicit creator passed by Python               |
| ----------------------------------------- | ----------------------------------------------- |
| epic / plan bead created from a plan file | the plan's `proposed_by`, else the acting agent |
| phase bead                                | nothing — core inherits from the parent epic    |
| task bead (and any other bead)            | the acting agent                                |

"The acting agent" is the current SASE agent's global name, or nothing when no agent is running — in which case core
falls through to the owner email, which is exactly the "or `bryanbugyi34@gmail.com`, if I created the bead manually"
case.

### Recording the plan's proposer

The plan file itself is the right place to record who proposed it: the value is written once at proposal time, travels
into the committed `sase--plans` sidecar copy, survives approval latency and machine changes, and is already the input
that deterministic epic creation reads. A new optional, system-managed frontmatter field is added:

```yaml
tier: epic
title: Workspace GC rewrite
goal: ...
proposed_by: bbugyi200.athena.q8--plan
create_time: 2026-07-31 12:43:46
status: wip
```

`sase plan propose` stamps it alongside the `bead` / `parent_bead` stamps it already writes, for both tiers. It is named
`proposed_by` rather than `agent` because `model:` in the same frontmatter already means "model for the agent that will
_run_ this plan"; an `agent:` key there would read as the executor, not the author. `proposed_by` also pairs directly
with the bead field it feeds.

Plans archived before this change have no `proposed_by`. Epics created from them fall through to the acting agent and
then to the owner email — the same value they get today. **No backfill of existing beads or plans is performed**; the
requirement is to start recording correctly, and rewriting historical provenance would be inventing it.

### One hazard this design must avoid

`discover_agent_identity` accepts `SASE_AGENT` as a _name_ source, but the launcher sets `SASE_AGENT=1` as a boolean
flag (confirmed in a live agent environment: `SASE_AGENT=1`, `SASE_AGENT_NAME=q8`). Worse, `tests/conftest.py`'s
`_clear_agent_env_vars` fixture scrubs `SASE_AGENT_*` and `SASE_ARTIFACTS_DIR` but not the bare `SASE_AGENT`, so an
agent-launched `just test` leaks it into every test. Naively trusting `identity.name` would record
`created_by: "bbugyi200.athena.1"` on beads created during test runs.

The attribution resolver therefore accepts only the `SASE_AGENT_NAME` and `agent_meta` identity sources and ignores the
`SASE_AGENT` source, and validates the discovered name before globalizing it. The underlying defect in
`discover_agent_identity` is tracked separately as task bead `sase-bp` and is **out of scope here**; this plan must not
change that shared helper's behavior for its other callers.

### Rendering

The shared detail renderer (`render_issue_detail`, used by `sase bead show`, `sase bead list --format full`, and
`sase bead search --format full`) gains a `CREATED BY` block placed immediately before the existing `PAGE` block, so the
two hosted links sit together and the flat header field stack is left undisturbed:

```text
○ sase-bm · Isolate model-alias tests from parallel state leakage   [OPEN]
Type: task · Owner: bryanbugyi34@gmail.com
Size: small

CREATED BY
  q8--plan
  → https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.q8/README.md

PAGE
  https://github.com/sase-org/sase--beads/blob/main/pages/sase-bm/README.md
```

A human creator renders without the arrow line, because there is no agent page to point at:

```text
CREATED BY
  bryanbugyi34@gmail.com
```

This reuses two idioms the renderer already owns: the uppercase two-space-indented section, and the `<value>` /
`→ <where it resolves>` pair used by the `PLAN` / `EPIC PLAN` block. It introduces no new visual vocabulary.

The URL is resolved only for `sase bead show`, mirroring how `page_url` is already resolved only there;
`list --format full` and `search --format full` render the name alone. The block is rendered whenever `created_by` is
non-empty, even when it equals `owner`, because "who owns this project" and "who created this bead" are different facts
that merely happen to coincide for manually created beads.

The published bead page gets the same fact as a real Markdown link on its existing ownership line:

```text
**Owner:** `owner@example.com` · **Created by:** [bbugyi200.athena.q8--plan](https://github.com/…) · **Assignee:** `alice`
```

The page keeps the global spelling unlocalized, because a published page is read outside the owner's context.

### Non-goals

- No backfill or migration of existing bead or plan records.
- No ACE TUI changes; ACE does not render `created_by` today.
- No change to `owner`, to `assignee`, or to how either is derived.
- No fix to `discover_agent_identity` itself (bead `sase-bp`).

## Phase core

Work in the `sase-core` repo. Open it with `sase repo open sase-core -r "<reason>"` and use only the path that prints.
Leave Cargo versions alone; release-plz owns them.

**Bead creation** — `crates/sase_core/src/bead/mutation.rs`:

- Add `#[serde(default)] pub created_by: Option<String>` to `BeadCreateRequestWire`. The PyO3 layer deserializes the
  whole request dict with serde (`bead_create_request_from_pydict`), so no binding change is needed — confirm that and
  say so rather than assuming it.
- In `create_issue`, replace the unconditional `created_by: owner` with a resolved creator, following the shape
  `append_issue_note` already uses for `author`:
  1. `request.created_by`, trimmed, when non-blank;
  2. otherwise, when `request.issue_type` is `phase` and `request.parent_id` names an issue present in the store, that
     parent's `created_by` when non-blank;
  3. otherwise `store.config.owner`.
- `issue.owner` keeps coming from `store.config.owner` and must not change.
- The resolved creator is also the actor passed to `append_issue_event` for the `IssueCreated` event and for the
  `ReferenceAdded` events emitted in the same call, preserving today's "actor equals `created_by`" invariant.

**Plan frontmatter** — `crates/sase_core/src/plan/validate.rs`:

- Add `"proposed_by"` to `SYSTEM_FIELDS` so it is accepted for both tiers instead of failing the `unknown-key`
  diagnostic.
- Add its `field_spec` to `plan_frontmatter_schema` next to the other system-managed entries: optional, described as the
  name of the SASE agent that proposed the plan, written by SASE, with an example such as `bbugyi200.athena.q8--plan`.
- Add `pub proposed_by: Option<String>` to `ValidatedPlanWire` and populate it in the validator alongside `bead` and
  `parent`. A present-but-blank value is an error diagnostic, consistent with the other non-empty-string system fields.

**Tests** — cover, in this repo's existing Rust test style:

- create with an explicit `created_by` records it and uses it as the `issue_created` actor;
- create with a blank or absent `created_by` still records the store owner (the current contract, unchanged);
- a `phase` create inherits its parent's `created_by`, and an explicit value still wins over inheritance;
- a `task` or `plan` create under a parent does **not** inherit;
- a phase whose parent is missing or whose parent has a blank `created_by` falls through to the owner;
- plan validation accepts `proposed_by`, returns it on the validated plan, still accepts a plan without it, and rejects
  a blank one.

Run `cargo fmt --all`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`. Land the
sase-core change before the dependent Python phases merge: this repo's CI resolves `sase_core_rs` from the sase-core
checkout, so Python code calling the new field needs the core change on `main` first.

## Phase attribution

Add `src/sase/bead/attribution.py` with the resolution logic and no I/O beyond reading a plan file's frontmatter:

- `acting_agent_name() -> str | None` — call `discover_agent_identity()`; return `None` when it yields nothing or when
  `identity.source == "SASE_AGENT"` (see the hazard above); otherwise validate the name with `validate_new_agent_name`
  and return `globalize_owned_agent_name(name)`. Any failure returns `None` rather than raising, so bead creation never
  breaks on a malformed identity.
- `plan_proposed_by(plan_path: Path) -> str | None` — read the file's frontmatter with `sase.sdd.frontmatter`
  `parse_frontmatter` and return a non-blank `proposed_by`, or `None` when the file is unreadable, has no frontmatter,
  or has no such key. This deliberately avoids full plan validation so `sase bead create` stays cheap and keeps working
  on a plan file that would not validate.
- `resolve_bead_creator(*, issue_type, plan_proposed_by=None) -> str | None` — implement the table from the design:
  `phase` → `None`; `plan` → the supplied proposer, else `acting_agent_name()`; anything else → `acting_agent_name()`.

Thread the value through, defaulting to `None` at every layer so existing callers are unaffected:

- `src/sase/core/bead_mutation_facade.py::create` — add a `created_by: str | None = None` keyword and include it in the
  request payload dict.
- `src/sase/bead/project.py::BeadProject.create` — add the matching keyword and forward it.

Surface the new plan field in Python:

- `src/sase/sdd/plan_validate.py` — add `proposed_by: str | None` to the validated-plan dataclass and populate it from
  the wire payload alongside `bead` and `parent`.

**Tests** — a new focused test module for the resolver plus additions to the existing bead facade/project tests:

- `acting_agent_name` returns the globalized name for `SASE_AGENT_NAME`, returns `None` when only the bare `SASE_AGENT`
  flag is set (assert explicitly against the `SASE_AGENT=1` case), returns `None` with no agent env, and resolves from
  `agent_meta.json` via `SASE_ARTIFACTS_DIR`;
- `plan_proposed_by` reads the field, and returns `None` for a missing file, a file without frontmatter, a plan without
  the key, and a blank value;
- `resolve_bead_creator` returns the documented value for each bead type;
- `BeadProject.create(created_by=…)` round-trips through the facade to the stored issue.

## Phase wiring

**`src/sase/main/plan_propose_handler.py::handle_plan_propose_command`** — add `proposed_by` to the `stamps` dict that
is already written with `set_frontmatter_fields` before the plan is formatted and archived, using `acting_agent_name()`
and skipping the stamp when it returns `None`. Stamp for both `tale` and `epic` proposals; a re-proposal overwrites the
field, which is correct because the latest proposer owns the archived copy.

**`src/sase/bead/cli_crud.py::handle_bead_create`** — resolve the creator once and pass it to `proj.create`. The handler
already parses `--type` into `(issue_type, plan_path, parent_id)`, so a `plan(<file>)` create can call
`plan_proposed_by(plan_file)` on the same resolved path it already validates, and everything else passes `None` for the
proposer.

**`src/sase/bead/epic_from_plan.py::create_and_launch_epic_from_plan`** — pass
`resolve_bead_creator(issue_type=IssueType.PLAN, plan_proposed_by=plan.proposed_by)` when creating the epic. The phase
loop passes nothing: core inheritance already gives every phase the epic's creator, which keeps the phase-creation call
unchanged and guarantees an epic and its phases can never disagree about their origin. Add a short comment saying so, so
the omission does not read as a bug.

**Tests**:

- `tests/test_plan_command_handler.py` — a proposal from an agent environment stamps `proposed_by` into the archived
  plan; a proposal with no resolvable agent leaves the frontmatter without it; the stamp does not disturb the existing
  `bead` / `parent_bead` stamping or the prettier formatting step.
- `tests/test_bead/test_epic_from_plan.py` — an epic created from a plan carrying `proposed_by` records it on the epic
  bead and on every phase bead; a plan without `proposed_by` falls back to the acting agent and then to the owner.
- Bead-create coverage in the existing CLI create tests — a task bead created inside an agent environment records that
  agent; a task bead created with no agent environment records the store owner; a `phase(<parent>)` create inherits.
- Confirm the `sase bead create` golden fixtures still pass unchanged; if any golden shifts, that is a signal the test
  environment is leaking agent identity and must be fixed in the test, not the fixture.

## Phase show

**`src/sase/bead/cli_detail.py`**:

- `render_issue_detail` — add a `creator_url: str | None = None` keyword mirroring the existing `page_url`, and emit the
  `CREATED BY` block described in the design immediately before the `PAGE` block, whenever `issue.created_by` is
  non-empty. The label is `present_agent_name(issue.created_by)`, guarded so any failure falls back to the raw stored
  value. The `→ <url>` line is emitted only when `creator_url` is present.
- Add `resolve_bead_creator_url(created_by: str) -> str | None` beside `resolve_bead_page_url`, resolving through
  `hosted_link_resolver(...).agent_url(...)` with the same never-raise contract; an email or an unresolvable name
  returns `None`.
- `render_issue_detail_json` — accept and emit `created_by_url` alongside `page_url` when it resolves. `created_by` is
  already present inside the `issue` object and does not move.

**`src/sase/bead/cli_query.py::handle_bead_show`** — resolve and pass `creator_url` for both the `full` and `json`
formats. Leave `list --format full` and `search --format full` unchanged; they render the name without a link, exactly
as they already render no `PAGE` block.

**Fixtures and docs**:

- Refresh the affected golden files under `tests/test_bead/golden/cli/` — the `show`, `show_phase_parent_epic_plan`,
  `list_full`, and `list_implicit_closed_full` text fixtures gain the `CREATED BY` block, and the `show_json` /
  `show_phase_json` fixtures are checked for the envelope key. These fixtures build beads with no agent environment, so
  the rendered creator is the store owner email and no arrow line appears.
- Add focused assertions in `tests/test_bead/test_cli_show.py`: an agent-created bead renders the localized name plus
  the arrow line when the URL resolves, renders the name alone when it does not, and the JSON envelope carries
  `created_by_url` only when resolved.
- Document the new block in the `sase bead show` material in `docs/beads.md`, including that a human-created bead shows
  an email and no link.

## Phase page

**`src/sase/bead_pages/rendering_identity.py`** — add a `Created by` fact to `_ownership_facts`, placed between `Owner`
and `Assignee`. Render the stored global name as the label, escaped, and wrap it in a Markdown link when the resolver
produces a URL; fall back to plain inline code when it does not, matching how `_reference_item` already degrades an
unresolvable reference to escaped text rather than a broken link.

`render_identity` currently takes a `plan_links: PlanLinkResolver | None`. Reuse the pattern already established in this
module: the concrete `HostedLinkResolver` passed in also satisfies the `_AgentLinkResolver` protocol, so narrow with
`isinstance(resolver, _AgentLinkResolver)` before calling `agent_url`, exactly as `_reference_url` does. Do not widen
the declared parameter type or add a second resolver argument.

**Tests** — refresh `tests/test_bead/golden/bead_pages/root.txt` and `descendant.txt`, and add assertions in
`tests/test_bead/test_bead_page_rendering.py` covering the linked case, the unlinked case, and a bead with an empty
`created_by` (which must render the ownership line exactly as it does today).

## Validation

Every phase runs `just install` first — workspace directories are ephemeral and dependencies may have moved — and then
`just check` before finishing. The `core` phase additionally runs `cargo fmt --all`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace` inside the sase-core checkout.

The whole feature is done when, from a SASE agent, `sase bead create -T task -t "…"` records that agent's global name,
`sase bead show <id>` prints the localized name with a working `agents` sidecar link, and an epic launched from a plan
proposed by an agent shows that same agent on the epic bead and on every one of its phase beads.
