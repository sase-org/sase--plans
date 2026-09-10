---
status: done
tier: epic
title: Modernize the quote.md and podcasts.md vault notes
goal: "~/bob/quote.md and ~/bob/podcasts.md read like the vault's recent hand-authored
  notes instead of zorg conversion output: clean frontmatter, real Markdown structure,
  human-readable block IDs, and — for podcasts — one note per show behind a Bases
  dashboard, with every existing inbound and outbound link still resolving.

  "
phases:
  - id: podcasts
    title: Extract podcasts into per-show notes with a Bases dashboard
    depends_on: []
    size: medium
    description:
      "podcasts: create the `[[podcast]]` type note, one `podcasts/<slug>.md` note for
      each of the 15 shows, a `podcasts.base` dashboard modeled on `eat.base`, and
      rewrite `podcasts.md` into a hub note that keeps all 17 legacy block anchors alive
      on its new lines."
  - id: quote
    title: Rewrite quote.md as Obsidian quote callouts
    depends_on: []
    size: medium
    description:
      "quote: convert all 32 zorg quote bullets in `quote.md` into `> [!quote]` callouts
      with human-readable block IDs, drop the dead zorg frontmatter and self-referential
      `[[quote]]` noise, and preserve every outbound wikilink and URL verbatim."
  - id: verify
    title: Verify link integrity and vault render
    depends_on:
      - podcasts
      - quote
    size: small
    description:
      "verify: re-check every inbound block reference, outbound wikilink, and URL
      touched by the two rewrites, confirm `podcasts.base` parses and renders, and
      confirm the vault working tree contains only the intended changes."
proposed_by: bbugyi200.athena.tt
create_time: 2026-09-09 20:00:13
---

- **PROMPT:**
  [prompts/202608/modernize_quote_and_podcasts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/modernize_quote_and_podcasts.md)

# Plan: Modernize `quote.md` and `podcasts.md`

## Context

`~/bob/quote.md` and `~/bob/podcasts.md` are still raw output of the 2026 zorg→Obsidian
migration (`convert_zorg_core.py`). Both carry a ten-line `zorg_*` provenance block in
frontmatter and a body made entirely of zorg bullets: `240410#CP` date/ID prefixes,
`ID::`/`LID::` declarations, `======` section banners, `|`-delimited continuation lines,
and machine-generated `^z-YYMMDD-xx` block anchors.

Bryan no longer uses zorg (per the `obsidian` long-term memory note), and the vault has
an established modernization idiom that these two notes have not received yet.

### Observed vault conventions (the "inspiration" this plan targets)

Surveyed from the vault's recently touched notes:

| Convention                                                                                                                      | Evidence                                                            |
| ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Frontmatter is `parent` + optional `type`, quoted wikilink style                                                                | `recur.md`, `mac_inbox.md`, `sase.md`, `bob.md`                     |
| `type` values are `[[project]] [[area]] [[ref]] [[day]] [[done]] [[restaurant]]`, each with a note whose `parent` is `[[type]]` | `type.md` children                                                  |
| Hub notes carry no `type`; only their per-entity children do                                                                    | `eat.md` has none; `eat/*.md` are `type: "[[restaurant]]"`          |
| `## Tasks` / `## Notes` / `## Future Work` / `## Legacy Notes` section names                                                    | `dev.md`, `body.md`, `cash.md`, `gtd.md`, `mac.md`                  |
| Block IDs are human-readable kebab slugs                                                                                        | `^rahway`, `^unemployment`, `^schedule-reason`, `^capture-priority` |
| Collections of like entities get one note each plus a `.base` dashboard                                                         | `eat.md` → `eat/` (92 notes) + `restaurant.md` + `eat.base`         |
| Bases are embedded with `![[x.base]]`                                                                                           | `dash.md:234`, `dash.md:238`                                        |
| Quotations are rendered as `> [!quote]` callouts with the block ID on its own line below                                        | `ref/blogs/harness_for_rsi.md`                                      |

Obsidian core `bases` is enabled (`.obsidian/core-plugins.json`), alongside community
`dataview`, `obsidian-tasks-plugin` and `obsidian-linter`.

