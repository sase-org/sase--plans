---
tier: epic
title: Name toobig_split agents with the `@` template suffix instead of a path hash
goal: 'Agents proposed by the `toobig_split` chop are named `split_file.<full_dotted_module>.@`
  (rendered as `<clan>.split_file.<module>.0`) instead of `split_file.<truncated_module>.<8-hex-hash>`,
  which requires SASE to support one `@` template marker in a chop proposal''s `clan`
  and another in its `agent_name`.

  '
phases:
- id: core-marker-rule
  title: Allow one template marker in each of a chop proposal's clan and member
  depends_on: []
  size: small
  description: '"Phase 1 - Core: allow one marker in each of clan and member" section:
    relax `validate_clan_member_identity` in the sase-core chop validator so a composed
    clan-member name may carry one `@` from the clan template and one from the member
    template, keeping the per-component "at most one marker" rule, and update the
    Rust unit tests that pin the old `ambiguous_agent_name_template` behavior.

    '
- id: planner-member-tokens
  title: Allocate chop member template tokens inside the concrete clan
  depends_on:
  - core-marker-rule
  size: medium
  description: '"Phase 2 - Planner: allocate member tokens inside the concrete clan"
    section: teach `_plan_clan_group` in `src/sase/axe/chop_proposals.py` to render
    the clan token first and then allocate each templated member name within the concrete
    clan on a trial reservation index, add regression tests, and document the two-marker
    rule in `docs/axe.md`.

    '
- id: toobig-chop-names
  title: Emit `split_file.<full_module>.@` member names from the toobig_split chop
  depends_on:
  - planner-member-tokens
  size: small
  description: '"Phase 3 - Chop: emit split_file.<module>.@ names" section: change
    `_agent_name` in the bugyi-chops `toobig_split` module to drop the path digest
    and the 48-character slug truncation and end the member name with `.@`, update
    its tests, and bump the `sase` floor to the release that contains the planner
    change.

    '
create_time: 2026-07-25 12:59:11
status: wip
bead_id: sase-9n
---

# Plan: Name `toobig_split` agents with the `@` template suffix

## Goal

The `toobig_split` chop currently proposes clan members named `split_file.<truncated_dotted_module>.<8-hex-digest>`,
which renders in the ACE Agents tab as entries like:

```
.split_file.src.sase.axe.run_agent_wait.77d2b8bf
.split_file.tests.test_run_agent_runner_slots.28a2150d
```

We want the meaningless hash replaced by SASE's `@` auto-name template marker and the module path no longer truncated,
so the same two members become:

```
.split_file.src.sase.axe.run_agent_wait.0
.split_file.tests.test_run_agent_runner_slots.0
```

The authored member name is `split_file.<full_dotted_module>.@`; SASE allocates the concrete token (`0`, `1`, ... `z`,
`00`, ...) at plan time inside the concrete clan.

## Background: how a `toobig_split` proposal becomes an agent name

1. The chop (external repo `gh:bbugyi200/bugyi-chops`, `src/bugyi_chops/toobig_split.py`) emits one proposal per
   oversized file with `clan="toobig-@"` (module constant `CLAN_TEMPLATE`) and `agent_name=_agent_name(path)` (lines
   294-297):

   ```python
   def _agent_name(path: str) -> str:
       stem = Path(path).with_suffix("").as_posix()
       slug = safe_fragment(re.sub(r"[/\\]+", ".", stem), fallback="file")[:48]
       return f"split_file.{slug}.{_path_digest(path)[:8]}"
   ```

2. `prepare_chop_proposals` (`src/sase/axe/chop_proposals.py:101`) validates each proposal through the Rust facade
   `validate_chop_proposal` (`src/sase/core/axe_chop_facade.py:34`).

3. `plan_chop_proposals` (`src/sase/axe/chop_proposals.py:267`) picks one concrete clan for the whole group in
   `_plan_clan_group` (`src/sase/axe/chop_proposals.py:205`) and composes the full agent identity as
   `f"{clan}.{agent_name}"` (`_full_name_template`, line 53).

4. `_scaffolded_prompt` (line 65) writes **concrete** names into the launch prompt: `%id:<clan>.<member>` for the
   clan-declaring proposal and `%id(<member>, clan=<clan>)` for the rest. No `@` survives into the prompt, so nothing is
   re-resolved at agent start.

