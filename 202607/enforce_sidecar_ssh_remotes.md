---
tier: tale
goal:
  Ensure every SDD sidecar clone uses an approved SSH or local Git remote,
  canonicalizing legacy GitHub HTTPS metadata and refusing any HTTP(S) URL before Git
  runs.
create_time: 2026-09-09 19:53:11
status: wip
---

# Plan: Enforce SSH remotes for sidecar clones

## Context

Fresh numbered-workspace launches now recreate the required plans sidecar from the
authoritative remote recorded in the durable SDD store. The clone helper currently
passes that recorded value directly to `git clone`. Although configured GitHub sidecars
are resolved elsewhere with canonical SSH URLs, a legacy store record can still contain
`https://github.com/<owner>/<repo>.git`; the resulting fresh clone and its `origin`
therefore use HTTPS. This is observable in the current plans checkout and violates the
repository's documented SSH transport policy.

The fix must preserve the authoritative-remote and fail-closed launch behavior from the
fresh-workspace change. It must not restore durable-primary clone seeding, add
interactive credential fallbacks, or make launch depend on first running
`sase repo init`.

## Design

1. Add one shared sidecar-remote policy helper near the existing hosted-remote parsing
   and canonical SSH utilities. Given the recorded remote plus its
   provider/host/repository identity, it should:
   - convert legacy GitHub HTTP(S) URLs to the canonical SSH form
     (`git@host:owner/repo.git`, or `ssh://git@host:port/owner/repo.git` for a
     configured SSH port);
   - preserve already-valid scp-style SSH, `ssh://`, and local filesystem remotes;
   - reject remaining HTTP(S) remotes with a clear materialization diagnostic instead of
     guessing provider-specific SSH details;
   - compare repository identity transport-neutrally so an existing HTTPS clone of the
     same GitHub repository can be retained and have its `origin` rewritten in place.

2. Apply that policy when resolving materialized SDD store metadata for both plans and
   research. A legacy GitHub record must yield canonical SSH URLs to every consumer,
   including launch-time `ensure_workspace_sdd_clone`, linked-sidecar materialization,
   inventory, and retained-clone synchronization. Keep read-only resolution free of
   durable file writes; the existing initialization path can continue persisting
   normalized records when it runs, while launches become safe immediately.

3. Add a defense-in-depth check at the actual remote clone boundary in
   `sase.sdd._store_link`. Before invoking Git, refuse `http://` and `https://` inputs.
   Strict launch/setup callers should receive `SddMaterializationError`; best-effort
   callers should warn and return without creating or leaving a target. Preserve
   `GIT_TERMINAL_PROMPT=0`, partial-clone cleanup, timeout/error diagnostics, and the
   no-local-fallback guarantee.

4. Update the sidecar storage, workspace launch, and configuration documentation to
   state that fresh clones use canonical SSH remotes, legacy GitHub HTTPS records are
   normalized at resolution time, and unresolved HTTP(S) sidecar metadata fails before
   Git executes. Clarify that rerunning `sase repo init` persists the migrated record
   but is not required to make a launch safe.

## Regression coverage

- Add store-resolution tests for plans and research records containing legacy
  `https://github.com/...` URLs, asserting canonical `git@github.com:...` output.
- Cover GitHub Enterprise host/port handling and already-canonical SSH URLs so
  normalization does not damage valid provider metadata.
- Extend the fresh-sidecar lifecycle regression to start from a legacy HTTPS plans
  record and assert that the captured `git clone` command and resulting `origin` use the
  exact canonical SSH URL, never the recorded HTTPS value.
- Add a direct clone-boundary regression proving an unexpected HTTP(S) URL cannot invoke
  Git, fails closed in strict mode, and leaves no partial target.
- Cover a retained matching clone whose origin is HTTPS and assert it is normalized to
  SSH in place without replacing the checkout or losing local state.
- Preserve local bare-repository fixtures and non-HTTP custom provider transports to
  ensure transport enforcement does not break offline or provider-owned workflows.

## Validation

Run the focused SDD store, sidecar remote normalization, linked-repository resolution,
and launch setup suites. Then run `just install` followed by the repository-required
`just check`, reporting any unrelated pre-existing failures separately from the new
transport-policy coverage.
