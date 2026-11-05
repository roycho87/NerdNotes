# Test 14: Anti-Burn LoRA strength on a moving scene (20 clips)

Test 13 ([static chair](../20261104/static_chair_lora_test.md)) showed no visible difference between no LoRA, 0.2, 0.4 and 0.6 on a still scene. This repeats the same four strengths on the moving beach run from [test 11](../20261104/antiburn_results.md), where the LoRA did help, to see which strength is best when the scene keeps moving.

## Setup

- **Scene:** the same 20-clip beach run as tests 10 and 11 (prompts: [prompts/noise_sweep](../20261101/prompts/noise_sweep)). She runs along a beach for clips 1–9, then the old prompts cut to new places for clips 10–20.
- **Chain:** 20 clips of 8 s, 480 × 864, Geiru Context 39 / drop last 17, seed 1234. Same workflow and settings as test 13.
- **Degradation protection on:** keystone + capstone from clip 1 (on for clips 2–9, off from clip 10 when the scene starts changing, as in tests 10 and 11), context noise linear 1, Late Lock capstone from step 5.
- **Four runs:**
  1. No LoRA
  2. Anti-Burn 0.2
  3. Anti-Burn 0.4
  4. Anti-Burn 0.6
- The LoRA is only on clips 2–20, never clip 1, so all four runs start from the same clip 1.

## What we're looking for

- Burn: crunchy, over-sharpened texture building up (skin, sand, waves).
- Smoothing: does fine texture go soft as the strength goes up?
- Colour drift from clip 1.
- Motion: does a higher strength change how she moves?

*(running, queued 2026-11-05 7:50 am)*
