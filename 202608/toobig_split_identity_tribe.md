---
tier: epic
title: Restore toobig_split keyed names and chop tribe membership
goal:
  Typed toobig_split admission preserves clan identity and @chop metadata while new
  agents use concise keyed basename names.
phases:
  - id: typed_identity
    title: Preserve grouped identity through typed launch planning
    depends_on: []
    size: medium
    description:
      "typed_identity: extend the Rust/Python typed wire, restore grouped directives,
      and resolve keyed markers once per batch."
  - id: axe_clan_admission
    title: Promote the first eligible chop member to clan declarer
    depends_on:
      - typed_identity
    size: medium
    description:
      "axe_clan_admission: make survivor promotion durable and prove launched chop
      members retain clan, tribe, and summary metadata."
  - id: keyed_basename_names
    title: Emit keyed basename templates from bugyi-chops
    depends_on: []
    size: small
    description:
      "keyed_basename_names: replace full path member names with stable collision-safe
      keyed basename templates in the plugin."
  - id: rollout_verification
    title: Deploy and exercise the repaired chop end to end
    depends_on:
      - axe_clan_admission
      - keyed_basename_names
    size: xsmall
    description:
      "rollout_verification: install the ordered releases and capture dry-run plus live
      evidence for names, tribe membership, and skips."
proposed_by: bbugyi200.athena.0c6
bead_id: sase-so
create_time: 2026-09-09 19:51:56
status: wip
---

- **PROMPT:**
  [prompts/202608/toobig_split_identity_tribe.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/toobig_split_identity_tribe.md)
