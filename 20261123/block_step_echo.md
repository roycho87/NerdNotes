# Test 22: Block Step Echo (setup, results to come)

Follow-up to [test 21](../20261119/block_map.md). The block map showed that brightness, saturation, warmth, contrast and sharpness all live in blocks 30–49, and that blocks 0–29 do nothing for any of them. This test asks: what if the two halves of the model are shown different versions of the held frames?

**Status:** the node and both test chains are built. No renders yet. This page is the plan; results get added here.

## The idea

In a chain, each new clip holds some frames from the clip before (Geiru Context locks them with a mask of 0). Normally every block sees those held frames as clean, finished frames.

Block Step Echo splits that:

* **Blocks 0–29** see the clean, locked held frames, as usual. These blocks set layout and motion.
* **Blocks 30–49** see the **echo**: what those same frames looked like inside the model, at this same sampling step, back when the clip before was being made.

So the top half of the model, the half that decides the look, gets held frames that are "developing" at the same pace as the new frames next to them, not frames that are already finished.

This comes from an older experiment, Step Echo (Nov 2), which saved each step's latent and laid it over the held frames while the next clip sampled. That version didn't work well. The block map is the reason to try again, this time inside the model and only on the blocks that matter.

## How it works

1. **Record.** While a clip samples, the state going into block 30 is saved at every step, for the frames the next clip will hold.
2. **Echo.** On the next clip, at block 30, the held frames' state is swapped for the saved one from the matching step.
3. **Switch the label.** H3 tells every block how noisy each frame is. Locked frames carry a "given frame" label (0.999, the same one pinned keyframes get). From block 30 up, the held frames' label is switched to this step's noise level, so the label matches the echo. Blocks 0–29 keep the normal label.
4. **Same noise.** The echo was made from the previous clip's noise, so the held frames need that same noise here. The node makes "timeline noise": each latent's noise comes from the seed plus its place in the whole video, not its place in the clip. Frames that overlap between two clips get identical noise. New frames get fresh noise, and nothing repeats.

Notes on the mechanics, from ComfyUI 0.45.0's H3 code:

* A fully locked (mask 0) latent goes into the model as 99.9% clean latent + 0.1% noise, the same mix as a pinned keyframe.
* Each block gets the per-frame labels as its own input, so they can be changed for some blocks and not others.
* ComfyUI's block replace hook hands a patch the block's input state and labels, which is all this needs.

## Has this been done?

Not this exact combination, as far as a Banodoco and paper search could find. The pieces exist:

* **[Ca2-VDM](https://arxiv.org/abs/2411.16375)** caches the attention keys and values of the previous clip's clean frames and reuses them. Clean frames only, not per step.
* **[RF-Solver-Edit](https://arxiv.org/abs/2411.04746)** and plug-and-play feature injection save internal features in one pass and inject them into another pass at the same step. Used for editing.
* **[ContextFlow](https://arxiv.org/abs/2509.17818)** injects saved attention into about 4 chosen blocks out of 40, picked by measuring how much each block responds, and found that different blocks do different jobs.

## Setup

* **Node:** Geiru Block Step Echo (experimental, not in the suite). `start_block` 30, label switch on.
* **Chain:** Geiru Context 39 with `drop_last` 17 (22 frames held) → Block Step Echo → sampler. No Keystone, Keystone Attention or Context Noise, so the echo is tested alone.
* **Render:** 30 clips of 124 frames, 480×864, 8 steps, the 8-step ref2v turbo LoRA, er_sde / beta. No other LoRAs. Timeline noise seed 1234.
* **Two scenes, 30 prompts each:**
  * **Moving:** one continuous take of her coming toward the camera along a lakeside park path, always walking, jogging or skipping, with the camera tracking backward at the same distance.
  * **Static:** she sits in an armchair by a window with the camera locked in place, moving naturally: sipping tea, reading, stretching, looking outside.
* **Overlap prompted:** every prompt opens with the action the clip before was doing in its held frames, then adds a new action, then starts the action the next clip will open on.
* **Comparison:** the same chain with the node's `enabled` off. The timeline noise stays on in both, so the only difference is the echo.

## What to look for

* Brightness, saturation, contrast and sharpness per clip, clip 1 to clip 30, with and without the echo.
* The joins: any jump or flash where the held frames meet the new ones.
* Motion: does the echo cost movement, like Keystone Attention did a little in [test 20](../20261115/keystone_attention_motion_and_hybrid.md)?
* A label switch on / off run, to see whether the label matters or only the swapped state does.

## Open questions going in

* The echo comes from the previous clip's frames at their old positions in that clip, placed at the start of this one. The state at block 30 already has that old position mixed in. Does that confuse the model?
* The sampler used here (er_sde) adds fresh noise every step, so only the starting noise matches between clips. Does that matter?
* The output for locked frames is thrown away and replaced by the clean latent. The echo can only act through how the new frames read the held frames in blocks 30–49. Is that enough to change anything?

## Results

To come.
