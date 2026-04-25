---
name: luna-session-state
description: Use at the start of any LUNA session operation when deciding how to read session state — before generating, placing, mixing, routing, or inspecting plugins. Also use when you're unsure whether to call export_session again or reuse an existing doc. Fires on any workflow that begins with "what's in the session" or requires knowing track layout, region positions, plugin chains, or version state before acting.
---

# Reading LUNA Session State Efficiently

## Never print the raw doc

The raw `doc` from `export_session` is ~35K tokens. Printing it wholesale fills context and buries the signal. Always use a view.

## View sizes (approximate)

| View | Chars | Use it for |
|---|---|---|
| `view_transport` | ~150 | BPM, time sig, sample rate — cheapest orientation call |
| `view_hud` | ~1,500 | First look at a session — tracks, buses, revision, transport in one compact block |
| `view_timeline` | ~1,350 | Placement decisions — where regions are, bar ranges, what's empty |
| `view_mix` | ~1,400 | Gain, pan, mute/solo state per track |
| `view_plugins` | ~2,200 | Plugin chains across all tracks — expensive, only when you need FX state |
| `view_track(doc, name)` | varies | Full slice for one track: regions + plugins + sends + versions |
| `context_for_placement` | ~2,000 | Composite before generate+place — replaces view_timeline + view_track combined |
| `context_for_mix` | ~2,400 | Composite before gain/pan/send changes |

## Which view before which operation

**Starting a session / first orientation:**
```python
print(view_hud(doc))  # revision, track list, bus list
```

**About to generate and place audio:**
```python
print(context_for_placement(doc, track_name_or_uid))
# Gives transport + timeline + track detail in one call
```

**About to change mix settings:**
```python
print(context_for_mix(doc, track_name_or_uid))
```

**Checking BPM before generation:**
```python
bpm = view_transport(doc)['tempo_bpm']  # no print needed, just read it
```

**Checking where regions are:**
```python
print(view_timeline(doc))
```

**Checking plugin chains:**
```python
print(view_plugins(doc))  # all tracks
# or
print(view_track(doc, 'Bass'))  # one track, full slice
```

## What views don't surface — go to raw doc

Two fields only exist on the raw track object, not in any view:

```python
doc = export_session(client, session_uid)
track = next(t for t in doc['tracks'] if t['name'] == 'Bass')

track['versions']             # full version/playlist list with regions per version
track['active_playlist_uid']  # which version is currently active
```

Use raw doc access when you need version state and you're already working with `doc`. Don't call `list_versions` unnecessarily — the data is already there.

## When to re-export vs reuse doc

Re-export after every `apply_patch` — the revision increments and any cached `doc` is stale. Within a single code block where no patch ops have run, reuse the same `doc`.

```python
doc = export_session(client, session_uid)
rev = doc['revision']

# ... decide what to do using views ...

result = apply_patch(client, session_uid, {'base_revision': rev, 'ops': [...]})

# doc is now stale — re-export before reading state again
doc = export_session(client, session_uid)
```
