---
tier: epic
title: Finish the remaining publication recovery proof
goal:
  Prove named historical publication requests on Athena and Apollo and close only the
  enclosing phase sase-1nv.5.
parent_bead: sase-1nv.5
phases:
  - id: exact-selection
    title: Add exact-request selection to the Rust publication policy
    depends_on: []
    description:
      "exact-selection: add an optional exact logical key to bounded drain selection,
      preserve existing default selection, and prove that missing or ineligible keys
      never fall through to unrelated work."
    size: medium
  - id: targeted-cli
    title: Expose exact publication retries through the existing CLI
    depends_on:
      - exact-selection
    description:
      "targeted-cli: wire the landed Rust selection into sase agent sync, add parser and
      real-outbox regression coverage, document the option, and ratchet the core pin."
    size: medium
  - id: readiness
    title: Verify host readiness and identify three publishable stopped requests
    depends_on:
      - targeted-cli
    description:
      "readiness: install only missing landed fixes through the approved update gate,
      recover Apollo replay only if still necessary, and publish a read-only preflight
      ledger naming one eligible stopped request per host/project."
    size: medium
  - id: athena-proof
    title: Prove one formerly failing Athena timeout request
    depends_on:
      - readiness
    description:
      "athena-proof: spend the single exact-request Athena/sase canary, prove
      publication within 120 seconds with revision/page/prompt and fetched-remote
      validation, and preserve unselected rows."
    size: medium
  - id: bob-proof
    title: Prove one formerly failing Apollo bob-cli request
    depends_on:
      - athena-proof
    description:
      "bob-proof: spend the single exact-request Apollo/bob-cli rebase-conflict canary
      and prove bounded publication, remote validity, and preservation of recovered
      unique content."
    size: medium
  - id: apollo-proof
    title: Prove one formerly failing Apollo SASE request
    depends_on:
      - bob-proof
    description:
      "apollo-proof: spend the single exact-request Apollo/sase manifest-mismatch
      canary, including retired eligibility when needed, and prove bounded publication
      without revising unrelated requests."
    size: medium
  - id: acceptance
    title: Publish the final recovery acceptance and downstream handoff
    depends_on:
      - apollo-proof
    description:
      "acceptance: consolidate the three canaries with existing checks 2-5, publish the
      updated report and remaining-backlog commands, and prepare closure evidence for
      the child land agent."
    size: small
proposed_by: bbugyi200.athena.sase-1nv.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-26 18:04:36
status: wip
---

- **PROMPT:**
  [prompts/202610/finish_publication_recovery_proof.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_publication_recovery_proof.md)

# Finish the remaining publication recovery proof

This child epic completes only the remainder of `sase-1nv.5`. Its previous workers
installed the fixes, repaired production revisions, and proved automatic publication,
but left acceptance check 1 missing. Note #2 and the 24 October report record three
failed canaries. This is substantive remaining work across a shared selector, two hosts,
and three publication targets; it does not clearly fit one coding agent. The user
explicitly requires a child epic in this situation. Do not repeat the completed
installation/repair investigation or launch another single-agent continuation loop.

No beads were created by this planner. The plan handoff owns child creation. Workers
record discovered work only as `PROPOSED FOLLOW-UP:` notes on their assigned phase.

## Evidence and fixed decisions

Read these through the audited interfaces before relying on them:

- `sase bead read sase-1nv.5 -r "Need the enclosing phase and its remaining work"`.
- `plan:202610/unblock_sase_1nf.md`, especially `pub-closeout`.
- `plan:202610/publication_recovery_proof.md`, especially the evidence contract.
- `research:202610/publication_recovery_acceptance.md`, including its 24 October
  follow-up. Open `sase--research` through `/sase_repo` if the report initially resolves
  as missing in a fresh workspace, then retry the audited artifact read.
- `file:explicit:ddc1ff91f98de4e168df0d63`, the complete recovery evidence archive.
  `file:explicit:0ad1354ad22ce3480a1a422c` is the earlier capture. Read artifact
  metadata with `sase artifact read` before inspecting archive members.
- `sase-1nf.11` for the downstream bulk-drain ownership and commands.

