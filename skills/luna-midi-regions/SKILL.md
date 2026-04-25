---
name: luna-midi-regions
description: Use when placing MIDI notes or patterns into a LUNA session — chord progressions, bass lines, drum patterns, or any pitched content expressed as note data rather than audio. Fires on phrases like "add a MIDI region", "write a chord progression", "place some notes", "create a MIDI bass line", "program a pattern", or any time the output is note data rather than generated audio.
---

# MIDI Regions

> **Status: write path not yet implemented.** `insert_midi_region` and `set_midi_content` pass schema validation but raise `midi_table_not_decodable` at execution time — the `ua:luna:midi_table` binary wire format has not been reverse-engineered. The op and tick math below are correct for when the write path lands. Do not attempt MIDI placement until this is resolved.

## Tick system

All note positions use **480 PPQ** (pulses per quarter note). At 4/4 time:

| Duration | Ticks |
|---|---|
| 1 bar | 1920 |
| 1 beat (quarter note) | 480 |
| 8th note | 240 |
| 16th note | 120 |
| 32nd note | 60 |

Bar/beat to tick offset within a clip:
```python
def to_ticks(bar, beat, subdivision=0):
    # bar and beat are 0-indexed within the clip
    return (bar * 4 + beat) * 480 + subdivision
```

## MIDI note numbers

Middle C = 60. Each semitone = 1.

```python
# Common notes
A3  = 57;  A4  = 69;  A5  = 81
D4  = 62;  E4  = 64
C4  = 60;  C5  = 72
```

## Placing a MIDI region

**Requires an instrument track** (`track_type='instrument'`). Audio tracks reject the op. Create the instrument track via `new_track` with a `preset_uid` — the server derives `track_type='instrument'` from the preset.

`channel` is marked optional in the schema (default 0) but enforced as required at runtime. Always include `'channel': 0` on every note.

```python
doc = export_session(client, session_uid)
rev = doc['revision']

result = apply_patch(client, session_uid, {
    'base_revision': rev,
    'ops': [{
        'op': 'insert_midi_region',
        'track_ref': {'kind': 'track', 'uid': instrument_track_uid},
        'region': {
            'name': 'Chord Progression',
            'start': {'musical': {'bars': 1, 'beats': 1}, 'clock': {'ms': 0}},
            'end':   {'musical': {'bars': 5, 'beats': 1}, 'clock': {'ms': 8000}},
            'notes': [
                # A major chord, bar 1, whole note
                {'pitch': 69, 'velocity': 80, 'start_tick': 0,    'duration_ticks': 1920, 'channel': 0},
                {'pitch': 73, 'velocity': 80, 'start_tick': 0,    'duration_ticks': 1920, 'channel': 0},
                {'pitch': 76, 'velocity': 80, 'start_tick': 0,    'duration_ticks': 1920, 'channel': 0},
                # D major chord, bar 2
                {'pitch': 62, 'velocity': 80, 'start_tick': 1920, 'duration_ticks': 1920, 'channel': 0},
                {'pitch': 66, 'velocity': 80, 'start_tick': 1920, 'duration_ticks': 1920, 'channel': 0},
                {'pitch': 69, 'velocity': 80, 'start_tick': 1920, 'duration_ticks': 1920, 'channel': 0},
            ]
        }
    }]
})
```

## Velocity conventions

- `80` — comfortable mid-level, suitable for most comping
- `100–110` — accented / downbeat emphasis
- `60–70` — ghost notes, off-beat fills
- `1` minimum, `127` maximum; `0` is note-off (don't use in NoteSpan)

## Updating notes in an existing region

Use `set_midi_content` to replace the notes in an existing region (requires the region UID):

```python
apply_patch(client, session_uid, {
    'base_revision': rev,
    'ops': [{
        'op': 'set_midi_content',
        'region_ref': {'kind': 'region', 'uid': region_uid},
        'notes': [...]  # replaces all existing notes
    }]
})
```

## Region end time

The `end` position must cover all notes. A safe pattern: compute end from the last note's `start_tick + duration_ticks`, convert to ms at current BPM:

```python
last_tick = max(n['start_tick'] + n['duration_ticks'] for n in notes)
end_ms = (last_tick / 480) * (60000 / bpm)
end_bar = int(last_tick / 1920) + 1
```
