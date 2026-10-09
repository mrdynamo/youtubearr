# Investigation: YouTubearr Dispatcharr Dummy EPG Regression

**Branch:** `dev/investigate-dummy-epg-regression` (off `main` @ `ac8a9d5` — v1.40.1)
**Created:** 2026-10-09

## Symptom (reported by user)

- YouTubearr plugin creates Dispatcharr channels for YouTube livestreams and assigns a `source_type="dummy"` EPG source to each.
- Plugin writes the real livestream title into `ProgramData.title` / `ProgramData.description`.
- **Downstream IPTV clients only see generic placeholder text** (e.g., from the Dummy EPG pattern) instead of the actual YouTube stream title.

## What YouTubearr does today

Plugin code paths:

- `plugin.py:1894-1900` — creates the EPG source: `EPGSource.objects.get_or_create(name=epg_source_name, defaults={"source_type": "dummy", "is_active": True})`
- `plugin.py:1907-1919` — creates/updates `EPGData` (one per channel) with `tvg_id=str(channel_number)`, `name=video_title`
- `plugin.py:1921-1924` — assigns `epg_data` to the channel and sets `channel.tvg_id`
- `plugin.py:1929-1938` — writes `ProgramData.update_or_create(title=video_title, description=video_title, start_time=now, end_time=now+12h)`
- `plugin.py:1941-1946` — stamps `custom_properties = {"owner": "youtubearr", ...}` on the program row
- `plugin.py:3281-3337` — `_refresh_epg_times()` runs on every poll cycle, but only updates `start_time`/`end_time`. The title is **NOT** re-fetched.
- `plugin.py:3762-3837` — `_generate_xmltv_cache()` writes `/app/media/cached_epg/{epg_source.id}.tmp` from the ProgramData rows. This is the ONLY surface proven to contain the real title. It only fires via `_trigger_webhook` after channel add/clean events.

Documented intent ([README.md:194](README.md#L194)):

> "The Dummy EPG source acts as a container for YouTubearr's programme data. The plugin creates `ProgramData` entries directly with the livestream title, bypassing the Dummy EPG's pattern-based generation."

## Smoking gun in YouTubearr's own CHANGELOG

[CHANGELOG.md L399-403](CHANGELOG.md#L399-L403) (v1.14.0, 2026-03-15):

> **Bug:** EPG data for YouTube streams showed correctly in Dispatcharr but not in Jellyfin.
> **Root cause:** Dispatcharr reads EPG directly from the database, but Jellyfin reads from XMLTV cache files at `/app/media/cached_epg/{source_id}.tmp`.

That sentence proves: **at v1.14.0, Dispatcharr's Dummy EPG source WAS reading the manually-written `ProgramData` rows and surfacing their `title`/`description` to clients.** That contract was the basis on which YouTubearr's design was built, and it has not changed in the plugin.

## What's stable in YouTubearr (searched with `git log -S`)

- `source_type="dummy"` first introduced in `73b32df` (v1.12.3, 2026-03-14). Never changed since.
- `ProgramData.objects.update_or_create(...)` writes title/description: present since v1.12.4 and only cosmetically edited.
- `_refresh_epg_times` (v1.15.0): only touches time fields, never titles.
- XMLTV cache generator (v1.14.0): writes `/app/media/cached_epg/{source_id}.tmp` correctly.
- `custom_properties={"owner": "youtubearr"}` added in **v1.20.0 (2026-06-05)** with the ominous note "**for future tooling**" — a candidate trigger if Dispatcharr later filters on `custom_properties.owner`.

→ Conclusion: **the regression is in Dispatcharr, not YouTubearr.**

## Hypothesis: what changed in Dispatcharr

Without access to Dispatcharr yet, the most likely candidates (in order):

1. **Dummy EPG renderer now ignores manually-written `ProgramData` rows** and only emits from its configured pattern template. Placeholder text clients see = operator's pattern template output.
2. **Filter on `custom_properties.owner`** — Dispatcharr may have shipped the "future tooling" the v1.20.0 note anticipated. A filter like "skip rows where `owner` is set" would make YouTubearr's tagged rows invisible, while still letting operator-authored rows through.
3. **`ProgramData` model rename/relocation** that the XMLTV serializer no longer reads.
4. **`source_type="dummy"` semantics change** — Dispatcharr's Dummy source no longer queries the program table at all and now serves only the configured pattern.

## How to verify (diagnostic checklist)

1. From the Dispatcharr shell, run:
   ```python
   python manage.py shell -c "
   from apps.epg.models import EPGSource, EPGData, ProgramData
   src = EPGSource.objects.get(name='YouTube Live')
   epg = EPGData.objects.filter(epg_source=src).first()
   print('source.source_type =', src.source_type)
   for p in ProgramData.objects.filter(epg=epg).order_by('-start_time')[:3]:
       print(p.start_time, '|', repr(p.title), '|', repr(p.custom_properties))
   "
   ```
   - If `p.title` shows the real video title and `custom_properties.owner == 'youtubearr'`: candidate #2 confirmed.
   - If the title is placeholder text: YouTubearr polling is broken (unlikely given v1.20.3 heartbeat fix).
2. Hit Dispatcharr's XMLTV endpoint directly (path: `/api/xmltv/<src_id>` or equivalent). If `<title>` is placeholder text in the wire format, the Dummy renderer is the culprit. If the wire XML has real titles, clients are reading something else.
3. Compare Dispatcharr versions: examine release notes / CHANGELOG for changes in `apps/epg/`, dummy/pattern rendering, and any field/filter related to `owner` or `custom_properties`.

## Likely fixes (in YouTubearr, no Dispatcharr upgrade needed)

1. **Strip the `owner: "youtubearr"` tag** from `ProgramData.custom_properties` at `plugin.py:1941-1946`. If Dispatcharr filters plugin-owned rows out, removing the tag makes those rows visible again. Minimal change.
2. **Switch source type from `dummy` to `xmltv`** and point Dispatcharr at the plugin's own XMLTV cache file at `/app/media/cached_epg/{id}.tmp`. Plugin already generates this correctly. Decouples YouTubearr entirely from whatever Dispatcharr's Dummy renderer does.

## Files referenced

- [plugin.py:1894-1900](plugin.py#L1894-L1900) — EPGSource creation
- [plugin.py:1907-1919](plugin.py#L1907-L1919) — EPGData create/update
- [plugin.py:1929-1938](plugin.py#L1929-L1938) — ProgramData write
- [plugin.py:1941-1946](plugin.py#L1941-L1946) — custom_properties ownership tag (added v1.20.0)
- [plugin.py:3281-3337](plugin.py#L3281-L3337) — `_refresh_epg_times`
- [plugin.py:3762-3837](plugin.py#L3762-L3837) — XMLTV cache generator
- [README.md:194](README.md#L194) — design intent
- [CHANGELOG.md:399-403](CHANGELOG.md#L399-L403) — v1.14.0 contract
- [CHANGELOG.md:136](CHANGELOG.md#L136) — v1.20.0 "for future tooling" note
- [CHANGELOG.md:72-74](CHANGELOG.md#L72-L74) — v1.20.3 ghost-heartbeat fix
