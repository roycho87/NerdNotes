# Test 20: Keystone Attention's cost to motion, and Keystone Attention in Hybrid Windows

Follow-up to [test 19](../20261113/attention_sweep.md). Geiru Keystone Balance is now **Geiru Keystone Attention**, released in the [Geiru Context Suite](https://github.com/roycho87/geiru_context_suite) (1.1), with balance 20 on the 13 lighting blocks (31-33, 36-44, 47) as the default.

## Short version

* With Keystone Attention on, hair moves less. No single block causes it: the stiffness tracks the brightness hold itself.
* On a jumping jacks chain the motion cost was small. Judged by eye, not enough to matter.
* Taking the keystone's extra attention from the whole pie instead of only from the carried frames made no real difference. Removed.
* Turning attention on the carried frames up or down in the motion blocks (0-19) did nothing.
* Geiru Hybrid Windows now keeps its full-length copies in system RAM and runs each window's own steps on just that window: about **half the GPU memory** while sampling, same speed, output identical bit for bit.
* Keystone Attention now works inside Hybrid Windows too.

## 1. Hair stiffness: which block?

Clip 10 of the beach chain, continued from the same clip 9. **Hair motion** = how much of the pink hair changes shape from frame to frame (a colour mask of the hair, lined up on its centre, frames 40-190; higher = more motion). **Brightness** = corner brightness against clip 1.

| Run | Hair motion | Brightness |
|---|---|---|
| Keystone + capstone | 0.70 | −22% |
| Keystone only | 0.71 | −22% |
| Keystone + attention, 13 blocks | 0.53 | −7.6% |
| Same, taking from the whole pie | 0.555 | −8.3% |
| 13 blocks minus 36-42 | 0.58 | −7.8% |
| 13 blocks minus 36-42 and 44 | 0.60 | −8.6% |
| Any one block alone (13 runs) | 0.70-0.73 | −19% to −22% |
| 13 blocks minus any one (13 runs) | 0.51-0.57 | −7% to −8% |

* No single block holds the brightness on its own, and no single block stiffens the hair: dropping any one of the 13 changes neither much.
* Wherever the brightness is held, the hair moves less. The stiffness is a side effect of the hold, not of one block.
* (An early run flagged a hair "flip" that turned out to be a seagull crossing the frame. The hair mask replaced the region difference after that.)

## 2. Jumping jacks: the motion cost

A 5-clip jumping jacks chain, made to show motion loss. Brightness for clips 2-5; hair motion averaged.

| Run | Clip 2 | Clip 3 | Clip 4 | Clip 5 | Hair motion |
|---|---|---|---|---|---|
| No protection | −3.3% | −7.6% | −10.6% | −14.6% | 0.469 |
| Keystone only | −2.9% | −6.2% | −7.8% | −12.8% | 0.453 |
| Keystone + attention | −4.1% | −6.2% | −7.1% | −8.7% | 0.409 |
| Keystone + attention, whole pie | −2.9% | −5.8% | −8.2% | −9.4% | 0.406 |

* Attention holds brightness (−8.7% by clip 5 against −12.8% for keystone only) for about 10% less hair motion.
* Watched side by side, the motion is affected, but not by enough to matter.
* Taking from the whole pie (prompt, references and sound give up a share too) looked and measured about the same, and so did limiting it to certain steps. Both settings were removed to keep the node simple.

## 3. The motion blocks

On Banodoco, blocks 0-19 are generally seen as the motion-heavy blocks (from LoRA training). Two ideas to win the motion back:

* **More attention on the carried frames in blocks 0-19** (Geiru Attention Balance, context strength 1.5 and 2): no effect.
* **Less attention on the carried frames** (balance below 1) on blocks 0-19, odd blocks and even blocks: no effect either.

So the carried frames' attention in the early blocks isn't where the motion is lost.

## 4. Geiru Hybrid Windows: half the GPU memory

Each window's own steps used to run the sampler on the full-length latent, with full-length copies (state, clean guesses, snapshots, keystone, capstone) kept on the GPU. Two fixes:

* **System RAM:** keep the full-length copies in RAM and move them to the GPU only for the math.
* **Crop:** run each window's own steps on just that window (video and sound), with the conditioning still full-length and the noise cut from the same full-length draw, so nothing changes.

Biker babe workflow, first 4 prompts (787 frames), Window Director re-rolling windows 2 and 3 twice:

| Run | Time per window | GPU memory while sampling |
|---|---|---|
| Original | ~143 s | ~930 MB |
| RAM fix | ~152 s | ~646 MB |
| Crop fix | ~148 s | ~667 MB |
| Both | ~145 s | ~510 MB |

* All four final videos are identical, frames and sound.
* The peak (about 1.7 GB in every run) is the VAE decode at the end, not the sampling.
* Both fixes are now how Hybrid Windows always works.

## 5. Keystone Attention inside Hybrid Windows

Hybrid Windows has a `keystone_attention` setting (needs `keystone_swap`), with the same `balance`, `block_mode` and `blocks`. It runs during each window's own steps; the blended steps have no keystone, so it's off there.

Check on the first two biker babe windows:

* Off: windows 1 and 2 identical to before the change, frames and sound.
* On (balance 20, lighting blocks): window 1 unchanged (the keystone comes from it), window 2 different, no errors.

Window Director also now lets the protection settings (keystone, keystone attention, capstone, context noise, fuse method, freenoise, crash protection) change between windows.

## Next

* Keystone + attention + context noise over 30 clips (the [Easily Distracted Clown](https://github.com/roycho87/geiru_context_suite/blob/main/example_workflows/easily_distracted_clown.json) setup).
* A longer Hybrid Windows render with keystone attention, measured on brightness.
* Kijai's Model Preview Override as a better preview for Window Director.

## Notes

* One scene and seed per test. Brightness is measured on corner patches; motion and look are judged by eye.
* All with the 8-step turbo LoRA and the Sage patch off.
