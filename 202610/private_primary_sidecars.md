---
tier: tale
title: Private primary repositories always get private sidecars
goal:
  sase init -M and sase repo init never create a public sidecar for a private primary
  repository.
size: medium
decisions:
  doctor_check:
    ask:
      Should `sase doctor` also warn when a private primary already has public sidecars?
    choices:
      add:
        New doctor WARN check probes recorded sidecar remotes; one more module plus
        tests
      skip: Fix init only; sidecars that are already public stay undetected
    default: add
    why:
      Init never re-checks existing remotes, so a sidecar made public earlier leaks
      silently
    answer: add
proposed_by: bbugyi200.athena.0zn
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 09:38:45
status: wip
---

# Private primary repositories always get private sidecars

## Problem

Running `sase init -M` in a SASE-managed project whose primary GitHub repository is
**private** prompts:

```
Create public GitHub sidecar repository bbugyi200/job--plans on github.com? [y/N]
```

That is wrong. Sidecars hold plans, beads, research, and agent transcripts derived from
the primary repository, so a private primary must never get a public sidecar. The prompt
should have been:

```
Create private GitHub sidecar repository bbugyi200/job--plans on github.com? [y/N]
```

### Root cause

Nothing on the sidecar-creation path looks at the primary repository's visibility. Every
layer defaults sidecar visibility to `public`:

- `src/sase/main/_repo_init_config.py` — `explicit_sidecar_config_update()` (what
  `sase init -M` / `sase repo init` writes into `sase/sase.yml`) declares
  `repos.sidecar.builtin.{plans,beads,agents}` and `repos.sidecar.custom.research` with
  no `visibility:` key.
- `src/sase/_linked_repo_config_entries.py` — `_merged_sidecar_entries_cached()` does
  `normalized_entry.setdefault("visibility", "public")`.
- `src/sase/_linked_repo_config_defaults.py` — `inject_default_linked_repos()` injects
  the hidden `agents` sidecar as `public`. It injects the public `attachments` sidecar
  whenever the beads sidecar's configured visibility is public, and beads defaults to
  public.
- `src/sase/main/_repo_init_config.py` — `configured_sidecar_specs()` builds
  `SidecarInitSpec(visibility=_entry_text(entry, "visibility") or "public")`.
- `src/sase/sdd/_sidecar_init.py` — `_sidecar_provider_options()` forwards
  `spec.visibility` as `options["sdd_visibility"]`. The sase-github provider echoes it
  into `SddSidecarPreflight.visibility` and later runs `gh repo create --public`.
- `src/sase/main/_repo_init_sidecars.py` — `_confirm_sidecar_creation()` renders
  `Create {preflight.visibility} {provider} sidecar repository …`.

## Invariant to establish

> A sidecar repository may be created **public** only when the primary repository is
> verified public. If the primary is private, or its visibility cannot be verified,
> every sidecar is created private. The public-only `attachments` sidecar is never
> created for such a primary.

This is a safety fix, not a preference, so it needs no feature flag and no config knob.
An explicit `visibility: public` on a sidecar of a private primary is overridden to
private, never honored.

## Scope and boundaries

- Python-only change in this repo. Sidecar config normalization and `sase repo init`
  sidecar creation already live in Python. No sase-core API is involved, so there is no
  `sase-core-revision.txt` bump.
- No sase-github change is needed. The provider already honors
  `options["sdd_visibility"] == "private"` (`gh repo create --private`) and echoes it in
  preflight. The existing mismatch guard in `preflight_sidecars()` stays correct because
  core now passes the clamped visibility.
- `src/sase/sdd/_auto_init.py` (first-launch auto-connect) creates nothing and stays
  unchanged.
- No new CLI options or config keys, so `src/sase/default_config.yml` is unchanged.

## Implementation

### 1. Resolve the primary repository's visibility

Add a small module, e.g. `src/sase/sdd/_sidecar_visibility.py`, with:

- A frozen dataclass `PrimaryRepoVisibility` holding `repo` (display form `owner/repo`,
  from the parsed origin) and `visibility` (`"public" | "private" | "unknown"`).
- `resolve_primary_repo_visibility(project_root: Path) -> PrimaryRepoVisibility | None`:
  1. Read the primary checkout's `origin` URL. Reuse
     `sase.workspace_provider._utils_git.run_git_remote_get_url()`, which already runs
     with the git-lock retry and a non-interactive env. A missing origin or a non-zero
     exit returns `None`.
  2. Parse it with `sase._git_remote.parse_hosted_git_remote()`. A local-path or
     otherwise non-hosted origin returns `None`. In that case there is no hosted primary
     to protect, so existing behavior is unchanged. This also keeps every existing
     `tmp_path` test, none of which has a hosted origin, on the old path.
  3. Otherwise call
     `sase.bead.attachments.remote_visibility.resolve_remote_visibility(origin_url)`.
     This is the existing anonymous smart-HTTP probe: 401/403/404 means `private`, a 200
     upload-pack advertisement means `public`, anything else means `unknown`. It already
     has the `set_remote_visibility_prober()` test seam and a disk cache. Import it
     lazily, matching `_repo_init_sidecars.py`.
- A pure function, e.g.
  `apply_primary_visibility_policy(specs, primary) -> SidecarVisibilityPolicy`. It
  returns the adjusted `specs`, the `forced_roles` whose visibility changed from public
  to private, and the `dropped_roles`:
  - `primary is None` or `primary.visibility == "public"`: specs unchanged.
  - `"private"` or `"unknown"` (fail closed): drop any `ATTACHMENTS_SIDECAR_ROLE`
    (`attachments`) spec, because that store is public by design. Rewrite every other
    spec to `visibility="private"` with `dataclasses.replace`. `attachments-private` is
    already private and is left alone.
- A helper that renders the user-facing note for a policy result, used in step 3. For
  example:
  - private:
    `note: acme/widget is private, so SASE creates its sidecar repositories as private`
  - unknown:
    `note: could not verify that acme/widget is public, so SASE creates its sidecar repositories as private`
  - dropped attachments:
    `note: skipping the public attachments sidecar because acme/widget is not verified public`

Keep the module's public surface limited to what callers use, so Symvision's
unused-symbol lint passes, and list it in `__all__`.

### 2. Persist `visibility: private` in `sase/sase.yml` for private primaries

Extend `explicit_sidecar_config_update(config_path, *, private_primary: bool = False)`
in `src/sase/main/_repo_init_config.py`. When `private_primary` is true:

- Each newly declared entry (`plans`, `beads`, `research`, `agents`) gets
  `visibility: private`.
- Each existing entry in `repos.sidecar.builtin` / `repos.sidecar.custom` that is not
  `disabled: true` and whose `visibility` is missing or `public` gets
  `visibility: private`. Skip the `attachments` and `attachments-private` roles, whose
  visibility is fixed by normalization. Record these roles in `updated_roles`, the same
  way the existing `plans` `ref` backfill does, so the action detail and the "updated …
  with repos.sidecar …" message mention them.
- Pass `private_primary=True` only for a definitive `"private"` result, never for
  `"unknown"`. A transient probe failure must not permanently rewrite a public project's
  config. The runtime clamp in step 3 still covers `unknown`.
- The update must be idempotent: a second run on an already-updated config produces no
  change.

Persisting the value keeps config-only readers consistent with reality, without probing.
Examples are `bead_store_visibility()` in `src/sase/bead/attachments/provenance.py` and
`_beads_public_for_injection()`. With a private beads entry, the public `attachments`
sidecar is no longer injected at all.

Update `sidecar_config_action_detail()`, or its call site, so the preview line says the
sidecars are declared private when that is the case. For example:
`declare the managed plans, beads, research, and agents sidecars (private: acme/widget is private)`.