The `@` marker itself is well understood by SASE: exactly one marker per name, rendered with the next free token from
`0..9, a..z, 00, 01, ...` (`sase-core` `crates/sase_core/src/agent_name_template.rs`, Python facade
`src/sase/agent/names/_templates.py`). A trailing `.@` needs no separator fixup because the prefix ends in `.`, and
`split_file.src.sase.axe.run_agent_wait.@` is a syntactically valid template today.

## Why the one-line chop change does not work today

Changing only `_agent_name` to return `split_file.<slug>.@` fails immediately, because the composed clan-member identity
would then hold **two** markers (`toobig-@` + `split_file....@`), and both layers reject that:

- **Rust validator.** `validate_clan_member_identity` in `crates/sase_core/src/axe_chop/validation.rs:403` computes
  `format!("{clan}.{member}")` and errors with `ambiguous_agent_name_template` when the composed string contains more
  than one `@` (lines 415-424). Every proposal would be rejected during `prepare_chop_proposals`, so the chop run would
  fail before any launch.

- **Python planner.** Even with that check removed, `_plan_clan_group` would crash. It computes
  `full_templates = [f"{clan}.{member}", ...]`, asks `is_agent_name_template(...)` (which is `True` only for _exactly
  one_ marker), gets `has_template = False`, and therefore iterates the single candidate `token = None` from
  `_group_token_candidates`. `_render_template_value` then hits `assert token is not None` while rendering the
  still-templated clan (`src/sase/axe/chop_proposals.py:188-194`). The `except Exception` handler in
  `src/sase/axe/chop_runner_script_result.py:323` would turn the whole run into a failure.

So the ask requires a real capability: **one marker in the clan template and one in the member template, allocated in
two stages.** Note that proposals _without_ a clan already accept a templated `agent_name` (it is passed through
verbatim and resolved by the child in `src/sase/axe/run_agent_directive_identity.py:136`); only the clan-scoped
composition is missing.

## Target semantics

For a proposal group sharing one raw clan:

1. Pick a clan token from the shared auto-name sequence and render the concrete clan (`toobig-@` -> `toobig-0`).
   Unchanged from today.
2. For each proposal, compose `f"{concrete_clan}.{agent_name}"`.
   - If `agent_name` holds no marker, the composed string is the concrete identity (today's behavior).
   - If `agent_name` holds one marker, allocate the lowest free token for that composed template **within** the concrete
     clan (`toobig-0.split_file.src.pkg.large.@` -> `toobig-0.split_file.src.pkg.large.0`).
3. A clan token is accepted only if the clan name, the clan namespace, and every member identity in the group are
   simultaneously available. Otherwise advance to the next clan token and retry the whole group, discarding any member
   tokens tentatively taken for the rejected clan token.
4. The member ID written into `%id(<member>, clan=<clan>)` is the concrete identity with the `<concrete_clan>.` prefix
   stripped, e.g. `split_file.src.pkg.large.0`.

Because each chop run gets a fresh clan, every member hood inside it is empty and the token is normally `0`. The token
is not decoration: it is what makes distinct scanned paths that collapse to the same slug (via `safe_fragment` character
folding) land on `.0` and `.1` instead of colliding, which is exactly the job the path digest used to do.

---

## Phase 1 - Core: allow one marker in each of clan and member

Repo: `sase-core` (open with `/sase_repo`, e.g. `sase repo open sase-core -r "<reason>"`). All paths below are relative
to that repo root.

### Change

In `crates/sase_core/src/axe_chop/validation.rs`, `validate_clan_member_identity` (around line 403):

- Keep both `validate_dotted_agent_name` calls. They already enforce, per component, "no empty dotted segments", "at
  most one `@`" (`invalid_agent_name_template`), and the `, ( ) =` / leading `!` directive guards.
- Remove the composed-string `composed.matches('@').count() > 1` rejection and its `ambiguous_agent_name_template` error
  (lines 415-424). One marker in the clan plus one in the member is now legal; two markers in either single component
  remains an error via `validate_dotted_agent_name`.
- Keep the `composed.contains("--")` family-separator rejection (lines 425-433) untouched.

Grep the crate for `ambiguous_agent_name_template` and remove any now-dead references; at authoring time it appears only
in `validation.rs:418` and `src/axe_chop/tests.rs:259`.

### Tests

In `crates/sase_core/src/axe_chop/tests.rs`, the invalid-case table around line 252 contains:

```rust
(Some("@"), "research.@", None, "ambiguous_agent_name_template"),
```

Move that combination to the valid-case set (a clan template plus a member template must now validate). Keep the
neighboring negative cases (`research@@` -> `invalid_agent_name_template`, `.research` and `worker..review` ->
`malformed_agent_hood`) exactly as they are, and add a positive case shaped like the real chop input: clan `toobig-@`,
member `split_file.src.pkg.large.@`.

### Done when

- `validate_chop_proposal` accepts `clan="toobig-@"` with `agent_name="split_file.src.pkg.large.@"`.
- `validate_chop_proposal` still rejects `agent_name="research@@"` and `clan="toobig-@@"`.
- The crate's own checks pass (follow the repo's `Justfile`/`cargo` conventions).

