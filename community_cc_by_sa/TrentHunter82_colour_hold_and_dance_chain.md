> **Contributor: TrentHunter82 (Flipping Sigmas)** ([TrentNodes](https://github.com/TrentHunter82/TrentNodes)). A summary of two of his test write-ups, shared with roycho on 2026-11-15: [Long Video Colour Hold](https://claude.ai/artifact/DrU8tCy4nCNhF9Ez8pRbwC) (31 Aug to 12 Nov 2026) and [H3 Dance Chain Findings](https://claude.ai/artifact/JCEdpADCNT7yUw79FduUrf) (12 to 14 Nov 2026). Licensed under CC BY-SA 4.0 at his request: if you share or adapt it, credit **TrentHunter82 (Flipping Sigmas)** as well as roycho / NerdNotes. All numbers below are his.

# TrentHunter82: holding colour down a long H3 chain, and a dance chain driven by a skeleton video

Both write-ups use one scene and one seed per test, so treat them as strong leads, not final results.

## 1. Long Video Colour Hold

**Goal:** a 10-clip (44 s) chain where the colours don't creep. Each clip is about 6 s and starts from the end of the clip before.

**How it's measured:**
- **Colour creep:** how much stronger the colours are at the end of clip 10 than at the end of clip 1. 0 is perfect; the chain with no fix scores +50.
- **Sky kept:** how bright the sky still is at the end, as a percent of the start.
- **Grain:** fine film grain at the start and at the end (a rising number means noise builds up).
- **Jump at the first join:** how far the red/blue balance moves in the one frame where clip 1 ends and clip 2 begins.

### The final recipe (run 16)

1. **Reference-to-Video for every clip.** The start picture is a reference the model looks at every time, so faces and clothes come from the original, not from the clip before.
2. **Clip Refiner** (from the noEmbryo pack) after each clip: it measures the colour and contrast of clip 1 and moves each later clip back to those values.
3. **First-frame pin:** each clip is anchored to the exact first frame of clip 1, passed along as a latent (no picture conversion). This is the keystone idea (roycho), built as Keystone Graft in TrentNodes.
4. **No timestamps inside a shot,** and a camera sentence that says the camera stays **at the same distance**.
5. **Colour match at each join:** Join Clips compares the last 4 frames of one clip with the first 4 new frames of the next, applies a full colour correction (all three colours together, so the sky stays blue while the skin is corrected) and fades it out over 2 seconds.
6. **Six steps with er_sde** for the draft (960 × 544), then a two-step pass that refines to 1920 × 1088 (Pika H3 Flex).

**Run 1 (no fix) → run 16:** colour creep +50.2 → +3.5, sky kept 53% → 92%, jump at the first join 21 → 0.1.

### The sixteen full-length runs (walking scene, 10 clips)

| Run | Change | Colour creep | Sky kept | Note |
|---|---|---|---|---|
| 1 | No fix | +50.2 | 53% | Grain 0.75 → 1.30 |
| 2 | No fix + first-frame pin | +41.7 | 59% | The drift travels through the 39 copied frames, so the pin alone has little to remove |
| 3 | Add the Clip Refiner | +14.4 | 81% | The biggest single gain; side effect: a colour jump at the first join |
| 4 | Clip Refiner + pin | +8.0 | 85% | Small extra gain |
| 5 | Let the Clip Refiner redraw fine detail (strength 0.4, 3 steps) | +24.7 | 76% | Worse: clip 1 soft and hazy, white clothes turned blue |
| 6 | Clip Refiner at strength 0.7 | +14.2 | 79% | Same as full strength |
| 7 | Reference-to-Video in place of Image-to-Video (both passes, matching LoRA, six-section prompts) | +7.2 | 87% | Halves the creep again |
| 8 | Reference-to-Video + pin | +3.2 | 90% | Flat from clip 4 to 10; least grain growth |
| 9 | Eight steps in place of six | +2.2 | 100% | Shot broke (sudden close-up in clip 5) |
| 10 | euler for the draft | +4.4 | 91% | Same cut in clip 5: the cause was the prompt |
| 11 | Prompts rewritten to the official format, timestamps removed | +11.5 | | Lesson: timestamps cause cuts |
| 12 | The new prompts with er_sde | +14.5 | 82% | No cut, but one of the prompt edits cost colour |
| 13 | Keep the fix, put the old camera sentence back ("glides backward… at the same distance") | +3.5 | 92% | No cut, and for the first time the grain doesn't build up. The official words "pulls out" had told the model the camera moves away, which changed the framing in every clip |
| 14 | Clip Refiner targets the end of clip 1 | +9.2 | 87% | Join jump fell by more than half, but creep and grain rose |
| 15 | Colour match at each join, first version | +3.6 | | The red/blue jump was fixed, but the frame after the join turned too orange |
| 16 | Colour match at each join, second version | +3.5 | 92% | **Winner so far.** Join jump 0.1; all nine joins cleaner |

Runs 9 and 10 only look good because the chain reset after the cut.

### What was tried before the full runs (and what didn't work)

- **One clip per queue, carried forward on disk** (31 Aug): the chain idea, using Video Helper Suite's Meta Batch Manager. Became the base of everything after.
- **Feeding the model's own output back as pictures:** dropped. Converting to pictures and back adds about 7 points of colour on every clip and removes grain.
- **Context noise** (from roycho's Geiru Context Suite) and **noise worked out backwards** (flip sigmas: run the old clip backwards through the sampler to find the noise that makes it, and start the copied frames from that noise): no clear difference in drift on a 4-clip fireplace test. The backwards noise was only about half as strong as real noise at any step count, because working backwards from a clean picture can only find the average noise.
- **The split method** (sample half the steps with the copied frames locked, swap in the unsampled old clip at the same noise level, then finish all frames together): looked like it halved the colour rise on the fireplace scene, but was overturned on 8 Nov. The woman moves nearer to the fire, so the colour changed with the content, not from drift. (Details: [TrentHunter82's split unsample test](../20261108/community/TrentHunter82_split_unsample.md).)
- **A final smoothing pass over the joined video** (Kijai's context windows): the joins became smooth, but the drift stayed, and the picture conversion alone added colour.
- **Hillside scene, 4 clips, every method ranked** (colour creep): pin on the split method 13.4, colour match to the start picture 16.3, keystone + colour match 19.3, noise worked out backwards 19.7, context noise 20.7, split method (half-finished copy) 21.7, plain chain 22.0, keystone 23.2, split method (backwards copy) 24.5. Only the pin and the colour match made a real difference.
- **The pin tested properly** (10 Nov, two seeds): it cut the drift by a third or more every time. Colour match added nothing on top.
- **A harder scene** (ten clips, two women stand up and walk, moving camera): the sky turned navy in every version. Plain chain +51.5, + anti-burn LoRA +47.9, + pin +42.8, split method + pin +43.0, pin + anti-burn LoRA +38.4. The pin slows the drift and the LoRA slows it a little but removes the film grain. Lesson: the drift travels through the 39 copied frames.

### Still open

- One scene and one seed; a second scene must confirm the recipe.
- Clip 1 is a little sharper than the later clips, so a small step in sharpness stays at the first join.

## 2. H3 Dance Chain Findings

**Goal:** a 44-second video of three androids that follows a real dance video frame by frame.

### The five main findings

1. **A skeleton video as Guide Latents gives exact motion.** The `MiniMaxH3AddGuide` node puts frame N of the guide on frame N of the clip; motion, camera and timing follow the dance.
2. **Two LoRAs can read the skeleton guide:** the Motion Transfer FFP LoRA and the Character Swap LoRA, both by Alissonerdx.
3. **The prompt must agree with the guide.** The prompt text decides where the scene comes from; a prompt that tells the model to keep the guide's background makes the room black.
4. **A colour matrix match removes the step at each join.** Join Clips compares four frames on each side of a join and fades the correction over 48 frames.
5. **One camera sentence controls the colour drift.** "At the same distance" holds the frame. Without it, saturation rose by 14 points over 10 clips.

### The test ladder (3 clips each, same seed, one change per run)

| Run | Motion input | Result |
|---|---|---|
| Start | Control net, thin skeleton video that includes the audience | Loose motion, rows of extra androids |
| A | Control net at 1.0, clean skeleton of the three dancers | Poses near; camera distance and body direction not exact |
| C | Guide = real dance video, plain Motion Transfer LoRA | Motion exact; picture goes into white fog |
| D | Same as C, pass 1 only | Same: the fog comes from pass 1 |
| E | Guide = real dance video, FFP LoRA, picture on guide frame 1 | Motion exact, look stays, but the real dancers' hair shows |
| F | Same as E, guide = skeleton video | Best: motion, camera and timing exact; bald androids, white studio |
| G | Same as F, joints from Sapiens (308 points), OpenPose drawing | Same precision; F's wrong arm is gone |
| H | Same as G, all 308 points drawn | No clear difference in clips 1–3 |
| I | Run G plus the control net | Worse: the skeleton drawing shows in the look |

**Why the skeleton beats the real video:** the guide frames are part of the clip, so the model copies what it sees in them. A real video has hair, clothes and a room, and they show in the result. A skeleton has only joints, so only the motion goes across.

**Making the skeleton video:** find the persons (a box per person per frame), track the 3 dancers through all 1059 frames, find the joints with Sapiens (308 points per dancer), draw coloured OpenPose limbs on black at 960 × 544, 24 fps. A tracker repair fixed a lost rear dancer in the floor section, and a hand filter removed stray finger points (163 of 1059 frames changed).

**Full runs (10 clips, 44 s):** motion follows in all 10 clips, floor section included. Brightness stays between 194.6 and 198.3 across the chain. Open fault: in clip 7 the front dancer hides the left dancer, the tracker jumps to a seated person in the audience, and the model draws a crouched android there. The model draws a body for every skeleton in the guide, so the repair is in the tracker, not the model.

**Character Swap LoRA on a pose map** (Alissonerdx's suggestion): with `body_swap:` and his own prompts it works (three androids, white studio, same poses). With the fixed `body_swap` prompt template, the room goes dark after clip 1 and the androids lose the face.

### Two recipes

| | FFP recipe (10 clips proven) | Swap recipe (3 clips only) |
|---|---|---|
| Add Guide, pass 1 | Skeleton video | Skeleton video |
| LoRA 1.0, pass 1 | Motion Transfer FFP | Character Swap v1.1 |
| Trigger | `vfx_edit:` | `body_swap:` |
| The look | Picture replaces guide frame 1 (clip 1 only); later clips get their first frames from the clip before | Picture in the reference input of each clip; the guide stays a skeleton in all frames |

### Rules

**Do:**
- Use a skeleton video as the guide, 960 × 544, 24 fps.
- Make the guide 141 + 102 × (clips − 1) frames long.
- Put the guide and the motion LoRA on pass 1 only.
- Describe the scene, the camera and the persons in the prompt.
- Keep "at the same distance" in the camera sentence.
- Set `match_colour_frames` to 48 on Join Clips.
- Look at the skeleton video before a long run.

**Don't:**
- Use a real video as the guide (hair and clothes show).
- Add the control net to the skeleton guide.
- Use the plain Motion Transfer LoRA in the chain (the picture goes to fog).
- Use the fixed `body_swap` template with a pose map.
- Write timestamps in a shot that has no cut.
- Let the prompt contradict the guide.

**Not checked:** a second seed; lip sync and beat match (they need a person who listens).

## How it lines up with NerdNotes

- Trent's first-frame pin is the keystone; in his chain it cut the drift by a third or more, and with Reference-to-Video it held creep at +3.2. Our [test 16](../20261107/beach30_protection_combos.md) and [test 19](../20261113/attention_sweep.md) found keystone (+ capstone, then + balance) does most of the work too.
- His context noise and flip-sigmas tests found no gain on colour, which matches [matlowai's](matlowai_h3_long_video_research.md) result (context noise adds nothing) and ours (context noise smooths the joins but doesn't hold the colour).
- The Clip Refiner is a colour correction after each clip toward clip 1, so it doesn't touch motion; it may fight a scene whose lighting is meant to change (not tested yet, by him or us).
- "Timestamps inside a shot cause cuts" and "the camera sentence changes the colour" are prompting findings worth checking against our own chains.
