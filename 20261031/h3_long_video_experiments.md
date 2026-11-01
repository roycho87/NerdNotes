# MiniMax H3 long-video experiments — what worked and what didn't

Notes from roycho00087's (roycho87 on GitHub) experiments fighting degradation in chained MiniMax H3 videos in ComfyUI, Oct 2026. Everything here was tested on an RTX 5090, usually with the same turbo LoRA and sampler on every run so the results can be compared.

Results are what I saw in my own renders. Nothing here is a controlled benchmark.

---

## Background: how the H3 latent is laid out

- H3 packs frames into 17-frame blocks of **1 + 4 + 4 + 4 + 4** frames. The first latent of every block holds a single frame; the others hold 4 frames each.
- Single-frame latents sit at latent 0, 5, 10… In a 192-frame clip the last one is latent 55 (frame 187).
- If you cut held frames out mid-block, you lose the clean single-frame latent the model seems to use as its detail reference.
- Clip 1's latent 0 is the only truly clean one. A later shot's latent 0 is also a single-frame latent, but it has already drifted.
- The VAE cuts the video into 17-frame chunks **starting at frame 0** (0–16, 17–33…). The leftover piece is at the **end** of the clip, padded out (that's the "5" in 5+17k). So at the start of a shot, 17 frames is one clean block, and 5 or 22 also land on a latent edge (any 1+4n count works). What has to line up is where the held frames start in the previous clip: 243 − 39 = 204 = 12 × 17.
- A fully unmasked latent gets scrambled on the first step, so unmasking 17 held frames is basically the same as cutting them off ("drop 17").
- Masks between 0 and 1 in H3 tended to give double exposures for me (two faces, four hands) when swapping characters. Keeping masks binary on the latent grid (drozbay's MaskVidExperiments) worked much better.
- When frames are pinned at both ends, H3 leans toward the quality of whichever end has more held frames.

---

## ✅ What worked

### Clown Keystone (+ Late Lock / capstone)
- **Keystone** = clip 1's first single-frame latent (frame 0). It gets swapped into slot 0 of every shot and masked. With Clown Context in front, the shot's own latent 0 is put back after sampling.
- First test (plain Context chain): shots were still crisp at shot 5+. Degradation still happens, but about 75% slower.
- 10-clip color drift test: **6.4** with nothing, **3.9** with keystone front, **2.9** with the late-lock capstone added. That's about 40% less drift, then about 60% less.
- It works with **masked latent context frames**, not guides. The 21 (or 38) real held frames behind frame 0 carry the motion, and frame 0 mostly adds detail. Without masked context behind it, frame 0 starts steering the picture.
- **Capstone** = clip 1's last single-frame latent. Pinning it at the back from the start was a no-go: the character walked back to that position. Adding it **late** (from around step 4, with the "Late Lock" step selector) added detail without pulling the motion.
- Keystone + capstone worked a lot better than Clown Prophecy, and they're now the recommended fix in Clown Context Suite.

### Clowntext Hybrid Windows
- Built on kijai's H3 context windows (in ComfyUI core) and JalenBrunson's HybridWindows (Rolling Diffusion): early steps run window by window in order, with each overlap pinned to the previous window; the last steps are blended context windows.
- My addition: window 1 finishes once to get the keystone and capstone, then rewinds. The keystone goes in every window's first slot, and the capstone is tacked on from a chosen step. `finished_handoff` passes each window's fully finished frames on to pin the next window.
- Degradation test (1433 frames, roller coaster through space, 8 windows): plain hybrid windows degraded, while keystone + capstone held the detail. Video in the Clown Context Suite README ("clowns for picard").
- Made for long, high-motion shots. On a static shot it just overbakes detail.

### Clown Context (block-aligned)
- A Motion Context replacement that copies the previous shot's last frames in as whole blocks, so no block gets cut in half.
- Handles motion much better than Motion Context.
- **drop_first / drop_last (steps of 17)**: freeing the outer 17 frames lets motion carry instead of freezing. Without it, it basically only worked for talking heads.
- Good settings: 39 frames, drop_last 17 (22 frames held).

### Clown Anchoring
- Pins the next shot's real latents at the **end** of a shot, masked, aligned to blocks. It fixed the brightness flashing in Static Vlog even shots.
- Rules that worked:
  1. The anchor must be clean.
  2. There must be more anchor frames than context frames (with more context, H3 drifts toward the context).
  3. Context and Anchoring must never hold the same frame.

### "No frame held twice"
- When Context and the previous clip's Anchoring both held the same frames, quality swung good → bad → good. Those frames were never resampled, so their degradation carried forward.
- 73-frame no-overlap layout (Context 73 drop_first 51, Anchoring 39): by shot 5 only soft, gradual degradation, and no bounce.

### TaoMate whole-clip refresh + regen chain
- 0.4 → 0.8 refresh, then regen. By clips 6–7, barely any degradation, only slight color drift.
- Locking the TaoMate reference to one good shot helped more than using each clip's first frame.

### SeedVR2 on the anchor tail
- SeedVR2 (0.4, 2x, then downscaled back) on only the 39 anchor frames gave more face detail.
- Its flaws don't matter here because pinned anchor frames get cut from the final video. They only steer the regen.
- Downside: red gradients creep into the color, so it needs color matching.

### From-scratch clips + FL2V seam repair
- Rendering each clip fresh with Ref2V along a song, then using FL2V to blend the seams, worked as a proof of concept (10 clips × 10 s).

### Clown Prophecy Sampler (partly)
- It renders a clean ending for the shot first (Ollama rewrites the prompt to describe only the ending), then pins it as the anchor so H3 fills in between.
- It worked on full shots, but it's too inconsistent to call finished. It moved to its own repo, Clown-Prophecy-Experimental-.

---

## ❌ What didn't work

- **Upscale prep pass before continuation** (upscaler at 0.1 denoise, resized back): continuation degraded *faster*.
- **17-frame blend continuation chain** (audio splitter → continuation → combiner): no real gain, so I dropped it.
- **Plain Clown Context chain alone**: still cloudy by shot 8. Conclusion at the time: you can't make a fresh latent that's exactly the same as another one, so copying generated frames forward always drifts.
- **Chaining backwards**: degrades the same way as forwards. That suggests the cause is copying generated frames forward, not the reference pictures.
- **Clown Ripple Repair** (render forward, make a clean ending, repair backwards with a growing anchor): degradation showed up in the middle of the stitched video and seemed to carry backwards. Intentionally degrading the context frames (so H3 "snaps clean" toward the anchor) looked promising at first but didn't save it.
  - Lesson: H3 only snaps to the clean anchor when the held frames are *much* worse than it. Subtle degradation gets followed.
- **Clown ControlNet** (depth control on only the first ~5 frames of the Prophecy ending): failed experiment for now.
- **Capstone pinned at the back from step 0**: the character returns to the capstone's position.
- **Clown Context Windows schedules** (my earlier windowed node):
  - diagonal: flips at the end
  - context motion: best motion, but degrades
  - reverse / center-first: just moves the degradation somewhere else
  - brick (windows shifted every other step): smooth continuous motion, "really good"
- **Windowed H3 with no reference**: the character drifts and moving backgrounds change a lot.
- **Clown Bookend** (put the capstone after filler latents so it lands in a single-frame slot): idea cancelled for now, not tested.

---

## 📚 Research that backs the frame 0 idea

These papers point to the first frame acting as an "anchor" that the rest of the video leans on. They were tested on other models (mostly Wan), not H3, so treat them as hints, not proof.

- **Analysis of Attention in Video Diffusion Transformers** (2025): found "attention sinks". Some attention heads keep looking at the same tokens, mostly in the first frame, no matter what's happening in the video. https://arxiv.org/abs/2504.10317
- **VMonarch: Efficient Video Diffusion Transformers with Structured Attention** (CVPR 2026): a speed-up paper that has to treat the first frame specially because so much attention goes to it. https://arxiv.org/abs/2601.22275
- **AdaState: Self-Evolving Anchors for Streaming Video Generation** (2026): long streaming video where the anchor updates as the video goes, instead of staying fixed to the first frame. Close to the "redraw frame 0 per shot" idea below. https://arxiv.org/abs/2605.30349
- **Adaptive Begin-of-Video Tokens** (2025): learned begin-of-video tokens that act as the video's anchor. https://arxiv.org/html/2511.12099v1
- **Pathwise Test-Time Correction for Autoregressive Long Video Generation** (2026): fixes drift by adding a few correction steps during sampling that use only the first frame as the reference. https://arxiv.org/html/2602.05871v1
- **Rolling Diffusion** / JalenBrunson's **ComfyUI-HybridWindows**: chained early steps + blended final steps. The base of Clowntext Hybrid Windows.
- **Banodoco:** others (Hevi, VK, QuintForms) clean up keyframes in an image model (Z-Image at 0.25–0.5) and run H3 again with them. Same goal as the keystone, but the fix is done outside H3.

---

## 💡 Tips from seitanism

seitanism (ComfyUI-H3-Motion-Context-MultiRef) has done a lot of H3 chaining work. The most useful points:

**Seams and overlap**
- Use a context length of **39, 90 or 141** frames, or audio and video won't both line up on the timeline.
- To hide the seam: latent-mask the held frames, feather the audio mask, linear-blend the overlap, and make the extension clip's video the one that wins.
- His layout: 39 context frames, **22 protected and 17 feathered out**, then a linear blend over the 22 protected frames. (Very close to my drop_last 17.)
- The audio VAE mangles the end of the audio, so he fades the noise mask over the last ~8 audio ticks so the new audio smooths over them.
- If you drop the overlap frames entirely, you can't blend or crossfade anymore, because you need the frames from both clips. The small color/brightness shift then becomes visible.
- Chained Linear Blend nodes eat RAM fast: each one in the chain stores all the frames before it.

- **Temporal mask feathering** (mask 1 on the held block, then 0.9, 0.8… stepping down one latent at a time): for video it gave consistency problems when the half-masked frames were kept, and did nothing when they were dropped. For **audio** it worked: feathering the audio noise mask toward the end fixed a subtle crackle at the seams.

**Masks and latents**
- A fully unmasked latent is basically random noise. It gets scrambled on the first step.
- Two ways to pin a keyframe: put it in the conditioning row (like Add Guide, re-injected after every step, but early on the model sees it as near-noise and it guides less), or encode it straight into the video latent with noise mask 0 (never denoised).
- ComfyUI rounds mask values above 0.996 up to 1. His granular fractional mask node keeps values like 0.9995.
- You can't cleanly mask one frame that's packed with 3 other frames into a 4-frame latent. After the VAE encode/decode it won't come back as the same frame. (This is why the single-frame latents matter for the keystone.)

**Anchors and degradation**
- A fixed anchor makes it snap back to the anchor instead of degrading, but that kills continuous moves. A 40-second slow zoom-out from a pier to a whole valley can't work if every clip is pulled back to the same frame. Same problem with odd/even (1-3-5 then 2-4) ordering: you don't know where clip 3 starts before clip 2 exists.
- Reference images don't help much with degradation in his experience.
- He hasn't really seen degradation *inside* a single clip of 15 s or less. It's the chaining.
- If the model has to match a bad-quality reference, it'll degrade to match.

**Quality and speed**
- Quality goes up pretty much linearly with resolution up to about 1440p, then the returns shrink and eventually it gets worse.
- er_sde gives good motion but shifts color grading (oversaturated or bright). Slicing in some euler steps at the start and end helps.
- Of the turbo LoRAs, pika H3 flex held up best at 6 steps for both video and audio.
- For upscaling a long video in chunks, each chunk needs the previous *upscaled* chunk's last 39 frames as fixed context, or it won't know how it's supposed to look. The upscale pass always adds some random detail.
- Latent upscale + resample at a higher resolution always improves quality a lot, but it's finicky with extensions. His two-pass chain: run all the low-res first passes on their own, latent-upscale each one, then overwrite the first 39 frames of each second-pass latent with the last 39 frames of the previous clip's second pass. On a static rainbow-pattern test the upscale didn't change the background in an inconsistent way, though normal chaining degradation was still there.
- Using the upscaled pass as context for the next clip's low-res first pass didn't work. Downscaling a latent directly (3D upscaler node) didn't work either; decode → resize → re-encode does.

**Consistency**
- To keep a room consistent when the camera sweeps around, use a 90-frame context window that holds a full 360° turnaround, or a ~3-second turnaround clip as a reference video. Anything not in the context frames comes back looking different.

**Test tip (from both of us):** for degradation tests, use a static background with rainbow stripes and abstract shapes. The rainbow shows color shift and the shapes show detail drift.

---

## 🔗 Other community work

- **xyzDist – H3-LongTakeNoCuts** (https://github.com/xyzDist/H3-LongTakeNoCuts): a second refine pass at ~0.5–0.6 denoise to bring back detail and fix the waxy/burnt look, a frame blend between frames 22–44, and a keyframe mask so the start connects to the last shot. Degradation shows up around segments 5–6 with no cuts (the "photocopy effect": each segment treats the last one as ground truth). Shots with cuts don't have the problem. Static/half shots still drift toward more contrast and saturation.
- **NikoDemon80 – ComfyUI-H3-Motion-Context** (https://github.com/NikoDemon80/ComfyUI-H3-Motion-Context): the original clip-chaining nodes. Clown Chain is a copy of its Chain node. My frame 0 write-up is issue #58 there.
- **Code review of Clown Keystone** (clown_context_suite issue #5, by sandpies): pointed out that latent 0 was only put back when Clown Context held it, that the mask didn't reach the capstone or Late Lock, and that the capstone position assumes 17n+5-frame clips. That led to Keystone being simplified to always use clip 1.

---

## 🗣️ NikoDemon80's feedback on Keystone (Motion Context issue #58)

Niko tested the frame 0 trick and dug into why it works:

- **It works great for locked-off shots** (background never changes).
- **Most color drift he measured builds up *inside* each clip while it generates, not at the handover.** The capstone gives each shot a clean target to land on, which he thinks is why the back slot took it from 40% to 60%.
- **A frame doesn't just carry color.** It carries the whole picture: pose, camera angle, background, lighting. On a moving shot (an arc around the subject, or a walk with the background always changing), the model drags the shot back toward clip 1's frame.
- **Putting clip 1's frames outside the clip didn't get around it.** Placed after the clip, the model still steered toward it: on an arc shot, clip 4 ended almost exactly on clip 1's opening frame. Placed far before the clip, it barely helped color and made the joins worse. His conclusion: any real frame from clip 1 pulls content with it.
- **On the AdaState paper:** a clean first frame is the strongest thing the model can attend to, so it anchors the whole video and motion dies down. That's the same mechanism that makes Keystone work and fight moving shots. Their fix needs training (a hidden frame the model is fine-tuned to use) and is built for models that stream small chunks with a running memory. H3 makes the whole clip at once, so there's nowhere to put it without retraining. They're also after the opposite trade: more motion at the cost of some consistency.
- **Suggestion:** Keystone writes into frame 0, which is also the first frame Clown Context holds from the last shot. Using only the capstone at the back might keep most of the gain.
- He won't add it to Motion Context because it only helps one kind of shot, but called it a solid approach for locked-off content.

**My reply / where I disagree:**
- With masked context latents behind it, my side-by-sides so far show frame 0 only changing detail; motion and picture stay the same. Still needs more side-by-side testing.
- Keystone only borrows the first slot while sampling. The rest of the held tail is real, so motion carries, and the real tail frame goes back in afterwards, so the saved clip joins like a normal chain.
- When the scene changes, it just helps less, and it's still better than a degraded frame 0.
- Capstone only at the back is worth testing, but locking the capstone from step 0 pulled her pose back toward the source, which is why Late Lock only brings it in for the last few steps. The front keystone was the bigger win.

---

## 🔬 Current test: how much does frame 0 matter, and why?

Based on nikodemon's feedback (does frame 0 carry content as well as detail?), this test checks whether the keystone helps because of its **quality** (detail), its **relation to the video** (same scene and character), or both.

Setup: same workflow, same seed and prompt, keystone front only, capstone off. Only the image in frame 0 changes (VAE Encode → Keystone input).

| | Related to the video | Unrelated to the video |
|---|---|---|
| **High quality** | clip 1's clean frame 0 (the normal keystone) | a sharp, detailed image of something else |
| **Low quality** | a degraded frame from the same scene | a degraded image of something else |

Plus a baseline with no keystone.

What to watch:
- Detail and sharpness over the chain (does it hold, and for how long?)
- Whether an unrelated frame 0 bleeds into the content (colors, objects, layout)
- Whether a low-quality frame 0 makes things worse than no keystone at all

If the unrelated sharp image helps as much as clip 1's frame, detail is what matters. If only the related one helps, the content is doing the work. If an unrelated frame shows up in the picture (objects, colors, layout), that backs Niko's point that a real frame pulls content with it.

Worth adding: a moving shot (an arc or a walk) next to the locked-off one, since that's where Niko saw the pull, and a capstone-only run, which was his suggestion.

---

## 🧪 Open / untested

- Handoff on vs off in Clowntext Hybrid Windows, same seed. Not compared yet.
- Redrawing frame 0 in its own pass (unmask only latent 0, let H3 make a fresh first frame, use it for the continuation, then swap the old one back in).
- A trained converter from a 4-frame latent to a 1-frame latent. I found no prior work on this.

---

## Links

- Clown Context Suite (Keystone, Late Lock, Clowntext Hybrid Windows, Clown Context, Anchoring, Shot Counter, Clown Chain): https://github.com/roycho87/clown_context_suite
- Clown Prophecy (experimental): https://github.com/roycho87/Clown-Prophecy-Experimental-
- All workflows: https://github.com/roycho87/workflows
- Built on work by kijai (H3 context windows), JalenBrunson (HybridWindows), seitanism and nikodemon (Motion Context / MultiRef), xyzdist (FL2VA resampling), obvpm (odd/even shot order).
