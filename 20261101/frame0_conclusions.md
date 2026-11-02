# Frame 0 Tests — Running Conclusions

What the frame 0 tests have shown so far, MiniMax H3 in ComfyUI. Updated as tests finish. The test plan is in [frame0_influence_test.md](frame0_influence_test.md).

All numbers are from single runs. Small motion differences between runs move them a little, so read the direction, not the exact values.

## How things are measured

- Colour drift: average Lab colour of clip 10 against clip 1, over 10 frames about 20 frames from the end. Higher = more drift.
- Sharpness: how strong the fine edges are over the same frames. Clip 1 is 24.8. Higher = more over-sharpened, the "baked" look.
- Same workflow, seed and prompts for every chain (clown walk: 8-second clips, 9:16 at 0.5 MP, Clown Context 39 with drop last 17).

## 1. One clip isn't enough to see anything

Phase 1 swapped a blurred, dark, green or degraded frame 0 into a single clip. Using the Difference blend mode against the clean run, the only change was thin outlines on her from tiny motion differences. No blur, darkness or green spread into the video, and the background was identical.

Degradation builds up over many clips, so single-clip tests can't measure it. **All further tests use 10-clip chains.**

## 2. Frame 0 only matters while sampling, not when decoding

From reading the H3 VAE code:

- Every latent decodes to 4 frames. The first latent of each 17-frame block keeps 1 of them, so latents 0, 5, 10… each end up as one frame.
- The decoder works on 7 latents at a time (one block plus 2 overlap latents), and all 7 affect each other. Latent 0 is only in the first one, so it can only change how about the first 22 frames are decoded.
- The encoder handles each 17-frame block on its own, so nothing carries across blocks when encoding.

The keystone and capstone gains were measured near the end of clip 10, far outside latent 0's reach in the decoder, so they come from sampling.

## 3. The capstone works just as well after it was taken out of the clip

The first Late Lock version wrote the capstone into the clip's last single-frame slot, where it got decoded. The current version only tacks it on during sampling, cuts it off, and puts the real frame 0 back. Same workflow, rerun for 10 clips:

| Run | Colour drift | Sharpness |
|---|---|---|
| No Keystone (baseline) | 6.83 | 49.2 |
| Keystone + capstone, old (capstone decoded in the clip) | 3.39 | 38.9 |
| Keystone + capstone, current (capstone cut off) | 3.35 | 40.0 |

![Clip 10, about 20 frames from the end: no Keystone, old capstone, current capstone](media/capstone_retest_clip10.png)

![Clip 10 playing side by side: no Keystone, old capstone, current capstone](media/capstone_retest_clip10.gif)

Full-quality video: [capstone_retest_clip10.mp4](media/capstone_retest_clip10.mp4)

Decoding the capstone added nothing. **The current version is confirmed:** about half the colour drift and much less over-sharpening, with nothing from clip 1 left in the clips.

## 4. A steady frame 0 matters most for colour; a clean one matters for sharpness

Phase 1b: the top row of the grid as 10-clip chains. The frame 0 each chain used, on every clip from 2 to 10 (all at full strength):

![The five frame 0 pictures: clean, blur, dark, green tint, degrade](media/frame0_pictures_used.png)

Clip 10 of each:

| Chain (keystone front only) | Colour drift | Sharpness |
|---|---|---|
| No Keystone (baseline) | 6.83 | 49.2 |
| Frame 0 clean | 5.17 | 47.5 |
| Frame 0 blur | 5.37 | 63.6 |
| Frame 0 dark | 6.02 | 65.4 |
| Frame 0 green tint | 5.27 | 59.6 |
| Frame 0 degrade | 5.68 | 58.5 |
| Keystone + capstone (clean, for comparison) | 3.35 | 40.0 |

![Clip 10 of each chain, about 20 frames from the end](media/frame0_chains_clip10.png)

![Clip 10 of every chain playing side by side: no Keystone, clean, blur, dark, green, degrade, keystone + capstone](media/frame0_chains_clip10.gif)

