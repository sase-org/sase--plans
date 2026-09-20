---
tier: tale
title: Re-host the sase-core-rs PyPI deletion gate on athena
goal:
  A pending gate shell on athena, backed by a committed wall-aware deletion driver in
  sase-core, lets the maintainer delete the reviewed sase-core-rs releases with a
  confirmation link they can actually open, taking the project back under the 10 GiB
  PyPI cap.
size: medium
proposed_by: bbugyi200.athena.0o4
create_time: 2026-09-20 12:11:28
status: wip
---

# Re-host the `sase-core-rs` PyPI deletion gate on athena

## Why this exists

`sase-core-rs` is at **265 releases / 1323 files / 10.30 GB**, i.e. **95.9% of the 10
GiB PyPI project cap** with roughly 0.4 GB (~5 releases) of headroom. Uploads already
fail mid-release when the cap trips (that is how `0.34.48` was published with 3 of its 5
files). Bead `sase-13t.1` ("Reclaim PyPI storage below the limit") is the phase that
frees the space, and it is blocking `sase-13t.3`.

Two credentialed deletion attempts have been made from **apollo**, both from gate
`custom-fcaeb8b6`. Both died with
`ValueError: No CSFR found in /manage/project/sase-core-rs/release/0.1.2/` **after** a
successful password + TOTP login, having deleted nothing. `docs/pypi-retention.md` in
`sase-core` already diagnoses this correctly: it is not a CSRF bug, it is PyPI's
unrecognized-device login-confirmation wall. PyPI serves the interstitial instead of the
release management form, and `pypi-cleanup` only checks that the post-login URL is not
the login URL, so it treats the unconfirmed session as a success and walks into the
first release page.

The confirmation email PyPI sends says, verbatim:

> A login attempt was made from an unrecognized device. To complete your login, please
> visit the following link **from the same device from which you attempted to log in**:
> `https://pypi.org/account/confirm-login/?token=...`

That is the blocker. The maintainer cannot open that link from apollo, so the wall can
never be cleared for a login that originated on apollo. The maintainer **can** open it
on **athena**. So the deletion has to be driven from a gate created on athena, where the
login attempt and the confirmation both originate from the same device.

This plan re-hosts that gate on athena, and hardens the deletion driver so the
confirmation is consumed **inside the same HTTP session that triggered it**, instead of
by a separate browser whose cookies the dead `pypi-cleanup` process could never have
used.

## Non-goals

- Do not change release cadence, wheel size, or the CI pre-flight guard. Those are other
  phases of epic `sase-13t` (`sase-13t.3`, `sase-13t.4`).
- Do not close the parent epic bead `sase-13t` or any ancestor.
- Do not cancel or touch apollo's gate `custom-fcaeb8b6`. It is not visible from athena
  and is not this plan's business.
- Do not delete anything from a script that runs without a human answering a gate.
  Deletion on PyPI is permanent: a deleted version and its filenames can never be
  re-uploaded.

## Ground truth already established (do not re-derive, but do re-measure)

- Retention tooling lives in the `sase-core` repo: `.github/scripts/pypi_retention.py`
  (`plan` / `regex` / `compare` / `verify`) and `docs/pypi-retention.md`. Both are
  committed (`f68baeb`, `0b48713`). Open that repo with `sase repo open sase-core`.
- `pypi-cleanup@0.1.10` is the newest stable release; `0.1.11.dev20260320034404` is
  byte-identical in `pypi_cleanup/__init__.py`. Upgrading is not a fix. Upstream issues
  #42 and #49 both resolve by confirming the emailed login; open PR #48 exists to handle
  the redirect.
- In `pypi_cleanup/__init__.py` the login happens **after** version selection and only
  when the selection is non-empty, so there is no credential-free way to provoke the
  confirmation email.
- Pin audit from 2026-09-20 (`sase`, `sase-github`, `sase-telegram`,
  `sase-research-artifacts`, `sase-core`): `sase` floor is `>=0.34.48`;
  `sase-research-artifacts` has floor `>=0.34.23` **and an exact `==0.34.23` pin** in
  its publish smoke test; `sase-telegram` has only a stale transitive `uv.lock` entry
  (`0.32.61`) that nothing reads. `sase-nvim` was not cloned and so was not grepped.
