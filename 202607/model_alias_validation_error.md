---
create_time: 2026-07-11 12:37:55
status: wip
prompt: 202607/prompts/model_alias_validation_error.md
tier: tale
---
# Repair the Models-panel alias edit validation failure

## Problem and root cause

Editing `@smarttest` produces a valid alias-only diff, but the shared config-edit planner validates the entire effective
configuration before allowing any write. The host-specific chezmoi overlay still defines
`chat_install.command: install_sase_github`. That field was intentionally removed when chat-driven updates moved to the
built-in `sase update --json` engine, so the current schema reports `chat_install.command` as an additional property and
the Models panel correctly refuses to write the otherwise-unrelated alias edit.

This is a stale user-configuration migration, not a defect in the user-defined model-alias schema. The fix should keep
the whole-config safety check intact and remove the retired override from the linked `chezmoi` repository.

## Implementation plan

1. Update the host-specific SASE overlay in the linked `chezmoi` repository by removing the obsolete
   `chat_install.command` setting. Since it is the only field in that section, remove the empty `chat_install` mapping
   as well, while preserving the rest of the host configuration exactly.

2. Check all effective SASE configuration layers for the other retired companion key, `chat_install.sync_workspace`, and
   for any remaining `chat_install.command` definition. Do not broaden the cleanup beyond retired chat-install keys;
   report any unrelated validation issue separately instead of silently changing user configuration.

3. Validate the repaired configuration against the current SASE schema and merged-layer behavior. Confirm that the host
   overlay remains valid YAML, that the effective config no longer reports the `chat_install.command` additional
   property, and that chat-driven updates still rely on the built-in update engine without a custom command.

4. Re-run the Models-panel edit flow without committing an incidental alias change: preview an edit to an existing
   user-defined alias, verify that the preview is valid and enables the write action, then cancel the preview (or use an
   equivalent read-only config-plan check). Review the chezmoi diff to ensure the retired setting is the only intended
   persistent change.

## Acceptance criteria

- The effective SASE configuration has no `chat_install.command` or `chat_install.sync_workspace` setting.
- Current schema validation passes without the screenshot's `property \`command\` is not allowed` diagnostic.
- A user-defined model alias edit can reach a valid, writable preview in the Models panel.
- The linked chezmoi change is limited to removal of the retired host override; no SASE or `sase-core` source change is
  introduced for this configuration-only repair.
