# Test 23: Context Attention (more attention on the held frames)

Follow-up to [test 21](../20261119/block_map.md) and [test 22](block_step_echo.md). Instead of showing the model a different version of the held frames, this leaves them as they are and changes **how much the model looks at them**, only on the blocks the block map ties to degradation.

**Status:** node and workflow built and checked on CPU. No renders yet.

## The idea

In a chain, the held frames (the "context group", locked in by Geiru Context) are the model's link to the clip before. Test 21 found brightness, contrast and sharpness all live in blocks 30–49. If every new frame pays more attention to the held frames on those blocks, the new frames should match the held frames' look more closely, and carry less of their own drift.

## Has this been done?

Close relatives:

* **[ckinpdx's Carry Attention](../20261113/community/ckinpdx_carry_attention.md)** turns attention on the carried frames **down** (and the reference pictures up), on all blocks. A bias of −2 held the joins with a slight bake; −3 broke them.
* **[Test 19](../20261113/attention_sweep.md)** turned the carry down alone, on all blocks, and it didn't slow the darkening.
* **Keystone Attention's "whole pie" mode** does the same maths for the keystone alone.

New here: turning the context group **up**, and only on the degradation blocks.

## The node: Geiru Context Attention (experimental)

* `strength` multiplies the attention every token pays to the held frames. 1 = off, 2 = double, 0.5 = half, 0 = ignored, no upper limit. Every other part (prompt, references, audio, new frames) gives up a little to make room, in proportion.
* Block presets from test 21's final sets:
  * **degradation** (brightness + contrast + sharpness together): 31–42, 44, 47
  * brightness: 31–33, 36–38, 40–41, 44, 47
  * contrast: 34–37, 39–40, 42, 44, 47
  * sharpness: 32–35, 37, 39–40, 42, 44
  * custom
* Only fully held latents count as the context group. Passes through on the first shot.
* Built on Keystone Attention's balanced attention, so it runs on plain, Sage, Kitchen and KJNodes' Low VRAM setups.
* The console prints the held frames' real share of attention before and after, on the first step.
* Note: blocks 31, 38 and 41 are in the brightness set but work against sharpness in the block map, so "degradation" pulls a little against sharpness on those three.

CPU check on a small random H3-shaped model, 19 of 19: the changes start exactly at the first chosen block, the maths matches by hand (held share 6.6% → 12.4% at strength 2, → 3.4% at 0.5, → 0% at 0, → 98.6% at 1000), and it gives the same output on top of a Low VRAM-style patch.

## Setup

* Geiru Context 39 with `drop_last` 17 (22 frames held) → sampler. Model → Geiru Context Attention → guider.
* No Keystone, Keystone Attention, Context Noise or Block Step Echo.
* The moving prompt set from test 22. 124 frames, 480×864, 8 steps, the 8-step ref2v turbo LoRA, er_sde / beta, seed 1234.
* Start: strength 2 on "degradation", 2-clip chains to look at the seam first.

## Results

To come.