The ancestor decisions remain final: `check1_canary = fresh_canary`,
`ssh_fix = apollo_identity`, `gc_measure_scope = close_now`, and `gc_soak = no`. This
plan keeps check 1 as required acceptance; it does not substitute the rejected defer
branch. No memory edits are needed or authorized.

The 24 October evidence establishes:

| Target         | Selected request                                                                             | Result                                                                                                                                                      |
| -------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Athena/sase    | `bbugyi200.athena.research.0o.cdx@95827e05e1dfefcf1d3a8b8c5563f5ff551384fb`, stopped timeout | Revived; page did not materialize; no push or fetched-remote validation. The roughly 44-second call failed; this was not proof of a new 120-second timeout. |
| Apollo/bob-cli | `bbugyi200.apollo.bob-cli-ac.land@1257c2e8d90c…`, active `unknown`                           | Rebase failed applying `a090a3a2c`; no historical rebase-conflict request was tested.                                                                       |
| Apollo/sase    | `bbugyi200.apollo.a4@69c625f8d9af403f7eacdb29c75762ece57910a0`, active `other`               | Page-completion gate failed. Fetched snapshot carried `bbb01e1faec78ac570ff58342aa201aec0cb75b2`, not the requested revision.                               |

All three limit-1 calls failed. No batch followed. Treat that old budget as closed; the
absence of a batch is not permission to run one after a regression.

The repair already preserved all unresolved rows and audited the supported changes.
Athena recorded 996 repaired, 469 linked, and 6 merges; Apollo/bob-cli recorded 56
repaired and 113 linked; Apollo/sase recorded 26 repaired and 52 linked. Do not run the
whole repair again merely because an old row remains unresolved. Do not broaden revision
association, invent commit evidence, or discard unresolved requests.

Check 5 already has the natural completion of `bbugyi200.athena.13y`, commit
`201d1cf4ef3b1e7c7045ae936ae5c89cc8b2266d`, run `run-19c27c1caa8a1818717f9afbdc3a7872`,
and fetched agents revision `70d09617414bfe14282ff9506ed5debc9ed01171`. Preserve and
audit that correlation; do not launch another agent just to replace adequate evidence.

Read-only checks on 26 October 2026 found both hosts installed at SASE
`f4591e9106830fb5e558d71f506b4a2d9fe9791b` and core
`d14a32c8242aea936f39711a8452ef0ba08b121d`, with clean reported source trees. Apollo
still greets `bbugyi200` under default SSH forwarding (`ssh -T` exits 1 for GitHub's
normal no-shell response). Athena's doctor snapshot showed sase with 3,467 requests: 0
active, 3,465 quarantined, and 2 retired. These are planning observations, not remote
publication proof. Refresh before any operation. The planner made no production retries,
repairs, installs, source edits, or memory edits. `sase bead epic-symbols sase-1nv.5`
reported no entries.

## Why exact selection is necessary

Current Rust `agent_publication_recovery/select.rs::select_publication_drain` selects
all active candidates before stopped candidates. `error_classes` filters stopped rows,
while active rows remain eligible regardless of that filter. This is existing deliberate
behavior, also described in the older proof plan; do not silently change default drains
to reverse it.

The wire has a `preferred_key`, but that preference falls through to other rows when the
requested key is absent or ineligible. The bounded CLI does not expose it.
`git_sync.py::_sync_bounded_publication_retries` calls
`select_and_revive_agent_publications`, then drains its exact returned keys. Keep that
pipeline and add an explicit exact-key constraint. This lets an operator inspect and
select a fulfillable historical request without spending a canary on an unrelated active
row or on an unresolved legacy row.

This is not a request to repair every historical research or legacy run. The old Athena
research request and Apollo `a4` may remain unresolved, with their evidence and durable
obligations preserved. A different named stopped request from the required historical
class satisfies the original acceptance contract when its evidence is real.

## Shared rules and verification

- Open every other repo with `/sase_repo`, use only its printed path, and read its
  `AGENTS.md`. Read sidecar context through `sase artifact read`. Mutation APIs own
  production outboxes, manifests, progress files, and the hidden machine clones.
