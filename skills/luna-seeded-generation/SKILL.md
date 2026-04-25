---
name: luna-seeded-generation
description: Use when generating multiple audio clips that need to sound like the same instrument — same bass player, same drum kit, same organ. Also use when the producer asks to generate alternate takes, chord variations, or segment a long loop into shorter parts while keeping the instrument voice consistent. Fires on phrases like "keep it sounding like the same X", "generate another take", "same instrument different chord", "alternate version of the bass", or any time you're about to call generate_audio multiple times for the same instrument role.
---

# Seeded Audio Generation

Use `generate_audio_seeded` — not `generate_audio` — whenever consistent instrument timbre across multiple clips matters.

## Why seed matters

`generate_audio` uses `/plugin/generate`. That endpoint does not accept seed — it is silently dropped. Every call produces a completely random take.

`generate_audio_seeded` uses `/agents/audio/text-to-audio`, which honors seed. Same seed + same prompt = perceptually identical audio. WAV hashes will differ (encoding metadata varies) but the waveform sounds the same — confirmed by ear.

## Basic pattern

Pick one seed per instrument role and hold it for all segments:

```python
BASS_SEED  = 42
KEYS_SEED  = 13
DRUMS_SEED = 7

# A section
bass_A = generate_audio_seeded(bass_prompt_A, bpm=120, duration_s=16, seed=BASS_SEED)
# D section — same seed, chord changes in prompt only
bass_D = generate_audio_seeded(bass_prompt_D, bpm=120, duration_s=16, seed=BASS_SEED)
```

## BPM

`/agents/audio/text-to-audio` does not accept a `bpm` field. Pass `bpm=` anyway — the helper auto-inlines it as `"120 BPM, ..."` prepended to your prompt. Do not add BPM to the prompt manually if passing `bpm=`.

## generate_continuation inherits seed automatically

Calling `generate_continuation(prev_result, prompt)` on a seeded result detects the seed in the envelope and routes to `generate_audio_seeded`. No extra wiring needed for chained segments.

## Prompt minimization — the most important rule

Seed locks the noise starting point. The prompt steers denoising. When the prompt changes significantly, the model follows the prompt and produces a different-sounding instrument even with the same seed.

**What survives seed:** Small swaps — chord name, key name, one descriptive word.

**What breaks seed:** Groove vocabulary changes, genre cue changes, different energy descriptors, different instrument lists.

To maximize timbre consistency, write a template and swap only the one variable:

```python
TEMPLATE = (
    "Isolated drum stem recording, kick snare hi-hat ride cymbal only, "
    "tight funk groove, {pattern}, "
    "16th-note hi-hats, dry close-miked studio drum recording, "
    "starts exactly on beat 1, no silent intro, strict tempo, "
    "loopable — the final beat leads cleanly back to the first beat, "
    "no bass guitar, no electric bass, no keyboards, no guitars, no melody"
)

groove_A = generate_audio_seeded(TEMPLATE.format(pattern="snare on 2 and 4, four-on-the-floor kick"), seed=7, bpm=120, duration_s=16)
groove_B = generate_audio_seeded(TEMPLATE.format(pattern="half-time feel, snare on beat 3, kick on 1 and 3"), seed=7, bpm=120, duration_s=16)
```

## Hard-won caveats

- **Seed does not lock the kit through large prompt changes.** Changing groove vocabulary (half-time vs driving 4/4) produces a noticeably different-sounding kit even with the same seed. Timbre is entangled with performance description in training.
- **"Closer" prompts = closer timbre.** The more vocabulary overlap between two prompts, the more similar the resulting sound. Swapping one clause beats swapping multiple.
- **Drum stems bleed bass.** Genre cues like "funk-rock" pull toward full arrangements. Use stem/isolation vocabulary: `"isolated drum stem recording"`, `"dry close-miked studio drum recording"`, enumerate specific kit pieces. Reduces but does not eliminate bleed.
- **`generate_audio`'s seed param is wired but silently ignored by the backend.** Use `generate_audio_seeded`.
