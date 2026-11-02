# Frame 0 Tests — Running Conclusions

What the frame 0 tests have shown so far, MiniMax H3 in ComfyUI. Updated as tests finish. The test plan is in [frame0_influence_test.md](frame0_influence_test.md).

All numbers are from single runs. Small motion differences between runs move them a little, so read the direction, not the exact values.

## How things are measured

Every chain is measured on clip 10, averaged over 10 frames about 20 frames from the end, against clip 1 at the same frames. Only two patches the woman never walks into are measured, because she stands in a different place in every run:

![The two measured patches on clip 1: wall (yellow) and floor (cyan)](media/measure_regions.png)

- **Wall colour drift** (yellow patch, the wall on the right behind her): how far its average colour has moved from clip 1. Higher = more colour drift.
- **Floor sharpness** (cyan patch, bottom-right floor): how strong the fine grit and edges are. Clip 1 is 3.3. Higher = more over-sharpened, the "baked" look.
- **Floor blotchiness** (same patch): how much the floor's brightness varies. Clip 1 is 5.0. Higher = blotchier.
- Same workflow, seed and prompts for every chain (clown walk: 8-second clips, 9:16 at 0.5 MP, Clown Context 39 with drop last 17).

An earlier version of these notes averaged the whole frame. That mixed in the woman and let different parts of the frame cancel out, so all tables below have been re-measured on these two patches.

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

| Run | Wall colour drift | Floor sharpness | Floor blotchiness |
|---|---|---|---|
| No Keystone (baseline) | 3.83 | 29.2 | 10.06 |
| Keystone only (no capstone) | 3.47 | 28.4 | 8.54 |
| Keystone + capstone, old (capstone decoded in the clip) | 2.52 | 22.0 | 6.64 |
| Keystone + capstone, current (capstone cut off) | 2.32 | 21.8 | 7.59 |

![Clip 10, about 20 frames from the end: no Keystone, keystone only, old capstone, current capstone](media/capstone_retest_clip10.png)

![Clip 10 playing side by side: no Keystone, keystone only, old capstone, current capstone](media/capstone_retest_clip10.gif)

Full-quality video: [capstone_retest_clip10.mp4](media/capstone_retest_clip10.mp4)

Keystone alone only helps a little; adding the capstone is what makes the big difference. Decoding the capstone added nothing. **The current version is confirmed:** less colour drift, a smoother floor and much less over-sharpening, with nothing from clip 1 left in the clips.

## 4. A bad frame 0 makes the chain bake much harder

Phase 1b: the top row of the grid as 10-clip chains. The frame 0 each chain used, on every clip from 2 to 10 (all at full strength):

![The five frame 0 pictures: clean, blur, black, green tint, degrade](media/frame0_pictures_used.png)

Clip 10 of each:

| Chain (keystone front only) | Wall colour drift | Floor sharpness | Floor blotchiness |
|---|---|---|---|
| No Keystone (baseline) | 3.83 | 29.2 | 10.06 |
| Frame 0 clean | 3.47 | 28.4 | 8.54 |
| Frame 0 blur | 2.64 | 46.6 | 10.14 |
| Frame 0 black (the "dark" effect came out nearly black) | 4.13 | 57.7 | 9.62 |
| Frame 0 green tint | 4.13 | 54.6 | 10.84 |
| Frame 0 degrade | 3.97 | 63.7 | 9.43 |
| Keystone + capstone (clean, for comparison) | 2.32 | 21.8 | 7.59 |

![Clip 10 of each chain, about 20 frames from the end](media/frame0_chains_clip10.png)

![Clip 10 of every chain playing side by side: no Keystone, clean, blur, dark, green, degrade, keystone + capstone](media/frame0_chains_clip10.gif)

Full-quality video: [frame0_chains_clip10.mp4](media/frame0_chains_clip10.mp4)

- **Sharpness:** this is the clear result. Every bad frame 0 (blur, black, green, degrade) made the floor bake far harder by clip 10: 47–64, against 28 with a clean frame 0. So frame 0's quality matters, and it shows as the over-sharpened look.
- **Colour:** a clean frame 0 on its own only nudged the wall colour (3.47 vs 3.83), and the bad versions landed on both sides of it. Frame 0 alone isn't what holds the colour.
- **The capstone does most of the work:** keystone + capstone has the least wall drift, the least over-sharpening and a smoother floor than keystone alone.

## 5. Moving held frames forward is clean; the drift comes from inside each clip

When Clown Context (or Motion Context) carries the held frames over on a block edge, each single-frame latent lands in a single-frame slot. Nothing is lost or squished in the move. The drift comes from each clip adding small errors while it samples, which are then in the tail that gets carried forward.

## 6. Redrawing the held single-frame latents ("Fresh Singles")

Idea: the held tail carries more than one single-frame latent (with 22 held frames: latents 0 and 5, frames 0 and 17), and each one is a full picture copied forward from the drifted clip. Clown Fresh Singles unlocks just those while sampling, so the model draws fresh ones, and puts the real ones back afterwards. The 4-frame latents stay locked and carry the motion. No Keystone, no capstone. Same walk chain, seed and prompts as everything above.

Two more versions were run with Keystone on: clip 1's clean frame 0 in latent 0, and only latent 5 redrawn (and put back afterwards). One with keystone front only, one with keystone + capstone.

| Run | Wall colour drift | Floor sharpness | Floor blotchiness |
|---|---|---|---|
| No Keystone (baseline) | 3.83 | 29.2 | 10.06 |
| Keystone only | 3.47 | 28.4 | 8.54 |
| Fresh Singles (latents 0 + 5 redrawn) | 3.72 | 19.4 | 9.33 |
| Keystone + Fresh Singles on latent 5 | 2.33 | 31.9 | 9.16 |
| Keystone + capstone | 2.32 | 21.8 | 7.59 |
| **Keystone + capstone + Fresh Singles on latent 5** | **1.40** | **15.9** | **7.26** |

