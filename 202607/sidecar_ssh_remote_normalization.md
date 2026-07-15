---
tier: tale
goal: 'GitHub sidecar repositories resolve, initialize, and converge on canonical
  SSH remotes instead of core-generated HTTPS remotes, including safe repair of existing
  sidecar records and clones.

  '
create_time: 2026-07-15 10:06:59
status: wip
prompt: 202607/prompts/sidecar_ssh_remote_normalization.md
---

# Plan: Normalize GitHub Sidecar Remotes to SSH

## Confirmed behavior and root cause

The HTTPS suspicion is confirmed in both code and live state:

- The host-primary SDD store record currently records both `sase--plans` and `sase--research` as
  `https://github.com/...` remotes, and both host-primary sidecar clones use those HTTPS origins. In the inspected
  numbered workspace, research also uses HTTPS while plans retains an older SSH origin.
- `src/sase/_linked_repo_config.py::_sidecar_repo_identity()` fabricates an HTTPS URL whenever it can derive a GitHub
  `owner/repo` identity but cannot reuse a compatible store URL.
- `src/sase/main/repo_init_handler.py::_configured_sidecar_specs()` forwards that hidden resolved URL into the sidecar
  initialization transaction.
- The GitHub provider already defaults sidecars to SSH, but correctly preserves a caller-supplied remote when it names
  the requested repository. Core's synthesized HTTPS value therefore overrides the provider default; cloning and the
  schema-v2 store record then make HTTPS persistent on later runs.

This was introduced by the generalized sidecar configuration path, not by the GitHub provider's ordinary unpinned
discovery path. The fix belongs in this repository's sidecar identity and reconciliation flow. The linked `sase-github`
repository should remain unchanged unless implementation proves that its existing canonical-SSH contract is
insufficient.

## Implementation

### 1. Make provider-derived GitHub sidecar identities canonicalize to SSH

Refactor sidecar remote derivation in `src/sase/_linked_repo_config.py` so an unpinned or repo-pinned GitHub sidecar
resolves to the provider-compatible SSH form for the selected host and repository, rather than hard-coding
`https://github.com/<owner>/<repo>.git`.

- Preserve the selected repository identity (`owner/repo`) independently from transport.
- Derive the SSH URL with the correct host shape, including the `ssh://` form when a host/port requires it; do not
  silently map an Enterprise host to `github.com`.
- Treat a matching stored HTTPS URL as stale transport when the same GitHub sidecar now has a canonical SSH URL, so
  inventory and initialization receive SSH and can self-heal the record. Continue preserving authoritative stored URLs
  for non-GitHub or otherwise non-derivable providers instead of applying a GitHub-specific rewrite to arbitrary
  remotes.
- Keep repository matching transport-insensitive: SSH and HTTPS URLs for the same host/repository are the same identity,
  while a different host, owner, or repository remains a real cutover and retains existing dirty-clone safety.

Avoid adding a user-facing remote URL setting to `repos.sidecar`; transport is provider-owned and the existing schema
intentionally exposes only the logical repo pin.

### 2. Converge existing clones and the durable store record

Carry the canonical SSH URL through the existing configured-sidecar init path without changing repository-creation
authorization or visibility behavior. `sase repo init` should use the SSH URL returned by resolution/provider, repoint a
clean protocol-equivalent clone with `git remote set-url` rather than recloning it, and rewrite the schema-v2 SDD store
record with the SSH URL.

Also tighten `src/sase/linked_repos.py` materialization so a clone whose origin names the correct host/repository but
uses stale HTTPS is normalized to the expected SSH URL when that sidecar is opened or auto-materialized. This must be an
idempotent local remote update, not destructive replacement. A genuinely mismatched repository must continue through the
current clean/dirty cutover checks, and dirty worktrees must never be discarded.

Do not mutate store records during read-only inventory collection. Durable record migration belongs to the explicit
initialization transaction; ordinary materialization may normalize the touched clone only.

### 3. Add regression and migration coverage

Update the focused tests around sidecar configuration, inventory, initialization, and linked-repo materialization:

- A new GitHub project with SSH or HTTPS primary-origin input resolves its derived `--plans` and `--research` sidecars
  to canonical SSH on the correct host, including an Enterprise/port case if that URL form is supported by the existing
  parser.
- Explicit `repo: owner/shared-sidecar` pins use SSH for that exact repo and do not regress repo-cutover identity
  checks.
- A schema-v2 record containing a matching HTTPS sidecar URL no longer poisons resolution: `sase repo init` supplies SSH
  to the provider, updates a clean existing clone in place, and records SSH afterward.
- Opening/materializing a protocol-equivalent HTTPS clone changes only its origin URL; local commits and files remain
  intact. Re-running the operation is a no-op.
- Non-GitHub/local stored remotes remain unchanged, while different-repository and dirty-clone refusal tests continue to
  pass.
- Replace existing assertions that accidentally enshrine core-generated GitHub HTTPS with the canonical SSH expectation;
  retain HTTPS fixtures where they intentionally test URL identity equivalence or non-GitHub custom remotes.

Add a short configuration/initialization note explaining that GitHub sidecars use provider-owned SSH remotes and that
rerunning `sase repo init` repairs legacy HTTPS store records. No schema or default-config change should be needed.

## Validation and rollout

Run `just install` first, then focused sidecar/config/materialization tests while iterating, followed by the
repository-required `just check`. Re-run the focused tests after the full check if formatting or generated artifacts
changed.

Validate migration against temporary repositories rather than relying only on mocked URL assertions: start with a clean
sidecar clone and schema-v2 record on an HTTPS origin, run the initialization/materialization path, and verify the same
checkout and content remain while both origin and rewritten record become SSH.

After the code is available in the user's installed SASE build, rerun `sase repo init` in the host-primary project to
migrate its durable record and host sidecar origins; then open/materialize plans and research in active workspaces so
their origins converge lazily. Report these rollout commands and their verification (`git remote get-url origin`,
`sase repo list`, and a clean `sase repo init --check`) rather than editing unrelated project or memory files.

## Acceptance criteria

- Newly initialized GitHub sidecars use SSH in their clone origins and SDD store records.
- Existing matching HTTPS sidecars converge in place to SSH without data loss, while mismatched or dirty repositories
  retain current safety guarantees.
- Inventory no longer fabricates GitHub HTTPS for configured sidecars.
- GitHub Enterprise host information is preserved, non-GitHub remotes are not rewritten, and sidecar creation
  confirmation semantics are unchanged.
- Focused tests and `just check` pass.
