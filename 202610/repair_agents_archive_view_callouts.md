---
tier: tale
size: small
title: Repair formatting-sensitive decision callouts in the Agents Archive plan
goal:
  The existing Agents Archive epic plan survives archival formatting and committed-plan
  validation without changing its decisions or implementation scope.
proposed_by: bbugyi200.athena.0zd
create_time: 2026-10-10 06:30:44
status: wip
---

# Repair the Agents Archive plan's decision callouts

## Diagnosis and selected fix

Repair `/home/bryan/.sase/plans/202610/agents_archive_view.md`. This is a small,
single-agent document repair. The user asked to fix this file's validation errors, or
correct validation logic if it was wrong.

The source passes both `validate_plan(content, "epic", mode="launch")` and
`validate_plan(content, "epic", mode="archived")` with no diagnostics. The failure
occurs after `archive_plan_file` in `src/sase/sdd/plan_archive.py` calls
`format_with_prettier`, before committed-plan validation. Prettier interprets an
underscore in each of two decision identifiers as emphasis paired with later italic text
in the same blockquote paragraph. It changes `view_name` to `view*name` and
`glossary_archive_term` to `glossary*archive_term`. The strict validator correctly
rejects these changed identifiers.

Use a standalone blockquote paragraph for each affected callout header, followed by a
blank blockquote line and its existing prose. This is accepted by the existing grammar
and remains stable through repeated Prettier passes. Keep the validator and formatter
implementations unchanged: this repair addresses the requested document using existing
supported syntax.

Planning-time experiments operated only on strings in memory. They established that the
two edits below preserve all frontmatter, five decisions, six callouts, eight phases,
and ten dependency edges. The repaired text passed archive preparation, including
create-time insertion and tier normalization, and two further formatting passes with no
committed-plan diagnostics.

## Implementation

1. Read the current target file and retain its original bytes for comparison. The
   investigated version had SHA-256
   `2c23a7e537b7713c2ff12d9c80d6eed7f9c3d474dd759aa33df26b256c9991df`. If it has
   changed, inspect the current callouts and apply the same localized repair without
   overwriting intervening work.
2. Make exactly these two body edits, retaining the rest of each paragraph:

   Replace the first line of the naming callout:

   ```markdown
   > [!decision] view_name = history Every user-facing "Archive" string reads "History"
   ```

   with:

   ```markdown
   > [!decision] view_name = history
   >
   > Every user-facing "Archive" string reads "History"
   ```

   Replace the first line of the glossary callout:

   ```markdown
   > [!decision] glossary_archive_term Add the `glossary:agent-archive` strand through
   ```

   with:

   ```markdown
   > [!decision] glossary_archive_term
   >
   > Add the `glossary:agent-archive` strand through
   ```

   The literal blank `>` line matters: an ordinary continuation line can be joined back
   into the header paragraph by Prettier. Preserve the current decision answers,
   acceptance attribution, phase descriptions, dependency graph, and all remaining
   prose. Write only the two localized changes to the target; avoid a whole-document
   formatting rewrite.

3. Review the resulting diff against the retained original. It should contain only the
   two paragraph splits. Compare parsed frontmatter and normalized decisions/phases
   before and after. Compare callout `(id, key, branch)` triples rather than line
   numbers, which necessarily shift.

## Verification

Use a Python interpreter that can import this checkout's `sase` modules and the
installed `sase_core_rs` extension. The interpreter in the `sase` executable's shebang
is available if no checkout virtual environment has been prepared; `PYTHONPATH=src`
selects the checkout for these short validation probes.

- Validate the repaired file in both `launch` and `archived` modes through
  `sase.sdd.plan_validate.validate_plan_file`; require success with no diagnostics. This
  file already contains system-written answers and `decided_by: auto`, so the
  authoring-only `sase plan validate` command reports expected system-field errors.
  Preserve that legitimate accepted state.
- Apply `sase.file_references.format_with_prettier` to an in-memory copy three times,
  checking `archived` validation and the six callout identities after every pass. Run
  with real Prettier present; an unavailable formatter's no-op fallback is insufficient
  evidence for this bug.
- Exercise the real `archive_plan_file` helper against a disposable source copy and a
  temporary `SddStore`, using `tier="epic"`, `yyyymm="202610"`, and an explicit
  temporary `primary_root`. Follow the isolated store setup in
  `tests/sdd/test_plan_archive.py`. Disable external hosted-link resolution in this
  probe so it cannot touch live sidecar state. Leave formatting and validation real.
  Require a written destination and validate its final bytes using
  `inspect_committed_plan` with no diagnostics. This covers both archive formatting
  passes and the intermediate header projection.
- Confirm the archived copy retains the original decisions and phase graph and that the
  actual target still has only the intended paragraph edits. All disposable outputs
  belong in temporary storage. No tracked source or test changes are needed, so
  repository-wide checks are unnecessary for this document-only repair.

## Completion

Report that the two callouts were repaired and the isolated archival path passed.
Provide the user's existing resume command:

```bash
sase bead work /home/bryan/.sase/plans/202610/agents_archive_view.md --yes
```

Do not execute that command as a test: it materializes and launches the separate
eight-phase Archive feature epic. This repair finishes when its input document has been
corrected and archive preparation has been verified in isolation.
