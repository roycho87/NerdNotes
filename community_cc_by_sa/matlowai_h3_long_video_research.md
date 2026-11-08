> **Contributor: Dylan (matlowai)** ([h3-long-video-research](https://github.com/matlowai/h3-long-video-research)). A summary of his research, which he published under CC BY 4.0 (it can be adapted under CC BY-SA 4.0 with credit). If you share or adapt it, credit "Dylan (matlowai), h3-long-video-research" as well as roycho / NerdNotes.

# Related research: Dylan (matlowai), h3-long-video-research

**Source:** [github.com/matlowai/h3-long-video-research](https://github.com/matlowai/h3-long-video-research), by Dylan ([matlowai](https://github.com/matlowai)) with Claude agents, published 2026-11-07. Text and figures are CC BY 4.0; this page summarizes it and links to it. Credit: "Dylan (matlowai), h3-long-video-research".

Open lab notes on making MiniMax H3 / Pika H3 Flex videos longer and more consistent. The extend studies ([U46–U46g](https://github.com/matlowai/h3-long-video-research/blob/main/studies/u46_extend.md)) measure the Geiru Context Suite directly: the latent carry (Geiru Context), Keystone + capstone, Late Lock and Context Noise. Their setup differs from ours: 832 × 480, the Pika H3 Flex 6-step LoRAs, about 6-second clips, chains of up to 24 clips (about 2 minutes), mostly one seed per test.

## What they found that matters for the suite

1. **Carry the latent, don't re-encode.** Decoding each clip and re-encoding its tail through the VAE causes the colour creep (saturation 1.42 → 1.10 over 3 clips). Carrying the sampler's own latent, as Geiru Context does, avoids it and is faster per clip.
2. **A plain latent carry holds about 40 seconds**, then grows a mosaic texture, and a face becomes a different person by about 90 seconds.
3. **Keystone + capstone is what holds 2 minutes.** Re-anchoring to clip 1 with keystone front + capstone back held the full 2 minutes, even when it was only used **every 3rd clip ("K3")** or every 6th ("K6"), at no extra cost. The face stayed the same person (ArcFace 0.75–0.85 against loop 1; a plain carry fell to 0.31).
4. **The rest of the bundle didn't add anything.** With the protection LoRA, context noise and Late Lock added (their "Strong" arm), the face held but the hair darkened by 19 L* and the background changed on a moving shot. Their recommendation is keystone + capstone only, with no LoRA, no context noise and no Late Lock.
5. **The Anti-Burn LoRA doesn't pay on a latent carry** up to 35 seconds. There's little burn to cancel, and it softens detail.
6. **The audio drone comes from the audio carry.** Long chains grow a sustained hum (tagged "music / violin" by an audio model). It comes from `hold_audio`, not from the keystone. Turning `hold_audio` off on the re-anchor clips, so they get fresh audio, took it to 0 of 24 clips with the picture unchanged.

## How it compares with our tests

- **It agrees with [test 16](../20261107/beach30_protection_combos.md) and [test 17](../20261107/forest30_protection_combos.md):** keystone + capstone does the real work, and the LoRA and context noise add little or nothing visible on top.
- **New for us:**
  - The keystone may not need to run on every clip. Every 3rd clip held 2 minutes in their tests.
  - The audio drone has a cause (the audio carry) and a free fix (`hold_audio` off on the anchor clips).
- **Late Lock:** they found it didn't help (their "L3" arm decayed almost like a plain carry). As they describe it, in that arm the **keystone** front slot was held empty until step 3 of 6, so the first steps ran on the plain carried latent. That's different from the suite's default, which uses Late Lock on the **capstone slot only**, with the keystone locked from the first step. So it says holding the keystone back hurts. It doesn't test the default capstone-only Late Lock, and that's still open.

## Caveats (theirs)

Mostly one seed per test and one or two scenes. Audio was judged by spectral features and an audio tagger, not by listening. Re-anchor periods other than 3 and 6, and chains past 2 minutes, are untested.

## Also in the repo

- [Prompt craft rules](https://github.com/matlowai/h3-long-video-research/blob/main/studies/prompt_craft_rules.md):
  - a reference picture acts as a target, so a "key" picture on an action cut makes the shot morph into it and freeze
  - pictures beat words
  - the `<Subject N>` form is tidiness, not a lever
- [FL2VA vs Ref2VA](https://github.com/matlowai/h3-long-video-research/blob/main/studies/factcheck_subject_vs_picture.md): they're separate checkpoints with separate adapters.
- [World-state design](https://github.com/matlowai/h3-long-video-research/blob/main/studies/world_state_design.md): references with roles and lifetimes, and a ledger.
- [Rolling references](https://github.com/matlowai/h3-long-video-research/blob/main/studies/u46h_rolling_refs.md): still in progress when it was published.
