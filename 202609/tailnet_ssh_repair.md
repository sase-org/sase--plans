---
tier: tale
title: Repair managed Tailnet SSH settings and remove MacBook duplicates
goal:
  Keep DNS-based SSH working through the managed config and remove redundant MacBook
  host blocks.
size: small
proposed_by: bbugyi200.athena.0gm
create_time: 2026-09-06 07:37:44
status: wip
---

# Repair the managed Tailnet SSH configuration and clean up the MacBook

## Objective and scope

Make the shared chezmoi-managed SSH host definitions sufficient on their own, retain
working Tailnet DNS names, and then remove the duplicate `athena` and `apollo` blocks
from the MacBook's `~/.ssh/config`. One implementation agent can complete this focused
configuration repair and its live verification; it does not need epic phases.

The durable source change belongs in the linked `chezmoi` repository at
`home/private_dot_ssh/private_tailnet.conf`. Open that repository with
`sase repo open chezmoi -r "Implement the approved Tailnet SSH repair"` and use its
returned path. Read its current `AGENTS.md` before editing. The repository's
`.chezmoiroot` is `home`. This plan authoring turn has made no configuration or
repository source changes.

## Findings established during planning

- The new shared file defines `athena`, `apollo`, `apollo-do`, and `mac`. Both Athena
  and the MacBook already have `Include tailnet.conf` as the first line of their main
  SSH config. The existing include script needs no change for this repair.
- The managed `athena` definition omits `Port 34857`. Athena's OpenSSH daemon listens on
  34857, and the managed file alone yields port 22. The MacBook's later duplicate
  `athena` block currently supplies the missing port, hiding this defect there.
- OpenSSH takes the first obtained value for each setting: the include already supplies
  the DNS hostname, while the duplicate supplies the missing port. Removing that block
  before fixing the include would break the working connection.
- The MacBook's duplicate `apollo` block names the public address `159.223.165.54`, but
  its effective config already uses `apollo.tail297af1.ts.net` from the include. The
  separate `apollo-do` alias deliberately retains public access outside Tailscale.
- The three Tailnet FQDNs below resolve successfully on both Athena and the MacBook.
  MacBook SSH authentication to Athena and Apollo succeeded using those DNS names and
  command-line `HostKeyAlias` overrides pointing to their existing trusted IP entries.
  MacBook authentication to `apollo-do` also succeeded normally.
- Fresh strict SSH connections from the MacBook to the DNS names initially fail because
  its known-host entries are still recorded under the old IP names. Carry verified trust
  forward during implementation; permanently adding IP-based `HostKeyAlias` settings or
  disabling host-key verification is unnecessary.
- The MacBook is reachable as `mac`, user `bbugyi`, and reports hostname `Kellys-MBP`.
  Its non-login shell does not find chezmoi on PATH; `/opt/homebrew/bin/chezmoi` works.
  Athena is user `bryan` and can reach the MacBook with the existing managed identity
  settings.

Expected effective settings after cleanup:

| Alias       | HostName                               | User     | Port    | Other settings                                                  |
| ----------- | -------------------------------------- | -------- | ------- | --------------------------------------------------------------- |
| `athena`    | `athena.tail297af1.ts.net`             | `bryan`  | `34857` | Add this explicit port to the shared source.                    |
| `apollo`    | `apollo.tail297af1.ts.net`             | `bryan`  | `22`    | Preserve existing settings.                                     |
| `apollo-do` | `159.223.165.54`                       | `bryan`  | `22`    | Preserve the existing public fallback.                          |
| `mac`       | `kellys-macbook-pro.tail297af1.ts.net` | `bbugyi` | `22`    | Preserve `IdentityFile ~/.ssh/id_rsa` and `IdentitiesOnly yes`. |

The MacBook also has `artemis`, `home`, `xhome`, `github.com`, and a `Match host *`
keepalive/timeout section. These are distinct settings and must survive byte-for-byte
apart from whitespace immediately adjacent to the two removed blocks. In particular,
`home` and `xhome` provide different routes to Athena and are not duplicate aliases.

## Implementation sequence

1. **Recheck and capture the baseline.** Read current source and live SSH config files
   again, check repository status, and capture effective `ssh -G` output for all aliases
   listed above plus the MacBook's remaining aliases. Confirm both first-line includes
   still exist. On each machine whose live files will change, make private, timestamped
   backups of those files, including the MacBook's `known_hosts` if updated. Record
   checksums so intervening user edits are detected before replacing a file. Keep the
   working Athena-to-Mac connection available throughout the migration. Use bounded SSH
   attempts with `BatchMode=yes`, `ConnectTimeout=5`, `ConnectionAttempts=1`, and
   disabled agent/X11 forwarding for these checks.

2. **Fix the authoritative source.** Add `Port 34857` to `Host athena` in
   `home/private_dot_ssh/private_tailnet.conf`. Retain every current hostname, user,
   fallback address, and Mac identity setting. No SASE application code, memory,
   include-script migration framework, or SSH daemon change is needed.

