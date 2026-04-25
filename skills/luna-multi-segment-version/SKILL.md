---
name: luna-multi-segment-version
description: Use when placing multiple generated clips as segments of the same named version on a LUNA track — e.g., 8 bars on A chord followed by 8 bars on D chord, both belonging to "Take 2" so the producer can audition the full sequence as one version. Do NOT use for single-clip versions (place_version handles that alone) or for placing clips on separate versions. Trigger phrases: "add both segments to the same take", "place this on the same version", "the second clip should be part of Take 2", "build out that version with the next section".
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