### Why the two notes get different treatments

`podcasts.md` is a list of **entities** with uniform structured attributes (title, URL,
subscribed vs. not). That is exactly the shape `eat.md` was modernized into, and a Base
pays for itself. `quote.md` is **content** — 32 one-to-six line quotations that are read
inline and embedded elsewhere via block references. Exploding those into 32 stub notes
would add friction without adding a queryable attribute worth filtering on, so
`quote.md` stays a single note and gets the callout treatment instead.

## Write target and repository access

**Every phase agent must run this first, from its own workspace directory, for the audit
trail:**

```bash
sase repo open gh:bobs-org/bob -r "<phase id>: modernize quote.md / podcasts.md"
```

**All file writes go to the live vault at `~/bob/`, not to the path that command
prints.** The printed path is a clone pinned to the last commit; the live vault is the
Obsidian Sync working copy and currently has uncommitted modifications (including to
`quote.md`). The GitHub auto-sync that would make the clone a viable write path is still
an open task (`bob.md#^auto-sync-for-sase`). Use the printed clone only if you need
committed history for comparison.

If the project owner would rather these edits land in the sase clone and flow back
through a PR, say so at approval time — that changes the write target for every phase.

### Preconditions and safety

- Obsidian (or `ob`) must not be actively editing `quote.md` or `podcasts.md` while a
  phase runs; Sync will fight the agent for the file.
- Before rewriting either file, snapshot it, because `git checkout --` would discard
  Bryan's _uncommitted_ edits:
  ```bash
  mkdir -p /tmp/bob_modernize_backup
  cp ~/bob/quote.md ~/bob/podcasts.md /tmp/bob_modernize_backup/
  ```
- Do **not** commit in `~/bob/`. The vault's own `bob bulk-git-commit` nightly job
  commits the working tree. Leave the changes staged-free and uncommitted, and do not
  run `sase commit` against the vault.

### Conventions that are explicitly out of scope

- `%person` shorthand (`%joshuai`, `%feynman`, `%yeats`, `%chad`, `%claude`, `%chop`)
  stays **verbatim**. It is a live vault-wide convention (763 occurrences of `%fscarpel`
  alone) and converting it is tracked separately in `bob.md`'s Future Work ("Use
  `type: [[person]]` …").
- Do not rewrite the 36 inbound `[[podcasts#^z-…]]` links scattered across the vault.
  This plan keeps those anchors working in place instead.
- `lit/podcasts.md`, `podcast_ref.md`, and `me.md` (whose `parent` is `[[quote]]`) are
  not modernized here.

---

## Phase `podcasts`: Extract podcasts into per-show notes with a Bases dashboard

Mirrors the `eat.md` → `eat/` + `restaurant.md` + `eat.base` modernization.

### 1. New type note `~/bob/podcast.md`

```markdown
---
parent: "[[type]]"
created: 2026-08-06 00:00
---

Podcast notes link to this note as their `type`.

Use this type for one-note-per-show records under [[podcasts]], including subscription
status, category, and show URL. See [[podcasts]] for the show index, [[podcasts.base]]
for the dashboard, [[podcast_ref]] for episode references, and [[lit/podcasts]] for
literature notes taken from episodes.
```

Set `created` to the actual date the phase runs. Match `restaurant.md`'s wording and
tone.

### 2. Per-show notes under `~/bob/podcasts/`

15 notes, filename = the show's original zorg `ID::` slug (short, stable, and already
what Bryan links by). Schema mirrors `eat/urban_burger.md`:

````markdown
---
parent: "[[podcasts]]"
type: "[[podcast]]"
title: "This Week in Tech"
status: subscribed
category: tech
url: https://twit.tv
zorg_id: twit
source_note: "[[podcasts]]"
source_anchors:
  - z-241027-07
source_dates:
  - "241027"
---

# This Week in Tech

## Notes

## Source

```text
- 241027#07 ID::twit This Week in Tech: https://twit.tv ^z-241027-07
```
````

The fenced `## Source` block holds the original zorg line(s) verbatim, exactly as
`eat/*.md` does. Leave `## Notes` empty unless the source line carried extra prose (see
the last column below).

