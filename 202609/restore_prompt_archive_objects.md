---
tier: epic
title: Publish remaining prompt-archive objects
goal: Restore and publish the remaining content-addressed objects linked by archived
  prompts so the new archive validator and just check pass.
parent_bead: sase-196
status: wip
phases:
- id: apollo-object
  title: Restore Apollo's missing prompt-archive object
  size: medium
  depends_on: []
  description: 'apollo-object: recover the exact object linked by an already-published
    sase prompt, then publish it through the fixed agents sync path.'
- id: bob-cli-objects
  title: Publish Bob's pending prompt-archive objects
  size: small
  depends_on: []
  description: 'bob-cli-objects: publish the two hash-valid pending bob-cli agents-sidecar
    objects through the fixed agents sync path.'
proposed_by: bbugyi200.athena.sase-196.land
create_time: 2026-09-25 12:06:22
bead_id: sase-196.6
---

- **PROMPT:** [prompts/202609/restore_prompt_archive_objects.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/restore_prompt_archive_objects.md)
- **PARENT:** [202609/agents_sidecar_orphan_objects.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_sidecar_orphan_objects.md)
- **BEAD:** [sase-196.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-196/sase-196.6.md)

# Remaining work for sase-196

The epic's code phases are complete. On 2026-09-25 the three previously untracked
objects in the `sase` agents sidecar were published as `0db40ded93`, but the validator
from this checkout still reports one `artifact-missing` error: archived prompt
`prompts/202609/bbugyi200.apollo.2.md` links to
`files/objects/sha256/41/412ed4ed462f3f76938f9846b24973ee9d2783d09fa7a3e747e24dd2581b6268`.
The exact object exists as an untracked file in Apollo's `sase` agents sidecar. Apollo's
installed SASE is at `1523580bc`, before the publication fix. The original epic plan
also records two pending objects in the `bob-cli` agents sidecar.

## Phase apollo-object

Use `sase repo open agents -r "..."` on athena and, through SSH, run Apollo's
`/home/bryan/.local/bin/sase repo open agents -p gh_sase-org__sase -r "..."` from its
`sase` primary checkout. Use only the printed paths. Check that the Apollo object is a
regular file and its SHA-256 equals its filename. Copy those exact bytes to the matching
path in athena's agents sidecar and verify the destination hash. Preserve the Apollo
copy. Do not invent content or alter the archived prompt.

Run the current checkout's `sase agent sync -p sase` (using its editable virtualenv
command if the globally installed `sase` is stale) to publish the restored object
through the new pre-pull pending-object sweep. Confirm the sidecar worktree is clean,
the object is tracked and pushed, and this checkout's `sase agent prompts validate`
reports no errors.

## Phase bob-cli-objects

Open the `bob-cli` agents sidecar with `sase repo open agents -p bob-cli -r "..."`. If
its two pending content-addressed objects remain, verify their hashes and publish them
with the current checkout's `sase agent sync -p bob-cli`. Confirm a clean sidecar and no
archive validation errors for that project. Avoid touching unrelated dirt.

## Child-epic landing

Run `sase tool run check` in the `sase` checkout. Run `just check` only; do not run
`just check-full`. Resolve any failure introduced by sase-196. Once the three old
objects and Apollo object are published and validation passes, close existing task
`sase-17u` with a note identifying the publication commits. Task `sase-190` was already
closed after phase `sase-196.5` fixed its placeholder-message guard.

Leave the close of `sase-196`, its Symvision check, and its plan status update to its
land agent after this plan completes.