### Note for the phase agent

Landing this phase alone leaves a narrow window where the validator accepts a two-marker composition that the Python
planner cannot yet render (it would raise `AssertionError`). That window is inert: no shipped chop emits such a
proposal, and the only adopter is Phase 3, which depends on Phase 2. Do not add a compensating guard in the Python layer
here.

---

## Phase 2 - Planner: allocate member tokens inside the concrete clan

Repo: this repo (`sase`). This phase also needs the Phase 1 commit present in the linked `sase-core` checkout, because
`just install` builds `sase_core_rs` from it (`Justfile` lines 14-24, 62-90). Pull the linked `sase-core` repo to a
revision that contains Phase 1 before running `just install`.

### Change

All in `src/sase/axe/chop_proposals.py`.

Rework `_plan_clan_group` (lines 205-264) to do two-stage rendering:

1. `clan_is_template = is_agent_name_template(raw_clan)`. Drive the token loop from that flag alone -
   `_group_token_candidates` must no longer inspect the composed full templates (which may now hold two markers and
   would report `False`).

2. Per candidate token, compute:
   - `concrete_clan` = the rendered clan (or `raw_clan` when it is not a template).
   - `clan_namespace` = today's namespace value: for a templated clan,
     `render_agent_name_template(agent_name_template_namespace_template(raw_clan), token)`; otherwise `concrete_clan`.
     Preserving this derivation matters for clan templates whose marker is not the final segment (e.g. `research.@.x`),
     where the reserved namespace is a proper prefix of the clan.
   - Reject the token when `concrete_clan in reserved_names` or when
     `reservation_index.candidate_available(concrete_clan, clan_namespace)` is false. Break out of the loop instead of
     continuing when the clan is not a template, matching today's behavior.

3. Allocate members on **trial copies** so a rejected clan token leaks nothing: `trial_reserved = set(reserved_names)`
   and a shallow clone of the index with copied `exact_names` / `occupied_namespaces` sets and the same `identity`
   (`dataclasses.replace` on `AgentNameNamespaceReservationIndex`, `src/sase/agent/names/_templates.py:61`). Per
   proposal, with `composed = f"{concrete_clan}.{proposal.agent_name}"`:
   - Templated member: `allocate_agent_name_template(composed, reserved=trial_reserved, index=trial_index)`
     (`src/sase/agent/names/_templates.py:233`). It validates the template, walks tokens in shortlex order, checks
     `candidate_available` against the composed name's own namespace, and registers the winner in both the pool and the
     index - which is exactly what makes two identical member templates in one group resolve to `.0` and `.1`.
   - Plain member: require `candidate_available(composed, clan_namespace)` and that `composed` is not already in
     `trial_reserved`, then add it to both trial structures.
   - Any failure abandons this clan token and moves to the next one.

4. **Do not** add `concrete_clan` to the trial index before allocating members. `add_name` inserts every dotted prefix
   of the name into `occupied_namespaces`, so registering the clan first would make
   `candidate_available(..., clan_namespace)` fail for every plain member. Commit the clan and all member names to the
   real `reserved_names` / `reservation_index` only after the whole group succeeds, as the current code already does
   (lines 255-260).