| File                                      | `title`                       | `status`     | `category`   | `url`                                                                            | `source_anchors` | `source_dates` | Extra content for `## Notes`                                                                                                                                                                                                                                   |
| ----------------------------------------- | ----------------------------- | ------------ | ------------ | -------------------------------------------------------------------------------- | ---------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `podcasts/techmeme.md`                    | Techmeme Ride Home            | subscribed   | tech         | `https://www.ridehome.info/show/techmeme-ride-home`                              | z-240911-1j      | 240911         | —                                                                                                                                                                                                                                                              |
| `podcasts/programming_throwdown.md`       | Programming Throwdown †       | subscribed   | programming  | `https://www.programmingthrowdown.com`                                           | z-240916-0b      | 240916         | —                                                                                                                                                                                                                                                              |
| `podcasts/talk_python.md`                 | Talk Python to Me             | subscribed   | python       | `https://talkpython.fm`                                                          | z-241001-0m      | 241001         | —                                                                                                                                                                                                                                                              |
| `podcasts/python_bytes.md`                | Python Bytes                  | subscribed   | python       | `https://pythonbytes.fm`                                                         | z-241217-0k      | 241217         | Show blurb: hosted by Michael Kennedy and Brian Okken; delivers headlines directly to your earbuds.                                                                                                                                                            |
| `podcasts/ted_radio_hour.md`              | TED Radio Hour                | subscribed   | ideas        | `https://www.ted.com/podcasts/ted-radio-hour`                                    | z-241103-0m      | 241103         | —                                                                                                                                                                                                                                                              |
| `podcasts/the_pete_and_sebastian_show.md` | The Pete and Sebastian Show † | subscribed   | comedy       | `https://peteandsebastianshow.com`                                               | z-241006-0g      | 241006         | `**Inspired by:** E168 of [[podcasts/programming_throwdown\|programming_throwdown]]`                                                                                                                                                                           |
| `podcasts/twit.md`                        | This Week in Tech             | subscribed   | tech         | `https://twit.tv`                                                                | z-241027-07      | 241027         | —                                                                                                                                                                                                                                                              |
| `podcasts/practical_ai.md`                | Practical AI †                | subscribed   | ai           | `https://practicalai.fm`                                                         | z-250513-0y      | 250513         | Show blurb: making artificial intelligence practical, productive & accessible to everyone.                                                                                                                                                                     |
| `podcasts/ai_daily_brief.md`              | The AI Daily Brief †          | subscribed   | ai           | `https://open.spotify.com/show/7gKwwMLFLc6RmjmRpbMtEO`                           | z-250524-0n      | 250524         | —                                                                                                                                                                                                                                                              |
| `podcasts/gtd_podcast.md`                 | Getting Things Done †         | unsubscribed | productivity | `https://gettingthingsdone.com/category/podcast-2`                               | z-240911-1f      | 240912, 240911 | —                                                                                                                                                                                                                                                              |
| `podcasts/the_changelog.md`               | The Changelog †               | unsubscribed | programming  | `https://changelog.com/podcast`                                                  | z-241004-0b      | 241004         | —                                                                                                                                                                                                                                                              |
| `podcasts/corepy.md`                      | Core.py †                     | unsubscribed | python       | `https://podcasters.spotify.com/pod/show/corepy`                                 | z-241004-0c      | 241004         | Blurb: podcast w/ `%lukasz` — "We talk about Python internals, because we work on Python internals." Plus `**Inspired by:** the "Free-threaded Python" episode of [[podcasts/the_changelog\|the_changelog]] on 2024-10-02, which had `%lukasz` on as a guest.` |
| `podcasts/productivity_cast.md`           | ProductivityCast †            | unsubscribed | productivity | `https://open.spotify.com/show/1My7mg48eHTnY6tgBivBSz?si=opxP18dDT-mNQUBxWmpNLQ` | z-241024-07      | 241024         | —                                                                                                                                                                                                                                                              |
| `podcasts/java_pub_house.md`              | Java Pub House †              | unsubscribed | programming  | `https://open.spotify.com/show/0laUgleS6VdmeeCk7hRemf?si=lu3cWdeaTJWK0fdIF58zZw` | z-241024-08      | 241024         | —                                                                                                                                                                                                                                                              |
| `podcasts/the_stackoverflow_podcast.md`   | The Stack Overflow Podcast †  | unsubscribed | programming  | `https://open.spotify.com/show/0e5eoM6w7eW9Wu7wMA04Tr?si=isiTI7EfTAWqF3NaKVgLJw` | z-241024-09      | 241024         | —                                                                                                                                                                                                                                                              |