### 3. Wire the policy into `sase repo init` (which `sase init -M` runs)

In `src/sase/main/repo_init_handler.py`:

- Add a module-level seam, `_primary_repo_visibility(project_root)`, that calls the
  resolver from step 1. Tests monkeypatch this seam, following the file's existing
  `_project_provider_sdd_policy` / `_configured_sidecar_specs` wrapper pattern.
- In both `plan_repo_init()` and `run_repo_init()`, resolve the primary visibility once
  per invocation and thread it through. `run_repo_init()` currently calls
  `plan_repo_init(args)` and then recomputes the config update. Avoid probing three
  times, either by passing the resolved value through or by relying on the probe's disk
  cache. Never probe for unmanaged or non-project directories, which return early.
- Pass `private_primary=` into `_explicit_sidecar_config_update()`. The config is still
  written before `_run_configured_sidecars()` re-reads specs, as today.
- Apply `apply_primary_visibility_policy()` to the specs in:
  - `_run_configured_sidecars()`, before `run_configured_sidecars()` runs preflight.
  - The `separate_repo` / materialized branch of `plan_repo_init()`, before
    `_plan_sidecar_actions()`, so the `-d` preview and the action details show private
    visibility and omit a dropped `attachments` action.
  - `_run_materialized_sidecars()` creates nothing and stays as is.

In `src/sase/main/_repo_init_sidecars.py`:

- Let `run_configured_sidecars()` accept the policy's notes or `forced_roles` /
  `dropped_roles` as an optional keyword argument, so existing callers and tests keep
  working. Print the note to stderr once, only when at least one forced role is actually
  missing (`not_found`) and about to be prompted, or when a role was dropped. A routine
  re-run of `sase repo init` on a fully initialized private project must stay quiet.
- No prompt-string changes are required. Once the spec is private, the provider
  preflight reports `private`, so the generic prompt renders exactly
  `Create private GitHub sidecar repository bbugyi200/job--plans on github.com? [y/N] `.
  The agents prompt renders `Repository visibility: PRIVATE.` and
  `Create private GitHub agents sidecar repository …`.
- In `plan_sidecar_actions()`, change the hidden-role detail from
  `with configured {visibility} visibility` to `with {visibility} visibility`. After the
  clamp, the value is no longer necessarily "configured".
- Keep `_confirm_attachments_sidecar_creation()`'s beads-remote probe as a backstop.

> [!decision] doctor_check = add

### 4. `sase doctor` check for existing public sidecars of a private primary

Add `src/sase/doctor/checks_sidecar_visibility.py`, modeled on
`src/sase/doctor/checks_attachment_store.py`, and register its specs in
`src/sase/doctor/runner.py` next to `attachment_store_check_specs`:

- Check id `project.sidecar_visibility`, group `project`, title `Sidecar visibility`.
- SKIP when there is no current SASE-managed project, when
  `resolve_primary_repo_visibility()` returns `None`, or when the primary is not
  definitively `private`.
- Otherwise read the materialized sidecar store record (`read_sdd_store_record` on the
  primary checkout) and probe each recorded sidecar `remote_url` with
  `resolve_remote_visibility()`. WARN when any sidecar is `public`, listing each role
  and repo. Include the remediation in the details, for GitHub
  `gh repo edit <owner/repo> --visibility private --accept-visibility-change-consequences`.
  Otherwise report OK. Treat `unknown` sidecar results as not public: say so in the OK
  summary, and never WARN on them.
- Never raise. Mirror the existing check's defensive `try`/`except` style.

> [!decision] doctor_check = skip

Skip step 4 entirely.

### 5. Docs

- `docs/init.md`, in the sidecar-creation section near "remote gets its own default-no
  prompt naming the visibility…" and the agents and attachments paragraphs: document the
  invariant, the fail-closed `unknown` behavior, the persisted `visibility: private`,
  and that the public `attachments` sidecar is skipped for private primaries. If
  `doctor_check = add`, mention the doctor check.
