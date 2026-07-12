---
create_time: 2026-07-08 21:53:11
status: done
prompt: 202607/prompts/plugin_topic_double_dash.md
tier: tale
---
# Plan: Switch the Plugin Catalog Topic to `sase--plugin`

## Problem & Product Context

SASE currently discovers plugins by querying GitHub repositories with the canonical repository topic `sase-plugin`:

```text
gh api --paginate -X GET "search/repositories?q=topic:sase-plugin&per_page=100"
```

The desired canonical marker is now `sase--plugin`. The implementation should make SASE look for that new topic
everywhere the plugin catalog is discovered or explained, without introducing a compatibility mode that keeps accepting
the old topic. A repository should appear in the catalog only when GitHub returns it for `topic:sase--plugin`.

The key implementation detail is that this is not just a string in the `gh` command. The catalog cache stores the query
used to populate it, but the loader currently treats any readable cache as valid. If we only change the search string,
users with an existing `topic:sase-plugin` cache would keep seeing old cached plugin results until they manually
refresh. That would violate the new canonical marker.

## Current Code Surface

- `src/sase/plugins/github_source.py`
  - Owns `GH_SEARCH_QUERY = "topic:sase-plugin"` and the `gh api` endpoint.
  - Its module docstring explains the canonical topic.
- `src/sase/plugins/catalog.py`
  - Writes `GH_SEARCH_QUERY` into fresh cache envelopes.
  - Defaults `PluginCatalog.query` to `GH_SEARCH_QUERY`.
  - Reads cached entries without checking whether `cached.query` still matches the active query.
- `src/sase/plugins/cache.py`
  - Persists the cache envelope and already exposes `CachedCatalog.query`.
  - Does not need a schema-shape change.
- `src/sase/main/parser_plugin.py`, `docs/plugins.md`, and `INSTALL.md`
  - User-facing text names `sase-plugin` as the canonical registry topic.
- Tests contain both real behavior assertions and sample fixture topics. Most topic fixtures should move to
  `sase--plugin`; unrelated strings such as fake package names like `sase-plugin-0` should remain unchanged.

## Proposed Design

### 1. Centralize the Topic and Query

In `src/sase/plugins/github_source.py`, introduce an explicit topic constant:

```python
SASE_PLUGIN_TOPIC = "sase--plugin"
GH_SEARCH_QUERY = f"topic:{SASE_PLUGIN_TOPIC}"
```

Keep `_GH_SEARCH_ENDPOINT` derived from `GH_SEARCH_QUERY`. This preserves the existing public `GH_SEARCH_QUERY` export
for callers/tests while making the topic itself obvious and harder to duplicate incorrectly.

Update the docstring to say the canonical GitHub repository topic is `sase--plugin`.

### 2. Reject Caches Built from the Old Query

Use the existing `CachedCatalog.query` field to make cache use query-aware.

Recommended loader behavior:

- A cache whose `query == GH_SEARCH_QUERY` is compatible and works exactly as today.
- A cache whose query is missing or differs from `GH_SEARCH_QUERY` is incompatible.
- Online non-refresh loads treat an incompatible cache like a cache miss: fetch `topic:sase--plugin`, write a fresh
  cache with the new query, and render the new result.
- Refresh loads ignore incompatible caches as fallback candidates.
- Offline loads with only an incompatible cache fail with an actionable message telling the user to rerun
  `sase plugin list --refresh` while online.
- If GitHub refresh fails, fall back only to a compatible cache. Do not show old `topic:sase-plugin` cache data under
  the new registry rule.

This avoids a schema bump because the cache envelope shape is unchanged. The query field already exists for this exact
kind of compatibility boundary.

### 3. Update User-Facing Text

Update CLI help and docs so the visible contract consistently names `sase--plugin`:

- `src/sase/main/parser_plugin.py`
- `docs/plugins.md`
- `INSTALL.md`

Keep wording precise: call it a GitHub repository topic, since the code uses the GitHub `topic:` search qualifier.

### 4. Update Tests and Fixtures by Meaning

Update behavior tests to assert the new topic/query:

- `tests/test_plugin_catalog_github_source.py`
  - Assert the `gh api` command uses `search/repositories?q=topic:sase--plugin&per_page=100`.
  - Update sample returned topics to include `sase--plugin`.
- `tests/test_plugin_catalog.py`
  - Update cached query expectations.
  - Add coverage for query-aware cache behavior:
    - matching cache avoids network;
    - old-query cache triggers a fetch on normal online load;
    - old-query cache is not used as fallback after a refresh failure;
    - offline old-query cache fails with a clear refresh-online message.
- `tests/test_plugin_catalog_cache.py`
  - Update round-trip query expectations to `topic:sase--plugin`.
- `tests/test_plugin_cli_list.py`
  - Update `list --json` query expectation.

Then update fixture topic tuples in plugin CLI/Admin Center/renderable tests where the topic is part of plugin catalog
metadata. Do this semantically, not with a blind replacement, so unrelated package-name examples are not changed.

If visual snapshot tests fail only because a displayed topic string changed from `sase-plugin` to `sase--plugin`,
inspect the generated actual/diff artifacts and update the goldens as an intentional visual text change.

### 5. Verification

Run focused tests first:

```bash
just install
just test tests/test_plugin_catalog_github_source.py tests/test_plugin_catalog.py tests/test_plugin_catalog_cache.py tests/test_plugin_cli_list.py tests/test_plugin_cli_show.py
```

Then run the repo-required full check because implementation files will change:

```bash
just check
```

Manual smoke check after tests:

```bash
.venv/bin/sase plugin list --refresh --json
```

Confirm the JSON payload reports:

```json
"query": "topic:sase--plugin"
```

## Rollout / Operational Note

This code change assumes plugin repositories have been moved to the new GitHub topic before users rely on the refreshed
catalog. Repositories that still only have `sase-plugin` will disappear from the catalog by design.

## Risks

- Existing users with only an old cache will need one online refresh before offline plugin browsing works again. This is
  intentional because the old cache was populated from the previous registry rule.
- If official plugin repositories have not yet been tagged with `sase--plugin`, a refresh will return an empty or
  incomplete catalog. The fix should not query both topics unless product direction changes.
- Visual snapshots may need small golden updates if topic strings are visible in Admin Center detail panels.