† = title inferred from the slug because the source line gave only an `ID::`. Keep
these; they are the shows' real published names.

Copy every URL character-for-character from `~/bob/podcasts.md` — several Spotify links
carry long `?si=` query strings that must survive intact.

### 3. `~/bob/podcasts.base`

Model directly on the known-good `eat.base`. Same `filters` / `formulas` / `properties`
/ `views` layout:

```yaml
filters:
  and:
    - file.inFolder("podcasts")
    - note.type == link("podcast")
formulas:
  name: if(title, file.asLink(title), file.asLink())
  subscription:
    if(status == "subscribed", "🎧 Subscribed", if(status == "unsubscribed", "🚫
    Unsubscribed", status))
  topic:
    if(category, if(category == "python", "🐍 Python", if(category == "programming", "💻
    Programming", if(category == "tech", "📰 Tech News", if(category == "ai", "🤖 AI",
    if(category == "comedy", "😂 Comedy", if(category == "productivity", "✅
    Productivity", if(category == "ideas", "💡 Ideas", "🎙️ " + category)))))))), "❓
    Uncategorized")
  listen: if(note.url, link(note.url, "🔗 Listen"), "")
  first_code: if(source_dates, source_dates.sort().slice(0, 1).join(""), "")
  added:
    if(formula.first_code, date("20" + formula.first_code.slice(0, 2) + "-" +
    formula.first_code.slice(2, 4) + "-" + formula.first_code.slice(4, 6)).format("MMM
    YYYY"), "")
  source: link("podcasts", "📄 podcasts.md")
properties:
  formula.name:
    displayName: Show
  formula.subscription:
    displayName: Subscription
  formula.topic:
    displayName: Topic
  formula.listen:
    displayName: Listen
  formula.added:
    displayName: Added
  formula.source:
    displayName: Source
  note.zorg_id:
    displayName: ID
views:
  - type: table
    name: 🎧 Subscribed
    filters:
      and:
        - status == "subscribed"
    groupBy:
      property: formula.topic
      direction: ASC
    order:
      - formula.name
      - formula.listen
      - formula.added
    sort:
      - property: formula.name
        direction: ASC
    summaries:
      formula.name: Count
  - type: table
    name: 🎙️ All Podcasts
    groupBy:
      property: formula.subscription
      direction: ASC
    order:
      - formula.name
      - formula.topic
      - formula.listen
      - formula.added
      - zorg_id
    sort:
      - property: formula.name
        direction: ASC
    summaries:
      formula.name: Count
  - type: table
    name: 🚫 Unsubscribed
    filters:
      and:
        - status == "unsubscribed"
    groupBy:
      property: formula.topic
      direction: ASC
    order:
      - formula.name
      - formula.listen
      - formula.added
    sort:
      - property: formula.name
        direction: ASC
  - type: table
    name: 🕐 Recently Added
    order:
      - formula.name
      - formula.topic
      - formula.subscription
      - formula.added
    sort:
      - property: formula.first_code
        direction: DESC
      - property: formula.name
        direction: ASC
```

### 4. Rewritten `~/bob/podcasts.md`

Drop the whole `zorg_*` frontmatter block (nothing in `bob-cli` reads it — verified by
grep across `src/`, `docs/`, `scripts/`). Re-parent from `[[zorg_ref_man]]` (an obsolete
zorg reference-manager note) to `[[mind]]`, which is the `h2_role` area that already
lists `[[quote]]` and `[[lit_notes]]` among its related notes. Give the hub no `type`,
matching `eat.md`.

