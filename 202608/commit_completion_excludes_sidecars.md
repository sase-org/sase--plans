---
tier: epic
title: '`@commit:` completion stops offering SDD sidecar commits'
goal: '`@commit:` completion — in the ACE prompt bar and in every LSP editor — offers
  only commits from repositories a human writes code in. SDD sidecar repositories
  (`plans`, `beads`, `agents`, `research`) contribute no completion rows, while a
  `@commit:<sidecar>@<sha>` reference typed in full still resolves at launch exactly
  as it does today.

  '
phases:
- id: core_kind_filter
  title: Repository kind on the wire, sidecars skipped in commit enumeration
  depends_on: []
  size: small
  description: 'core_kind_filter: add a defaulted `kind` field to `ArtifactRefRepositoryWire`
    and skip `kind == "sidecar"` repositories in `append_commit_candidates` before
    spawning `git log`, leaving `resolve_commit` and every other kind untouched.

    '
- id: host_repo_kind
  title: The host labels each repository and proves both surfaces agree
  depends_on:
  - core_kind_filter
  size: small
  description: 'host_repo_kind: carry `RepoRecord.kind` into `ArtifactRefRepository`
    and its wire form in `artifact_ref_context()`, extend the prompt-bar/LSP parity
    test with a sidecar repository, assert sidecar resolution still succeeds, and
    correct the two documentation sentences that promise commits from every local
    checkout.

    '
- id: core_floor
  title: Raise the published sase-core-rs floor
  depends_on:
  - host_repo_kind
  size: small
  description: 'core_floor: once the release carrying `core_kind_filter` is published,
    raise the `sase-core-rs` lower bound in `pyproject.toml` so an installed sase
    cannot silently run the unfiltered enumeration.'
proposed_by: bbugyi200.athena.si
create_time: 2026-08-03 06:31:58
status: wip
bead_id: sase-ek
---

- **PROMPT:** [prompts/202608/commit_completion_excludes_sidecars.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/commit_completion_excludes_sidecars.md)
- **BEAD:** [sase-ek](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ek/README.md)

# Plan: `@commit:` Completion Excludes SDD Sidecar Commits

## Goal

`@commit:` completion is meant to answer one question: _which of my recent commits do I want this agent to read?_ Today
it answers a different one, because SASE's own bookkeeping repositories are in the corpus. Sidecar commits are
machine-written (`chore(agents): sync from …`, `chore(beads): close sase-eg`, `Archive approved plan …`), they are the
most frequent commits on the machine, and the menu is recency-ranked — so they arrive first and they arrive constantly.

After this plan, a repository SASE classifies as a **sidecar** contributes zero commit-completion rows in both surfaces.
Nothing else about `@commit:` changes: the grammar, the ranking, the row shape, and launch-time resolution are all
untouched.

## Current state

Established by reading and running the code. Implementing agents should trust the behavior and re-verify line numbers,
which drift.

### Measured impact

Run from a `sase` workspace against the real machine state:

```python
from collections import Counter
from pathlib import Path

from sase.artifact_refs import artifact_ref_context
from sase.sdd.plan_refs import workspace_context_for_plan_resolution
from sase_core_rs import artifact_ref_payload_inventory

workspace, workspace_num = workspace_context_for_plan_resolution(Path.cwd())
context = artifact_ref_context(workspace, workspace_num, "sase")
inventory = artifact_ref_payload_inventory("commit", context.to_wire())
rows = inventory["payloads"]
print(len(rows), inventory["truncated_payloads"])
print(Counter(row["scope"] for row in rows))
```

Observed on 2026-08-03:

| Scope         | Kind     | Rows |
| ------------- | -------- | ---- |
| agents        | sidecar  | 200  |
| beads         | sidecar  | 200  |
| plans         | sidecar  | 200  |
| research      | sidecar  | 27   |
| sase          | primary  | 200  |
| sase-core     | linked   | 98   |
| chezmoi       | linked   | 53   |
| sase-telegram | linked   | 9    |
| bob-cli       | external | 13   |

**627 of 1000 rows are sidecar bookkeeping.** Ranks 0 through 9 are all sidecar rows; the newest `sase` commit lands at
rank 10. The merged timeline is at its `ARTIFACT_REF_COMMIT_MAX_ROWS = 1000` cap with `truncated_payloads = 878`, so
sidecar rows are not merely noisy — they consume the budget that would otherwise carry older commits from the primary
and linked repositories past the cap.