- **BEAD:**
  [sase-so](https://github.com/sase-org/sase--beads/blob/main/pages/sase-so/README.md)

# Restore `toobig_split` identity and tribe semantics

## Verified diagnosis

The screenshot is consistent with one precise regression. The currently installed
`bugyi-chops 0.7.0` is already sourced from commit
`8b2785d5e336ac5ff900fc0fb7e79e382d98888f`, so yesterday's conditional-admission change
was committed, pushed, and installed; this is not an old-package problem.

The plugin emits structured proposals with `clan="toobig-@"`, a templated member
`agent_name`, repeated `clan_summary`, and an admission `%if`. Before typed admission,
SASE plans a concrete clan batch correctly: the first proposal gets a full `%id` plus
`%clan(<concrete>, tribe=chop, ...)`, while later proposals get
`%id(<member>, clan=<concrete>)`.

Typed planning then destroys that grouping information:

- Rust `AgentUnitWire` stores only the positional `identity`; `parse_id_directive`
  ignores `clan=`, `family=`, and `tribe=`, and the typed classifier removes `%clan`
  without retaining its name, tribe, or summary.
- `agent_unit_dispatch_prompt` reconstructs `%id:<identity>` only. A clan joiner thus
  becomes an unrelated agent with its member ID as the complete name, and every typed
  member loses the `@chop` assignment.
- If the statically selected declarer is condition-skipped, the first surviving unit is
  a former joiner. The observed `split_file.tests.test_query_profile.0` under `@default`
  is exactly that output. Even when the original declarer launches, the reconstructed
  prompt omits `%clan(..., tribe=chop)` and loses the tribe.
- The bridge regression test only asserts that `%if` is absent and accepts a mocked
  launcher result whose expected name is supplied by the mock. It never parses the
  actual dispatched identity/clan directives, so the loss passed verification.

The requested newer naming policy is also absent. The current plugin still authors
`split_file.<full-dotted-path>.@`; neither its Aug 23 commit nor the approved
conditional-admission plan changed that to a keyed basename template. The existing
two-stage chop planner already accepts one marker in the clan and one in each member,
including `{@<id>}`, so the plugin can safely produce final names shaped like
`toobig-3j.<basename>.0` once typed dispatch preserves the planned group.

## Design constraints

- Treat identity/group parsing and dispatch reconstruction as shared backend behavior.
  Extend the Rust wire and Python mirror instead of teaching AXE a second parser for
  `%id` and `%clan` syntax.
- Keep condition evaluation, waits, and admission journaling unchanged. A false `%if`
  still allocates no runner, workspace, model request, or agent.
- Make clan-declarer promotion durable. A closure-local "first launch" boolean is
  insufficient because a detached admission coordinator may resume in a fresh process.
- Preserve the legacy non-typed chop launch path byte-for-byte and stay within the
  existing `typed_launch_units` feature flag. This is a repair of enabled beta behavior,
  not a new user choice or a reason to add another flag.
- Keep `toobig-@` clan allocation owned by SASE. The plugin owns only the readable
  basename and a stable keyed member marker; it must not inspect live agents or choose
  concrete tokens itself.
- Use `/sase_repo` for every `sase-core` and `bugyi-chops` checkout access. Do not edit
  installed site-packages or generated memory/provider instruction files.

## Phase 1: Preserve grouped identity through typed launch planning

Repositories: `sase-core`, then this `sase` repo for the Python wire/adapters.

### Shared typed identity contract

- Extend the Rust `AgentUnitWire` with a backward-compatible, structured representation
  of the complete agent identity binding needed after admission. It must distinguish a
  plain ID, clan membership, family attachment, direct tribe membership, and an
  auto-named tribe member without conflating the positional member ID with the final
  agent name.
- Preserve a `%clan` declaration as structured agent-unit data, including its concrete
  clan name, `tribe`, literal summary, and summary-script form where present. Missing
  fields from schema-v1 bundles must deserialize to today's plain-identity behavior.
- Update typed classification to retain all supported `%id` keywords and `%clan`
  arguments while still removing directives from model-facing prose. Reuse the canonical
  directive argument rules and emit field-specific diagnostics for malformed or
  conflicting identity forms rather than silently discarding them.
- Update `agent_unit_dispatch_prompt` to recreate the equivalent `%id` and optional
  `%clan` directives, followed by the existing model/effort/auto/final/hide/wait-runner
  directives. It must continue omitting admission-only `%if` and logical dependency
  waits.
- Mirror the new optional fields in `src/sase/core/agent_launch_wire.py`, JSON
  serialization/deserialization, and PyO3 parity tests. Keep older durable bundles
  readable and stable.

### Dispatch-scoped keyed markers

- Resolve keyed agent-name markers once across the complete expanded typed batch before
  it is split into durable logical units. Persist the concrete result in the plan so a
  delayed or restarted coordinator never reallocates the same key independently per
  unit.
- Preserve literal markers inside fenced code and xprompt-disabled regions, and retain
  the existing collision/namespace rules. Add a regression where a shared keyed clan
  marker appears in a declarer, joiner, wait/reference, and prose and resolves to one
  token for the entire typed dispatch.

### Tests and documentation

- Add Rust unit tests for parse/serialize/dispatch round trips of plain, clan-member,
  clan-declaration-with-tribe/summary, family, and direct-tribe forms, plus legacy JSON
  defaults and invalid conflicts.
- Add Python wire-parity and direct typed-launch tests proving the reconstructed prompt
  retains grouping and that `extract_prompt_directives` reports the same effective
  identity metadata as the submitted prompt.
- Update the typed-admission documentation to state that all identity directives, not
  only the positional name, survive admission and that keyed markers resolve at batch
  creation time.
- Run the `sase-core` repository's focused agent-launch tests and full prescribed check.
  In `sase`, run `just install` before focused typed-launch tests and `just check`.

## Phase 2: Promote the first eligible chop member to clan declarer

Repository: this `sase` repo. Depends on Phase 1's typed identity wire and formatter.

### Durable AXE dispatch metadata

- Extend AXE's per-logical-unit host metadata with the repeated proposal `clan_summary`,
  intended `tribe` (`chop`), concrete clan/member identities, and the originally planned
  declaration role. Keep the metadata keyed by logical unit and derived from
  `PlannedChopProposal`, never from mutable prompt prose or agent names.
- At dispatch time, determine whether that concrete clan generation has already been
  successfully declared. Use durable admission receipts plus the registered clan
  identity as the source of truth so the decision is correct after coordinator restart
  and cannot depend only on in-memory ordering.
- The first eligible member of an undeclared chop clan must dispatch as
  `%id:<full-name>` plus `%clan(<clan>, tribe=chop, summary=[[...]])`. Every later
  eligible member must dispatch as `%id(<member>, clan=<clan>)`. A skipped or
  condition-error unit must never consume the declarer role; an all-skipped batch must
  create no clan.
- Preserve the planned concrete member names, logical waits, model/effort/auto settings,
  chop ownership environment, admission fingerprints, launch descriptors, and once-per
  bookkeeping. Do not re-run name-template allocation inside a detached unit.

### Regression coverage

- Strengthen `tests/test_axe_chop_proposal_launch.py` so injected launchers parse and
  assert the actual dispatched `%id`/`%clan` directives instead of returning an
  unverified expected name.
- Cover: original declarer eligible; original declarer skipped and the next member
  promoted; several leading skips; all skipped; a condition error; a fresh coordinator
  process resuming before the first launch; resume after a prior member launched; and a
  launch failure without an accidental second declaration.
- Add an end-to-end batch shaped like `toobig_split` and assert launched metadata shows
  the concrete `toobig-*` clan, `clan_tribe == "chop"`, the preserved summary, and no
  member under `@default`. Keep existing non-typed proposal tests unchanged.
- Update AXE docs to explain that admission-time filtering promotes the first surviving
  proposal to declarer, matching once-per filtering's existing behavior.
- Run `just install`, the focused AXE/typed-admission suites, and `just check`. Because
  this changes a durable launch entry point and the Rust wire, run `just check-full`
  only through `/sase_monitor` with `TESTING`/`TESTED` statuses and a concrete
  follow-up.

## Phase 3: Emit keyed basename templates from `bugyi-chops`

Repository: `gh:bbugyi200/bugyi-chops`, opened through `/sase_repo`. This phase can be
implemented in parallel with Phases 1-2 but cannot be deployed until Phase 2 lands.

### Naming policy

- Change `_agent_name(path)` to use the sanitized file basename without its `.py`
  suffix, followed by one qualified keyed marker. Use the existing stable path digest
  (or the proposal's equivalent alphanumeric stable ID) as the marker key, for example
  `test_query_profile.{@a1b2c3d4e5f6}`. Keep `CLAN_TEMPLATE = "toobig-@"` so SASE's
  two-stage allocator produces `toobig-3j.test_query_profile.0`.
- Do not hardcode `.0`: two different paths with the same basename must allocate `.0`
  and `.1` within the same concrete clan. Do not include `split_file.` or parent
  directory segments in the new member name.
- Keep the digest-based proposal ID, wait chain, `%if`, `@medium` routing, priority,
  workspace, summary, and scanner behavior unchanged.

### Tests, docs, and package contract

- Assert raw proposals contain exactly one keyed marker whose key is stable for a path,
  distinct paths with the same basename remain collision-safe, and SASE planning yields
  concise concrete names with one shared clan.
- Extend the SASE-bridge integration test to inspect the actual dispatched prompt and
  effective clan/tribe metadata, not a mocked name. Test a skipped first proposal so the
  next basename member becomes declarer.
- Update the README's concrete-name examples and ownership explanation. Adjust package
  version/dependency metadata and `uv.lock` only as required by this repository's
  release convention and the first SASE release containing Phases 1-2.
- Run `just install` and `just check` in the opened `bugyi-chops` checkout against the
  compatible SASE/core sources or release.

## Phase 4: Deploy and exercise the repaired chop end to end

Depends on Phases 2 and 3.

- Land/publish `sase-core` first, then refresh and land SASE against that core, then
  update `bugyi-chops`. Reinstall the plugin into the same managed uv tool environment
  as the running SASE executable; verify its direct-url/version metadata points at the
  new commit rather than editing site-packages.
- Run `sase axe chop doctor` and a verbose `toobig_split[sase]` dry run. Confirm the raw
  result contains the keyed basename template, repeated clan metadata, and `%if`, while
  the planned preview contains concrete concise names.
- With no active `toobig-` clan, perform one controlled live run. Verify ACE and
  `sase agent list` show agents named `toobig-<token>.<basename>.<token>` inside the
  `@chop` panel with the clan summary intact.
- Exercise the regression path by making or selecting a later queued target whose
  predicate skips after its predecessor. Confirm the first surviving eligible member
  declares the clan, no skipped unit allocates resources, later members join the same
  clan, and the AXE run settles successfully.
- Capture the run ID, admission summary, and launched identities as durable verification
  evidence. Do not hand-edit run history, clan registry state, or once-per state.

## Acceptance criteria

- Typed launch plans preserve and reconstruct every supported `%id` relationship and
  `%clan` declaration; old durable plans remain readable.
- Keyed markers are resolved once per complete typed dispatch, not once per admitted
  unit or coordinator process.
- Conditional AXE filtering promotes the first eligible chop proposal to declarer. Every
  launched member has the same concrete `toobig-*` clan, `clan_tribe=chop`, and the
  intended summary; no member falls into `@default`.
- `bugyi-chops` authors a stable keyed basename template and produces names shaped like
  `toobig-3j.test_query_profile.0`, with duplicate basenames safely disambiguated.
- Focused tests, `just check` in each changed repository, monitored SASE
  `just check-full`, doctor/dry-run validation, and one controlled live admission all
  pass.

## Risks and mitigations

- **Durable wire compatibility:** new Rust/Python fields are optional with explicit
  legacy defaults, and parity fixtures cover old and new JSON.
- **Declarer races or coordinator restart:** promotion consults durable receipts and
  registered clan state under the existing admission/launch serialization; it does not
  rely on process-local memory.
- **Keyed names becoming stale while waiting:** resolve once into the durable plan and
  retain normal launch-time collision failure rather than silently choosing a different
  clan/member after approval.
- **Cross-repository rollout skew:** publish core before SASE and SASE before the
  plugin; doctor and dry-run checks precede the live run.
- **Ambiguous basenames:** the keyed suffix, not parent directories or a hardcoded
  number, provides deterministic collision-safe allocation within the clan.
