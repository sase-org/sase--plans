---
tier: tale
title: Keep repository aliases on the checkout used by host hooks
goal:
  Prevent agents from editing an accidental duplicate of a configured repository and
  then failing finalization against a different checkout.
size: medium
proposed_by: bbugyi200.athena.0ja
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ja](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ja.md)
- **COMMITS:**
  - [354dcd4](https://github.com/sase-org/sase-core/commit/354dcd4d22b69f4c463503ad31fd0ee824d3326e)
    — feat(repo): add canonical repository resolver

# Keep repository aliases on the checkout used by host hooks

## Outcome and scope

When `sase-core` is configured as a linked repository for the current project,
`sase repo open sase-core`, `sase repo open gh:sase-org/sase-core`, and
`sase repo open sase-org/sase-core` must select the same configured checkout in the
current workspace. Opening the repository, testing local changes, and running the host's
commit hook must agree on its identity. Truly external repositories retain their
existing atomic clone and nondestructive reopen behavior.

This is a `tale` with `size: medium`: one coding agent can implement and verify the
bounded resolver change across Rust and Python. It does not need independently scheduled
phases. New shared identity and selection rules belong in `sase-core`; Python owns
inventory collection, subprocesses, checkout materialization, audit records, and CLI
messages.

The task is prevention of the failure found in `sase-zl.2`, not implementation of the
continuation epic. Do not replay that phase, change its status, restart its dependents,
or duplicate its code. Recheck recovery evidence before suggesting any operational
action. Do not change retry budgets, weaken binding validation, change published version
bounds to hide missing capabilities, or delete duplicate checkouts as part of this
repair.

## Investigation and evidence

The failed run is `sase-zl.2`, started on 2026-09-11 at 07:25:33 EDT. Its stable runtime
evidence is under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/11/20260911072533/`.
Inspect `done.json`, `opened_linked_workspaces.json`, `tool_calls.jsonl`,
`final_submission.json`, `finalizer_result.json`, and
`finalizers/commit/attempt-1.main.stderr` as needed. The saved error transcript is
recoverable with:

```sh
sase chat show --basename gh_sase_org__sase-tmp_260911_072714-workflow_tmp_260911_072714_main_ERROR-260911_072718
```

The named `sase chat show --agent sase-zl.2` lookup failed during investigation; the
content-filtered chat list found this error transcript. The published agent page was
stale (`active`), whereas completed `done.json` authoritatively reported `failed`. Use
the latter to interpret this incident.

The causal chain is supported by the completed records:

1. The agent opened `gh:sase-org/sase-core`, which produced a separate external
   checkout. It implemented the twelve new `continuation_*` exports there.
2. Its successful build and verification commands explicitly set `SASE_CORE_DIR` to that
   external checkout. The binding validator exited successfully at 12:19:40 UTC. Later
   checks used the same command-local override.
3. The agent closed its phase bead at 12:39:46 UTC and submitted commit decisions for
   the main repository and the external core repository at 12:40:58 UTC.
4. The host finalizer attempted the main repository first. Its `just fix` hook rebuilt
   from the different, linked core checkout. That source did not contain the uncommitted
   external continuation work. The hook failed with twelve missing `continuation_*`
   bindings, including `continuation_wire_schema_version`, `continuation_validate_node`,
   and `continuation_plan_budget`.
5. The finalizer emitted `stitch_failed`, then `stitch_retry_skipped_identical_inputs`.
   The skipped retry was a consequence; it did not cause the source mismatch. The
   recorded unrelated lint and full test failures were not the terminal hook error.

There is subsequent recovery: opening the configured core repository during this
investigation obtained commit `a5d2609b31ed13143602ae94801f0cbfa1f2c680`, authored at
08:47:19 EDT, titled
`feat: Define the Rust continuation and result contracts (sase-zl.2)`. The PyO3 exports
are present in that committed source. This establishes recovery of the missing Rust
source, not proof that every old environment or the main Python changes have been
recovered. No reinstall or original-agent retry was performed.

The generic defect still exists in inspected SASE commit `ac55d0c87`:
`src/sase/main/repo_handler_common.py::match_repo_record` compares configured names,
slugs, and paths but never remote identity. A read-only reproduction using a linked
`RepoRecord` with remote `git@github.com:sase-org/sase-core.git` returned:

```text
sase-core: configured match = sase-core
gh:sase-org/sase-core: configured match = None; external fallback
sase-org/sase-core: configured match = None; external fallback
```

`repo_open_external.py` consequently materializes a second checkout. The `Justfile`
correctly prioritizes explicit `SASE_CORE_DIR`, then workspace-linked environment paths
and fallbacks; a command-local override cannot update the parent host's environment. Fix
the divergent repository selection at its source.

The audited context also includes `agent:sase-zl.2` and
`plan:202609/monitor_continuations.md`. Read artifacts through `sase artifact read` and
prior chats through the SASE chat skill. Open `sase-core` with the repo skill and
`sase repo open sase-core -r '<specific reason>'`; use only the returned path. Paths
beginning `crates/` below are relative to that opened core repository. All other source
paths are relative to the main SASE checkout.

## Implementation

### 1. Define the identity and selection contract in Rust

Add a focused repository-resolution module in `crates/sase_core/src/`, with a narrow
PyO3 API in `crates/sase_core_py` and a typed adapter under `src/sase/core/`. Reuse
existing suitable normalization helpers where possible, but do not use a basename or a
slugified workspace key as proof of repository identity. Avoid migrating unrelated
inventory or project-key behavior.

The pure operation accepts the requested repository reference and host-observed
candidate identities. It produces a unique configured match, no match, or an ambiguity
result with candidate identifiers. Matching includes the provider and host, owner, and
repository. For GitHub, normalize `gh:owner/repo`, existing bare `owner/repo` input, and
observed HTTPS, SSH URL, and SCP-style GitHub remotes; account for `.git`, a trailing
slash, and GitHub case insensitivity. Reject malformed identities and do not merge
different owners, hosts, or providers. Other provider schemes keep the existing external
fallback.

Keep exact configured names/slugs/paths and their existing ambiguity precedence. Remote
matching is an additional step before external materialization. Multiple configured
candidates with the same verified remote require explicit selection; never pick the
first inventory entry. No Python implementation of the new domain rules or fallback for
a missing Rust binding.

Scope candidates to the current host project and select the caller's workspace; an alias
must never redirect into another numbered workspace. Keep canonical matching keys
separate from the configured display name used in audit records.

### 2. Collect evidence and use the existing configured-open path

Integrate the decision into `src/sase/main/repo_handler_common.py`,
`repo_handler_open.py`, and `repo_open_external.py` as appropriate. Candidate collection
should run only when identity matching is needed, rather than adding Git subprocesses to
every inventory list or TUI refresh.

`RepoRecord.remote_url` is currently populated for sidecars but can be `None` for linked
repositories. Handle the real linked-repo case: inspect the selected workspace clone's
origin, or the configured primary source if the clone is absent, through existing
bounded, read-only Git helpers. Use explicit sidecar remote metadata where available. Do
not infer identity from directory names or recursively follow arbitrary local origins.
Unavailable or local-only origin evidence must not create a false match; distinguish a
failed probe from a proven different remote in diagnostics.

For a unique match, call the existing configured preparation path with
`preparation="none"`. Preserve workspace selection, hidden sidecar rules, dirty files,
and audit behavior. Record the configured repository name/kind and its actual checkout
path, so finalizer discovery sees one repository obligation. Keep stdout as the single
returned path and send any explanation to stderr. Do not add a special `sase-core` path
shortcut or propagate ad hoc shell variables into the host finalizer.

### 3. Handle pre-existing duplicate checkouts explicitly

Before redirecting a provider alias, inspect the current workspace's existing external
identity/marker and its standard destination. If a distinct valid external checkout
already exists for the same identity, return an actionable collision error naming both
paths and the configured name before either checkout is prepared. Require explicit
selection/recovery through existing mechanisms; do not silently abandon a previously
opened external repository, merge changes, clean files, rewrite old markers, or remove
directories. A conservative error even for a clean old duplicate is acceptable for this
bounded fix.

Check for the duplicate before any configured checkout refresh or successful-open marker
write. Recognize equivalent GitHub spellings in existing external records as well as the
standard destination, so case or `.git` normalization cannot hide an already opened
clone. A rejected provider alias must not register a new finalizer obligation. This work
diagnoses old duplicates; it does not migrate them.

Exact configured-name opens remain available to select the linked checkout. External
repositories that have no configured identity match must still reopen without cleaning,
retain their audit identity, and preserve clone-failure cleanup. This is an
unconditional correction to configured-repository selection; no temporary compatibility
branch or feature flag is planned.

### 4. Update the contract documentation

Update the relevant `docs/workspace.md` / `docs/cli.md` repository-open section to
explain that provider references reuse matching configured repositories, how
ambiguity/duplicate errors are resolved, and that build commands and host hooks should
share the configured checkout. Keep the existing CLI syntax; no new subcommand or option
is required. Do not edit canonical memory or deployed skills.

## Regression coverage and verification

Use Rust unit tests, real PyO3 contract tests, and focused Python integration tests.
Avoid network clones or compiling the whole Rust project inside Python fixtures.

- Cover remote normalization, same-repository aliases, different owners/hosts,
  missing/malformed identity, ambiguity, and existing exact-name/path precedence.
- Extend `tests/main/test_repo_handler_open.py` and
  `tests/main/test_repo_handler_open_external.py`: a linked record with
  `remote_url=None` and a GitHub origin must route both provider spellings to its
  configured workspace path, never invoke the external clone hook, preserve dirty files,
  and write canonical opened-repo/audit records.
- Test absent workspace clones using configured source evidence, sidecar remote
  metadata, unreadable origins, and unsupported providers. Verify no unrelated
  repository is selected by matching only its basename.
- Test a pre-existing duplicate with uncommitted content and with a local commit:
  collision exits before mutation, reports both locations, leaves data and markers
  intact, and creates no successful-open record for the rejected request.
- Add an integration regression connecting repository-open output, the recorded
  linked-repo obligation, and `Justfile` selection. Starting with a configured
  `sase-core` GitHub origin, provider-alias open and a fresh subprocess evaluating
  `just --evaluate sase_core_dir` must resolve to the same workspace checkout without
  `SASE_CORE_DIR`. A stubbed hook/validator should see a sentinel change made through
  the returned checkout. This is the behavioral regression for the failure, beyond
  testing string normalization alone.
- Preserve the current external atomic-clone, provider error, nondestructive reopen,
  configured-open, finalizer-linked-repo, and Justfile selection tests.

Build/install the updated Rust binding from the returned linked checkout, then run the
binding scanner and validator and the targeted Python tests. The new binding must also
be covered by `tools/validate_sase_core_rs`. Do not manually change release-managed
Cargo versions. Run the core repository's required full `just check` /
`scripts/check.sh`, including PyO3 tests, and the main repository's required
`just check`. Apply the `lint_and_test.md` escalation rules; if a full lane or a long
command is necessary, use the SASE monitor skill. Record actual unrelated failures
separately without claiming an all-green result.

Before completion, recheck the original phase's published recovery state and report what
is recovered versus still unknown. Submit repository decisions through the host-owned
finalizer for both touched repositories. Acceptance requires the resolver reproduction
to choose one configured checkout, the open-to-hook regression to pass, and genuine
external opens to retain their existing behavior.