- The keep window **slides**. The apollo-reviewed list was 232 versions
  (`0.1.2 .. 0.34.18`); `plan --keep 30` today selects **235** (`0.1.2 .. 0.34.21`),
  keeping `0.34.22 .. 0.34.68`. The list must be regenerated from live data at gate
  build time and again immediately before deleting. Never reuse a stale list.
- Gate command execution facts, from `src/sase/notification_gates/command_runner.py`:
  commands run with `shell=False`, `cwd` set to the gate bundle directory, the gate
  input arrives as canonical JSON on **stdin**, and there is **no execution timeout**.
  Gate _actions_ (`operations`) take no inputs, so no action may require a secret.
  Resource roles are `attachment`, `command`, `editable`, `preview`
  (`src/sase/notification_gates/model_request.py`).
- A gate whose selected option's command fails stays pending and answerable — that is
  how apollo's gate survived two failed attempts. The retry loop in this plan depends on
  it.
- `gog` (`/home/bryan/bin/gog`) gives read-only Gmail access on athena;
  `gog gmail search 'from:pypi.org newer_than:7d' -p` already lists the two
  `[PyPI] Unrecognized login to your PyPI account` messages. `-p` truncates bodies, so
  read the full body with `-j` (or the raw message) before extracting the URL.
- athena has `uv`, `uvx`, `/usr/bin/python3` (3.13), `xdg-open`, `firefox`, and
  `google-chrome`.

## Step 1 — Re-audit the consumer pins

Open every consumer that could pin a doomed version and re-check it, because the keep
window has slid three versions since the last audit:

```bash
sase repo open sase-research-artifacts -r "Re-audit sase-core-rs pins before an irreversible PyPI deletion"
sase repo open sase-nvim -r "Re-audit sase-core-rs pins before an irreversible PyPI deletion"
sase repo open sase-telegram -r "Re-audit sase-core-rs pins before an irreversible PyPI deletion"
sase repo open sase-github -r "Re-audit sase-core-rs pins before an irreversible PyPI deletion"
```

