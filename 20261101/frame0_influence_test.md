# Frame 0 Influence Test — Proposal

## Goal

Find out how much frame 0 (the first single-frame latent) affects a MiniMax H3 shot. We test two things: how closely frame 0 matches the video, and how good its quality is. This follows nikodemon's feedback. If detail turns out to be what matters most, the next step is to generate fresh replacements for frame 0.

## Base setup

- One reference image with the clown in it.
- Base clip: one 5-second generated chain. Its existing shots are not used as results; the front is snipped off and everything is regenerated.
- Held context: 22 frames from that clip (the minimum that still carries motion). Frame 0 is the one that gets swapped; the other 21 stay the same in every run.*
- Every run is a new 5-second generation from the same 21 held frames. The only change is frame 0. The control is regenerated too, using the original frame 0.
- Keep the seed, prompt, reference image, steps and settings the same for every run.
- Frame 0 goes in through Clown Keystone (keystone front only, capstone off). The real frame 0 is put back after sampling.*

## Phase 1: explore effects on the control

Phase 1 is only for exploring: try several methods on the control image and see which, if any, have an impact. Make each effect as heavy as possible while the picture can still be made out.

- Effects: blur, very dark, green tint (a colour not in any image, so it can only come from frame 0), simulated degradation (~~made in Photoshop~~ made with TrentNodes' Video Degradation*).
- ~~Apply every effect through the same mask in Photoshop at the same opacity, so each one changes the same pixels.~~ Every effect is applied to the whole frame by the Clown Frame 0 Test node, at full strength (max_strength 1.0).*
- ~~Check each against the original with the Difference blend mode to keep the amount of change about equal.~~ Each run's frame 0 picture is saved next to its video so the strength can be checked.*

## Phase 2: test grid (25 runs)

Pick the one effect from Phase 1 with the most impact, and run it at increasing strength to push the image as far as possible. Every content version of frame 0 is run at every strength.

| Content \\ Strength | Clean | Level 1 | Level 2 | Level 3 | Level 4 (strongest) |
| --- | --- | --- | --- | --- | --- |
| Control: clown, mid shot, red background, dog | Control run |  |  |  |  |
| Blue background |  |  |  |  |  |
| Child instead of dog |  |  |  |  |  |
| Close-up instead of mid shot |  |  |  |  |  |
| All three: close-up, blue background, child |  |  |  |  |  |

The clown stays the same and in the same position in every image, as far as possible.

**Changed to 10-clip chains.*** A single clip showed no measurable difference (see Phase 1b), so every cell is now a 10-clip chain instead of one clip, and every effect from Phase 1 is run on every row instead of one effect at increasing strength. The top row was run first, as Phase 1b.*

## Phase 1b: the top row as 10-clip chains*

Added after Phase 1.*

- Five chains: clean, blur, dark, green tint, degrade. All at full strength.*
- Each chain: clip 1 is shared (the same seed makes it identical), then shots 2–10, each with that chain's frame 0 through Keystone (front only, capstone off). Same seed, prompts and settings throughout (the clown walk workflow: 8-second clips, 9:16 at 0.5 MP, Clown Context 39 with drop last 17).*
- Measured on clip 10, averaged over 10 frames about 20 frames from the end, against clip 1 at the same frames:*
  - colour drift: distance between the average Lab colour of the two*
  - sharpness: how strong the fine edges are (higher than clip 1 = over-sharpened / baked)*
- Compared against the existing no-Keystone baseline and the keystone + capstone run of the same workflow.*

## Phase 3: frame 0 on a degraded continuation

Phase 3 swaps the Phase 2 frame 0 latents into a chain that is already degraded, to see whether frame 0 makes the degradation better or worse. Method still to be decided.

1. Pin only the first frame and prompt that the scene stays motionless, still and unmoving.
2. Continue it at least 10 times, until the latent is clearly degraded.
3. Swap in each frame 0 from the Phase 2 grid: the clean related and unrelated images, the in-betweens, and the degraded versions chosen in Phase 2.
4. Compare how degraded each result is with the degraded control.

## Later tests: how the held frames are fed in*

Added after the frame 0 tests.* Reading the H3 code showed that the frames Clown Context holds are handed to the model almost clean (0.999) at every step, even on step 1 when the new frames are pure noise. These tests change how the held frames are fed in. All are 10-clip chains of the same clown walk workflow (seed, prompts, Clown Context 39 / drop last 17). Results are in [frame0_conclusions.md](frame0_conclusions.md), section 7.*

**Step Echo** (node: Clown Step Echo). Save the clip's latent at every one of the 8 steps. On the next clip, at each step, swap the held frames for the same frames at the same step of the last clip, so they sit at the same noise level as the new frames. Put the real held latents back after sampling.*

1. Every held frame echoed, no Keystone. *(done)*
2. Frame 0 of the held context (from the last clip) locked and finished, the rest echoed, FreeNoise on. *(done)*
3. Both held single-frame latents (0 and 5) locked, the 4-frame latents echoed, FreeNoise on. *(done)*

FreeNoise (a Step Echo option): the held frames get the last clip's exact starting noise, so the noise print runs straight across the join; the new frames reuse that noise reshuffled instead of fresh random noise.*

**Context noise, after Lumifel's method** (node: Clown Context Noise). Lumifel (Banodoco, Aug 2026) gave the held context part of the sampler's current noise at every step, fading to none at the end of the context so the join stays clean. He reported no burn-in up to about 60 s, but the colour still drifted warmer. Here it's done with H3's partial masks: a mask value m puts a held latent at m × the current noise level, and the model is told so. The held latents ramp from 0.8 at the start of the held part to 0 at the join. The real held latents are put back after sampling.*

4. Context noise + Keystone (clip 1's clean frame 0 locked at latent 0, no capstone). The question: does a clean frame 0 fix the colour drift Lumifel had left? *(done)*
5. Context noise + Keystone + Late Lock capstone. *(done)*
6. Test 5 extended to 20 clips on the walk (clips 11–20 repeat the 10 walk prompts). *(done)*
7. Test 5 on a locked-off static shot (the red studio scene from the grid), 20 clips. *(done)*
8. Grid 2: the five clean frame 0 pictures (rows) × baseline / keystone + capstone / context noise / context noise + keystone + capstone (columns), studio scene, 20 clips each. Keystone and capstone from the row's picture; baseline and noise-only run once. *(done, see [conclusions, section 9](frame0_conclusions.md))*

Compared against: no Keystone, keystone only, keystone + capstone, keystone + capstone + Fresh Singles on latent 5, and the Step Echo runs.*

## Kept the same in every run

- Prompt: the control prompt, unchanged. If it pulls runs back towards the control, that is a result.
- Reference image: the clown only. Everything else is prompted.
- Seed and audio.
- Encoding: the control frame is decoded and re-encoded like the others.

## What to look for

- Phase 1: does the effect spread into the shot, or does the shot clean it up?
- Phase 2: does the changed content show up in the shot, for how many frames, and how does the effect strength change that?
- Phase 3: does each frame 0 make the degradation better or worse than the degraded control? Compare every run to the control side by side.
- Chains: does a bad frame 0 make the chain drift or bake faster by clip 10?*

## Follow-up

A second workflow decodes every single-frame latent at each sampling step, to see when frame 0's influence kicks in.

## Open questions

- ~~Should there also be a baseline run using an empty latent?~~ Added as its own run outside the grid: frame 0 unmasked (Keystone off), so H3 draws its own frame 0. Whatever is put in frame 0 doesn't matter once it's unmasked, so it's one run, not a row.*

## Other ideas to test

- Solid colours placed in latent 1 (frames 1–4; frame 1 can't be swapped on its own).*
- Latent 1 completely denoised.*
- Frame 0 as pure random noise.
- Frame 0 as an all-zero latent.
- Frame 0 from an unrelated image (landscape, no clown).
- Swap the single frame at the start of a later block instead of frame 0, to see if position matters.
- The other single-frame latents in the held context.* The held tail usually contains more than one single-frame latent (with Clown Context 39 / drop last 17: latents 0 and 5, frames 0 and 17). Keystone only replaces latent 0, so latent 5 is copied forward from the drifted tail every clip. Test whether it has an impact too: swap a clean or effected latent into slot 5 as well, or unlock it, and compare over a 10-clip chain.
- Redraw every single-frame latent in the held context, then put the real ones back.* While sampling, unmask latents 0, 5, 10… inside the held context so the model draws its own fresh versions instead of copying the drifted ones forward; the 4-frame latents stay locked and carry the motion. After sampling, the real single-frame latents are swapped back in (like Keystone / Late Lock does for frame 0), so the held part of the video is untouched and the join still lines up. Idea: a continuation where the only things copied from the last clip are the 4-frame latents might behave much more like a fresh generation. Compare over a 10-clip chain against the baseline, Keystone, and Keystone + capstone. Risk: the redrawn latents may just match the drifted frames around them. Prior work: nothing found on Banodoco or the web that unmasks the single-frame latents inside a held context; the closest is advice to put injected keyframes at frames 0, 17, 34… so they land on single-frame latents.

---

\* Addendum: changed or added after the proposal was written, as the test was run. Struck-through text is the original plan it replaces. Results and conclusions are in [frame0_conclusions.md](frame0_conclusions.md).