- `docs/configuration.md`: update the `repos.sidecar.*.<role>.visibility` table row to
  say "default `public`; forced to `private` when the primary repository is private or
  its visibility cannot be verified". Also update the prose near the `agents` /
  `attachments` defaults.
- Touch `docs/agents_sidecar.md` and `docs/beads.md` only where they state that sidecars
  default to public without qualification.

## Tests

No test may perform a real network probe. Use `set_remote_visibility_prober()`, and
reset it in teardown, or monkeypatch `repo_init_handler._primary_repo_visibility`.

1. **Resolver.** Use a `tmp_path` git repo with no origin, a local-path origin, and
   `git@github.com:acme/widget.git` with the prober injected as `public` / `private` /
   `unknown`. Assert `None`, `None`, and the matching
   `PrimaryRepoVisibility(repo="acme/widget", …)`.
2. **Pure policy.** For `None` and public, specs are unchanged. For private and unknown,
   `plans` / `beads` / `research` / `agents` become private, explicit `public` is
   overridden, `attachments` is dropped, `attachments-private` is untouched, and
   `forced_roles` / `dropped_roles` are correct.
3. **Config update** (extend `tests/main/test_repo_init_plan.py`, or wherever
   `explicit_sidecar_config_update` is covered):
   - With `private_primary=True` on an empty config, every declared entry carries
     `visibility: private`.
   - On the config shape a previous `sase init -M` wrote (no visibility keys), the
     missing visibility is backfilled and the roles are reported as updated.
   - Explicit `public` is rewritten.
   - `disabled: true` entries and `attachments` / `attachments-private` entries are
     untouched.
   - A second run is a no-op.
   - `private_primary=False` output is byte-identical to today's.
4. **Handler, run path** (alongside `tests/main/test_repo_init_handler_creation.py`):
   - With a private primary (seam monkeypatched) and default public specs, the preflight
     options or specs received are private.
   - The prompt list equals
     `["Create private GitHub sidecar repository acme/widget--plans on github.com? [y/N] "]`.
   - `initialize_sidecars` receives private specs.
   - The stderr note appears once.
   - A fully materialized private project, with no missing sidecars, prints no note.
   - An `attachments` spec is never prompted for.
5. **Handler, plan path.** With a private primary, `plan_repo_init()`'s config action
   `new_content` contains `visibility: private` and the hidden-role details say
   `private`.
6. **Regression.** With a public or `None` primary, the existing assertions stay green,
   including the `"Create public GitHub sidecar repository acme/widget--research …"`
   prompt in `tests/main/test_init_onboarding_interactive.py`.
7. **If `doctor_check = add`.** Cover SKIP for a public or `None` primary, OK when all
   recorded sidecars probe private, WARN listing the public ones with the `gh repo edit`
   remediation, and no exception when the store record is missing.

## Verification

Follow the `lint_and_test` memory note (read it with `/sase_memory_read`). Run the
repo's standard check recipe through `sase tool run`, and fix any Symvision
unused-symbol or private-misuse findings the new module introduces.

## Acceptance criteria

- On a SASE-managed project whose GitHub origin is private, `sase init -M` and
  `sase repo init`:
  - prompt
    `Create private GitHub sidecar repository <owner>/<repo>--<role> on github.com? [y/N]`
    for every missing sidecar;
  - create each confirmed sidecar private;
  - write `visibility: private` into the project's `repos.sidecar` entries;
  - never offer or create the public `attachments` sidecar.
- When the primary's visibility cannot be verified, sidecars are still created private,
  but the config is not rewritten.
- Public-primary and non-hosted-origin projects behave exactly as before.
- If `doctor_check = add`, `sase doctor` warns about any recorded sidecar that is public
  while the primary is private.