Grep each printed checkout (plus this project's own checkout) for `sase-core-rs` in
`pyproject.toml`, `uv.lock`, `requirements*.txt`, Justfiles, and CI workflow YAML.
Record every exact `==` pin and every floor. `sase-nvim` was never audited before —
audit it now.

An `==X` pin that exists in order to **fail** (a negative test) does not need
protecting; say so explicitly in the note if you find one.

Carry the result forward as an explicit `--keep-version` list. At minimum expect
`--keep-version 0.34.23` (the `sase-research-artifacts` exact pin) and
`--keep-version 0.34.48` (this project's floor). Pass them even when they currently fall
inside the keep-30 window: more releases will publish before the gate is answered, and
the window will slide past them.

## Step 2 — Add a wall-aware deletion driver to `sase-core`

Write `.github/scripts/pypi_delete.py` in the `sase-core` checkout, as a self-contained
`uv run --script` file in the same style as `pypi_retention.py` (PEP 723 header,
`requires-python = ">=3.11"`, dependency on `requests`). It must be committed. The last
session's tooling was left uncommitted and was wiped out of the checkout; do not repeat
that.

Why a driver instead of `uvx pypi-cleanup@0.1.10 --do-it`: `pypi-cleanup` exits the
moment it hits the wall, and its `requests.Session` dies with it. Opening the
confirm-login link in a browser afterwards cannot confirm a session that no longer
exists, which is exactly why two apollo attempts and two confirmation emails have
produced zero deletions. The driver keeps the session alive across the confirmation.

It reimplements only the small login/delete flow that `pypi_cleanup/__init__.py` already
performs — read that file first (`uv venv` + `uv pip install pypi-cleanup==0.1.10`, or
read it from the sdist) and mirror it:

1. `GET /account/login/`, parse the `csrf_token` hidden input.
2. `POST /account/login/` with `csrf_token`, `username`, `password` and a
   `referer: https://pypi.org/account/login/` header. A response URL still equal to
   `/account/login/` means bad credentials — fail loudly and distinctly.
3. If the response URL starts with `/account/two-factor/`, parse that page's CSRF and
   `POST` `{csrf_token, method: "totp", totp_value: <totp>}` with a `referer` of the
   two-factor URL. A response URL unchanged from the two-factor URL means an invalid or
   expired code — fail loudly and distinctly, because that is the failure a fresh TOTP
   fixes and it must never be reported as the wall.
4. `GET /manage/project/sase-core-rs/release/<first version>/` and look for the
   `confirm_delete_version` CSRF field.
5. **Wall handling**, when that field is absent:
   - Search Gmail for the newest `[PyPI] Unrecognized login to your PyPI account`
     message whose internal date is **at or after** the moment this run started its
     login, using `gog gmail search ... -j --no-input`. Never accept an older message:
     there are already two stale confirmation emails from the apollo attempts, and
     consuming one of those would silently do nothing.
   - Extract the `https://pypi.org/account/confirm-login/?token=...` URL from the full,
     untruncated body.
   - `GET` that URL **with the same `requests.Session`**. This is the whole point of the
     driver: the pending login and its confirmation then share one cookie jar, so the
     "same device" requirement is satisfied by construction rather than by hoping a
     separate browser lands on the same session.
   - Also print the URL on stderr and best-effort `xdg-open` it, so the maintainer can
     click it themselves if the in-session `GET` is rejected. Treat a missing `DISPLAY`
     or a missing `xdg-open` as non-fatal.
   - Re-`GET` the release page, polling every 5 seconds for up to 10 minutes (make the
     budget a flag), until the delete form appears. If it never appears, exit non-zero
     with a message that names the wall, the email timestamp used, and the confirm URL.
6. Delete loop, one version at a time, mirroring `pypi-cleanup`: `GET` the release page,
   parse the `confirm_delete_version` CSRF, `POST`
   `{csrf_token, confirm_delete_version: <version>}` with the release page as `referer`.
   Log each deletion to stderr. A `404` on a release page means the version is already
   gone — count it as such and continue; anything else aborts.
7. Print exactly one JSON object on **stdout** and nothing else. Diagnostics all go to
   stderr. Include at least `status`, `deleted`, `already_gone`, `failed`, `requested`,
   `wall_encountered`, `confirmation_source`, `oldest_kept`, and `bytes_used_after`.

Hard requirements on the driver:

- It reads `username`, `password`, `totp` and the delete list **from its own arguments
  and environment**, never from an interactive prompt. It must never call `input()` or
  `getpass`, because it runs with no TTY.
- The password arrives via an environment variable, never via `argv`.
- It refuses to delete when the delete list is empty, when any listed version is newer
  than a `--keep-version` it was told to protect, or when `--confirm` is absent.
- It never writes the password or the TOTP to stdout, stderr, or any file.

Extend `docs/pypi-retention.md` in the same commit: document `pypi_delete.py`, and
rewrite the wall entry in **Troubleshooting** to record the real resolution — the
confirmation must be consumed by the same session that triggered it, which is why a
browser click after `pypi-cleanup` has exited does not help, and why the login must
originate from a machine whose operator can reach the emailed link.

Keep `pypi_retention.py` untouched and keep using it for `plan`, `regex`, `compare`, and
`verify`; the credential-free dry run must keep proving the selection with upstream
`uvx pypi-cleanup@0.1.10 --query-only`, so the reviewed set is still what the audited
third-party tool selects.

Run `just check` in `sase-core` only if you changed Rust or Python that it covers; a new
`.github/scripts` script plus a Markdown file does not require it (and the last session
recorded the same judgement). Say which you did.

## Step 3 — Regenerate the delete list and prove it credential-free

From the `sase-core` checkout:

```bash
uv run --script .github/scripts/pypi_retention.py plan --keep 30 \
  --keep-version 0.34.23 --keep-version 0.34.48 \
  --output delete-list.txt
REGEX=$(uv run --script .github/scripts/pypi_retention.py regex --list delete-list.txt)
uvx pypi-cleanup@0.1.10 --package sase-core-rs --query-only --version-regex "$REGEX" 2>&1 \
  | uv run --script .github/scripts/pypi_retention.py compare --list delete-list.txt --allow-subset
```

`compare` must report a match. If it reports extra versions, stop and re-derive — extra
versions are never tolerable. Record the exact counts, the byte totals, the delete
boundary (`<oldest> .. <newest deleted>`), and the oldest version that survives; the
gate presentation quotes all of them.

Do not commit `delete-list.txt`. It is regenerated on every use.

## Step 4 — Author the gate request

Write a schema-version 3 `kind: "custom"` gate request. Follow the `/sase_gate` skill
for the exact shape. The decision:

```text
delete OR manual OR defer
```

with `primary_branch: ["delete"]`.

**Presentation.** `icon: "🧹"`. `title` is the one-line decision naming the live count
and that it is irreversible. `panel: "pypi"` with `panel_icon: "📦"`.
`sender: "pypi-retention"`. `chip`: glyph `📦`, label `pypi`. `origin_agent` set to your
own agent name. `tags`: `pypi`, `retention`, `irreversible`. The `notes` lines are the
maintainer's entire view of the decision, so they must state, in this order:

- the live measurement, the delete/keep counts, the delete boundary, the oldest
  survivor, and the headroom this buys;
- that deletion is permanent and the filenames are burned forever;
- that this gate runs **on athena**, and that the confirmation link PyPI emails must be
  reachable from athena — that is the entire reason this gate is not apollo's;
- that a failed attempt leaves the gate answerable, so the fix for a wall or an expired
  code is to answer again with a **fresh** TOTP, not to restart anything;
- that the TOTP must be generated immediately before submitting.

**Options.**

- `delete` (`🗑️`, `feedback: "optional"`). Inputs: `username` (`line`, required),
  `password` (`line`, required, `secret: true`), `totp` (`word`, required,
  `secret: true`). Its command regenerates the delete list from live PyPI with the same
  `--keep`/`--keep-version` policy, re-runs the credential-free `--query-only` compare
  with `--allow-subset`, refuses if `compare` reports extra versions or if the newest
  doomed version is not strictly below every protected version, and only then invokes
  `pypi_delete.py`. `result_schema` requires `status` with `const "deleted"`, so any
  wall, bad password, or expired code exits non-zero and leaves the gate answerable.
- `manual` (`✍️`, `feedback: "optional"`, no inputs) — "I deleted them myself; just
  verify". Runs `pypi_retention.py verify` and returns its measurements.
- `defer` (`✋`, `feedback: "required"`, no inputs) — a no-op command that prints
  `{"status": "deferred"}`.

**Actions** (`operations`; repeatable, credential-free, never settle the gate):

- `dry_run` (`🔍`, key `d`, `display: "text"`) — re-measure live PyPI, regenerate the
  list under the pinned policy, run the upstream `--query-only` compare, and print the
  counts, bytes, boundary, projected headroom, and the `compare` verdict. It rewrites
  the stored list, so it must declare that resource under `targets`.
- `login_confirmation` (`📧`, key `c`) — find the newest
  `[PyPI] Unrecognized login to your PyPI account` message via `gog`, print its
  timestamp and the full `confirm-login` URL in the action body, and best-effort
  `xdg-open` it on athena. Print the timestamp prominently: there are already stale
  apollo-era confirmation emails, and opening one of those does nothing. This is the
  maintainer's manual escape hatch when the driver's in-session confirmation is refused.
- `measure` (`📊`, key `m`) — the cheap live measurement alone, with no `uvx` run.

**Resources.** `commands/*` with role `command`, each `#!/usr/bin/env python3` (athena
has `/usr/bin/python3` 3.13; fall back to an absolute interpreter if the executor's
`PATH` turns out not to carry one). Attach `.github/scripts/pypi_retention.py`,
`.github/scripts/pypi_delete.py`, and `docs/pypi-retention.md` into the bundle as
`attachment` resources via `source`, so the gate stays self-contained if the checkout
moves or is wiped — that is how the last session recovered its tooling. The mutable
delete list is an `editable` resource named in `dry_run`'s `targets`.

Commands run with `cwd` set to the bundle, so reference the attachments by
bundle-relative path. Resolve `uv` and `gog` with `shutil.which` and fall back to
`~/.local/bin/uv` and `~/bin/gog`; do not assume the executor's `PATH` matches an
interactive shell's. Do not hard-code any workspace path — resolve the `sase-core`
checkout at build time with `sase repo path sase-core` (or the path `sase repo open`
printed) and copy the files into the bundle, so nothing in the durable gate depends on a
workspace that may not exist when the gate is answered.

**Shell block.** `workspace: "inherit"`, `next.output: ["results"]`,
`next.fork: "family"`, `pending_status: "PYPI"`, `settled_status: "PYPI OK"`, and an
explicit `gate_timeout_seconds` of `604800` (7 days) — the default 24 hours is short for
a decision that has already waited most of a day, and a timeout throws the work away.
Branch follow-ups:

- `delete` → the verification prompt in Step 6.
- `manual` → the same verification prompt.
- `defer` → `{ "prompt": null }`; the family simply ends.

Leave `timeout`, `stopped`, and `failed` unset.

## Step 5 — Rehearse, then create the gate shell and stop

Before creating anything, rehearse every credential-free path by hand so the maintainer
never meets a broken button:

- run each `commands/*` script directly with a representative JSON payload on stdin and
  confirm it prints exactly one JSON value on stdout that satisfies its `result_schema`;
- run the `dry_run`, `login_confirmation`, and `measure` action commands end to end —
  `login_confirmation` should find the 2026-09-20 15:43 UTC message and print its full
  URL;
- exercise `pypi_delete.py`'s argument validation and its refusal paths (empty list,
  missing `--confirm`, a protected version present in the list) without credentials.

Then create the gate shell in the **foreground**:

```bash
sase gate create --shell < gate-request.json > gate-descriptor.json
```

Print the descriptor. **Creating a gate shell ends your turn** — do not wait, poll,
`sase gate wait`, or run bundle commands by hand. If the call returns anything other
than a descriptor, your turn has not ended: read the error and report it.

Report in your response: the live measurement, the delete/keep counts and boundary, the
pin-audit result including `sase-nvim`, the `sase-core` commit, and a one-paragraph
statement that the maintainer must open the PyPI confirmation link **on athena** and
answer with a fresh TOTP.

## Step 6 — Follow-up after the gate settles (the gate shell's `next`)

The follow-up agent receives the option's validated JSON results. It must:

1. Re-measure and assert the outcome from the `sase-core` checkout, deriving
   `--expect-oldest` from the `oldest_kept` value in the results rather than hard-coding
   it (the keep window slides, and `verify --expect-oldest` tolerates releases published
   since the deletion):

   ```bash
   uv run --script .github/scripts/pypi_retention.py verify --list delete-list.txt \
     --require-version 0.34.48 --expect-oldest <oldest_kept from results>
   ```

   Add `--require-version` for every exact pin Step 1 found.

2. Append the outcome to bead `sase-13t.1` with `sase bead note`: the measured before
   and after, the number deleted, whether the wall was hit and how the confirmation was
   consumed, and the `sase-core` commit.

3. Close `sase-13t.1` **only if** `verify` passes and
   `sase bead epic-symbols sase-13t.1` reports no leftover `--epic-symbol` entries:

   ```bash
   sase bead epic-symbols sase-13t.1
   sase bead close sase-13t.1 --note "<what you verified>"
   ```

   This bead was launched on apollo and this agent is not its assignee, so the close may
   be refused. If it is, leave the note standing, do not force anything, and report that
   the phase is complete but must be closed from apollo. Never close `sase-13t` or any
   ancestor.

4. If `verify` fails, or if the wall was never cleared, record the failure as a note on
   `sase-13t.1` and report it. Do not retry deletion outside a gate.

## Risks and the fallback if the wall still wins

- **The in-session confirmation may be refused.** PyPI may bind `confirm-login` to a
  browser session rather than to the signed token's pending login. The driver covers
  this by printing and `xdg-open`-ing the URL and then polling the release page, so a
  manual click on athena still unblocks the same live session. That is strictly better
  than today, where the session is already dead when the maintainer clicks.
- **If both paths fail**, the documented fallbacks, in order: install `pypi-cleanup`
  from upstream PR #48, which exists precisely to handle this redirect; or sign in to
  PyPI in a browser on athena first, clear the confirmation, and only then answer the
  gate, so the device is already recognized when the driver logs in. Record whichever
  worked in `docs/pypi-retention.md`.
- **Partial deletion is safe to resume.** Every run regenerates the list from live data
  and `compare --allow-subset` tolerates versions that are already gone, so a run that
  dies after 80 deletions is resumed simply by answering the gate again.
- **Headroom is ~5 releases.** If a release publishes and trips the cap before the gate
  is answered, it will land partial like `0.34.48` did. That is a reason to get the gate
  in front of the maintainer quickly, not a reason to delete without one.

## Definition of done

- `sase-core` has a committed `.github/scripts/pypi_delete.py` and an updated
  `docs/pypi-retention.md` whose Troubleshooting entry records the real resolution.
- A pending gate shell exists **on athena** whose `dry_run`, `login_confirmation`, and
  `measure` actions have all been rehearsed and work.
- The gate's notes state the live numbers, the irreversibility, the athena
  device-confirmation requirement, and the fresh-TOTP retry loop.
- Answering `delete` either deletes the reviewed versions or fails in a way that leaves
  the gate answerable, and the follow-up agent verifies the result and records it on
  `sase-13t.1`.