Full-quality video: [frame0_chains_clip10.mp4](media/frame0_chains_clip10.mp4)

- **Colour:** even a heavily blurred or green frame 0 holds the colour almost as well as a clean one, and still beats no Keystone. Having the same fixed frame 0 in every clip seems to be what stops the colour drifting, not how good that frame looks.
- **Sharpness:** a bad frame 0 makes the chain bake much harder: about 58–65 by clip 10, against 47.5 with a clean one. All four bad versions (blur, dark, green, degrade) did it. So frame 0's quality does matter, for the over-sharpened look.
- **The capstone does more than half of the colour work:** keystone alone 5.17, keystone + late capstone 3.35.

## 5. Moving held frames forward is clean; the drift comes from inside each clip

When Clown Context (or Motion Context) carries the held frames over on a block edge, each single-frame latent lands in a single-frame slot. Nothing is lost or squished in the move. The drift comes from each clip adding small errors while it samples, which are then in the tail that gets carried forward.

## 6. Redrawing the held single-frame latents ("Fresh Singles")

Idea: the held tail carries more than one single-frame latent (with 22 held frames: latents 0 and 5, frames 0 and 17), and each one is a full picture copied forward from the drifted clip. Clown Fresh Singles unlocks just those while sampling, so the model draws fresh ones, and puts the real ones back afterwards. The 4-frame latents stay locked and carry the motion. No Keystone, no capstone. Same walk chain, seed and prompts as everything above.

**Correction:** my first number for this run (whole-frame colour drift 3.18) made it look as good as keystone + capstone. Looking at the clips, it isn't: the floor is as blotchy as the no-Keystone run. The whole-frame average hides that, because brightness differences in different parts of the frame cancel out. So I re-measured by region, clip 10 against clip 1:

| Run | Floor colour drift | Floor blotchiness (clip 1 = 6.1) | Ceiling colour drift | Whole-frame drift (first measure) |
|---|---|---|---|---|
| No Keystone (baseline) | 12.45 | 10.81 | 6.43 | 6.83 |
| Keystone only | 8.03 | 10.71 | 5.25 | 5.17 |
| Fresh Singles (latents 0 + 5) | 9.81 | 10.64 | 3.46 | 3.18 |
| Keystone + capstone | **5.34** | **8.63** | **3.08** | 3.35 |

(Floor = the bottom 28% of the frame, below her feet. Blotchiness = how much the floor's brightness varies. Ceiling = the top 8%.)

![Clip 10, about 20 frames from the end: no Keystone, keystone only, Fresh Singles, keystone + capstone](media/fresh_singles_clip10.png)

![Clip 10 playing side by side](media/fresh_singles_clip10.gif)

Full-quality video: [fresh_singles_clip10.mp4](media/fresh_singles_clip10.mp4)

- **Fresh Singles helps some colour** (floor 9.81 vs 12.45, ceiling 3.46 vs 6.43), but it does **not** stop the floor going blotchy, which stays at the no-Keystone level.
- **Keystone + capstone is clearly the best** on every regional measure.
- The whole-frame average used in sections 3 and 4 is too crude on its own. From here on, results use the regional measures; the earlier tables still show the right order between runs, but the gaps are better read from the regional numbers.

Running next: Keystone + Fresh Singles on latent 5, then keystone + capstone + Fresh Singles on latent 5.

## Running now

The full grid as 10-clip chains, in the studio scene the grid pictures came from (clown, red background, dog), so the rows really are "more and more different from the video":

- 5 rows (control, blue background, child instead of dog, close-up, all three) x 5 effects (clean, blur, dark, green tint, degrade) = 25 chains.
- Clip 1 is the studio control clip, shared by every chain. Shots 2–10 use the control prompt and that chain's frame 0 through Keystone (front only, capstone off). 5-second clips.
- 225 renders, about 10 hours. The top row is redone here so the rows can be compared within one scene.
