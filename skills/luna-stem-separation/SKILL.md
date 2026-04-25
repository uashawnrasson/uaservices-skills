---
name: luna-stem-separation
description: Use when separating a mixed audio file into individual stems (drums, bass, vocals, other) to place them on separate LUNA tracks. Fires on phrases like "separate this into stems", "pull the drums out of this", "stem separate", "isolate the bass", or any workflow starting from a full mix WAV that needs individual stems in the session.
---

# Stem Separation

## Output shape

`separate_stems` returns `ok({stems, stem_paths, source_wav, provider, config})`.

`stems` is a dict keyed by stem name — always `drums`, `bass`, `vocals`, `other`. Values are absolute file paths. Use `stems` (the dict) in code; `stem_paths` is a convenience list of the same paths in insertion order.

```python
result = separate_stems(client, '/path/to/mix.wav')
stems = result['data']['stems']
# stems['drums'] → '/path/to/mix_drums.wav'
# stems['bass']  → '/path/to/mix_bass.wav'
# stems['vocals']→ '/path/to/mix_vocals.wav'
# stems['other'] → '/path/to/mix_other.wav'
```

## Critical: stems land next to the source WAV

Output files are written as siblings of the input file — `{source_stem}_{name}.wav` in the same directory. The source WAV must be in a location writable by UA Services (local filesystem, not a temp path that gets cleaned up).

## Full workflow: separate → place on tracks

```python
from pathlib import Path

# 1. Separate
result = separate_stems(client, source_wav_path)
if not result.get('ok'):
    print('Failed:', result['error'])
stems = result['data']['stems']

# 2. Import and place each stem
for stem_name, wav_path in stems.items():
    track = next((t for t in doc['tracks'] if stem_name in t['name'].lower()), None)
    if not track:
        print(f"No track found for {stem_name} — create one first")
        continue
    imp = import_audio(client, Path(wav_path), session_uid)
    if not imp.get('ok'): continue
    d = imp['data']
    apply_patch(client, session_uid, {
        'base_revision': rev,
        'ops': [{'op': 'insert_region',
            'track_ref': {'kind': 'track', 'uid': track['uid']},
            'region': {
                'name': f'{stem_name} stem',
                'start': {'musical': {'bars': 1, 'beats': 1}, 'clock': {'ms': 0}},
                'end':   {'musical': {'bars': end_bar, 'beats': 1}, 'clock': {'ms': end_ms}},
                'source_ref': {
                    'sha256': d['sha256'], 'file_uid': d['file_uid'],
                    'source_start_ns': 0, 'source_end_ns': d['duration_ns'],
                    'path_hint': f"audio/{d['file_uid']}.wav"
                }
            }
        }]
    })
    rev = ...  # read from patch result
```

## Provider not ready

If you get `provider_not_ready`, the Demucs model files are missing. Fix:
1. Install via LUNA / UA Connect
2. Call `reload_models(client)` — no server restart needed
3. Retry `separate_stems`

## Providers

- `"demucs"` (default) — local on-device, ~5-10s on Apple Silicon neural compute
- `"audioshake"` — cloud-based, requires credits

`compute="neural"` uses Apple Silicon NPU. Fall back to `compute="cpu"` only if neural fails.
