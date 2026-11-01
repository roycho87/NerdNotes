# Frame 0 Influence Test — Proposal

## Goal

Find out how much frame 0 (the first single-frame latent) affects a MiniMax H3 shot. We test two things: how closely frame 0 matches the video, and how good its quality is. This follows nikodemon's feedback. If detail turns out to be what matters most, the next step is to generate fresh replacements for frame 0.

## Base setup

- One reference image with the clown in it.
- Base clip: one 5-second generated chain. Its existing shots are not used as results; the front is snipped off and everything is regenerated.
- Held context: 22 frames from that clip (the minimum that still carries motion).
- Every run is a new 5-second generation from the same 21 held frames. The only change is frame 0. The control is regenerated too, using the original frame 0.
- Keep the seed, prompt, reference image, steps and settings the same for every run.

## Phase 1: explore effects on the control

Phase 1 is only for exploring: try several methods on the control image and see which, if any, have an impact. Make each effect as heavy as possible while the picture can still be made out.

- Effects: blur, very dark, green tint (a colour not in any image, so it can only come from frame 0), simulated degradation (made in Photoshop).
- Apply every effect through the same mask in Photoshop at the same opacity, so each one changes the same pixels.
- Check each against the original with the Difference blend mode to keep the amount of change about equal.

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

## Phase 3: frame 0 on a degraded continuation

Phase 3 swaps the Phase 2 frame 0 latents into a chain that is already degraded, to see whether frame 0 makes the degradation better or worse. Method still to be decided.

1. Pin only the first frame and prompt that the scene stays motionless, still and unmoving.
2. Continue it at least 10 times, until the latent is clearly degraded.
3. Swap in each frame 0 from the Phase 2 grid: the clean related and unrelated images, the in-betweens, and the degraded versions chosen in Phase 2.
4. Compare how degraded each result is with the degraded control.

## Kept the same in every run

- Prompt: the control prompt, unchanged. If it pulls runs back towards the control, that is a result.
- Reference image: the clown only. Everything else is prompted.
- Seed and audio.
- Encoding: the control frame is decoded and re-encoded like the others.

## What to look for

- Test A: does the changed element show up in the shot, and for how many frames?
- Test B: does the blur, darkness, green or degradation spread into the shot, or does the shot clean it up?
- Compare every run to the control side by side.

## Follow-up

A second workflow decodes every single-frame latent at each sampling step, to see when frame 0's influence kicks in.

## Open questions

- Should there also be a baseline run using an empty latent?

## Other ideas to test

- Solid colours placed in frame 1.
- Frame 1 completely denoised.
- Frame 0 as pure random noise.
- Frame 0 as an all-zero latent.
- Frame 0 from an unrelated image (landscape, no clown).
- Swap the single frame at the start of a later block instead of frame 0, to see if position matters.