5. Keep the duplicate-identity guard (lines 244-246). Templated members can no longer collide with each other, but two
   plain members still can, and that must remain a `ValueError`.

6. Return `members[proposal.index] = (concrete_name, member_id)` where `member_id` is `concrete_name` with the
   `f"{concrete_clan}."` prefix removed. Deriving it by stripping the prefix keeps `%id(<member>, clan=<clan>)`
   consistent with the allocated name for both plain and templated members.

Adjacent cleanups in the same file:

- `_render_template_value` (line 188) and `_group_token_candidates` (line 197) exist only to serve the old single-marker
  composition; fold them into the new flow or delete them if unused. Do not leave a helper whose
  `assert token is not None` can be reached with a templated clan and `token=None`.
- Leave `_full_name_template` (line 53) as the raw composition helper. It is still used by `_scaffolded_prompt`'s
  fallback and by `_fallback_plan` (line 363), which runs only in `proposal_previews` when planning already failed; a
  preview string containing raw `@` markers on that error path is acceptable. Do not route it through template parsing.

### Tests

Add to `tests/test_axe_chop_clan_launch.py` (its `_write_clan_result` helper already uses `clan="toobig-@"` with members
`split_file.first` / `split_file.second`):

- A dry-run preview where both members are `split_file.<mod>.@`, asserting `agent_name` values
  `toobig-0.split_file.first.0` and `toobig-0.split_file.second.0`, `clan_role` `["declare", "join"]`, the
  `%id(split_file.second.0, clan=toobig-0)` join directive, and the `%wait:toobig-0.split_file.first.0` chain.
- A mixed group: one plain member and one templated member under the same clan.
- Two proposals whose member templates are byte-identical, asserting they resolve to `.0` and `.1` rather than raising
  the duplicate-identity error.
- A case where `toobig-0` (or a descendant of it) is already reserved, asserting the group moves to `toobig-1` as a
  whole and that no member token from the abandoned attempt leaks into the reservation set.

Keep every existing assertion in `tests/test_axe_chop_clan_launch.py`, `tests/test_axe_chop_result_protocol.py`, and
`tests/test_axe_chop_name_collisions.py` passing unchanged - plain-member clan proposals must behave exactly as they do
today.

### Docs

`docs/axe.md` lines 364-369 describe the proposal fields and say the runner owns "concrete clan allocation plus the full
`<clan>.<member>` identity". Extend that paragraph: `clan` and `agent_name` may each carry at most one `@` template
marker; the clan token is chosen for the whole group first, then each templated member is allocated inside the concrete
clan, and a clan token is only taken when the clan and every member identity are free together.

### Done when

- `just install && just check` passes.
- A proposal with `clan="toobig-@"` and `agent_name="split_file.src.pkg.large.@"` plans to
  `toobig-0.split_file.src.pkg.large.0`.
- Existing chop clan behavior for non-templated members is byte-identical.

---

## Phase 3 - Chop: emit `split_file.<module>.@` names

Repo: `gh:bbugyi200/bugyi-chops` (open with `/sase_repo`, e.g. `sase repo open gh:bbugyi200/bugyi-chops -r "<reason>"`).
All paths below are relative to that repo root.

### Change

In `src/bugyi_chops/toobig_split.py`, `_agent_name` (lines 294-297):

```python
def _agent_name(path: str) -> str:
    stem = Path(path).with_suffix("").as_posix()
    slug = safe_fragment(re.sub(r"[/\\]+", ".", stem), fallback="file")
    return f"split_file.{slug}.@"
```

Two edits, both intended:

- **Drop `_path_digest` from the name.** Keep `_path_digest` itself: it still builds
  `proposal_id = f"split-{_path_digest(path)}"` in `build_result` (line 447), and `_dedupe_key` (line 300) still hashes
  file content. Only the agent name stops using it.
- **Drop the `[:48]` truncation** so the name carries the full dotted module path, as the goal states. This also removes
  a latent failure: the slice is applied _after_ `safe_fragment` has stripped trailing `._-`, so a path whose 48th
  character lands on a `.` produces a slug with a trailing dot, and the resulting name (`...axe_display..1a2b3c4d`) is
  rejected by the core validator with `malformed_agent_hood`, failing the entire run. Length is not a concern: the hard
  limit is 512 bytes (`MAX_AGENT_NAME_BYTES`) and the longest current sase module path is well under 100 characters.

