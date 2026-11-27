> **Contributor: TrentHunter82 (Flipping Sigmas)** ([TrentNodes](https://github.com/TrentHunter82/TrentNodes)). Shared in [NerdNotes issue #7](https://github.com/roycho87/NerdNotes/issues/7) on 2026-11-26, with his full running results page, every run with its video (7 to 26 Nov 2026): [Long Video Run Log](https://claude.ai/artifact/CRNqPxfQEaPaWMXRiRw9HA). Licensed under CC BY-SA 4.0 at his request: if you share or adapt it, credit **TrentHunter82 (Flipping Sigmas)** as well as roycho / NerdNotes. All numbers below are his.

# TrentHunter82: a 30-clip chain with one long pass 2 in context windows, plus an audio pin

## Short version

30 batched clips (about 2 min 9 s, 3099 frames at 24 fps), two scenes, no visible joins, no colour drift, lip sync kept. The method:

1. **Pass 1 (960×544) with the Video Chain**, one clip at a time: Reference to Video, the keystone (clip 1's first frame, refreshed every 6 clips), unsample E on the held overlap, Clip Refiner, Join Clips with a colour match.
2. **One long pass 2 (1920×1088) over the whole joined pass 1 video**, not one pass 2 per clip: VAE Encode, latent upscale, then **kijai's MiniMax H3 Context Windows** (141 frames, overlap 39, pyramid, one prompt per window with `split_conds_to_windows`), 2 steps, er_sde, sigmas **0.75, 0.52, 0**.
3. **Audio pin in pass 2**: the pass 1 sound is encoded and put into the latent with a noise mask of 0 (VAE Encode Audio → SolidMask 0 → Set Latent Noise Mask → Concat AV Latent). Masked audio costs no tokens.

## What he found

- **The jumps at the joins came from pass 1, not the camera.** In a per-clip chain, each clip re-draws the overlap frames it continues from, so trees and faces change shape at every join (23 of 29 joins over 1.6× the normal frame-to-frame change). Image-side fixes (affine align, RAFT morph) made it worse or ghosted.
- **One long pass 2 hides the joins.** The windows (141 frames, overlap 39) step by 102 frames like the clips, so each clip join sits inside a window overlap, where the pyramid blend mixes the two windows' predictions at every step. The re-drawn frames are refined together with the frames on both sides, so the join smooths out. Hillside joins over 1.6×: 23 of 29 → 0 of 29 (mean step 1.90 → 1.14 at sigma 0.75, 1.09 at 0.8), colour unchanged.
- **Pass 2 ruins the lip sync without the audio pin.** It refined the mouths against an empty audio latent. With the pin and a lower start sigma, the mouths follow the speech again. New measure: the ComfyUI core MediaPipe face landmarker, `jawOpen` per frame (head motion filtered out), correlated with the draft and with the speech loudness.

| Desert scene, long pass 2 | Mouth match with draft | Mouth match with speech (draft 0.27) | Mean join step | Joins over 1.6× |
|---|---|---|---|---|
| sigma 0.8, no pin | 0.62 | 0.21 | 1.10 | 4 of 29 |
| sigma 0.8, audio pin | 0.72 | 0.25 | 1.10 | 4 of 29 |
| sigma 0.7, audio pin | 0.77 | 0.26 | 1.14 | 4 of 29 |

On the hillside scene (harder for the joins), sigma 0.7 gave 2 joins at 1.6×; **sigma 0.75 gave 0** with the same speech match as 0.7, so 0.75 is his default.

- **The keystone edge drifts.** Refreshed keystones made the right edge more colourful clip after clip (+34 by clip 30). Remember Clip Ending now matches the edge's tint and colour strength to the pixels just inside it (edge boost stays +0 to +3).

## Tried and dropped

- Locking the whole join (5 latents) from the unsample: seamless joins, but a dark rounded border grew (an exact copy carries its flaws forward, the old decay).
- Moving the join to the end of the overlap (cut frame 39): no gain.
- kijai's sequential context windows (ComfyUI PR 17993) over all 30 windows: seamless, but with no anchor the colour ran from day to night.

## Nodes and workflow

TrentNodes (commit [346aa30](https://github.com/TrentHunter82/TrentNodes/commit/346aa30)):

- **Window Prompts**: the same `=====`-separated clip prompts as the chain → one Reference to Video cond per clip, joined in order for `split_conds_to_windows`.
- **Decode Long Video**: decodes a long H3 latent in parts that start on a multiple of 5 latents (17 frames each), with a 5-latent warm-up per part, so the parts join exactly; writes straight to an mp4 with the sound. A one-piece decode of 3099 frames at 1920×1088 needs about 77 GB of RAM.

The workflow has both passes in one graph, with a Fast Groups Bypasser to switch between them. He can share it on request.

## Limits

One seed per scene and two scenes. The hillside lip measure is weak (two faces that turn away); the desert scene (one face to the camera) is the reliable one. The long pass 2 uses about 90 GB of system RAM for 30 clips; kijai's tip `--fp16_intermediates` should halve the intermediate cost, but it isn't tested yet.
