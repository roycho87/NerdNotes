# Test 18: making the audio hold up down a chain

The picture side of degradation is mostly solved by keystone + capstone ([test 16](../20261107/beach30_protection_combos.md), [test 17](../20261107/forest30_protection_combos.md)). The audio isn't: with `hold_audio` on, each clip continues the previous clip's held sound, and over a long chain it can turn dull, build up or turn into a droning hum ([Dylan's measurements](../community_cc_by_sa/matlowai_h3_long_video_research.md); NikoDemon80 describes the same thing in Motion Context's notes). These tests look for a way to make the audio hold up longer with the held audio on.

## Short version

- **Use clip lengths that are multiples of 3** (141, 192, 243, 294, 345). With 175-frame clips the sound built up about 17 dB over 10 clips; with 192-frame clips it moved much less.
- **`audio_drop_last` 14 vs 15 barely matters.**
- **Re-encoding the held audio stopped the build-up** on the off-grid length, and was the best by ear for birds, especially with `audio_feather_ticks` 8.
- **It didn't fix speech.** Re-encode + feather 8 on speech went uneven late in the chain. Audio degradation is still open.
- The re-encode and guide modes are **experimental only** (private pack); nothing here is in the public suite. The suite README now recommends multiples-of-3 lengths.

## The idea: the audio grid

H3's audio runs at 40 steps a second against 24 video frames, so one frame is 5/3 of an audio step. Only frame counts that are multiples of 3 land exactly on an audio step.

- With 192-frame (8 s) clips the held audio starts exactly on a step (192 − 39 = 153 frames = 255 steps).
- With 175-frame clips it starts at frame 136 = 226⅔ steps, so the held audio is cut a third of a step off from where it belongs.
- `audio_drop_last` 15 holds 24 frames = exactly 40 steps; 14 holds 25 frames = 41⅔ steps.

## Setup (the same for every run)

| | Setting |
|---|---|
| Clips | 10 clips, 192 frames (8 s) or 175 frames (7.3 s) |
| Picture | 32 × 32 (the picture isn't judged; this keeps it fast, about 22 s a clip) |
| Geiru Context | 39 frames, drop last 17, held audio on, `audio_drop_last` 15 unless noted |
| Degradation protection | off (plain chain; keystone and context noise don't touch the audio) |
| Sampling | seed 5678, 8 steps er_sde / beta, the 8-step turbo LoRA |
| Prompts | one speaker repeating "This is a test." ([prompts/audio18](prompts/audio18)); a quiet park with a fountain ([prompts/audio18amb](prompts/audio18amb)); loud birdsong in a forest ([prompts/audio18birds](prompts/audio18birds)). The same prompt every clip. |

## Runs

| Run | Length | What changed |
|---|---|---|
| Speech | 192 | drop 15 and drop 14 |
| Park | 192 | drop 15 and drop 14 |
| Birds | 192 | drop 15 and drop 14 |
| Birds, off grid | 175 | drop 15 (normal) and drop 14 |
| Timing-label patch | 175 | moved the held audio's position label by the ⅓ step (birds) |
| Feather 8 | 175 | `audio_feather_ticks` 8 (the suite default is 0) (birds) |
| Guide | 175 | the held audio given as a guide at its exact time, Motion Context style, instead of being held (birds) |
| Re-encode | 175 | decode the previous clip's audio, cut it at the exact sample, re-encode it, hold that (birds) |
| Re-encode + feather 8 | 175 | both (birds, then speech) |
| Speech, off grid | 175 | normal vs re-encode + feather 8 |

## Results

**By ear (roycho):**

- **192 speech:** no buzz at all. drop 14 vs drop 15 sounded the same.
- **175 vs 192:** the off-grid length "definitely did" build up. drop 14 vs 15 still made little difference.
- **Timing-label patch:** no change.
- **Guide:** terrible overlaps at the joins.
- **Re-encode:** the best by far.
- **Feather 8:** smoother joins.
- **Re-encode + feather 8 (birds):** came out pretty good, the best of the birds runs.
- **Speech:** still degrades, with either setting.

**Birds, 175 frames. Loudness build-up, clip 1 → clip 10:**

| Setup | Build-up | Cut vs normal | Listen |
|---|---|---|---|
| Normal (baseline) | +17.0 dB | | [m4a](media/audio18_birds175_drop15.m4a) |
| drop 14 | +16.1 dB | 5% | |
| Timing-label patch | +16.3 dB | 4% | [m4a](media/audio18_birds175_fix.m4a) |
| Guide | +15.0 dB | 12% | [m4a](media/audio18_birds175_guide.m4a) |
| Feather 8 | +13.4 dB | 21% | [m4a](media/audio18_birds175_feather8.m4a) |
| Re-encode + feather 8 | +2.7 dB | 84% | [m4a](media/audio18_birds175_reenc_feather8.m4a) |
| Re-encode | −0.4 dB | 100% | [m4a](media/audio18_birds175_reenc.m4a) |

For comparison, birds at 192 frames: +6.6 dB at clip 10 with drop 15 ([m4a](media/audio18_birds_drop15.m4a)), −1.4 dB with drop 14, both wandering up and down along the way (peaks of +12 and +7 dB).

**Speech, 175 frames, normal vs re-encode + feather 8** ([normal](media/audio18_speech175_normal.m4a), [re-encode + feather 8](media/audio18_speech175_reenc_f8.m4a)):

| Setup | Loudness, clip 1 → 10 | Hum sign, clip 1 → 10 | Treble, clip 1 → 10 |
|---|---|---|---|
| Normal | +1.6 dB | 0.034 → 0.014 | 6% → 12% |
| Re-encode + feather 8 | −1.5 dB | 0.034 → 0.001 | 6% → 5% |

Speech doesn't build up in loudness the way birds do. Normal gets steadily thinner and brighter. Re-encode + feather 8 swings up and down, and by clips 8–10 its hum sign is near zero, meaning it's turned tonal.

**What it means:**

- **The off-grid cut causes the build-up.** At 175 frames the held audio is ⅓ step off, and the error adds up clip after clip. Multiples-of-3 lengths avoid it with no extra nodes.
- **Re-encoding at the exact sample removes the build-up** even on off-grid lengths. Feather 8 smooths its joins.
- **It doesn't stop the slow loss of quality.** Speech still drifts, and re-encode made speech less steady late in the chain. The held audio still degrades a little with every copy; this test doesn't solve that.

## Ideas to try next (roycho)

Neither of these has been built or tested.

- **Post-sampling audio pass:** render each clip with its own fresh audio, made in sync with its picture, and carry no audio over from the context. Afterwards, do a second pass at each join with the picture locked. That pass anchors the previous clip's last 39 frames of sound, like Geiru Anchoring, and regenerates only the new clip's audio around the join so the sound lines up. Nothing gets copied from clip to clip, so nothing can build up down the chain. The cost is an extra sampling pass per join.
- **Cheap audio pass first:** make the audio in a quick 32 × 32 render first (for example at 32 steps), grab its end, and lock that audio in the full-size render. With the audio locked, the model should draw the picture to fit the sound, the same way a talking clip follows a recorded voice.

**Caveats:**

- One seed, 10 clips, 32 × 32 picture.
- The park runs turned tonal with both drop settings, and the speech runs at 192 changed slowly; neither tested a fix.
- The 175 and 192 birds runs start from different first clips (the length changes clip 1), so compare their build-up, not their loudness.

**How it's measured:** every clip after the first is measured after its 1.625 s held head, so only the new audio counts. Loudness is RMS in dB. Hum sign is spectral flatness (50 Hz–12 kHz): noisy, natural sound scores higher; a hum or tone scores close to 0. Treble is the share of energy above 4 kHz. Raw numbers for every clip: [audio18_metrics.csv](media/audio18_metrics.csv).