- Shared selection policy belongs in `sase-core`; Python is a thin binding and IO
  adapter. Preserve default wire behavior with additive serde defaults and binding
  round-trip tests. Update sase's core revision pin after the Rust commit lands.
- Read `cli_rules.md` before adding the option, `docs_writing.md` before editing docs,
  and `lint_and_test.md` before finishing tracked sase changes. Run
  `sase tool run check` in each changed code repo. Never run `check-full` or bare cargo.
  Use the documented recipe for local bindings and `/sase_monitor` for long builds,
  tests, recoveries, profiles, and canaries.
- Use `/sase_gate` for necessary `sase update` operations, with the exact missing landed
  commits identified first. Never global-install directly. Apollo commands use the
  `tailnet.md` login-shell convention.
- Preserve exact revision matching, prompt obligations, foreign-owner content,
  publication protection checks, and the ordinary 120-second drain bound. Never clear
  protection records or bypass scanning to obtain a passing canary.
- Do not full-sync, retry all, drop retired rows, bulk-drain, or increase a bound. The
  large backlog belongs to `sase-1nf.11`. Do not stop or relaunch waiting agents.
- Evidence for each mutation includes UTC start/end, argv and exit code, installed
  code/core revisions, before/after outbox snapshots, selected keys and prior states,
  requested/original/fulfilling revision and scope, attempts and prompt obligations,
  stage timings, fetched remote SHA, validation result, and representative paths.
  Preserve raw evidence as explicit artifacts on the responsible child phase.
- Compare every preexisting unselected key's lifecycle, attempts, revision, and error
  history. Distinguish concurrent additions/automatic publications using their evidence;
  do not hide changes behind aggregate totals.
- Remote acceptance uses `load_validated_publication` against a tree at the recorded
  fetched revision, through an opened repo. A modified local tree or an absent outbox
  row alone is insufficient. Never spoof completion or acknowledgment.
- Before closing a child phase, resolve its `epic-symbols`. A check failure proven
  identical on a clean base is a `PROPOSED FOLLOW-UP:` with evidence, not a reason to
  strand otherwise completed phase work. Behavioral acceptance failures remain failures;
  do not mark an unproven canary passed.

## Phase exact-selection

Implement only the shared selector extension in the linked `sase-core` repo.

1. Add an optional exact logical-key constraint to
   `PublicationDrainSelectionRequestWire` in
   `crates/sase_core/src/agent_publication_recovery/wire.rs`. Use the existing
   `PublicationLogicalKeyWire`, with `#[serde(default)]`; absence preserves current
   behavior. Keep `preferred_key` semantics for existing callers. Reject ambiguous
   simultaneous exact and preferred constraints rather than silently ignoring one.
2. In `select_publication_drain`, constrain eligibility to that key when supplied. Never
   fill unused capacity with another key. Preserve lifecycle permissions, existing key
   validation/deduplication, and error-class filtering for stopped rows. An absent or
   ineligible exact key returns an empty selection without touching another row. An
   active exact key may resume only itself; it does not establish formerly-stopped
   acceptance without the durable prior-state ledger.
3. Test a matching stopped row behind unrelated active rows, missing key, wrong error
   class, retired row without permission, duplicate/invalid identities, active exact
   resumption, and default behavior without the new field. Exercise the binding round
   trip and old-wire deserialization. A populated preferred key must remain a preference
   when no exact key was supplied.
4. Run the core check and land through `/sase_final`. No production queue calls.

## Phase targeted-cli

Expose the landed policy through the existing bounded sync command; no alternative
publisher or Python selector.

1. Read the CLI rules and select an available short alias. Add `--retry-key` taking one
   `GLOBAL_AGENT@REVISION` identity, with the full revision as stored in the outbox.
   Reject malformed identities and combinations without exactly one project,
   `--retry-limit 1`, and an applicable retry lifecycle flag. Do not add positional
   fallback or implicit full sync.
2. Thread the parsed identity through the command handler, `sync_agents`,
   `_sync_bounded_publication_retries`, `select_and_revive_agent_publications`, and the
   Rust facade. Selection/revival stays within the existing outbox transaction; the
   existing selected-key drain remains the publication path.