**The 17 `^z-…` anchors must survive**, because 36 links elsewhere in the vault point at
them (7 alone at `^z-241027-07`). Carry each anchor onto its new modern line rather than
parking the old bullets in a `## Legacy Notes` section — the anchor is just an ID, and
moving it keeps the file free of a duplicated legacy list.

```markdown
---
parent: "[[mind]]"
---

# Podcasts

Notes on the podcasts I listen to. One note per show lives under `podcasts/`; browse
them all in [[podcasts.base]].

Related notes:

- [[lit/podcasts]]
- [[podcast_ref]]

![[podcasts.base]]

## Subscribed

- [[podcasts/techmeme|Techmeme Ride Home]] ^z-240911-1j
- [[podcasts/programming_throwdown|Programming Throwdown]] ^z-240916-0b
- [[podcasts/talk_python|Talk Python to Me]] ^z-241001-0m
- [[podcasts/python_bytes|Python Bytes]] ^z-241217-0k
- [[podcasts/ted_radio_hour|TED Radio Hour]] ^z-241103-0m
- [[podcasts/the_pete_and_sebastian_show|The Pete and Sebastian Show]] ^z-241006-0g
- [[podcasts/twit|This Week in Tech]] ^z-241027-07
- [[podcasts/practical_ai|Practical AI]] ^z-250513-0y
- [[podcasts/ai_daily_brief|The AI Daily Brief]] ^z-250524-0n

## Unsubscribed

- [[podcasts/gtd_podcast|Getting Things Done]] ^z-240911-1f
- [[podcasts/the_changelog|The Changelog]] ^z-241004-0b
- [[podcasts/corepy|Core.py]] ^z-241004-0c
- [[podcasts/productivity_cast|ProductivityCast]] ^z-241024-07
- [[podcasts/java_pub_house|Java Pub House]] ^z-241024-08
- [[podcasts/the_stackoverflow_podcast|The Stack Overflow Podcast]] ^z-241024-09

## Notes

- **Taking a fleeting note from Spotify:** 3 dots → Share → More → `#gkeep`. The share
  includes a Spotify link that jumps to the exact timestamp you shared from.
  ^z-240913-04
- **Comedy podcast recommendations** — @CHAT w/ @LLM, prompt: "What are some of the best
  comedy podcasts I should consider listening to?" ^z-241007-09
  - `%chad`: https://chatgpt.com/share/6703fb1c-338c-8010-84a2-1b9e7118dd85
  - `%chop`: https://chatgpt.com/share/6703fb24-7ab0-8010-9d63-009d9fd14350
  - `%claude`: https://claude.ai/chat/dce45626-8ff5-458d-ad9a-97b64d291ff1
