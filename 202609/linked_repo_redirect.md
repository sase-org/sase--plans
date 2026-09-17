---
tier: tale
title: Prefer linked checkouts for external repository references
size: medium
goal:
  Make external references to configured linked repositories open the linked checkout
  automatically, with a clear explanation and preservation of existing work.
proposed_by: bbugyi200.athena.0ma
create_time: 2026-09-17 10:05:09
status: wip
---

# Prefer linked checkouts when opening external repository references

## Outcome and scope

When an agent runs `sase repo open gh:sase-org/sase-core -r "Inspect bindings"` from a
project that links that GitHub repository, open the configured `sase-core` checkout for
the selected host workspace. Explain the substitution on stderr and print only the
resulting absolute path on stdout. A previously opened external copy must not prevent
this operation or be modified by it.

Cover both provider references and references to another registered SASE project whose
primary repository corresponds to a configured linked repo. Use the same deterministic
resolution for agent and interactive callers: the CLI already prefers configured
identities for both, and an environment-dependent choice of checkout would make
troubleshooting harder. Do not introduce an agent-detection heuristic, configuration
switch, or force-external option.

This is one medium tale: a bounded change to the existing resolver, its Python adapter
and CLI presentation, tests, and skill source. One implementation agent can complete the
coordinated changes in `sase` and its linked `sase-core` repo.

## Existing behavior and root cause

- `src/sase/main/repo_handler_open.py::handle_open_command` tries configured inventory
  matching before calling `repo_open_external.open_external_repo`. Configured successes
  already use `prepare_opened_checkout(preparation="none")`, record the canonical repo
  name and kind, and print a path.
- `src/sase/main/repo_handler_common.py::match_repo_record` sends exact-name/path
  candidates, then remote identities, through
  `src/sase/core/repository_resolution_facade.py`. It filters out inventory rows of kind
  `external`, which is correct. Identity matching already recognizes `gh:owner/repo`,
  bare `owner/repo`, and supported GitHub HTTPS/SSH remote forms.
- In `sase-core`, `crates/sase_core/src/repository_resolution.rs` returns
  `match_reason`, `matched_id`, and `requested_identity`. The Python helper discards the
  match metadata, so a successful provider-to-linked substitution is silent. Existing
  tests already prove the basic linked selection works.
- After a remote match, `_ensure_no_external_collision` raises an error if a distinct
  external checkout exists. The test
  `test_repo_open_provider_alias_rejects_existing_external_duplicate` explicitly
  enforces this behavior. It makes previous mistakes block automatic correction.
- Registered project references are resolved later in
  `src/sase/main/repo_open_external.py`. A project alias or differently named project
  therefore bypasses linked matching unless it happens to equal the configured linked
  name. It can clone a second copy of the same repository.

## Resolution contract

1. Preserve existing exact configured path/name/slug precedence, host-project filtering,
   and explicit ambiguity errors. Direct linked names remain the preferred spelling and
   require no substitution message.
2. For supported provider identities, preserve the Rust canonicalizer and require an
   unambiguous full identity match. Never infer correspondence from basename alone or
   equate different owners, hosts, or provider schemes. Keep the existing exact-name
   fast path free of remote probes.
3. If configured matching fails and the external reference uniquely names another
   registered project, inspect its primary checkout identity before cloning. Match it to
   the current host's linked repos by normalized primary checkout path, then by
   supported canonical remote identity. An exact source-path match is stronger than a
   remote match. Multiple matches at the winning level are errors with selectable
   configured names/paths; never choose the first entry. Preserve registry
   name/display-name/alias resolution and host exclusion.
4. Once a linked match is established, always take the existing configured open path,
   including lazy materialization, workspace selection, linked markers, and finalizer
   tracking. A linked preparation failure is an error, not permission to fall back to an
   external clone. Identity lookup must precede external materialization, provider
   loading, and successful-open recording.
5. An existing external copy is advisory evidence, not another configured candidate.
   Open the linked repo and warn about the external copy. Preserve that copy's files,
   index, HEAD, untracked files, and any existing open markers; do not delete, reset,
   move, merge, or mark it as newly opened.
6. With no linked match, retain external cloning/reopening and error behavior. Missing
   or unsupported identity evidence is not a match. This change does not expand external
   clone syntax or provider support. Preserve existing primary and sidecar matching and
   collision behavior; the new duplicate override is specifically for linked repos.

## Implementation

### Keep shared decisions in Rust

Open `sase-core` with `/sase_repo` using its configured name and use only the printed
path. Follow that checkout's `AGENTS.md`.

Extend the existing repository-resolution request with an optional resolved
external-project descriptor containing its normalized primary source path and locally
obtained remote URLs. Existing callers can omit it. Resolve these inputs against linked
candidates in Rust after ordinary configured matching has failed; add distinct match
reasons for project-path and project-remote correspondence. Python supplies normalized
filesystem facts and registry records, while Rust owns matching, precedence, and
ambiguity. Keep all existing GitHub parsing in Rust.

Update the wire contract, serde/PyO3 coverage, and facade together. Follow existing
schema-version conventions when changing the contract, and ensure legacy requests
without the optional descriptor still work. The binding lives in
`crates/sase_core_py/src/lib.rs`. Do not introduce a Python fallback or manually bump
crate release versions.

