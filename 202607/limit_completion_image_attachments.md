---
create_time: 2026-07-10 10:33:25
status: done
prompt: 202607/prompts/limit_completion_image_attachments.md
tier: tale
---
# Plan: Limit Agent Completion Image Attachments

## Problem and root cause

Completed agents discover added or modified image files and retain their paths in the execution result, `done.json`, and
the persistent artifact index. The same image list is passed to
`src/sase/axe/run_agent_runner_finalize.py::send_completion_notification()`, which currently appends every path to the
shared notification's `files` list without a count check.

Markdown-derived PDFs take a different path. SASE counts the discovered Markdown sources before rendering and, when
there are more than `MAX_MARKDOWN_PDF_ATTACHMENTS` (10), renders and attaches none of them and adds a completion note
explaining why. Telegram therefore receives no generated PDFs in the over-limit case. For images, Telegram receives the
unbounded shared `files` list and correctly sends each supported path as a photo or animation. The missing policy is in
the shared SASE completion-notification producer, not in Telegram's transport loop.

The intended behavior is:

- Attach all automatically discovered completion images when there are 10 or fewer distinct images.
- Attach none of those images when there are more than 10, and explain the omission in the completion message.
- Continue retaining every discovered image in agent metadata and persistent artifact storage so ACE and other artifact
  consumers remain complete.
- Leave chat transcripts, diffs, generated PDFs, videos, failure reports, logs, and explicitly created artifacts on
  their existing paths and policies.

## Implementation design

1. Define a named image-attachment threshold alongside the completion image discovery contract in
   `src/sase/axe/image_attachments.py`. Keep it separate from the Markdown PDF constant so each attachment class has an
   explicit policy even though both currently use a threshold of 10.

2. Update `send_completion_notification()` in `src/sase/axe/run_agent_runner_finalize.py` to deduplicate the automatic
   image candidates against files already selected for the notification before applying the threshold:
   - At or below the threshold, append the image candidates in their existing stable order.
   - Above the threshold, append none of the automatic image candidates.
   - Add a human-readable note containing the distinct image count and the configured limit when images are skipped.

   Apply this only while assembling the outbound notification. Do not truncate `AgentExecResult.image_paths`,
   `done.json["image_paths"]`, or default artifact persistence. This keeps the delivery safety policy from degrading the
   artifact record and mirrors the product-level placement used for Markdown PDF notifications.

3. Do not add a second filter in `sase-telegram`. Telegram should continue treating `Notification.files` as the shared
   attachment contract; central enforcement keeps all notification consumers consistent and avoids transport-specific
   divergence.

## Test plan

Extend `tests/test_run_agent_runner_notifications.py` with boundary and regression coverage:

- Exactly 10 distinct automatic image paths are included, in order, with no limit note.
- 11 distinct automatic image paths are all omitted, while non-image completion files remain attached.
- The over-limit notification note reports both the image count and the limit.
- Duplicate or already-present paths are counted according to the deduplicated notification candidates, preventing
  duplicate inputs from triggering or bypassing the policy.
- Existing explicit-artifact, PDF, video, failure-file, ordering, and deduplication behavior remains unchanged.

Where useful, extend the existing finalization attachment test to confirm that an over-limit image set is still written
to `done.json` and remains available through the artifact index even though the completion notification omits it.

No `sase-telegram` source change is planned; its existing single-image and mixed-attachment tests establish that it
sends every image present in the shared notification payload.

## Verification

Run the focused notification and finalization tests first, then the repository-required checks:

```bash
just install
pytest tests/test_run_agent_runner_notifications.py tests/test_axe_run_agent_exec_finalize_attachments.py
just check
```

Review the final diff to ensure only delivery assembly is capped and the stored agent artifact record remains lossless.
