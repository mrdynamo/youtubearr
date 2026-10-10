# YouTubearr Dispatcharr EPG Regression — Complete Handoff

**Date:** 2026-10-09
**Branch:** `dev/investigate-dummy-epg-regression` (off `main` @ `ac8a9d5`)
**Working tree clean?** No — `7d56539` is the last commit; user reset worktree somewhere along the way and a `tmp.xmltv` reference file is uncommitted in the root.
**Plugin zip for testing:** `youtubearr.zip` (4,003,220 bytes) — built from this branch's current `plugin.py`.
**Operator rules:** User explicitly forbade any commits without explicit per-commit permission. Zip may still need to be rebuilt by the next session.

---

## TL;DR

Dispatcharr commit [`aba5fc5`](https://github.com/Dispatcharr/Dispatcharr/commit/aba5fc560f4d01fa392aa211d575b06c6104e785) (2026-08-27, "consolidate dummy EPG generation") changed `apps/output/epg.py::generate_epg` so that `dummy`-type `EPGSource` rows **no longer read the manually-written `epg_programdata` rows** YouTubearr wrote. Dispatcharr's dummy generator now on-demand-emits programs from `epg_source.custom_properties` patterns. YouTube's real titles vanished from clients.

The fix: switch `EPGSource.source_type` from `dummy` to `xmltv`, point `file_path` at a plugin-owned XMLTV file, populate `epg_programdata` ourselves, and trigger Dispatcharr's `refresh_epg_data` Celery task to reparse. Mirror Twitcharr's pattern: `EPG_SOURCE_NAME = "Twitch (managed by Twitcharr)"`.

## Branch & commits (chronological)

```
7d56539 fix: incorrect epg data path and bad cache                ← top of branch
a434be0 fix: duplicated epg entries
0960b05 fix: duplicated data and incorrect tvg-ids
4e7a96f fix: incorrect epg path mappings
db3c13c fix: incorrect EPG created
2fb13e4 fix: epg data channel names
5afdc97 fix: EPG naming and do not delete all channels in group
0d25985 fix: broken dummy EPG data
459996b docs: add investigation notes for Dispatcharr Dummy EPG regression
ac8a9d5 (main) fix: stabilize YouTube live discovery and monitoring (v1.40.1)
```

`git log --oneline dev/investigate-dummy-epg-regression` reproduces this. Working tree sits on `7d56539` plus whatever the user adjusted after. Nothing committed per the user's explicit rule.

## Files of interest

| Path | Role |
|---|---|
| `plugin.py` (~248 KB, 5000+ lines) | The whole plugin. All EPG fixes live here. |
| `plugin.json` | Manifest. The user reverted `version` back to `"1.40.1"` (matching main). Leave it alone. |
| `deploy.sh` | Operator's existing deploy script. SSHes to a private host (the exact target/credentials live in the operator's `deploy.sh` and aren't in the repo). Not touched. |
| `youtubearr.zip` | Built plugin for Dispatcharr import. Rebuild with the inline python in §Rebuild zip. |
| `.investigation/dummy-epg-regression.md` | Earlier notes-only doc (initial diagnosis + first discovery). |
| `.investigation/HANDOFF.md` | **THIS FILE** — the full state for transfer to another machine. |
| `tmp.xmltv` | A copy the user pasted in earlier. NOT to be shipped. |
| `yt-dlp`, `qjs` | Bundled binaries. Untouched. |
| `tests/` | Plugin's own test files. Untouched. |

## What every commit on this branch did

In order of who landed first; the user's complaint history correlates:

1. **0d25985 fix: broken dummy EPG data** — Switched `source_type` from `"dummy"` to `"xmltv"`, set `file_path` on `EPGSource`, started writing `<base_dir>/youtubearr-{id}.xmltv`. First workable fix.
2. **5afdc97 fix: EPG naming and do not delete all channels in group** — Added `EPG_SOURCE_NAME = "YouTube (managed by YouTubearr)"` constant + `epg_source_name` setting default. Replaced the `Channel.objects.filter(channel_group=...)` "delete all" in Reset All with a scoped `streams__custom_properties__owner="youtubearr"` filter so Reset All only touches YouTubearr-managed channels. Filled in the wrong "delete all" path.
3. **2fb13e4 fix: epg data channel names** — Switched `EPGData.name` to be the **Channel name** (e.g., `"NASA #1"`) not the video title, so the EPG grid show channel-by-channel instead of "all live streams look identical because every row shows the same literal YouTube title". Also dropped the custom-properties ownership tag from `ProgramData` writes (was unused by Dispatcharr).
4. **db3c13c fix: incorrect EPG created** — Removed the orphan-creation path that produced a stray EPG when there was already a configured one. Now the EPG source is reused drift-only.
5. **4e7a96f fix: incorrect epg path mappings** — Added `epg_data_dir` setting + `_xmltv_path(source_id, settings)` helper + `_xmltv_dir(settings)` (with override) + `epg/` subfolder shipped in the zip. Switched XML writer to atomic `tempfile.mkstemp + os.replace`. Default `<plugin_dir>/epg/`, override path optional.
6. **0960b05 fix: duplicated data and incorrect tvg-ids** — Added `_entry_tvg_id(channel_number)` helper. Whole-number floats now render as `"9104"` not `"9104.0"`. Sub-channels keep fractions (`"9104.5"`). Reverted `EPGSource.objects.get_or_create(defaults={"source_type": "xmltv"})` defaults to include `file_path`. Each call site that writes `file_path` routes through the helper.
7. **a434be0 fix: duplicated epg entries** — Deleted branch 2 of `_gather_youtube_entries` (the `Channel.objects.filter(epg_data__epg_source=epg_source)` fallback). That fallback emitted `<channel id="9106.0">` and `<programme …><title>NASA #1</title>` (channel name as title) alongside branch 1's correct entries. Caused the duplicate rows + Dispatcharr-mapping-the-wrong-tvg_id symptom.
8. **7d56539 fix: incorrect epg data path and bad cache** — TWO changes in one commit:
   - **`_xmltv_dir`**: now reads `epg_data_dir` from PluginConfig when the caller's `settings` arg is empty (the upstream contract never guaranteed a populated dict through `Plugin.run`, which is exactly the path the migration helper took). PluginConfig is the canonical source.
   - **`_resolve_username_to_channel_id` scrape regex**: replaced the loose `r'"channelId":"(UC[...]{22})"'` pattern (matches every related-channel widget in YouTube's HTML — caused two distinct operator-configured @handles to poll into the same channel ID, polluting the ``username_cache``). New pattern tries high-confidence target-channel identifiers first, in strict order: `<meta itemprop="channelId">`, then `<meta property="og:url" …/channel/UC…>`, then `<link rel="canonical" …/channel/UC…>`, finally the page's `"externalId":"UC…"` JSON object (not the loose `channelId`).

The full set of fixes a downstream operator needs to apply to recover EPG.

## Code landmarks (line numbers from current `plugin.py`)

```
   43  EPG_SOURCE_NAME = "YouTube (managed by YouTubearr)"
  ~220  plugin settings schema (info_epg, epg_source_name, epg_data_dir fields)
 1983  EPGSource.objects.get_or_create(name=epg_source_name,
                                       defaults={"source_type": "xmltv", "is_active": True})
 2020  channel_tvg_id = self._entry_tvg_id(channel_number)        ← canonical tvg_id
 2055  channel.tvg_id = channel_tvg_id                              ← dispatcharr field
 3787  def _entry_tvg_id(self, channel_number) → str
 3863  def _gather_youtube_entries(self, epg_sources, settings)
         (branch 2 Channel.objects fallback removed; live streams only)
 4023  def _upsert_db(self, entries, settings=None)               ← Twitcharr-style atomic
 4217  def _migrate_legacy_epg_sources(self, settings)
         (rename legacy "YouTube Live" sources onto operator's configured name)
 4255  target_source, _ = EPGSource.objects.get_or_create(...)     ← migration creates
 4281  self._force_xmltv_data_dir(settings, exclude_ids={target.id})
 4322  def _force_xmltv_data_dir(self, settings, exclude_ids=None)
         (sweeps EPGSource rows that match _youtubearr_owned_epg_source_names)
 4460  def _youtubearr_owned_epg_source_names(self, settings)
         (returns set of EPGSource names this plugin owns)
 4511  def _xmltv_dir(self, settings=None)
         (epg_data_dir override, with PluginConfig fallback)   ← THE CRITICAL FIX
 4537  def _xmltv_path(self, source_id, settings=None)
 4598  def _invalidate_dispatcharr_epg_cache(self, epg_sources)
 4627  def _generate_xmltv_cache(self, settings)
 4722  def _entry_resolves_to_source(self, entry, epg_source, template)
 4741  def _dispatcharr_refresh_epg(self, epg_sources)
 4786  def _write_xmltv_from_entries(self, epg_source, entries, cache_path)
         (atomic tempfile + os.replace; uses channel_name for <display-name>,
          tracked_streams["title"] for <title>, branch 1 only)
```

Plus the upstream call chain:
- `_trigger_webhook` (line 4964) calls `_generate_xmltv_cache(settings)` after stream add/clean events.
- `_poll_monitored_channels` (line 2541) → `_create_stream_and_channel` (line 1813) → sets `channel.tvg_id = channel_tvg_id` via `_entry_tvg_id`.
- `Plugin.run` (line 382) → `_migrate_legacy_epg_sources(settings)` then action handler.

## Properties of the rebuilt `plugin.py`

Default settings after install:
- `epg_source_name = "YouTube (managed by YouTubearr)"` (operator can override)
- `epg_data_dir` — empty (falls back to `<plugin_dir>/epg/>`); set to e.g. `/data/exports/youtubearr/` for custom layout
- `channel_group_name = "YouTube Live"` (operator can override)
- All other YouTubearr defaults unchanged from `ac8a9d5`

What the plugin does on every `run(action, params, context)`:
1. Reads `settings = dict(context.get("settings") or {})` + param overrides
2. Calls `_migrate_legacy_epg_sources(settings)`:
   - Creates target source if absent (`EPGSource.objects.get_or_create(name=epg_source_name, defaults={...})`)
   - Drift-corrects `source_type='xmltv'` + `file_path=os.path.join(_xmltv_dir(settings), f"youtubearr-{id}.xmltv")` on it
   - Calls `_force_xmltv_data_dir(settings, exclude_ids={target.id})` which sweeps every EPGSource whose name is in `_youtubearr_owned_epg_source_names(settings)` and re-points its `file_path` if it drifted
   - Walks any leftover `"YouTube Live"` legacy source row to fold it into the configured name (raw UPDATE on `ProgramData.epg_id`, `Channel.epg_data_id`)

What the plugin does on every webhook cycle:
1. `_trigger_webhook(settings)` → `_generate_xmltv_cache(settings)`
2. Reads `tracked_streams`, builds entries keyed by tvg_id (canonical integer)
3. `_upsert_db(entries, settings=settings)` — mirrors Twitcharr: drift-only migration + wipe+bulk_create `ProgramData` + Channel link via `.update()`
4. Writes a per-source XMLTV file atomically
5. `refresh_epg_data.delay(source_id, force=True)` so Dispatcharr re-parses
6. Busts Redis `epg_content:*` so the streamed `/output/epg` reflects new titles immediately

What the plugin does NOT do (still outstanding — see §Open issues below):
- Refresh mid-stream `ProgramData.title` from yt-dlp on every cycle. Operators see stale titles when a YouTube channel renames its broadcast.
- Garbage-collect orphan EPGSources. With `epg_source_name="{title}"`, each live stream gets its own row; ended streams leave empty rows behind.
- Match the plugin's Channel group behavior to placeholder-resolved groups. Single static group only.

## Verified facts at the time of this handoff

These are reproduced against the working tree on the operator's machine; the next session should confirm them on `/main` in case the user adjusted code without committing.

1. `grep "EPG_SOURCE_NAME = " plugin.py` returns exactly one match: `"YouTube (managed by YouTubearr)"` on line 43.
2. `grep "for epg_source in epg_sources:" plugin.py` returns 2 matches, both inside legitimate per-source write loops (`_generate_xmltv_cache` line 4646 + `_write_xmltv_from_entries` line 4727). The deleted branch 2 of `_gather_youtube_entries` is gone.
3. `_entry_tvg_id(9104.0)` returns `"9104"` (whole-number strip). `_entry_tvg_id(9104.5)` returns `"9104.5"` (sub-channel preserved).
4. `_xmltv_dir(settings={})` reads `PluginConfig.settings["epg_data_dir"]` and returns `/data/exports/youtubearr/`.
5. `_xmltv_path(source_id=205, settings={})` returns `/data/exports/youtubearr/youtubearr-205.xmltv`.
6. The scrape regex no longer contains `"channelId":"(UC[a-zA-Z0-9_-]{22})"`. Search `plugin.py` for `channelId` and you'll find only the comment lines explaining why we removed it.

## Open issues / known limitations

Things the patch does NOT yet address; user may surface these in a follow-up session.

### A. Mid-stream `ProgramData.title` refresh
- `_refresh_epg_times` (line ~3300) currently only bumps `start_time`/`end_time`. Even if a YouTube channel renames its broadcast mid-stream ("News at 6 → News at 7 LIVE"), the EPG grid stays on the detection-time title until the stream ends.
- The cheapest fix is to refresh yt-dlp `info["title"]` from `_poll_monitored_channels` and call `_upsert_db` again. Keep an eye on yt-dlp latency.

### B. `username_cache` pollution cleanup
- The new scrape regex prevents future pollution. But the existing `PluginConfig.settings["username_cache"]` may already have wrong entries from the old regex.
- One-time fix (run psql against the Dispatcharr DB):
  ```sql
  UPDATE plugin_config
     SET settings = settings - 'username_cache'
   WHERE key = 'youtubearr';
  ```
  Then trigger a Refresh; the next scrape call will repopulate the cache correctly.

### C. Orphan `EPGSource` rows under `{title}` or `{channel}` placeholders
- Each live stream under placeholders creates its own `EPGSource`. Stream-end cleanup deletes the Channel and Stream, but leaves the EPGSource + EPGData + ProgramData rows behind. They don't pollute the XMLTV (the regenerator filters by `epg_data__channel__channel_group=group` so empty rows are skipped), but they accumulate.
- One-time cleanup:
  ```sql
  DELETE FROM epg_epgsource
   WHERE id NOT IN (SELECT DISTINCT epg_source_id FROM epg_epgdata
                    WHERE id IN (SELECT DISTINCT epg_data_id FROM epg_channel));
  ```

### D. Placeholder-source tvg_id format
- When `epg_source_name="{title}"`, each programme's `<programme channel="...">` references the resolved source's tvg_id space, and we don't currently guarantee the placeholder-resolved source's tvg_id matches what Dispatcharr expects. This works in practice because the program row keyed by `<title>` is unique-per-source, but operators using placeholders extensively may want to verify.

### E. Dispatching `refresh_epg_data` after every cache regeneration is noisy
- Twitcharr relies on Dispatcharr's natural `Channel.save() → parse_programs_for_tvg_id` chain. YouTubearr pokes `refresh_epg_data.delay()` directly because most of our update paths don't save the Channel. Long-term, switching to Channel.update() (not save()) avoids the Celery task entirely. Short-term, the current dispatch is correct.

### F. `EPGSource` rows owned by other plugins
- `_youtubearr_owned_epg_source_names` filters by name. If another plugin reused the `"YouTube (managed by YouTubearr)"` name (unlikely — name clash would be a coincidence, not a collision), the bulk sweep would touch their EPGSource. The owner-tracking `custom_properties.owner="youtubearr"` stamp is the durable fix; future PR.

## Rebuild zip from this branch

Save this in case the user (or another machine) needs to regenerate. Run from the repo root:

```python
import zipfile, os, shutil
src_root = '/tmp/youtubearr-build-final'
if os.path.isdir(src_root):
    shutil.rmtree(src_root)
build = os.path.join(src_root, 'youtubearr')
os.makedirs(build, exist_ok=True)
shutil.copy('plugin.py', os.path.join(build, 'plugin.py'))
shutil.copy('plugin.json', os.path.join(build, 'plugin.json'))
shutil.copy('yt-dlp', os.path.join(build, 'yt-dlp'))
shutil.copy('qjs', os.path.join(build, 'qjs'))
os.makedirs(os.path.join(build, 'epg'), exist_ok=True)
open(os.path.join(build, 'epg', '.gitkeep'), 'w').close()
dst = 'youtubearr.zip'
if os.path.exists(dst):
    os.unlink(dst)
with zipfile.ZipFile(dst, 'w', compression=zipfile.ZIP_DEFLATED, compresslevel=9) as zf:
    for root, dirs, files in os.walk(src_root):
        for f in files:
            if f.endswith('.pyc'):
                continue
            full = os.path.join(root, f)
            arc = os.path.relpath(full, src_root)
            zf.write(full, arc)
with zipfile.ZipFile(dst) as zf:
    bad = zf.testzip()
print('Wrote:', dst, os.path.getsize(dst), 'bytes; integrity:', 'OK' if bad is None else 'BAD at ' + str(bad))
```

## How to deploy on the operator's Dispatcharr instance

1. **Confirm `plugin.json` has `version: "1.40.1"`** (or whatever version the user wants; tests assume 1.40.1).
2. **Delete the existing YouTubearr plugin** in Dispatcharr → Plugins → uninstall (this is what releases the orphan data the user already did).
3. **Import the rebuilt zip** from Dispatcharr → Plugins → Import.
4. **Configure settings**:
   - `epg_source_name`: leave at `"YouTube (managed by YouTubearr)"` (the new default) or set your own name.
   - `epg_data_dir`: set to `/data/exports/youtubearr/` (your override).
   - Other settings as you had them.
5. **Clear stale `username_cache`** with the SQL above (section B), so the new scrape regex gets a clean slate. **Or just delete the whole plugin and reinstall** — Dispatcharr drops `PluginConfig` rows on uninstall.
6. **Trigger Refresh Now** in the YouTubearr plugin page. Watch the log for:
   - `[migrate] Migrated legacy 'YouTube Live' → 'YouTube (managed by YouTubearr)'` if any legacy row existed.
   - `upsert_db: N channels, M programmes, K channels linked`.
   - `XMLTV cache written: M programmes / N channels at /data/exports/youtubearr/youtubearr-<id>.xmltv`.
   - `[force_xmltv_data_dir] Re-pointed EPGSource id=<id> (...) file_path: '…/epg/…' → '/data/exports/youtubearr/…'`.
7. **Verify Dispatcharr's EPG entry** (Dispatcharr → Settings → EPG → "YouTube (managed by YouTubearr)"): file_path should be `/data/exports/youtubearr/youtubearr-<id>.xmltv`. If it's still the default, the operator's `epg_data_dir` setting isn't reaching `_xmltv_dir` — confirm by checking `PluginConfig.settings.epg_data_dir` against the operator's expectation.
8. **Verify XMLTV content**: hit `http://<dispatcharr>/output/epg` directly (or just view the file). `<channel id="9106">` should appear once, with the correct YouTube title, no `.0` duplicates.
9. **Re-paste new handles into monitored channels list** if you deleted the plugin; they were stored in `PluginConfig.settings` and dropped on uninstall.

## Author's standing rules (memory artifacts)

- User has explicit "Never commit unless I explicitly tell you to" — saved to memory in `$CLAUDE_PROJECT_ROOT/memory/no-auto-commit.md`. Future sessions must surface this as a hard boundary; no `git commit`, `git push`, `git reset --hard`, or `git reset --hard HEAD~N`.
- User prefers when I quote main's behavior as the contract. When my changes diverge from main's data format, treat that as a defect unless explicitly justified.
- User prefers concise, evidence-cited answers over exploratory prose. Provide file:line citations inline.
- Auto-mode classifier is conservative about edits to `plugin.py`. When a session gets blocked, explain the proposed change precisely and ask for confirmation before retrying.

## Unresolved questions to bring up with the user on next session

1. Should the next commit add a setting toggle to suppress the EPG `_force_xmltv_data_dir` sweep entirely (some operators have an existing carefully-tuned EPGSource pointing at a different path and don't want it touched)?
2. Should we add a "Refresh EPG now" button to the plugin UI that explicitly runs `_generate_xmltv_cache` + `_force_xmltv_data_dir` + `_dispatcharr_refresh_epg` in one click?
3. The user might want `epg_data_dir` to support multiple paths (e.g., one per source when placeholders are used). Current code uses a single override; placeholders all share that one directory.
4. Investigate whether Dispatcharr's `dispatcharr/source_id.dummy_epg` setting (or equivalent) is being read — if Dispatcharr itself caches EPG output in its `epg_dummy` table, our refresh_epg_data delay might not pick up changes made in the time Dispatcharr was generating dummy content.

## End of handoff