### Carry resolution details to the CLI

Introduce a small structured Python resolution result containing the selected
`RepoRecord`, original request, Rust match reason/identity, and any existing external
checkout paths. Keep the record-only helper as a thin compatibility wrapper where
useful, including its `repo path` consumers; do not print from the matcher. Route
`repo open` through the richer result without repeating matching or Git probes just to
construct a message.

Separate registered-project lookup from external checkout preparation in
`repo_open_external.py` so the command can reuse the same resolved project for either
linked selection or ordinary external cloning. Collect only the source identity needed
for this decision. Retain bounded, noninteractive local Git probes and the
configured-remote/selected-clone/primary-clone lookup behavior.

Refactor external-collision collection into data used by the open handler. Limit it to
the current host project and selected workspace, deduplicate paths, and exclude paths
resolving to the selected linked checkout. Cover both inventory entries and the existing
standard-path probe, plus the registered-project external destination when that spelling
was used. Do not scan every checkout's Git history, discover arbitrary directories, or
contact the network to match an identity. Retain the existing collision error for
primary/sidecar matches.

Continue using the existing configured preparation path with `preparation="none"`.
Record one successful open with the canonical linked name, `repo_kind="linked"`,
selected workspace, actual returned path, and the user's original audit reason. Do not
create an external marker or invoke an external clone provider for a redirect. Retain
any external marker from an earlier open so existing work remains visible to completion
handling.

### Explain the successful substitution

After successful preparation, print a concise informational message to stderr when the
match reason denotes external-to-linked correspondence. Include the requested reference,
configured linked name, host project, actual selected path, and the reason for using it.
For example:

```text
Info: 'gh:sase-org/sase-core' matches linked repo 'sase-core' in project 'sase'. Opened its configured checkout at <path> so edits and repo tracking use the project's linked repo. Next time, use `sase repo open sase-core -r "<reason>"`.
```

If a distinct external copy exists, add:

```text
Warning: An external checkout of this repo also exists at <external-path>. It was left untouched. Continue in the linked checkout printed on stdout; any work in the external copy remains there.
```

Render actual paths and shell-quote suggested repo names correctly. Use the resolved
checkout path rather than predicting it from an inventory row. Preserve stdout as
exactly one path plus newline. Do not announce a successful open if preparation fails.
Direct configured opens and genuinely external opens should not receive this
informational message. `repo path` should not gain open-specific messages as a side
effect of sharing the resolver.

Update `src/sase/xprompts/skills/sase_repo.md` to teach the configured-name spelling and
explain automatic linked selection, stderr notices, and preserved external copies.
Update existing CLI help/docs where they describe resolution precedence; no new command
or option is needed. Edit only the skill source. Generated provider skills are deployed
later from a clean, landed revision according to the generated-skills memory.

## Verification and acceptance

Use small local repository fixtures and mocked providers; do not clone live GitHub repos
or change real external duplicates to test this behavior.

- Rust and binding contract tests: existing exact-match precedence and identity
  normalization; optional descriptor omitted; registered project source-path and remote
  matching; unsupported identity; same basename under another owner/host; ambiguous
  linked candidates; serialized match reasons and schema compatibility.
- Extend `tests/main/test_repo_handler_open_resolution.py` and
  `tests/main/test_repo_handler_open_configured.py` for provider and registered alias
  redirects. Exercise mixed-case GitHub identities, optional `.git` suffix, bare
  shorthand, HTTPS and SSH spellings, lazy linked clones, and a nonzero selected
  workspace. Assert the actual linked path and informational stderr.
- Replace the existing linked-duplicate rejection test with successful selection plus
  warning. Exercise an inventory-listed duplicate and a standard-path-only duplicate.
  Include dirty tracked/untracked files in both checkouts, capture HEAD/index state, and
  verify reopening preserves them. Another workspace's or host project's external copy
  must not generate a warning for this open.
- Verify one canonical linked audit event and linked marker, no external clone call/new
  external marker, and preservation of a preexisting external marker. Use the real
  marker/audit path in at least one integration test instead of mocking all preparation.
  A failed linked open records no success and never falls through to external cloning.
- Retain coverage in `tests/main/test_repo_handler_open_external.py` for unmatched
  provider/project references, idempotent external reopening, missing providers, and
  atomic clone failures. Keep `tests/main/test_repo_path.py` and the configured
  primary/sidecar behavior green. A direct linked-name open has path-only stdout and no
  redirect notice, with or without agent environment variables.

Rebuild/install the changed local Rust binding into the implementing workspace using the
repository's supported Justfile flow, explicitly selecting the checkout opened by
`/sase_repo`; verify that Python tests exercise the new binding. Run `just check` in
`sase-core` (it includes workspace and PyO3 tests; core-only cargo tests are
insufficient). Run focused Python tests above, then `just check` in `sase`, following
`lint_and_test.md`. Use `/sase_monitor` if verification needs a long-running handoff,
and observe the required full-check policy before landing.

Acceptance is an automatic, explained linked open for both supported external reference
classes, including when an external duplicate exists, with correct
workspace/audit/marker behavior and preserved work. Existing duplicate cleanup, new
providers, broad repository-management refactors, and global skill deployment are
outside this tale. No feature flag is needed for this complete correction.
