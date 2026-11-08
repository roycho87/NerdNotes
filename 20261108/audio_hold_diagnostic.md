# Test 18: making the audio hold up down a chain (diagnostic)

The picture side of degradation is mostly solved by keystone + capstone ([test 16](../20261107/beach30_protection_combos.md), [test 17](../20261107/forest30_protection_combos.md)). The audio isn't: with `hold_audio` on, each clip continues the previous clip's held sound, and over a long chain it can turn dull or into a droning hum ([Dylan's measurements](../community_cc_by_sa/matlowai_h3_long_video_research.md); NikoDemon80 describes the same thing in Motion Context's notes). These tests try to find a way to make the audio quality hold longer with `hold_audio` on.

We're starting with a diagnostic run and going from there.

## One idea being tested: lining the held audio up with the audio grid

H3's audio runs at 40 steps a second against 24 video frames, so one frame is 5/3 of an audio step. Only frame counts that are multiples of 3 land exactly on an audio step.

- With a 39-frame overlap and `audio_drop_last` 17, the held audio is 22 frames = 36⅔ steps, rounded to 37. It ends a third of a step past the picture join.
- `audio_drop_last` 15 holds 24 frames = exactly 40 steps. `audio_drop_last` 0 holds the full 39 frames = exactly 65 steps.
- With 192-frame (8 s) clips the held audio starts exactly on a step (192 − 39 = 153 frames = 255 steps).

## Diagnostic run

| | Setting |
|---|---|
| Clips | 10 clips of 8 s (192 frames) |
| Picture | 32 × 32 (the picture isn't judged; this keeps it fast) |
| Prompt | one speaker repeating "This is a test." for the whole clip ([prompts/audio18](prompts/audio18)) |
| Geiru Context | 39 frames, drop last 17, `hold_audio` on, `audio_drop_last` 15 |
| Degradation protection | off (plain chain; keystone and context noise don't touch the audio anyway) |
| Sampling | seed 5678, 8 steps er_sde / beta, the 8-step turbo LoRA |

## What we're listening and looking for

- Whether the voice stays clear clip after clip, or gets duller, muffled or turns into a hum.
- Jumps at the joins (volume or timing).
- roycho's ears make the final call; measured with spectral flatness, pitch centre and loudness per clip.

*(running, started 2026-11-08)*