Collision safety is preserved by the `@` token: two paths that `safe_fragment` folds to the same slug now get `.0` and
`.1` instead of colliding.

### Dependency bump

`pyproject.toml` pins `sase>=0.12.0,<0.13.0`. Confirm which published `sase` release first contains the Phase 2 planner
change and raise the floor to it if `0.12.0` predates it. The chop's own dev environment resolves `sase` from PyPI via
`uv sync --group dev`, so the tests in this repo cannot pass until that release exists (or until the environment is
pointed at a local `sase` checkout through `BUGYI_CHOPS_VENV_BIN`, which `just install` supports).

### Tests

In `tests/test_toobig_split.py`:

- `test_scan_deduplicates_files_and_emits_stable_wait_chain` (around line 306) currently asserts the hashed names and
  then `assert all("@" not in proposal["agent_name"] ...)` (line 311). Replace the expected list with
  `split_file.src.pkg.large.@`, `split_file.src.pkg.shared.@`, `split_file.tests.large.@`, and invert the marker
  assertion: every `agent_name` ends with `.@` and contains exactly one `@`.
- `test_sase_planning_emits_one_summary_and_promotes_a_surviving_tail` (around line 328) already drives the real
  `prepare_chop_proposals` / `plan_chop_proposals`. Add assertions on the planned names so this test is the end-to-end
  proof that a templated clan plus a templated member resolves to `toobig-0.split_file.<module>.0`.
- Add a case for a path long enough that the old `[:48]` slice would have cut it, asserting the full dotted module path
  survives.

### Done when

- `just check` passes in the chop repo.
- The chop proposes `agent_name="split_file.<full dotted module>.@"` for every scanned file, with `clan` still
  `toobig-@`.

---

## Rollout and ordering

The phases must land in order, and Phase 3 additionally waits on a published `sase` release:

1. Phase 1 merges to `sase-core` master.
2. Phase 2 merges to `sase` master (its workspace must build `sase_core_rs` from a `sase-core` revision that includes
   Phase 1).
3. A `sase` release containing Phase 2 is published.
4. Phase 3 merges to `bugyi-chops` with the floor bump.
5. The chop runs from the `sase` uv tool environment, so the new `bugyi_chop_toobig_split` only takes effect after that
   environment is reinstalled/refreshed against the updated `bugyi-chops` revision and the `sase` editable checkout it
   overrides.

Agents already running or queued under an existing `toobig-NN` clan keep their hashed names; the change only affects
clans created after step 5. The chop is configured under `axe.lumberjacks.run_every.chops.toobig_split` in the
chezmoi-managed `sase_athena.yml`; no config change is needed for this work.

## Rejected alternatives

- **Drop the suffix entirely** (`split_file.<module>`, one marker total, from the clan). This needs only the Phase 3
  edit and produces the shortest names, but it is not what was asked for and it removes the automatic disambiguation
  that makes two slug-equal paths safe. Worth reconsidering only if Phases 1-2 turn out to be more invasive than scoped
  here.
- **Hardcode `.0` in the chop.** Cosmetically identical in the common case, but it is a literal name, so a slug
  collision inside one clan raises "clan contains duplicate member identities" and fails the whole run.
- **Render the member marker with the clan's token** (`toobig-0.split_file.x.0` driven by one shared token). Still needs
  the same core and planner changes, and the member token then carries no information and cannot resolve collisions.
- **Make the clan concrete in the chop** (e.g. `toobig-<run-token>`) so the member marker is the only one. Avoids the
  SASE changes but replaces readable `toobig-00` clan names with opaque ones and makes freshness the chop's problem.

## Verification

End-to-end check after Phase 3 lands and the tool environment is refreshed: let the `toobig_split` chop fire (or run
`bugyi_chop_toobig_split --context <file>` against a scratch context) and confirm the ACE Agents tab shows clan members
as `.split_file.<full dotted module>.0` with no hash segment, one clan declaration, and the sequential `%wait` chain
intact.