3. Emit clear selected/no-match diagnostics including the exact key. Missing,
   concurrently acknowledged/rekeyed, or ineligible keys are bounded no-ops; never
   silently substitute a new request. A preview is not an authorization to ignore
   current state at mutation time.
4. Add parser/handler tests and real-outbox regression tests showing a stopped timeout
   target is selected despite unrelated active rows, that unrelated
   quarantined/retired/active and referenced-by rows remain unchanged, and that a
   missing key does no publication. Keep existing failed-push/resumption tests. Tests
   must exercise the real Rust selection/revival policy, not stub it green.
5. Document the additive option and exact-key preflight in
   `docs/contribute/agents-sidecar-sync.md` and the applicable CLI/reference docs. Keep
   the existing active-first description for commands without this option. Ratchet
   `sase-core-revision.txt`, install the workspace binding through the prescribed
   recipe, run the required check, and finalize. No production retry.

## Phase readiness

This is one operational preparation unit. It does not spend a retry canary.

1. Compare fresh installed revisions with the two landed phases. Request only necessary
   updates through one gate, then verify the actual CLI option and core on both hosts in
   its follow-up. Reconfirm Apollo identity.
2. Open Apollo's bob-cli agents target through `sase repo open agents -p bob-cli` and
   inspect current HEAD/upstream/worktree plus the saved recovery progress. Determine
   whether the `a090a3a2c` replay conflict still exists; the old report already found
   that commit on the fetched remote, so do not assume it is unique unpublished content.
   Do not reset a healthy clone to historical state.
3. If stale replay persists, use the existing `tools/agents_publication_recover_replay`
   report-first/apply protocol against the observed full HEAD (`-H`), with a dated
   durable verified backup. Preserve unique owner runs and pending objects, require
   empty unknown/refused divergence lists, and resume the same backup if interrupted.
   Apply through the existing recovery API; never hand-reset, hand-commit, or manually
   resolve production conflicts. Verify fetched-remote content and second-apply
   `already_complete`. If already healthy, capture equivalent current evidence and skip
   mutation.
4. Build a read-only candidate ledger for Athena/sase `timeout`, Apollo/bob-cli
   `rebase_conflict`, and Apollo/sase `manifest_mismatch`. Select one stopped row per
   target whose local source run, exact effective revision (or linked scope), page
   generation, and prompt obligation are actually supported by current inventory and the
   saved repair evidence. Use existing inventory/validation and pure Rust selection
   APIs; do not revive rows during inspection. A dry selection of the full queue with
   the new exact key must return exactly that eligible identity.
5. Save each full original/effective key, lifecycle/class, attempts, source run,
   commits, prompt evidence, hood, repair audit identity if applicable, and the exact
   future CLI command. Inspect local candidate payload and current protection status
   before declaring ready. Do not infer source absence means no prompt. Reject
   unresolved candidates such as the old research/a4 rows unless new independent
   evidence actually resolves them. Do not fix thousands of rows to find one qualifying
   row.
6. If a target has no provably fulfillable historical candidate, preserve that finding
   and identify the smallest concrete source/repair defect with a local reproducer
   before further planning. Do not spend the canary to diagnose a known missing
   revision. Do not close readiness while its ledger is incomplete.
7. Publish an explicit readiness artifact and ledger on this phase. Code changes
   discovered necessary here are not silently folded into operations: a narrow repair
   needs its regression/check evidence; a remainder too large for one coder requires the
   same child-epic rule as the enclosing task.

## New canary authorization and command ledger

Approval of this child plan authorizes **one new limit-1 invocation per target**, after
readiness passes, constrained to its ledger's exact key. This is an explicit new
allowance after the failed 24 October allowance, not reuse of unspent batches. There is
no additional batch allowance and no budget reset across continuations. All three proof
phases are serialized because the hosts share publication remotes.

The planned commands are:

```sh
sase agent sync --project sase --retry-quarantined --retry-limit 1 --retry-error-class timeout --retry-key '<athena-ledger-key>' --json
sase agent sync --project bob-cli --retry-quarantined --retry-limit 1 --retry-error-class rebase_conflict --retry-key '<apollo-bob-ledger-key>' --json
sase agent sync --project sase --retry-quarantined --retry-retired --retry-limit 1 --retry-error-class manifest_mismatch --retry-key '<apollo-sase-ledger-key>' --json
```