3. **Validate and deploy the managed file before deleting duplicates.** Render the
   revised file with `chezmoi --source "$opened_repo" cat "$HOME/.ssh/tailnet.conf"` and
   inspect the diff. Parse the rendered file with `ssh -G -F <rendered-file>` for every
   managed alias and compare against the table. Apply only this target on Athena using
   `chezmoi --source "$opened_repo" apply --exclude scripts "$HOME/.ssh/tailnet.conf"`;
   the existing first-line include is already installed. Deploy the same changed source
   to the MacBook with chezmoi as well. To deploy the approved content before host-owned
   commit/publication, stage only a copy of the revised
   `private_dot_ssh/private_tailnet.conf` in a private temporary source directory on the
   MacBook, preview it with
   `/opt/homebrew/bin/chezmoi --source <stage> diff "$HOME/.ssh/tailnet.conf"`, and
   apply that single target with `--exclude scripts`. This staging directory is not a
   repository checkout and needs no other dotfiles. Verify the deployed bytes match the
   source and the SSH file has private permissions. Do not run a broad remote update
   against an older published revision during this staging interval. Remove the
   temporary source after successful verification.

4. **Preserve host identity under the DNS names.** Revalidate the MacBook's existing
   trusted entries with `ssh-keygen -F '[100.87.31.114]:34857'` and
   `ssh-keygen -F 159.223.165.54`, and repeat strict authenticated DNS connections using
   those names as temporary command-line `HostKeyAlias` overrides. Planning verified
   Athena's ED25519 fingerprint as `SHA256:Mn7q4SaQQWmPyDPzi2l4gp84q4E5lRf6dvMUx+z+vGs`
   and Apollo's as `SHA256:QABt0A1VMJPylSIxV6D9RWYQ6G+UvhuAj9cAwElpH4I`. If the current
   DNS entries are absent, add the reverified ED25519 public keys under
   `[athena.tail297af1.ts.net]:34857` and `apollo.tail297af1.ts.net` in the MacBook's
   `known_hosts`, retaining the old entries. Make this idempotent. If a DNS entry is
   already present, verify it; never replace a mismatched key without resolving the
   identity discrepancy. Do not trust an unverified `ssh-keyscan` result. Confirm fresh
   DNS-based connections now work with `StrictHostKeyChecking=yes`, `UpdateHostKeys=no`,
   and no `HostKeyAlias` override before proceeding.

5. **Remove only the MacBook duplicates.** Build and inspect a candidate main config
   deleting the complete standalone `Host apollo` and `Host athena` blocks. Preserve
   `Include tailnet.conf` at the top and every other host and Match section. Compare
   `ssh -G -F <candidate>` against the captured effective settings, including the
   remaining aliases and keepalive options. Abort replacement if the live config has
   changed since the baseline. Install the reviewed candidate atomically while
   preserving private permissions. This is a one-time cleanup of the unmanaged main
   config; the host definitions remain authored only in the chezmoi source.

6. **Verify the final state and complete repository handoff.** Run the acceptance checks
   below, inspect `git diff --check` and the final source diff, and record the backup
   paths and results. Submit the linked repository's source change through the normal
   `sase_final` host-owned completion flow; do not manually create a commit, branch, or
   PR. Its `AGENTS.md` requires `chezmoi update -a --force` after commits: honor that
   requirement in the host-owned post-commit flow and ensure it uses the published
   repair. The normal managed source must carry this fix so later updates preserve the
   deployed content on the MacBook and other machines. The temporary deployment is not a
   substitute for the durable source change.

## Acceptance checks

- Both machines' `ssh -G athena` show the Tailnet FQDN, user `bryan`, and port `34857`.
  Parsing the managed file alone produces the same settings. The other three managed
  aliases retain the table's effective values.
- The MacBook's `~/.ssh/config` contains one first-line `Include tailnet.conf` and no
  standalone `Host athena` or `Host apollo` definition. Its `artemis`, `home`, `xhome`,
  GitHub, and Match settings retain their original effective behavior and content.
- Fresh MacBook-to-Athena, MacBook-to-Apollo, and MacBook-to-`apollo-do` authenticated
  sessions succeed as `bryan`; Athena-to-Mac still succeeds as `bbugyi`. For acceptance
  use normal aliases, strict host-key checking, no port/hostname/host-key overrides, and
  `ControlMaster=no`, `ControlPath=none` so an existing multiplexed connection cannot
  hide a defect. Use a harmless `hostname; id -un` remote command.
- Tailnet FQDNs still resolve on both machines. No Tailnet host is switched to a literal
  address. `apollo-do` remains the intentional pre-existing public-IP exception.
- Scoped chezmoi verification/diff on each deployed target reports agreement with the
  revised source, and a repeated scoped apply produces no further target changes.
- Use these direct parsing, diff, and connection checks for this configuration-only
  repair; a new unit-test suite or a whole SASE application test run is unnecessary.

## Failure handling

If the MacBook is offline when implementation starts, preserve its local definitions and
report that deployment/cleanup remains incomplete. Do not claim completion from a source
edit alone. If parsing or fresh authentication fails, leave the duplicate blocks in
place until the cause is understood. If a failure occurs after replacement, restore only
the affected live files from the recorded backups after checking for intervening edits,
then retest the prior route. Never relax authentication checks to make a test pass.
Report exactly which work is complete and which verification is still blocked.