An empty `@commit:` query is therefore useless on this machine, and a query narrow enough to escape the sidecars is a
query the user could have typed the reference for.

### Both surfaces already share one enumeration

There is exactly one place commits are enumerated, which is why this is a small change:

- `sase_core::editor::completion::build_artifact_ref_payload_inventory`
  (`crates/sase_core/src/editor/completion.rs:420`) dispatches `kind == "commit"` to `append_commit_candidates`
  (`crates/sase_core/src/editor/completion.rs:480`), which iterates `context.repositories`, takes `checkout_paths[0]`,
  and runs a bounded `git log` in each.
- **Prompt bar:** `load_prompt_commit_snapshot` (`src/sase/ace/tui/widgets/prompt_commit_inventory.py:49`) calls that
  function through the `artifact_ref_payload_inventory` binding on a background worker.
- **LSP:** `cached_at_reference_payload_inventory` (`crates/sase_xprompt_lsp/src/server.rs:843`) calls the same function
  over the catalog JSON that `_materialize_artifact_ref_catalog` (`src/sase/integrations/xprompt_lsp.py:310`) writes at
  `sase lsp` startup. `sase-nvim` is a pure consumer of that server and needs no change.

### The host knows what a sidecar is; the wire throws it away

`context.repositories` is assembled in exactly one producer, `artifact_ref_context()`
(`src/sase/artifact_ref_context.py:68`), from `collect_repo_inventory()` records. Each `RepoRecord` carries
`kind: Literal["primary", "sidecar", "linked", "external"]` (`src/sase/_repo_inventory_models.py:8`) — the same field
`sase repo list --json` already prints — and the comprehension keeps only `name`, `slug`, and checkout paths.
`ArtifactRefRepositoryWire` (`crates/sase_core/src/artifact_ref/wire.rs:143`) has no field for it either.

That is the entire defect: **core cannot exclude sidecars because the host never tells it which repositories are
sidecars.**

### Resolution reads the same list and must keep working

`resolve_commit` (`crates/sase_core/src/artifact_ref/mod.rs:595`) finds the repository by `name` or alias in
`context.repositories`, and `_resolve_for_launch` (`src/sase/artifact_ref_prompt.py:173`) covers the empty-`shas` case
by resolving `<sha>^{commit}` through that repository's checkout. So `@commit:plans@<sha>`, typed in full, resolves and
expands today. Removing sidecars from `context.repositories` would turn that into an `unknown_repo` launch failure.

## Design

### D1 — The host labels each repository; core owns the policy

Add `kind: String` to `ArtifactRefRepositoryWire`, `#[serde(default)]`, and to the Python `ArtifactRefRepository`
dataclass and its `to_wire()`. The host copies `RepoRecord.kind` verbatim. Core decides what to do with it: a repository
whose `kind` is `"sidecar"` contributes no commit-completion rows.

This is the Rust core backend boundary rule applied literally. An editor integration must agree with the TUI about which
commits exist, so the rule belongs in `sase_core`, where both hosts already read it from one function. Splitting the
decision — data in Python, policy in Rust — is what keeps the two surfaces identical by construction instead of by
discipline.

The default is the empty string, which matches no policy and therefore filters nothing. A catalog JSON written by an
older `sase` and read by a newer server behaves exactly as it does today.

**Rejected: a boolean `sidecar: bool`.** It reads well right now and ages badly: it encodes today's single policy
question into the wire, so the next question (should `external` repos contribute? should `primary` outrank `linked`?)
needs another boolean. `kind` is the datum the host already has, under the name it already publishes.

**Rejected: filter in Python, by dropping sidecar records from `context.repositories`.** Two independent reasons. It
breaks launch-time resolution of a fully typed `@commit:<sidecar>@<sha>` (see Current state), and it puts a rule both
frontends must obey in only one of the two paths that reach them — the TUI snapshot loader and the LSP catalog writer —
so the surfaces can drift.

**Rejected: infer sidecars inside core from checkout paths or the `<project>--<role>` naming convention.** The `beads`
and `agents` sidecars have no corresponding entry in `context.document_roots`, so paths cannot identify them; the naming
convention is host policy that core has no business re-deriving, and the wire row carries the bare role name (`plans`)
rather than the slug (`sase--plans`) anyway.

### D2 — Skip before spawning `git log`, not after

The filter goes at the top of the `for repository in &context.repositories` loop in `append_commit_candidates`, ahead of
the checkout probe and the child process. Three consequences, all wanted:

- Four fewer `git log` subprocesses per inventory build on a typical project — this runs on a background worker in the
  TUI and inside a two-second-TTL cache in the LSP, and both get cheaper.
- `truncated_payloads` stays honest. Sidecar commits never enter the merged timeline, so they cannot be counted as
  "additional payloads not shown", and the freed budget goes to real commits.
- The `seen` dedupe set never observes a sidecar payload, so a commit reachable through both a sidecar record and a
  non-sidecar record is still offered under the non-sidecar name.

### D3 — Exclusion is unconditional, because the inventory is query-independent

`build_artifact_ref_payload_inventory` is documented as query-independent and is cached as such: the LSP holds one
`AtReferencePayloadIndex` per `(project, kind)` and the prompt bar holds one TTL-revalidated snapshot per project. Fuzzy
matching happens after the corpus is built.

**Rejected: keep sidecar rows and reveal them only when the query names the sidecar explicitly** (`@commit:plans@…`). It
sounds like the generous option, and it is the expensive one: the inventory would become query-dependent, which
invalidates the caching model in both hosts, and the "hidden unless named" tier would have to be threaded through the
payload index and the shared menu comparator. The rows it would recover are already reachable — a user who knows the
sidecar name and the SHA can type the reference, and it resolves (D4).

**Rejected: a configuration knob** (`ace.artifact_refs.commit_completion.include_sidecars` or similar). A knob here lets
the prompt bar and an editor disagree about what `@commit:` contains, which is precisely the class of bug this
enumeration was centralized to prevent. There is no demand for it; add it if one appears.

### D4 — Only completion changes

`resolve_commit`, `_resolve_for_launch`, `canonicalize_artifact_ref`, the reference grammar, and every non-`commit`
payload kind are untouched. `context.repositories` keeps every repository, so a sidecar commit reference that a user
types, that an older agent left in a plan, or that `sase artifact` recorded, still resolves and expands at launch. This
is asserted mechanically in `host_repo_kind`, not just asserted here.

### D5 — Scope boundaries

- Only `kind == "sidecar"` is excluded. `primary`, `linked`, and `external` repositories all keep contributing; a linked
  plugin repo or an opened external repo is a place humans commit code, which is the whole criterion.
- The hidden `agents` sidecar is excluded by the same rule as the visible ones — it needs no special case, and it is
  currently the single loudest contributor.
- A long-running editor session that started before the upgrade keeps its already-written catalog JSON, which has no
  `kind` field, so it keeps offering sidecar commits until `sase lsp` restarts. This is inherent to the catalog design
  and is the correct trade — the alternative is invalidating on schema shape, which no other field does.

> **Invariant.** A repository excluded from commit completion is still fully resolvable. For every repository in
> `context.repositories`, `@commit:<name>@<sha>` continues to resolve at launch whether or not that repository
> contributes completion rows.

---

## Phase `core_kind_filter` — Repository kind on the wire, sidecars skipped in commit enumeration

Work in the linked `sase-core` repository. Open it with the `/sase_repo` skill and use the path it prints. Do not edit
`[workspace.package].version`, crate versions, or path-dependency pins — release-plz owns those.

### Wire

Add to `ArtifactRefRepositoryWire` (`crates/sase_core/src/artifact_ref/wire.rs:143`):

```rust
#[serde(default)]
pub kind: String,
```

`""` is the default and means "unclassified"; it must not be treated as any policy value.

`crates/sase_core/src/artifact_ref/mod.rs:1500` constructs this struct exhaustively and will stop compiling. Add
`..Default::default()` there rather than listing the new field, matching the style every other test construction in the
crate already uses.

### Filter

In `append_commit_candidates` (`crates/sase_core/src/editor/completion.rs:480`), skip sidecar repositories at the top of
the loop, before `checkout_paths.first()` and before `commit_log_output`. Introduce a named constant and a small
predicate rather than an inline string comparison, so the rule is greppable from the wire definition:

```rust
const ARTIFACT_REF_SIDECAR_REPOSITORY_KIND: &str = "sidecar";
```

Update the doc comment on `build_artifact_ref_payload_inventory` to state that `commit` enumeration covers every
non-sidecar repository in the context, and why: SDD sidecars are machine-written stores whose commits are not what a
human means by "my recent commits".

### Tests

Extend the existing commit-inventory tests in `crates/sase_core/src/editor/completion.rs` using the crate's real
`git init` fixtures with pinned `GIT_AUTHOR_DATE` / `GIT_COMMITTER_DATE`:

- A context holding one `kind: "sidecar"` repository and one unclassified repository, both with commits, where the
  sidecar's commits appear in neither `payloads` nor `truncated_payloads`, and the surviving rows are re-ranked from
  zero with no gap.
- A sidecar repository whose commits are strictly newer than every other repository's — it must not take rank 0, which
  is the exact failure the plan exists to remove.
- `kind: "primary"`, `kind: "linked"`, `kind: "external"`, and `kind: ""` all still contributing.
- A context whose only repository is a sidecar yielding an empty inventory rather than an error.
- `resolve_artifact_ref` still resolving `commit:<sidecar-name>@<sha>` from a context whose repository is marked
  `sidecar` — the D4 invariant, asserted in core.
- Every existing commit-inventory and at-reference test passing unmodified. Rows built by the `repository()` helper get
  `kind: ""` from `..Default::default()`, so a golden that changes is a defect in this phase.

Run the crate's full test suite before opening the PR. The release that carries this phase is what `core_floor`
consumes.

---

## Phase `host_repo_kind` — The host labels each repository and proves both surfaces agree

Work in this repository. It depends on `core_kind_filter` being merged to `sase-core` `master`, because CI builds
`sase_core_rs` from that checkout and the parity test below asserts the filtered behavior.

### Carry the kind

- `ArtifactRefRepository` (`src/sase/artifact_ref_models.py:209`): add `kind: str = ""` and emit `"kind": self.kind`
  from `to_wire()`. The default keeps every existing construction — in `src/` and in tests — valid.
- `artifact_ref_context()` (`src/sase/artifact_ref_context.py:68`): pass `kind=record.kind`. Use the typed attribute
  directly, consistent with the neighbouring `record.name` and `record.slug`; the defensive `getattr` style in this
  module is reserved for the duck-typed clone probing.
- `tests/artifact_refs/test_context.py:38` fakes a record as `SimpleNamespace(name=..., slug=...)`. Give it
  `kind="primary"` and assert the assembled repository carries it, so the context test covers the new field rather than
  merely tolerating it.

Nothing else in `src/` constructs an `ArtifactRefRepository` except the `replace(..., shas=...)` call in
`src/sase/artifact_ref_prompt.py:192`, which preserves unnamed fields.

### Prove it end to end

`tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py:244` already asserts that the LSP payload sequence and
the prompt-bar candidate sequence are identical for `@commit:`, over real git fixtures. That is the right anchor —
extend it rather than adding a parallel test:

- Add a third fixture repository built with `kind="sidecar"`, holding the **newest** commit of the three.
- Assert the two sequences still match each other, and that neither contains the sidecar commit — one sidecar row
  slipping into either surface fails the test that guards both.
- Assert `snapshot.truncated_payloads` is unaffected by the sidecar's commits.
- Keep the existing loop that resolves every offered row through `_resolve_for_launch`, and add one assertion outside
  it: `_resolve_for_launch` on the sidecar's own `@commit:<name>@<sha>` still returns a non-`missing` status with a
  locator. That is the D4 invariant, asserted from the host side.

### Documentation

Two sentences currently promise commits from every local checkout:

- `docs/editor.md:80` — "`commit` payloads are enumerated from local git checkouts."
- `docs/getting_started.md:164` — "The prompt bar and editor LSP both complete `@commit:` from local git checkouts…"

Correct both to say that SDD sidecar repositories are excluded from enumeration, and state in `docs/editor.md` that a
sidecar commit reference still resolves when written out in full, so the exclusion reads as a curation decision rather
than a capability gap.

### Verification

Run `just install` before anything else — this workspace may be stale and the phase depends on a freshly built
`sase_core_rs`. Then `just check`.

---

## Phase `core_floor` — Raise the published sase-core-rs floor

Work in this repository, after the `sase-core` release carrying `core_kind_filter` is published to PyPI.

Raise the lower bound of `sase-core-rs` in `pyproject.toml` (currently `>=0.17.13,<0.18.0`) to the released version,
following the existing `build(deps): require sase-core-rs <version>` commit convention. Leave the upper bound alone.

`just validate` runs `tools/validate_sase_core_rs_version --published-minimum`, which queries PyPI, so this phase fails
by design until the release is actually available. If it is not yet published when this phase is picked up, stop and
report that — do not work around the check.

Without this bump an installed `sase` resolves against a core that still enumerates sidecar commits, so the user-visible
fix is not delivered until this phase lands.

Verify with `just check`.