The latter two run on Apollo using the documented login shell. Before invoking, record
and revalidate the key, current lifecycle/class, command, and budget state in durable
evidence. Natural publication of that row before invocation is not a reason to retry a
missing key or silently choose another; prove it from equivalent timing/remote evidence
if available, otherwise update the read-only ledger before spending the still-unused
call. No new allowance is created if an invocation has already started. A monitor
continuation reads the original command result; it does not start another command.

## Phases athena-proof, bob-proof, and apollo-proof

Each phase owns only its corresponding command above and the shared evidence contract.
Read predecessor evidence and do not run the next canary after a regression.
Source/identity checks and the complete queue snapshot happen first; run the long
operation through `/sase_monitor` from the start.

Require an acknowledged, named, formerly stopped request and elapsed drain time at or
below 120 seconds. Prove its exact effective revision and run/page, resolve its prompt
obligation, fetch and validate the remote revision, and compare unselected rows.
Preserve the recorded original revision after repair. Report stage durations,
selected/result diagnostics, push outcome, and remaining counts. Check 1 is not
satisfied by unrelated active-row publication or by queue absence.

For bob-cli, revalidate the preserved unique runs and pending archive objects from the
readiness replay against the remote; a backup alone does not establish preservation. For
Apollo/sase, validate the repaired manifest and all referenced payloads, including the
selected run. Confirm the fate of `f2d32a476b` from fetched ancestry/content evidence,
or state exactly why it remains unconfirmed. Confirm `a090a3a2c` similarly without
force-pushing either clone.

If a canary fails or a new regression appears, stop further retries, preserve the
pending obligations and command evidence, and diagnose it with local reproduction. Never
claim check 1 or close an incomplete proof phase merely to wake the bulk drain. The plan
does not authorize another production invocation after failure; any necessary further
allowance must be explicit in the ensuing review, after the cause is fixed. Do not leave
an unsupported promise to resume later.

## Phase acceptance

Open the research sidecar, audit the existing report, and append a dated closure section
through normal finalization. Keep historical failures intact. Include:

- All five checks, with the three named check-1 request proofs and artifact refs.
- The existing production replay and real-Git fixture evidence for checks 2-4. Run
  relevant existing fixtures if recovery implementation changed; do not repeat completed
  production repairs to manufacture new evidence.
- The check-5 `13y` natural-completion evidence and exact revision correlation.
- Original repair counts, current per-target backlog counts with UTC timestamps,
  unresolved/protected counts, and a complete old/new canary command ledger.
- Exact bounded next-step commands for `sase-1nf.11`, using the landed selection
  semantics and documented preflight; leave its bulk draining to that phase.
- The status of both formerly stranded Apollo commits and any preserved objects.

Attach the final report and immutable evidence to this child phase and `sase-1nv.5`.
Prepare a concise ancestor-closeout evidence note for `sase-1nv`'s land agent covering
`sase-1nj`, `sase-1nf.10.1.9.1`, and `sase-1nf.10.1.9`. Do not close those beads or edit
their plan status as a child phase worker.

## Child land agent

Verify all child work landed, all required checks have evidence, and the acceptance
report is durable. Audit proposed follow-ups under the enclosing epic's rules. Complete
this child epic through its normal land workflow.

The user explicitly delegates closing the enclosing phase to this child land agent. Run
`sase bead epic-symbols sase-1nv.5`; resolve or re-key every exemption to a genuinely
open owner. Then close **only the enclosing assigned phase** with:

```sh
sase bead close sase-1nv.5 --note '<verified checks, three named canaries, report and evidence references>'
```

Record the prepared ancestor-closeout handoff on `sase-1nv.5` for the owning `sase-1nv`
land agent. Do not close `sase-1nv`, `sase-1nj`, `sase-1nf.10.1.9.1`, `sase-1nf.10.1.9`,
any higher ancestor, or `sase-1nf.11`, `sase-10x`, `sase-1nf.14`, or `sase-1f3` from
this child epic. Instructions in older plans to close other beads are preparation and
evidence for their authorized land agents, not an expansion of this delegation.
