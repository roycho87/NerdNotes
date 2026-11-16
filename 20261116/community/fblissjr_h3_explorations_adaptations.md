# How fblissjr adapted NerdNotes and the Geiru Context Suite (ComfyUI-h3-explorations)

**By fblissjr** ([ComfyUI-h3-explorations](https://github.com/fblissjr/ComfyUI-h3-explorations), MIT, clean-room: written from the ideas, not the code). Summarised here from his "What We Borrowed & How We Adapted It" notes (Nov 16 2026). His credits are in [`docs/wiki/upstream/periphery.md`](https://github.com/fblissjr/ComfyUI-h3-explorations/commit/4040e4dc50d1f7104a475d3f2a04a201b2981669). These are his results on his own setup (6-step Pika runs, his own audio pipeline), not re-tested here.

## Context noise taper at the seam

* **From:** Lumifel's (ethanfel's) context noise and the Geiru Context Suite.
* **His version:** H3's video VAE decodes in chunks of 5 tokens with 2 overlapping, and all his context lengths (39, 90, 141 frames → 12, 27, 42 latents) leave a remainder of 2 when divided by 5. So his opt-in `seam_taper` ramps the context noise down to 0, and holds the last two context latents at exactly 0, which locks the VAE's chunk overlap across the cut. It also stops short timeline windows re-denoising the context twice.

## Early vs late blocks

* **From:** the NerdNotes block sweep ([test 19](../../20261113/attention_sweep.md)): blocks 0–29 do nothing for brightness, 31–47 carry it (38 and 44 the strongest).
* **What he found:** it matches his own telemetry, which puts brightness in blocks 38–43. On his 6-step Pika runs, scaling audio attention on all blocks collapsed the picture to static noise, while keeping it to blocks **0–29** restored the scene (blocks 30–49 cause the collapse). He also fixed a double-scaling bug in his attention code along the way.

## A frame 0 anchor that lets go early

* **From:** Geiru Keystone.
* **What he points out:** a keystone held through all the steps of every clip holds brightness (his reading of the 30-clip test: −7.4%), but pulls a moving camera back toward frame 0.
* **His version:** a frame 0 guide latent in the DiT only, released after the first third of the steps, set per timeline section. It carried about 78% of a brightened frame 0's exposure into the clip (+23.1% of +29.5%) without locking the camera in place.

## Attention balance between two sets of keys

* **From:** Geiru Keystone Attention and ckinpdx's Carry Attention ([MMH3Tools](https://github.com/ckinpdx/ComfyUI-MMH3Tools)).
* **His version:** the same idea (re-split the attention share between two sets of keys, everything else unchanged), rewritten for speed on his SageAttention fork: **5.9× faster** on an RTX 4090 (88 ms against 519 ms), and safe past ~100k tokens. Shared across his audio, Ref2VA and song-loop nodes.

## What he skipped, and why

* **Carried audio latents (`hold_audio`):** caused a +17.6 dB audio runaway in his pipeline. He slices a pre-encoded master track for each clip and muxes the audio losslessly instead, which already solves audio drift for him.
* **Separate window-cropping nodes:** already built into his song-loop node.