```

Indent nested bullets with a tab, matching `body.md` / `cash.md` / `sase.md`.

### 5. Loose end to leave alone

`zorg_ref_man.md` still lists `[[podcasts]]` in its "Related notes". That link keeps
resolving after re-parenting, so leave it. Optionally add `- [[podcasts]]` to
`mind.md`'s "Related notes" list; that is the only edit outside the podcasts files this
phase may make.

### Phase acceptance

- `podcast.md`, `podcasts.base`, and 15 `podcasts/*.md` notes exist.
- All 17 original `^z-…` anchors are present in the new `podcasts.md`, and only there
  (no duplicates).
- Every URL in `podcasts.md`'s pre-rewrite backup appears in exactly one per-show note.
- `python3 -c "import yaml; yaml.safe_load(open('/home/bryan/bob/podcasts.base'))"`
  succeeds.

---

## Phase `quote`: Rewrite `quote.md` as Obsidian quote callouts

### Link-safety precondition

Re-run this before touching the file. It must print `0`:

```bash
grep -rho '\[\[quote#\^[a-z0-9-]*' ~/bob --include='*.md' | grep -v '^$' | wc -l
```

At planning time no note anywhere in the vault block-references `quote.md`, which is why
the `^z-…` anchors can be replaced with readable slugs. If the count is non-zero, stop
and report instead of renaming those anchors.

### Target shape

```markdown
---
parent: "[[mind]]"
---

# Quotes

Quotes worth keeping. Each one carries a stable block ID, so other notes can pull a
quote in with `![[quote#^<block-id>]]`.

Related notes:

- [[me]]
- [[lit_notes]]

## Quotes

> [!quote] You drown not by falling into the river, but by staying submerged in it. —
> Extraction (2020)

^drown-in-river

> [!quote] Hurting yourself is easy; living is hard. — Loudermilk, S01E10

^living-is-hard
```

Rules:

- One `> [!quote]` callout per quote, quote text on the callout title line, attribution
  as a `> — Source` line.
- Commentary, "see also" links, and source URLs become extra `>` lines inside the same
  callout, separated from the attribution by a `>` spacer line — the shape
  `ref/blogs/harness_for_rsi.md` uses.
- The block ID goes on its own line after a blank line, following the callout (again per
  `harness_for_rsi.md`).
- Preserve chronological order, which is the file's existing order.
- Drop the zorg scaffolding: the `240410#CP` prefixes, the `|` continuation pipes, the
  `* ` sub-bullet marker for what is really commentary, and the self-referential leading
  `[[quote]]` link that appears on many entries.
- Drop the entire `zorg_*` frontmatter block and the `Quotes` bare-text first line (the
  `# Quotes` heading replaces it).
- **Preserve verbatim:** every outbound wikilink and every URL. Specifically
  `[[podcasts#^z-241001-0m|talk_python]]`, `[[podcasts#^z-241027-07|twit]]` (×4),
  `[[podcasts#^z-250524-0n]]`, `[[eat#hop]]`, `[[old_ref#^one_thing|one_thing]]`,
  `[[work_ref#^z-250721-0v|decision_frameworks]]`,
  `[[bobdoto_ref#^z-250319-0f|bobdoto_comparisons]]`,
  `[[nvim_plugins#^z-250418-0v|dashboard_nvim]]`, `[[goog#^z-241024-0l|youtube]]`, the
  readwise URL, the quora URL, and the businessinsider URL. All of these were confirmed
  to resolve today; do not "fix" or shorten any of them.
- Keep `%yeats`, `%feynman`, `%joshuai` exactly as written.
- The `INSPIRED BY:` lines become `> **Inspired by:** …` inside their callout.

### Block ID assignment

All 32 quotes, in file order, with the old anchor they replace:

| #   | Old anchor  | New block ID                    | Quote / attribution                                                                          |
| --- | ----------- | ------------------------------- | -------------------------------------------------------------------------------------------- |
| 1   | z-240410-cp | `drown-in-river`                | "You drown not by falling into the river…" — Extraction (2020)                               |
| 2   | z-240410-cq | `living-is-hard`                | "Hurting yourself is easy; living is hard" — Loudermilk S01E10                               |
| 3   | z-240410-cr | `injustice-anywhere`            | "Injustice anywhere is a threat to justice everywhere" — MLK                                 |
| 4   | z-240410-cs | `politicians-and-diapers`       | "Politicians and diapers must be changed often…" — Mark Twain                                |
| 5   | z-240501-1m | `genius-enthusiasm`             | "Every production of genius…" — fortune cookie, `[[eat#hop]]`                                |
| 6   | z-240603-0x | `books-are-useful`              | Bob Nystrom, Flutter Podcast E41                                                             |
| 7   | z-240811-06 | `rage-and-serenity`             | "True power lies somewhere between rage and serenity" — Charles Xavier, _X-Men: First Class_ |
| 8   | z-240827-0l | `hatred-ended-by-love`          | "Hatred is never ended by hatred but by love" — Buddha                                       |
| 9   | z-240906-0z | `honesty-blunt-instrument`      | Robert Greene, _The 48 Laws of Power_                                                        |
| 10  | z-240910-0r | `trump-lost-2020`               | AP-sourced passage; keep the readwise URL                                                    |
| 11  | z-241001-0n | `two-hard-problems`             | Episode 404 of `[[podcasts#^z-241001-0m\|talk_python]]` at ~21:00                            |
| 12  | z-241027-08 | `love-is-like-a-fart`           | heard on `[[podcasts#^z-241027-07\|twit]]`                                                   |
| 13  | z-241109-0e | `skate-to-the-puck`             | Wayne Gretzky                                                                                |
| 14  | z-241111-0n | `thinking-without-writing`      | Leslie Lamport                                                                               |
| 15  | z-241126-0f | `make-the-iron-hot`             | `%yeats`                                                                                     |
| 16  | z-241128-0r | `quantity-is-quality`           | businessinsider URL                                                                          |
| 17  | z-250112-0d | `questions-without-answers`     | `%feynman`                                                                                   |
| 18  | z-250207-0c | `pobodys-perfect`               | Eleanor, _The Good Place_                                                                    |
| 19  | z-250407-0u | `deck-chairs-on-titanic`        | `[[podcasts#^z-241027-07\|twit]]` + quora explainer                                          |
| 20  | z-250410-0m | `diplomacy-barrel-of-gun`       | `[[podcasts#^z-241027-07\|twit]]`                                                            |
| 21  | z-250423-0j | `satisficing`                   | _The Organized Mind_                                                                         |
| 22  | z-250425-0k | `follow-the-masses`             | `%joshuai` whiteboard                                                                        |
| 23  | z-250505-0a | `clever-vs-wise`                | Albert Einstein; note `fortune.nvim` / `[[nvim_plugins#^z-250418-0v\|dashboard_nvim]]`       |
| 24  | z-250516-0k | `dekoi`                         | `%joshuai` whiteboard                                                                        |
| 25  | z-250517-0l | `powerful-habits`               | `[[old_ref#^one_thing\|one_thing]]`                                                          |
| 26  | z-250802-0k | `indecision-vs-wrong-decision`  | Tony Soprano; inspired by `[[work_ref#^z-250721-0v\|decision_frameworks]]`                   |
| 27  | z-250808-0v | `founders-vs-followers`         | UNKNOWN; inspired by `[[bobdoto_ref#^z-250319-0f\|bobdoto_comparisons]]`                     |
| 28  | z-250910-0l | `starving-man-eats-from-floor`  | `[[podcasts#^z-241027-07\|twit]]`                                                            |
| 29  | z-250910-0m | `risks-in-barbed-wire`          | _Blackish_                                                                                   |
| 30  | z-251116-0k | `exercise-discipline-affection` | a `[[goog#^z-241024-0l\|youtube]]` comment                                                   |
| 31  | z-251121-0p | `small-minds-great-minds`       | Blaise Pascal                                                                                |
| 32  | _(none)_    | `secret-between-three`          | Benjamin Franklin, _Poor Richard's Almanack_, via `[[podcasts#^z-250524-0n]]`                |

### Worked example

Before:

```
- 250505#0a [[quote]] "A clever person solves a problem. A wise person avoids it." - Albert Einstein ^z-250505-0a
  * I saw this on a quote-of-the-day from fortune.nvim (used by [[nvim_plugins#^z-250418-0v|dashboard_nvim]])!
```

After:

```markdown
> [!quote] A clever person solves a problem. A wise person avoids it. — Albert Einstein
>
> Saw this on a quote-of-the-day from `fortune.nvim` (used by
> [[nvim_plugins#^z-250418-0v|dashboard_nvim]])!

^clever-vs-wise
```

### Phase acceptance

- All 32 quotes present, in order, with the block IDs above and no `^z-` anchors left in
  `quote.md`.
- Every wikilink and URL from the backup copy still present, character-for-character.
- No `generated_from_zorg` / `zorg_*` keys remain in the frontmatter.

---

## Phase `verify`: Verify link integrity and vault render

Run after both phases land. Nothing in this phase edits content; it reports, and files a
task bead through `/sase_new_task` for anything it cannot fix trivially.

1. **Inbound podcast anchors still resolve.** Every `[[podcasts#^…]]` target in the
   vault must exist in the new `podcasts.md`:
   ```bash
   cd ~/bob
   grep -rho '\[\[podcasts#\^[a-z0-9-]*' --include='*.md' . | sed 's/.*\^//' | sort -u | while read -r a; do
     grep -q "\^$a\b" podcasts.md || echo "MISSING ANCHOR: $a"
   done
   ```
2. **No content lost.** Diff the backups against the rewrites for dropped URLs and
   wikilinks:
   ```bash
   for f in quote podcasts; do
     diff <(grep -oE 'https?://[^ )"]+' /tmp/bob_modernize_backup/$f.md | sort -u) \
          <(grep -rhoE 'https?://[^ )"]+' ~/bob/$f.md ~/bob/podcasts/ 2>/dev/null | sort -u)
   done
   ```
   Every URL from a backup must still appear somewhere in the rewritten set.
3. **New wikilinks resolve.** Extract every `[[…]]` target from `quote.md`,
   `podcasts.md`, `podcast.md`, and `podcasts/*.md`, and confirm a matching `.md` (or
   `.base`) file exists.
4. **Base parses and renders.**
   `python3 -c "import yaml; yaml.safe_load(open('/home/bryan/bob/podcasts.base'))"`,
   then open the vault (`ob`) and confirm all four `podcasts.base` views populate with
   15 rows total, 9 subscribed and 6 unsubscribed, and that the `![[podcasts.base]]`
   embed in `podcasts.md` renders.
5. **Frontmatter is well-formed.** Parse the frontmatter of every new/changed note;
   confirm `parent`/`type` use the quoted-wikilink style the rest of the vault uses, and
   that `type: "[[podcast]]"` resolves to the new `podcast.md`.
6. **Working tree contains only intended changes.** `git -C ~/bob status --short` should
   show `M quote.md`, `M podcasts.md`, `?? podcast.md`, `?? podcasts.base`,
   `?? podcasts/`, optionally `M mind.md`, plus whatever unrelated modifications were
   already present before the epic started (capture that baseline in phase `podcasts`).
   **Do not commit.**

### Epic acceptance

- `quote.md` and `podcasts.md` contain no zorg frontmatter, no `NNNNNN#XX` prefixes, no
  `ID::`/`LID::` declarations, and no `======` banners.
- Zero broken links introduced, inbound or outbound.
- `podcasts.base` renders 15 shows.
- The vault working tree is uncommitted and otherwise untouched.

---

## Alternatives considered and rejected

| Alternative                                                                                               | Why rejected                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Explode `quote.md` into `quote/<slug>.md` notes + `quotes.base`, mirroring `eat`                          | 32 stub notes whose only attributes are author and date. Quotes are read inline and embedded by block reference; the Base would add navigation friction without a useful filter axis. Worth revisiting if quotes ever grow tagged themes.                 |
| Keep the legacy bullets under `## Legacy Notes` in `podcasts.md` (as `dev.md` / `body.md` / `cash.md` do) | Those notes have legacy content with no modern equivalent. Here the legacy bullets _are_ the podcast list, so keeping them would duplicate the new list. Carrying the `^z-…` anchors onto the new lines preserves every inbound link with no duplication. |
| Rewrite all 36 inbound `[[podcasts#^z-…]]` links to point at `[[podcasts/<slug>]]`                        | 36 edits across ~20 unrelated notes for cosmetic gain, and a much larger blast radius. The anchors keep working as-is. Reasonable follow-up task bead.                                                                                                    |
| Convert `%joshuai` / `%feynman` / `%yeats` to `[[person]]` links                                          | Vault-wide convention change already tracked in `bob.md`'s Future Work. Out of scope.                                                                                                                                                                     |
| Keep the `zorg_*` frontmatter (as `dev.md`, `body.md`, `cash.md`, `gtd.md`, `mac.md` still do)            | Those notes were modernized incrementally and simply never had it cleaned up. Nothing reads these keys — confirmed by grep across `bob-cli`'s `src/`, `docs/`, and `scripts/`. Removing dead provenance is part of the ask.                               |
| Write to the `sase repo open` clone and land via PR                                                       | The clone is pinned to the last commit and the live vault has uncommitted edits; the auto-sync that would make this workable is still an open task (`bob.md#^auto-sync-for-sase`). Flagged above as an approval-time decision.                            |
