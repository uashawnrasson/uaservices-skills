---
name: luna-take-versions
description: Use when placing multiple generated audio clips as auditionable takes on a LUNA track — alternate takes the producer can switch between, multi-segment versions where different clips need to land on the same named version, or any time you need to manage versions/playlists on a track. Fires on phrases like "add as a new take", "add another version", "place this on the same version", "let the producer audition", "alternate take", or any workflow involving place_version, place_generated_clip across multiple takes, or list_versions.
---

# LUNA Take & Version Placement

## Two distinct features — pick the right one

**Comp lanes** (`insert_region` with `layer`) — stacked clips on one playlist for comping. The producer cuts between lanes to build a composite. Not for auditioning whole alternate takes.

**Versions / playlists** (`place_version`) — separate named playlists (V1, Take 2, Take 3...) the producer switches between wholesale via the version dropdown. Use this for generated alternatives.

For generated audio auditioning, always use the version/playlist model.

## The multi-take placement pattern

### Step 1 — Check if V1 is empty in the target region

```python
track = view_track(doc, track_uid)
v1_has_content = any(r for r in track.get('regions', []))
```

### Step 2 — Place accordingly

**If V1 is empty:** place take 1 with `place_generated_clip` (lands on V1), then `place_version` for each subsequent take.

```python
# Take 1 → V1
place_generated_clip(client, gen_1, session_uid,
    track_ref={'kind': 'track', 'uid': track_uid},
    start_ns=0, region_name='Take 1')

# Take 2 → new named version
place_version(client, gen_2, session_uid,
    track_ref={'kind': 'track', 'uid': track_uid},
    start_ns=0, version_name='Take 2', region_name='Take 2')
```

**If V1 already has content:** use `place_version` for all takes. Never overwrite existing work on V1.

## Multi-segment versions

When a version needs multiple clips end-to-end (e.g., 8 bars on A chord, 8 bars on D chord):

1. `place_version` for the first segment — this creates the new playlist
2. `switch_version` to the new version
3. `place_generated_clip` for each subsequent segment
4. **Do not switch back to V1** — the producer owns version state

```python
# Create version with segment 1
r = place_version(client, gen_A, session_uid,
    track_ref={'kind': 'track', 'uid': track_uid},
    start_ns=0, version_name='Take 3', region_name='seg 1')
v = r['data']['version']

# Switch to the new version, add segment 2
switch_version(client, session_uid, track_uid, v)
place_generated_clip(client, gen_D, session_uid,
    track_ref={'kind': 'track', 'uid': track_uid},
    start_ns=16_000 * 1_000_000, region_name='seg 2')

# Do NOT switch back — producer decides what's active
```

## Hard rules

- **Never switch back to V1 after placing takes.** The producer owns what's active. Switching back without being asked is the version equivalent of touching mute.
- **Never use `place_version` twice for the same multi-segment take.** It creates a new playlist on every call. Use the switch_version → place_generated_clip pattern for segments 2+.
- **Check clip_count from `list_versions`** to verify each version has the expected number of segments before reporting done.

## Checking version state

```python
versions = list_versions(client, session_uid, track_uid)['data']['versions']
for v in versions:
    print(f"{v['name']}: {v['clip_count']} clips")
```

## Cleaning up stray versions

```python
for v in versions:
    if v['name'] == 'stray version':
        delete_version(client, session_uid, track_uid, v['version'])
# Note: cannot delete version 1 (V1 / primary playlist)
```
