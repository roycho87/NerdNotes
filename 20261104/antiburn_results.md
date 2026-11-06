# Anti-Burn LoRA results: burn and colour (tests 11 and 12)

> **Update (2026-11-05):** the colour gain seen here couldn't be replicated in [test 14](../20261105/beach_lora_strength_test.md). The suspected cause is double compression of one of the initial side-by-side videos during stitching. The measurements below come from the original clips, and on the beach clips they show small differences. Geiru Protection LoRA is now in [Geiru-Protection-LoRa-Experimental](https://github.com/roycho87/Geiru-Protection-LoRa-Experimental).

Follows [20261101 tests 11 and 12](../20261101/frame0_influence_test.md). Burn and colour are measured; the motion verdict is roycho's, from watching the videos.

## What ran

- **Test 11:** the 20-clip beach run at linear 1 (the new default), with Imagineer13's [H3 Anti-Burn LoRA](https://huggingface.co/Imagineer13/H3-Anti-Burn-LoRA) (`h3_antiburn_universal_v1`) at 0.6 on clips 2–20. Clip 1 is shared and has no LoRA. Compared with test 10's linear 1 run (same settings, no LoRA).
- **Test 12, two versions:** both branch at clip 9 and use the turning prompts for clips 10–20 ([prompts/noise_sweep_turn](../20261101/prompts/noise_sweep_turn)): she turns into a new place every clip. Keystone and capstone off from clip 10, context noise on.
  - **With Anti-Burn 0.6:** continues test 11's clip 9.
  - **No LoRA:** continues test 10's linear 1 clip 9. Same prompts and seed.

Stitched videos (on roycho's PC, `output\minimax\noise_sweep\stitched\`):
- `antiburn_060.mp4`: test 11 in full.
- `turn_antiburn.mp4` and `turn_no_lora.mp4`: test 12, all 20 clips each.
- `turn_side_by_side.mp4`: Anti-Burn on the left, no LoRA on the right.

## How it's measured

Same as [section 11 of the conclusions](../20261101/frame0_conclusions.md):
- **Colour change:** average Lab colour of the four corners, compared with clip 1 (lower = closer to clip 1).
- **Grit:** fine texture left after a soft blur (lower = less burn). Corners on the beach clips; the full frame once the scene changes, since the corners show different things.

## Test 11: the beach clips (3–9)

| Run | Colour change (avg) | Colour change at clip 9 | Corner grit | Full-frame grit, clips 3 / 5 / 7 / 9 |
|---|---|---|---|---|
| Clip 1 (reference) | – | – | 2.16 | 4.65 |
| Plain chain | 9.91 | 12.99 | 2.47 | 5.25 / 5.77 / 5.99 / 6.32 |
| Linear 1 | 8.38 | 8.42 | 2.33 | 4.93 / 4.96 / 5.16 / 5.21 |
| **Linear 1 + Anti-Burn 0.6** | **7.85** | **7.94** | **2.08** | **4.65 / 4.53 / 4.63 / 4.56** |

- **Anti-Burn is the best run of the whole sweep on both measures.** Colour change is about 0.5 lower than linear 1, and grit drops below clip 1's own level.
- **The full-frame grit stays flat at clip 1's level** (4.5–4.65) all the way to clip 9. Without the LoRA it creeps up from 4.9 to 5.2; the plain chain climbs to 6.3.
- The later clips of test 11 (the old prompts, which cut to new places) are also lower: full-frame grit at clips 12 / 16 / 20 is 6.58 / 5.85 / 4.69, against 6.82 / 6.57 / 5.16 without the LoRA.

## Test 12: turning into new places (clips 10–20)

Full-frame grit:

| Clip | 10 | 12 | 14 | 16 | 18 | 20 | Average |
|---|---|---|---|---|---|---|---|
| No LoRA | 6.17 | 5.35 | 7.07 | 6.29 | 5.91 | 4.65 | 5.91 |
| **Anti-Burn 0.6** | **5.73** | **4.26** | **6.60** | **5.41** | **5.53** | **4.27** | **5.30** |

- **Anti-Burn is lower at every clip measured,** by about 0.6 on average (roughly 10%).
- The clip-to-clip swings (4.3 to 7.1) come from the places themselves: a crystal cave or jungle has more fine detail than a lavender field. Compare the two runs at the same clip, not clips against each other.

## roycho's verdict (by eye)

Watching the videos, the Anti-Burn LoRA eliminated virtually all remaining colour drift. The measured numbers below don't show that, but they're from a moving shot, where the camera showing different parts of the scene gets counted as colour change, so they're not reliable for colour. roycho's eye is the verdict for now.

**Next:** a few 30-clip tests to gauge exactly how much the LoRA affects quality and what the best strength is. The notes get updated after that.

## Colour and contrast drift, clip 1 to clip 9 (whole frame)

How much of the plain chain's drift each setup takes care of, measured on the whole frame at clip 9 (plain chain = 100% of the drift):

| Setup | Colour drift taken care of | Contrast drift taken care of |
|---|---|---|
| Plain chain | 0% | 0% |
| Degradation protection (context noise linear 1 + keystone + capstone) | 40% | 63% |
| Degradation protection + Anti-Burn LoRA 0.6 | 36% | 93% |

- **Contrast (burn):** the LoRA keeps it almost exactly at clip 1's level.
- **Colour:** almost all the colour change is the picture getting darker. The beach is a moving shot, so some of that is the camera showing different parts of the scene, not drift. On this measure the LoRA doesn't add anything for colour.
- **More testing coming:** a locked-off shot (where the scene doesn't change, so any colour change really is drift), with and without the LoRA, to get colour numbers we can trust.

## Next test: is the LoRA's smoothing real?

Some people see a slight softening with the Anti-Burn LoRA. roycho's theory: the LoRA isn't causing it. Degradation normally shows up as burn (H3 over-sharpening as the chain drifts); the LoRA stops the over-sharpening, so the detail that's already been lost shows up as softness instead. The same degradation in a different form. Against that, Imagineer13 found the LoRA can over-smooth when it's too strong for the turbo ([his notes](../community_cc_by_sa/Imagineer13_H3_antiburn_notes.md)).

The test: a locked-off shot, the LoRA at 0.3, 0.6 and 1.0, measuring real fine detail and burn crunch separately (several texture sizes), against clip 1.
- If the softness follows the chain length whatever the strength, it's degradation.
- If it gets worse as the strength goes up, it's the LoRA.

**Result ([test 13](static_chair_lora_test.md)):** no smoothing at any strength. So the smoothing some people saw with this LoRA is suspected to be the detail loss that comes with normal degradation. Because the LoRA preserves colour, that loss showed up as smoothing instead of the usual crushed blacks and waxy skin we're all familiar with.

## Caveats

- One seed. Differences of about 1 in colour change are within noise; the grit gap on the beach is small but holds at every clip.
- After clip 9 the two test 12 runs drift apart and show slightly different versions of each place, so some of the grit gap there could be content, not burn. It goes the same way at all six clips, though.
- Grit measures fine texture. It can't tell burn from real detail like sand or leaves, which is why the reference is clip 1's own level.

## Motion (roycho's review)

- **Motion looks great in both turning runs,** with and without the LoRA. Anti-Burn doesn't hurt the motion.

## Takeaway

- Anti-Burn 0.6 on top of linear 1 eliminates virtually all remaining colour drift (by eye) and burn, without costing motion. 30-clip tests to come for quality and the best strength.
- **Later:** the colour result couldn't be replicated (see the update at the top).