My first number for Fresh Singles (whole-frame drift 3.18) made it look as good as keystone + capstone. Looking at the clips, it isn't: its floor is nearly as blotchy as the no-Keystone run. That's what led to measuring only the two patches above.

![Clip 10, about 20 frames from the end: no Keystone, keystone only, Fresh Singles, keystone + capstone](media/fresh_singles_clip10.png)

![Clip 10 playing side by side](media/fresh_singles_clip10.gif)

Full-quality video: [fresh_singles_clip10.mp4](media/fresh_singles_clip10.mp4)

The Keystone versions next to keystone only and keystone + capstone:

![Clip 10, about 20 frames from the end: clip 1, no Keystone, keystone only, keystone + fresh 5, keystone + capstone, keystone + capstone + fresh 5](media/fresh5_clip10.png)

![Clip 10 playing side by side: no Keystone, keystone only, keystone + fresh 5, keystone + capstone, keystone + capstone + fresh 5](media/fresh5_clip10.gif)

Full-quality video: [fresh5_clip10.mp4](media/fresh5_clip10.mp4)

- **Fresh Singles alone** gave the least floor grit of any run (19.4), but the wall colour barely moved and the floor stayed blotchy.
- **Keystone + Fresh Singles on latent 5** matched keystone + capstone on wall colour (2.33 vs 2.32), the best colour of anything without a capstone, but the floor still baked and stayed blotchy.
- **Keystone + capstone + Fresh Singles on latent 5 is the best so far, on all three patches**: the wall drifted about 40% less than keystone + capstone (1.40 vs 2.32), the floor baked less (15.9 vs 21.8) and it was a little less blotchy (7.26 vs 7.59).
- So latent 5 matters too: copying it forward from the drifted clip adds drift, and redrawing it on top of the keystone and capstone removes some of that.

## 7. New direction: Step Echo (held frames at the same noise level as the new ones)

Reading the H3 code showed something about how held frames are fed in: at every step, the frames Clown Context holds go to the model almost clean (0.999), even on step 1 when the new frames are still pure noise. The idea: the drift may come from the model copying finished held frames that aren't forming along with the noise around them.

**Clown Step Echo** (private pack) saves the clip's latent at every one of the 8 steps. On the next clip, at each step the held frames are swapped for the same frames at the same step of the last clip, so they sit at the same noise level as the new frames and "grow" with them, like the overlap in context windows. The real held latents are put back after sampling, so the join lines up.

Prior work: the closest is Lumifel on Banodoco (20 Aug 2026), who added the sampler's current noise to the held context each step, fading it out near the join. He said it removed the burn-in and artifacts past 60 s but didn't fully fix the colour drift. Step Echo uses the last clip's real in-between latents instead of fresh noise on finished frames. Nothing found that replays the saved steps.

First run: the plain walk chain (no Keystone), Step Echo from shot 3 (clip 1 was rendered before Step Echo existed, so shot 2 only recorded).

| Run | Wall colour drift | Floor sharpness | Floor blotchiness |
|---|---|---|---|
| No Keystone (baseline) | 3.83 | 29.2 | 10.06 |
| Keystone + capstone + Fresh Singles on latent 5 | 1.40 | 15.9 | 7.26 |
| **Step Echo** | 7.35 | **14.2** | **3.74** |

![Clip 10: clip 1, no Keystone, keystone + capstone + fresh 5, Step Echo](media/step_echo_clip10.png)

![Clip 10 playing side by side: no Keystone, keystone + capstone + fresh 5, Step Echo](media/step_echo_clip10.gif)

Full-quality video: [step_echo_clip10.mp4](media/step_echo_clip10.mp4)

Every clip of the Step Echo chain, about 20 frames from the end:

![Step Echo chain, clips 1 to 10](media/step_echo_every_clip.png)

- **No degradation.** The floor is the cleanest of any chain by clip 10, with no baking and no blotches. It looks like a fresh generation.
- **But it doesn't hold the scene.** The framing jumps in and out from clip to clip, the ceiling lights come and go, and by clip 10 the wall behind her reads more like a door. The wall colour number (7.35) is high because the wall itself changed, not because of colour drift.
- Why: on H3's schedule the saved steps sit at noise levels of about 1.0, 0.99, 0.98, 0.96, 0.92, 0.86, 0.72 and 0.42, so for most of the render the echoed held frames are still mostly noise. The model can't copy them, which stops the baking, but it also has almost nothing to hold the scene to.
- This points to the drift coming from the model copying finished held frames.

Running next (ahead of the rest of the grid):

- Step Echo with the first held frame (frame 0 of the held context, from the last clip) kept locked and finished as an anchor, and the other held frames echoed. No Keystone.
- With a new noise option, FreeNoise: the held frames get the last clip's exact starting noise, so the noise print runs straight across the join, and the new frames reuse that noise reshuffled instead of fresh random noise.

## Running now

After the Step Echo runs, the grid continues:


The full grid as 10-clip chains, in the studio scene the grid pictures came from (clown, red background, dog), so the rows really are "more and more different from the video":

- 5 rows (control, blue background, child instead of dog, close-up, all three) x 5 effects (clean, blur, dark, green tint, degrade) = 25 chains.
- Clip 1 is the studio control clip, shared by every chain. Shots 2–10 use the control prompt and that chain's frame 0 through Keystone (front only, capstone off). 5-second clips.
- 225 renders, about 10 hours. The top row is redone here so the rows can be compared within one scene.
