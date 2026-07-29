- **PLAN:** [../202607/notification_release_report.md](../notification_release_report.md)

 Can you help me fix the `ci_watch` notification action (see the error
in the toast in ~/tmp/screenshots/20260729_093926.png for context) so we instead
see a report of all recent (and pending releases--i.e. times that the `ci_watch`
chop merged a release-please/release-plz PR and/or existing release PRs that
still need to be merged)?

- The lumberjack chop associated with `ci_watch` is defined in my bugyi-chops
  GitHub repo (and configured in my chezmoi repo).
- Make sure that we display a good preview in the right pane of the notification
  panel when one of these notifications is selected.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 