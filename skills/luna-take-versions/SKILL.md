---
name: luna-take-versions
description: Use when placing multiple generated audio clips as auditionable takes on a LUNA track — alternate takes the producer can switch between, multi-segment versions where different clips need to land on the same named version, or any time you need to manage versions/playlists on a track. Fires on phrases like "add as a new take", "add another version", "place this on the same version", "let the producer audition", "alternate take", or any workflow involving place_version, place_generated_clip across multiple takes, or list_versions.
---

# LUNA Take & Version Placement

## Multi-segment versions

When a version needs multiple clips end-to-end, `place_version` alone won't work — it creates a new playlist on every call. Use this pattern instead:

```python
# place_version creates the playlist and places segment 1
r = place_version(client, gen_A, session_uid,
    track_ref={'kind': 'track', 'uid': track_uid},
    start_ns=0, version_name='Take 2', region_name='seg 1')
v = r['data']['version']

# Switch to it, place remaining segments with place_generated_clip
switch_version(client, session_uid, track_uid, v)
place_generated_clip(client, gen_D, session_uid,
    track_ref={'kind': 'track', 'uid': track_uid},
    start_ns=16_000 * 1_000_000, region_name='seg 2')

# Do NOT switch back to V1 — the producer owns what's active
```

## Version state is in the export

No need for a separate `list_versions` call when you already have `doc`:

```python
track = view_track(doc, track_uid)
versions = track.get('versions', [])
# [{'version': 1, 'name': 'V1', 'is_active': True, 'regions': [...]}, ...]
```

`list_versions` is useful for a lightweight check between operations when you don't need a full export. The export `versions` field includes full region data per version; `list_versions` returns clip_count only.
