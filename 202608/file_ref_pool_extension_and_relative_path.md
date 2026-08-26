---
tier: tale
title:
  Give captured @file pool copies a real extension and render them workspace-relative
goal:
  A captured `@file:<path>` reference lands in the pool as `<sha12>-file-ref.<ext>`
  mirroring the source file's extension, and its prompt expansion names the copy as
  `.sase/artifacts/pool/<sha12>-file-ref.<ext>` instead of an absolute workspace path.
size: medium
proposed_by: bbugyi200.athena.0e5
---

- **AGENTS:**
  - [bbugyi200.athena.0e5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0e5.md)
- **COMMITS:**
  - [a2e9f2e](https://github.com/sase-org/sase/commit/a2e9f2e145b71eecba8f7a39ef1527e8c3ba6cda)
    — fix(artifacts): preserve captured file-ref suffixes

# Plan: Give captured `@file` pool copies a real extension and render them workspace-relative

## Problem

`#sshot` expands to `@file:~/tmp/screenshots/<name>.png`. That reference is captured
into the workspace-local prompt-artifact pool, and today the agent sees this in its
prompt:

```text
the /home/<user>/.local/state/sase/workspaces/<org>/<project>/<workspace>/.sase/artifacts/pool/5713d120c7da-file-ref file
```

Two things are wrong with that string:

1. **No extension.** The pooled copy is named `5713d120c7da-file-ref` even though the
   source was a PNG. Nothing downstream — the agent's own file reader, `file`, an image
   viewer, a MIME sniffer keyed on suffix — can tell a screenshot from a Markdown note.
   This is the only capture path in the codebase that loses the extension;
   `stage_prompt_artifact` already passes `source.name` through, and the VCS
   materialization path already passes an explicit `suffix`.
2. **Absolute path.** The expansion prints a long absolute path that is specific to one
   ephemeral workspace. Every other workspace-local staging path SASE puts in a prompt
   is already relative — `process_file_references` rewrites `@~/diagram.png` to
   `@.sase/artifacts/home/diagram.png` — so the pool path is the odd one out.

## Root cause

**Extension.** `capture_prompt_file_ref` in `src/sase/core/prompt_artifact_staging.py`
(line 236) hardcodes the pool label:

```python
pool_filename = str(
    require_rust_binding("prompt_artifact_pool_filename")(
        sha256,
        "file-ref",
    )
)
```

The Rust `artifact_pool_filename` already preserves and sanitizes an extension when the
name it is handed has one (`stage_prompt_artifact` at line 142 relies on exactly that by
passing `source.name`). The literal `"file-ref"` simply has no extension to preserve.

**Absolute path.** `_capture_file_path_ref` in `src/sase/artifact_ref_prompt.py`
(line 604) builds an absolute pool path and substitutes it into the resolution:

```python
captured_path = Path.cwd().expanduser().resolve(strict=False)
captured_path = captured_path / ".sase" / "artifacts" / pool_relpath
```

`_replacement_for_candidate` (line 421) then does
`resolution = replace(resolution, resolved_path=captured_path)`, and `_path_file_text`
in `src/sase/artifact_ref_prompt_rendering.py` (line 178) renders
`_FILE_EXPANSION_FORMAT` (`"the {checkout_path} file"`) with `str(resolved_path)`.

The same `resolved_path` is also what feeds staging suppression
(`_stage_artifact_references`) and the artifact-consumption ledger, so it must stay
absolute. Only the prompt-facing string should change.

## Approach

Two small, independent changes. Neither needs a Rust core change:

- The pool **naming algorithm** (sanitizing, bounding, disambiguating) stays where it
  already lives, in `sase_core`'s `artifact_pool_filename`. Only the _label_ handed to
  it changes, exactly as `stage_prompt_artifact` already chooses `source.name`. No new
  `require_rust_binding` name is introduced, so the published-core floor gate
  (`tools/check_sase_core_rs_bindings`, which statically scans `require_rust_binding`
  call sites against the pinned `sase-core-rs` floor) stays green and no cross-repo
  release sequencing is required.
- The prompt-facing **display path** is threaded explicitly from the capture site to the
  renderer rather than inferred by relativizing against the current directory inside a
  generic renderer. Threading keeps the change surgical: only a captured `@file:<path>`
  copy is ever rendered relative. Document, chat, VCS-materialized, and
  `@file:<artifact-id>` expansions keep their current absolute or sidecar-relative prose
  untouched.

## Implementation

### 1. Preserve the source extension in the pool filename

In `src/sase/core/prompt_artifact_staging.py`, inside `capture_prompt_file_ref`, replace
the hardcoded `"file-ref"` label with one that carries the source's suffix:

```python
pool_filename = str(
    require_rust_binding("prompt_artifact_pool_filename")(
        sha256,
        f"file-ref{source.suffix}",
    )
)
```

Keep the `file-ref` stem: the pooled name deliberately does not carry the user's local
basename (the manifest row's `logical_path` / `authored_path` already record where the
bytes came from). Only the extension is added.

Add a short comment stating why the stem is fixed and the suffix is not, so a future
reader does not "fix" it back to `source.name`.

Behavior of the composed label under the existing Rust naming rules — confirm these in
the tests below rather than assuming them:

| Source basename                 | Label passed   | Pool filename                             |
| ------------------------------- | -------------- | ----------------------------------------- |
| `shot.png`                      | `file-ref.png` | `<sha12>-file-ref.png`                    |
| `gtd.md`                        | `file-ref.md`  | `<sha12>-file-ref.md`                     |
| `README` (no suffix)            | `file-ref`     | `<sha12>-file-ref` (unchanged from today) |
| `.bashrc` (`Path.suffix == ""`) | `file-ref`     | `<sha12>-file-ref`                        |
| `bundle.tar.gz`                 | `file-ref.gz`  | `<sha12>-file-ref.gz`                     |

Ordinary alphanumeric suffixes need no sanitizing, so `artifact_pool_filename` does not
append its disambiguator hash and the name is exactly `<sha12>-file-ref.<ext>`. A
pathological suffix (spaces, punctuation, absurd length) still flows through the
existing Rust sanitizer/bounder and yields a safe single path component; that is a
correctness backstop, not a case to special-case in Python.

**Deliberate consequence to record in the docstring:** pool dedup becomes per
`(digest, extension)` rather than per digest. Two byte-identical sources with different
extensions now occupy two pool files. This is intended — the extension is the point —
and it does not affect the durable archive, which is addressed purely by digest through
`artifact_object_relpath` and stays one object.

### 2. Render the pool copy as a workspace-relative path

**`src/sase/artifact_ref_prompt.py`**

- `_capture_file_path_ref` (line 604): return the prompt-facing relative path alongside
  the absolute one. Widen its return to a 3-tuple
  `(record, captured_path, display_path)` where:
  - `captured_path` stays `<workspace>/.sase/artifacts/<pool_relpath>` (unchanged), and
  - `display_path` is `Path(".sase") / "artifacts" / pool_relpath`.

  Build both from the same `pool_relpath` so they can never disagree. Return
  `(record, None, None)` / `(None, None, None)` on the existing early-return branches.

- `_replacement_for_candidate` (line 421): keep
  `resolution = replace(resolution, resolved_path=captured_path)` so the absolute path
  still reaches staging and telemetry, and pass `display_path=display_path` into
  `_artifact_ref_replacement`.

- `_artifact_ref_replacement` (line 475): accept a keyword-only
  `display_path: Path | None = None` and forward it.

**`src/sase/artifact_ref_prompt_rendering.py`**

- `artifact_ref_replacement` (line 57): accept keyword-only
  `display_path: Path | None = None`. Document it as "prompt-facing override for the
  path rendered into the expansion; the returned resolved path is unaffected." Forward
  it to `_replacement_text`.
- `_replacement_text` (line 89): forward it to `_path_file_text` on the
  `kind_type in {"file", "chat"}` branch.
- `_path_file_text` (line 178): render `str(display_path or resolved_path)`. Keep the
  existing `resolved_path is None` guard — the raise must still fire when there is no
  resolved path at all, so check `resolved_path` first and only then choose the string
  to render.

Do **not** change `artifact_resolved_path` or the value `artifact_ref_replacement`
returns as its second element. The absolute path is what `_stage_artifact_references`
adds to `staged_file_paths` (suppressing a re-stage in the following plain-`@path` pass)
and what `_record_artifact_ref_consumption` writes into the consumption ledger. Both
must keep absolute paths.

Net effect on the `#sshot` example:

```text
the .sase/artifacts/pool/5713d120c7da-file-ref.png file
```

## Tests

`tests/test_prompt_artifact_staging.py`

- Extend `test_file_ref_capture_writes_captured_copy_and_metadata` (or add a sibling) to
  assert `record["pool_relpath"]` matches `pool/<12 hex chars>-file-ref.md` for a `.md`
  source — that is, both that the extension is present and that no disambiguator hash
  was inserted.
- Add a `.png` case asserting the pooled filename ends in `-file-ref.png` and the pooled
  bytes match the source.
- Add an extensionless-source case (`README`) asserting the filename is exactly
  `<sha12>-file-ref` with no trailing dot — the pre-existing shape must not regress.
- Add a case for two byte-identical sources with _different_ extensions asserting two
  pool files exist and both records share one `sha256` and one `object_relpath`. This
  pins the intended dedup change;
  `test_file_ref_capture_reuses_pool_for_duplicate_bytes` (two `.md` files) stays valid
  and must keep passing unchanged.

`tests/artifact_refs/test_preprocessing_effects.py`

- `test_file_ref_expands_to_captured_copy` currently asserts
  `expanded == f"Read the {captured_path} file."` against the absolute staged path.
  Update it to assert the expansion is
  `"Read the .sase/artifacts/pool/<sha12>-file-ref.md file."` while `staged_paths` still
  holds the **absolute** path, and that reading that absolute path still yields the
  source bytes. The test already `chdir`s into `tmp_path`, so it can also assert the
  rendered relative path resolves to the same file as the staged absolute path — that is
  the property that actually matters to the agent.
- Add a `.png`-source case asserting the expansion ends with `-file-ref.png file.`, so
  both halves of this plan are covered end to end by one assertion.
- Assert the rendered text contains no absolute path — e.g. that the expansion does not
  contain `str(tmp_path)`.

Regression coverage to leave alone (verify still green, do not edit):

- `tests/artifact_refs/test_preprocessing_expansion.py::test_expands_document_chat_file_and_fragments`
  and `tests/test_artifact_file_e2e.py` (around line 343) both assert absolute paths for
  `@chat:` and `@file:<artifact-id>` expansions. Those payloads are not `file_path`
  captures, so they must keep rendering absolute paths. If either changes, the
  `display_path` threading has leaked past the capture path.

## Docs

- `docs/agent_images.md`, the `pool/<sha12>-<basename>` bullet in "Prompt Artifact
  Staging and Archive": document that a captured `@file:<path>` reference is pooled as
  `pool/<sha12>-file-ref<.ext>` — fixed stem, extension mirrored from the source — while
  staged `@path` and artifact references keep `pool/<sha12>-<basename>`. Note that the
  published object store is unchanged and remains digest-addressed with no extension.
- `docs/llms.md`, the staging paragraph in the preprocessing-pipeline section (near the
  `.sase/artifacts/pool/` mention): state that a captured `@file:<path>` reference
  expands to a workspace-relative `.sase/artifacts/pool/...` path, matching the
  `.sase/artifacts/home/` convention already used by the plain `@path` pass.

## Verification

Run `just install` first if this workspace has not been used recently, then
`just check`. If the scoped test lane escalates or reports an unusual selection, run
`just check-full` through the `/sase_monitor` skill with the `TESTING` / `TESTED` status
pair.

Targeted runs while iterating:

```bash
just test tests/test_prompt_artifact_staging.py \
          tests/artifact_refs/test_preprocessing_effects.py \
          tests/artifact_refs/test_preprocessing_expansion.py \
          tests/test_artifact_file_e2e.py
```

Manual confirmation, if a screenshot is available: expand a prompt containing `#sshot`
and check that the rendered sentence reads
`the .sase/artifacts/pool/<sha12>-file-ref.png file` and that the named file exists
relative to the workspace root and opens as a PNG.

## Risks and notes

- **Relative path usability.** The rendered path is relative to the agent's workspace
  root, which is the agent's working directory — the same assumption
  `_capture_file_path_ref` already makes when it builds the absolute path from
  `Path.cwd()`, and the same convention `@.sase/artifacts/home/...` references have used
  for a long time. A provider whose file-reading tool demands an absolute path can still
  resolve it against its working directory. If this turns out to be friction in
  practice, the fix is to render the absolute path again for that provider, not to
  revert the extension work — the two changes are independent.
- **Pool files already on disk.** Existing extensionless `<sha12>-file-ref` files stay
  valid: they are still referenced by their manifest rows' `pool_relpath` and are still
  garbage-collected by `_prune_prompt_artifact_pool`. No migration, no backfill. A
  re-referenced source is simply re-captured under the new name, which costs one extra
  pooled copy of that file once.
- **`Path.suffix` takes only the final component**, so `bundle.tar.gz` pools as
  `file-ref.gz`. That is the standard interpretation of "the extension" and keeps the
  rule trivially predictable; capturing compound extensions is not worth the special
  case.

## Out of scope

Do not fix these here; file task beads with `/sase_new_task` if they are worth pursuing.

- `_capture_file_path_ref` passes `expanded_ref=raw_ref` to `capture_prompt_file_ref`,
  but the expanded prompt contains the rendered prose, not the raw `@file:` token. So
  `rewrite_prompt_artifact_links` — which matches manifest rows against the prompt by
  `raw_ref` and `expanded_ref` — never links a captured `@file:` reference in the
  published prompt archive. This plan does not change that behavior in either direction,
  but the relative path introduced here is what a future fix would want to record as
  `expanded_ref`.
- `_print_captured_file_ref_notice` reports the logical path but not where the bytes
  landed. Adding the pool relpath to that stderr notice would be useful and is a
  one-line change, but it is not part of this plan.
- The digest-addressed object store (`artifact_object_relpath`) deliberately stays
  extensionless. Do not add a suffix there.
