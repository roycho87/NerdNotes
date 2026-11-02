# MiniMax H3: How Latents Work, and Where Long-Video Degradation Comes From

A reference for anyone chaining MiniMax H3 clips into long videos in ComfyUI. It covers:

1. [How H3 turns frames into latents](#1-how-h3-turns-frames-into-latents) (read from the ComfyUI H3 code)
2. [How the model samples, and what "held" frames really are](#2-how-the-model-samples-and-what-held-frames-really-are)
3. [What degradation looks like](#3-what-degradation-looks-like)
4. [Where degradation comes from](#4-where-degradation-comes-from)
5. [How continuations evolved](#5-how-continuations-evolved)
6. [xyzDist's method (H3-LongTakeNoCuts)](#6-xyzdists-method-h3-longtakenocuts)
7. [What people do about it today](#7-what-people-do-about-it-today)
8. [What the research papers say](#8-what-the-research-papers-say)
9. [Practical checklist](#9-practical-checklist)
10. [Sources](#10-sources)

Our own test results are in [../20261101/frame0_conclusions.md](../20261101/frame0_conclusions.md) and [../20261031/h3_long_video_experiments.md](../20261031/h3_long_video_experiments.md).

Everything about H3 internals below comes from reading ComfyUI's native H3 code (`comfy/ldm/minimax/vae.py`, `comfy/ldm/minimax/model.py`, `comfy/model_base.py`, `comfy/samplers.py`) as of late October 2026. If the code changes, details may change.

---

## 1. How H3 turns frames into latents

### The short version

- H3 works on **24 frames per second**.
- The VAE squeezes every **16 × 16 pixels** into one latent "pixel" with **24 channels**.
- It squeezes time by **4×**, but in a special pattern: it works in **17-frame blocks**, and each block becomes **5 latents** shaped **1, 4, 4, 4, 4** frames.
- So in every block, the **first latent holds a single frame**, and the other four each hold four frames.

```
frames:   0 | 1 2 3 4 | 5 6 7 8 | 9 10 11 12 | 13 14 15 16 | 17 | 18 19 20 21 | ...
latents:  0 |    1    |    2    |     3      |      4      |  5 |      6      | ...
          └──────────── block 0 (17 frames) ────────────────┘  └── block 1 ...
```

Single-frame latents sit at latent **0, 5, 10, 15, …** = frames **0, 17, 34, 51, …**

### Valid clip lengths

Because of the block pattern, clean clip lengths are **17n + 5 frames**, which give **5n + 2 latents**:

| Frames | Seconds (24 fps) | Latents |
|---|---|---|
| 22 | 0.92 | 7 |
| 39 | 1.63 | 12 |
| 56 | 2.33 | 17 |
| 124 | 5.17 | 37 |
| 192 | 8.00 | 57 |

Other lengths get padded: the encoder repeats the last frame to fill the last block, then drops 3 latents off the end (`token_drop = 3`).

### The encoder (pixels → latents)

- A **3D causal CNN**. "Causal" means a frame can only see frames before it, never after.
- It encodes each **17-frame block on its own**. Nothing carries across blocks while encoding.
- Inside a block, the first latent only has one real frame to look at. The time padding is zeros at the front, so latent 0 of each block is effectively a still picture of that one frame.
- Downsampling: space ×2 four times (= 16×), time ×2 twice (= 4×).
- Latents are normalized with a fixed per-channel mean and scale before the model sees them.

### The decoder (latents → pixels)

- A **ViT3D** (a transformer, not a CNN).
- Every latent decodes to **4 frames**. For the first latent of each block, 3 of the 4 are thrown away, leaving the single frame.
- It decodes in chunks of **7 latents** (one 5-latent block plus 2 overlap latents) and attention sees all 7 at once. So one latent only affects the frames around it, roughly the first ~22 frames for latent 0.
- Large frames are decoded in spatial tiles (256 px with 64 px overlap) and blended.

**What this means for chaining:** if you cut a clip on a block edge, the single-frame latents line up with single-frame slots in the next clip, and the 4-frame latents with 4-frame slots. Copying latents forward on block edges is clean. Cutting anywhere else forces a re-encode or a mismatch.

### The audio latent

- Stereo audio, **32 channels at 40 latent steps per second**, generated in the same pass as the video.
- Audio runs on its own noise schedule (shift 3.0, versus 12.0 for video) that's tied to the video's.

### The text side

- H3 reads text through **Qwen3-VL** (hidden states from layer 50).
- The model packs everything into one sequence: `[text | reference / condition rows | audio | video]`.

---

## 2. How the model samples, and what "held" frames really are

### Flow matching with a strong shift

H3 is a flow-matching model: each step moves the latent from noise toward a clean picture. The video schedule is **heavily shifted (12.0)**, so most steps sit at very high noise. With the common 8-step `er_sde` / `beta` setup we measured these noise levels:

```
1.00 → 0.99 → 0.98 → 0.96 → 0.92 → 0.86 → 0.72 → 0.42 → 0
```

So **the visible picture only really forms in the last two steps.** For six of eight steps, everything is mostly noise.

But NikoDemon80 logged the model's guess at the finished clip after every step and found **contrast and colour are already decided at step 0**, before there's any picture. Saturation is settled by step 1; only fine detail is added later, around steps 2–4. So the look of a clip is part of the model's plan from the start, and last-step tricks only reach detail. This held even for a clip with nothing pinned: clip 1 on its own gained 8–16 brightness levels over 124 frames. ([his notes, section 4](community/NikoDemon80_H3_drift_notes.md))

H3's turbo / PDD schedules are tuned for `er_sde` + `beta`; community tests found generic samplers degrade them.

### Held frames (masks) in H3

When you continue a clip, you "hold" some frames from the last clip: copy their latents into the new clip and mask them so the sampler keeps them. In H3 that works like this:

1. **Every step, the held frames are pasted back in almost clean**: 99.9% clean latent and 0.1% noise (`VISUAL_COND_TIMESTEP = 0.999`). This happens even on step 1, when the new frames are pure noise.
2. The model is told those rows are **finished** (their timestep label is pinned near "clean").
3. After each step, whatever the model predicted for the held rows is thrown away and the original is pasted back.
4. Partial masks work: a mask value `m` puts that row at `m × current noise`, so a half mask means half the noise level.

So H3 sees **finished frames sitting next to pure noise for most of the render**. It was trained this way for keyframes and references, so it's "correct". But it means the model leans on the held frames very hard, and copies whatever is in them.

### Guides versus masked latents

There are two ways to carry frames forward:

- **Guides / keyframes** (Motion Context, Add Guide): the held frames go in as extra conditioning rows, not as part of the video being sampled.
- **Masked latents** (Clown Context, seitanism's masked context nodes): the held latents are copied straight into the start of the new video and locked with a mask.

Both feed the model clean, finished frames.

---

## 3. What degradation looks like

When you chain clips without cuts, quality slips a bit with every clip. People describe it as:

- **Burnt / fried / baked:** too much contrast, over-sharpened grit, crunchy textures.
- **Colour drift:** saturation creeps up or flattens out; the picture gets warmer, or darker, clip by clip.
- **Waxy skin** and lost fine detail on faces.
- **Blotchy surfaces:** floors and walls go mottled.
- **Washed out** by the end of a long chain: flatter colour range and lower sharpness.

When it shows up:

- Usually noticeable around **clip 3–6** (about 30–45 s). Several people report about **30 s** clean with good setups.
- **Static and half-static shots are worst.** Moving shots hide it better because the background keeps getting replaced.
- **Hard cuts reset it.** A new shot with a new camera angle starts fresh.

---

## 4. Where degradation comes from

Several causes stack. Here's what we know, roughly from best-supported to more speculative.

### 4.1 Each clip adds a little error, and the next clip copies it

This is the core of it, and the papers call it **exposure bias** or **error accumulation**. The model was trained to continue from clean, real video. In a chain it continues from its **own** output, which has small errors. It treats those errors as truth, adds its own, and passes everything on. xyzDist calls it the **"photocopy effect"**: a copy of a copy of a copy.

Where the drift happens depends on the shot (NikoDemon80, [his notes, section 3](community/NikoDemon80_H3_drift_notes.md)):

- **Moving shots:** 85–95% of contrast and 88–97% of saturation drift builds up **inside each clip** while it generates. The handover just passes it on.
- **Locked-off shots with a subject doing something:** a lot of it is a **step at each join**, about 1% brightness and 1.5% colour per join. The first new frames sat about 2.8× further from the held frames than normal motion. Correcting only that step held the levels.
- **Still subject, locked camera, calm scene:** contrast stayed flat over 5 clips, but saturation still rose, mostly at the joins.

So "static shots are worst" depends on what's static: a still subject, a still camera and a still background are different cases.

He also measured the handover itself: H3 returns the held picture almost exactly (0.9975 correlation) but about **1.027× hotter**. That doesn't simply feed forward, though: each clip's tail comes back settled, and the held latent is only about 1.003× hotter link to link. The rows that come back hot are the ones that get trimmed.

### 4.2 The model copies finished held frames very closely

Because H3 hands the held frames over almost clean at every step (section 2), it treats them as ground truth and matches them closely: colour, contrast, grain, all of it.

Our Step Echo test supports this ([frame0_conclusions.md, section 7](../20261101/frame0_conclusions.md)). When the held frames were swapped at each step for the last clip's matching in-between step, so they were mostly noise for most of the render like the new frames, **the baking disappeared**. But the model also lost the scene, because it had nothing finished to hold onto. Locking the single-frame latents (0 and 5) brought the scene and motion back and kept most of the cleanup, but some colour drift came back with them.

### 4.3 Frame 0 matters a lot

The first single-frame latent of a clip acts like the "starting image" of an image-to-video generation, and H3 pays a lot of attention to it. In our tests:

- A **clean frame 0** from clip 1 ("keystone") cut colour drift and degradation noticeably over 10 clips.
- A **bad frame 0** (blurred, darkened, tinted, degraded) made the floor bake **about twice as hard** by clip 10.
- The **other single-frame latents** in the held context (latent 5 = frame 17) matter too. Redrawing latent 5 fresh helped colour.

### 4.4 VAE round trips add loss, but they're not the whole story

Chaining through pixels (decode, save as video, load, re-encode) loses detail every time, and saving intermediates as compressed mp4 adds more. Saving as lossless video (ffv1) or passing latents directly avoids that part.

But Ablejones showed that **chaining purely in latents still degrades**, faster than expected. So the VAE isn't the main cause. NikoDemon80 ruled out more suspects one by one: a lossy codec (same drift on lossless), int8 vs fp16 video VAE (identical to 3 decimals), where in the clip the decode happens, clip length, audio grid alignment, the FastH3 distillation (20 steps with no LoRA drifted *more*), resolution up to ~0.8 MP on a calm scene, and the reference image (for contrast). Above ~0.8 MP is untested; joeygambino reported drift growing with canvas size. Ryko (Nov 2026) found the same: saved latents that were never VAE-decoded still degrade, so it seems to be anything that serves as a guide latent (any clean, finished frames handed to the next clip) that causes it. That matches 4.2.

### 4.5 Settings that make it worse

- **Turbo / speed LoRAs** reduce motion, prompt adherence and audio quality, and can add crunch.
- **Resolution above ~1 MP.** H3 is trained around 1 MP with a 768 px short side, and goes off the rails above that.
- **Wrong sampler** for the schedule (see section 2).
- **Re-using `[Shot N]` tags for the same scene.** H3 treats each shot tag as a scene change and re-invents the environment, which shows up as geometry and lighting drift.

---

## 5. How continuations evolved

A rough timeline of how people have extended H3 clips since the open weights came out (August 2026).

| Stage | How it works | Known problems |
|---|---|---|
| **One long shot** | Generate the whole thing in one pass (people reached 30–45 s in a single pass on big GPUs). | VRAM; the model can lose the plot over very long takes. |
| **Last-frame image-to-video** | Use the last frame of clip N as the first frame of clip N+1 (the WAN-era trick). | One frame carries no motion, so the motion breaks at the join. Colour shifts every handover. |
| **First + last frame (FL2VA)** | Pin both ends of a clip. | Colour drift at the ends; seams. |
| **Motion Context** (NikoDemon80) | Carry the last N frames (often 22) into the next clip as guide keyframes, block-aligned. Also Context Loop workflows. | Smooth motion, but the photocopy effect starts after a few clips. |
| **Masked latent continuation** (seitanism, Clown Context) | Copy the tail latents straight into the new clip and lock them with a mask. Hard mask on video, soft feather on audio. | Same accumulation; more direct control. |
| **Reference / Add Guide extends** | Feed the previous video through the reference lane. | toyxyz measured degradation on every pass, even on a steady scene. |
| **Fixes on top** | Refine passes, clean anchors, colour matching, context windows, rectifying the tail (sections 6 and 7). | Each fixes part of it. |
| **Context windows as a second stage** (Kijai) | Extend normally first, then run context windows over the whole thing for the last steps to even it out. | About 2× the compute; mainly useful as a second stage. |

---

## 6. xyzDist's method (H3-LongTakeNoCuts)

**Repo:** [github.com/xyzDist/H3-LongTakeNoCuts](https://github.com/xyzDist/H3-LongTakeNoCuts). It's built on the native Motion Context example workflow, with as few changes as possible, and has examples up to a 3-minute single shot.

**The problem, in his words:** with Motion Context or latent save/load, degradation starts around segments 5–6 without cuts. Each segment loads the previous latent as ground truth, and every generation adds a little drift and detail loss: the photocopy effect.

**The method:**

1. **Refine resample.** After each segment, run a second short sampling pass at **0.5–0.6 denoise**. This brings back detail and fixes the waxy / burnt look. Optionally, mask the character with SAM3 so only the character is refined.
2. **Blend back by frames, in latent space.** His `H3 Blend Latents By Frames` node blends the original segment into the refined one over time. The default keyframes `0:0, 22:0, 44:1` mean:
   - frames 0–22 stay original, so the join to the last shot is untouched (Motion Context holds those 22 frames anyway)
   - from frame 22 to 44 it fades over to the refined version
   - after frame 44 it's fully refined
3. **Keyframe mask** (`15:1, 22:0, 30:1`) keeps the character the same at the start so it connects to the previous shot.
4. **Mask Levels** node to tune the character mask.

**Why it works:** each segment gets a partial re-generation that pulls it back toward what H3 thinks a clean picture looks like, before it's handed to the next segment. The handover frames are left alone, so nothing jumps.

**Limits he lists:**

- It's a workaround, not a true fresh regeneration.
- Works very well on moving shots.
- **On half-static and static shots the subject still drifts toward more contrast and saturation.**
- Audio through Motion Context still needs checking.

---

## 7. What people do about it today

### Cut more often

The simplest fix. A hard cut or a new camera angle every ~30 s resets the drift. RuneX (Nov 2026) puts it the same way: a new camera angle is basically a fresh video with the same subject and scene, and long continuous shots are rare in real films anyway. He also tried tail latents with a mask, plus a latent guide or an image guide, and found none of them perfect. Ethanfel plans a camera cut about every 30 seconds and uses "semantic anchors" in the prompts so details carry over.

### Colour correction / colour matching

Very common, and it helps the colour part. It doesn't fix the baked texture.

- **ComfyUI's native Color Match** (Kijai's, merged into core in April 2026, runs on GPU). Kijai's older KJNodes `ColorMatch` offers the methods `mkl` (default), `hm` (histogram matching), `reinhard`, `mvgd`, and the hybrids `hm-mvgd-hm` and `hm-mkl-hm`. Match each clip to a clean reference frame from clip 1.
- **MiniMax H3 Stitch Advanced** (djbfilmz) colour-matches start and end images when joining clips.
- **By hand** in DaVinci Resolve or similar. JalenBrunson found fixing only saturation and tint reduced the overcooked look, though colour still shifted.
- **Watch out:** colour matching to a whole frame can make things worse if the frames have different content. mdkb's red bus threw off the match. Match on a masked area or a similar frame.

### A refine pass

- **xyzDist:** 0.5–0.6 denoise per segment, blended back (section 6).
- **Low-denoise pass:** suggested for long static shots that decay on every extend.
- **Two-pass with the H3 latent upscaler:** a low-res pass, the latent upscaler, then a second pass at ~0.4 denoise. Mainly for quality and resolution, but it also re-generates detail.

### Rectify the tail

KingGore's approach for chains that get dark and blurry after ~3 links: only fix the degraded end. Mask the start, rectify the last frames (with an image model at low strength, or let H3 itself redo them), then lock start and end and regenerate the middle.

### Clean anchors

- **Keystone / capstone** (Clown Context Suite): put clip 1's clean frame 0 at the start of every shot (keystone), and clip 1's last single-frame latent tacked onto the end during sampling (capstone). The capstone does most of the colour work.
- **Anchor 5 frames as context** (Ethanfel): pack composition context as 5 frames of one clip plus 17 of others.
- Niko's caveat: a frame carries pose, camera and background, not just colour. On moving shots, a clip-1 anchor drags the shot back toward clip 1. All our keystone/capstone tests so far hold the camera still; how it behaves with a moving camera is still open.

### NikoDemon80's stack (Motion Context)

From [his notes, section 5](community/NikoDemon80_H3_drift_notes.md). It slows drift; it doesn't stop it.

- **Join Match:** corrects the step at each join on locked-off shots. Over 6 clips: darkening 5% → 0.3%, contrast loss 2.4% → 0.5%, colour loss 7% → 2.4%. Texture unchanged. It stands down on moving shots.
- **One-step Refine Blend, then Join Match:** a refine of just the last step (split at step 7 of 8), blended in. Detail growth +14.9% (was +23.5%), texture +11.4% (was +23.4%), saturation and brightness flat. Order matters: Join Match has to run after the refine. A 0.5-denoise refine aged the subject 5–10 years by clip 5 in his setup.
- **Sigmas Tail** (`tail_steps` 3, strength 0.5): texture build-up 18% vs 23–26% without; faces stopped aging on calm backgrounds. It's tuned to FastH3's schedule (last jump 0.63 → 0); untested on the 0.42 schedule above.
- **Reference images:** a character sheet on a solid background helps pull the subject back. A clip-1 frame as the reference repeats the background on moving shots.

**Prompt levers** (free):

- State colour, lighting, contrast, saturation and level of detail as constant.
- Describe detail as natural and steady. Dropping "sharp" and "crisp" cut build-up by about a third. Avoid "saturated" and "bright highlights".
- Name hair colour and wardrobe in every clip.
- H3 reads hex colour codes: good on backgrounds and major colours, poor on hair and near-black.
- Without a look/detail recipe his moving chains ran 1.12–1.21× on contrast; with one, 0.94–0.97×.

**What failed for him:** Pin Gain, Pin Shift, Noise Aug, Level lock, Detail Match, Pin Noise, Chain Memory, a motion-aware Join Match, and masking instead of pinning (picture matched, but the audio dipped 3–30 dB at the join). Pin Noise wrote a *fixed* noise pattern into the held latent and the model copied it; that's a different thing from Lumifel's context noise, which changes every step and tells the model the noise level.

**Seed noise is large:** four chains of the same prompt ranged 0.98× to 1.13× on contrast, more than the effect of the setting being tested. One chain vs one chain can be luck, and that includes our tests.

**Audio** is the least solved part. How well held audio carries through depends on the material (0.45–0.96 correlation on the same settings); dialogue holds, thin ambience doesn't. An 18-frame audio pin worked best by ear. For music, use a real audio file.

Two unrelated scenes moved the same latent channels the same way (20 of 24), which suggests the drift comes from the model, not the scene.

### Context windows

- Several overlapping windows are denoised each step and blended. By default this gives no length degradation, but Kijai notes it's mainly useful as a second stage or with a control signal.
- **Extend first, window last** (Kijai, Oct 2026): of 8 steps, the first 6 extend with latent inpainting, then the last 2 run full context windows over everything to stabilise it and remove degradation.
- **Hybrid windows / Rolling Diffusion** (JalenBrunson, `h3-hybrid-resume` branch): split the steps between windows and a refining pass.

### Noise tricks on the held frames

- **seitanism:** noise masking plus a linear seam blend for the fried look.
- **Lumifel (Aug 2026):** at each step, add some of the sampler's current noise to the held context, fading it out toward the join. He reported it removed burn-in and artifacts past 60 s, but didn't fully fix colour drift.
- **Step Echo** (our test): use the last clip's real in-between step latents for the held frames instead of fresh noise. Results are in our [conclusions, section 7](../20261101/frame0_conclusions.md).

### Community tests: ucost4 (Reddit, Nov 2026)

ucost4 replied on r/StableDiffusion with four approaches he tested:

1. **PNG of the last frame** as the next clip's start. PNG is lossless, so it avoids mp4 loss, but a single frame carries no motion, so movement can break at the join.
2. **A very similar text prompt with the same sigmas and seed** for each clip, to keep the clips consistent.
3. **The previous clip's video latent** as the start of the next one. This is the same idea as Motion Context and Clown Context.
4. **Writing the last clip's prompt so the next clip is an angle cut** that makes sense by the rules of cinema.

He found 3 the most promising and plans more tests on it. For now he uses 4 (cutting to a new angle), which resets the drift.

### Plumbing that helps

- Pass **latents** between clips, or save intermediates as **lossless** video (ffv1), not mp4.
- Keep the frame rate at **24 fps**. A 30 fps reference is read as 25% too long.
- Cut on **block edges** (17n + 5 frames) so single-frame latents land in single-frame slots.
- Turbo LoRA off on the pass that matters, or at least on the first pass.
- Stay around **1 MP**.

---

## 8. What the research papers say

Video models in general have the same problem, and research has a few names for it.

- **Exposure bias / train–test gap.** Models are trained to continue from clean, real frames, but at generation time they continue from their own imperfect output. Errors pile up. ([Self Forcing](https://arxiv.org/abs/2506.08009), [Stable Video Infinity](https://arxiv.org/abs/2510.09212))
- **Diffusion Forcing.** A training method where every frame gets its own noise level, so the model learns to continue from noisy, uncertain context instead of perfectly clean frames. ([Self Forcing paper's summary](https://arxiv.org/html/2506.08009v1))
- **Self Forcing.** Trains the model on its own rollouts, so it learns to handle its own mistakes.
- **Stable Video Infinity (SVI).** "Error recycling" fine-tuning: during training it feeds the model its own errors on purpose, so it learns to correct them instead of copying them. Only a LoRA is needed, with no extra cost at inference. People in the WAN community have used SVI LoRAs; there's no H3 version as far as we know.
- **FramePack.** Defines "drifting" as quality loss from errors building up over time. Its fixes:
  - compress older frames more, so more history fits
  - **generate the endpoints first and fill in between**, so every section is anchored to something that hasn't drifted

  This is the same idea as our capstone.
- **Rolling Diffusion.** A sliding window where later frames are noisier than earlier ones. Frames enter noisy and leave clean, so the video can go on forever. This is what JalenBrunson's hybrid windows are based on.

The common thread: **models drift because they trust their own previous output too much.** The fixes either train the model to expect errors (SVI, Self Forcing, Diffusion Forcing), give it clean anchors to land on (FramePack endpoints, keystone/capstone), or keep the context partly noisy so it can't copy it exactly (Rolling Diffusion, Lumifel's noise, Step Echo).

---

## 9. Practical checklist

- Cut on block edges (17n + 5 frames)
- Pass latents, or save lossless (ffv1)
- 24 fps
- ~1 MP, 768 px short side
- er_sde + beta for turbo / PDD schedules
- Turbo LoRA off on the important pass
- One `[Shot]` tag per scene; timestamps for camera changes
- Hard cut or new angle every ~30 s if the story allows
- Clean anchor: keystone + capstone, or a clean reference frame
- Refine pass at 0.5–0.6 denoise, blended in after the held frames (xyzDist)
- Colour match to a clean clip-1 frame, on a masked or similar area
- Context windows as a second stage for long takes
- Locked-off shots: correct the step at each join (Join Match)
- Prompts: state the look as constant, drop "sharp"/"crisp", name hair and wardrobe every clip
- Watch static shots most closely

---

## 10. Sources

**Code**
- ComfyUI native H3 implementation: `comfy/ldm/minimax/vae.py`, `comfy/ldm/minimax/model.py`, `comfy/model_base.py` (MiniMaxH3), `comfy/samplers.py` (KSamplerX0Inpaint)
- [xyzDist / H3-LongTakeNoCuts](https://github.com/xyzDist/H3-LongTakeNoCuts)
- [NikoDemon80 / ComfyUI-H3-Motion-Context](https://github.com/NikoDemon80/ComfyUI-H3-Motion-Context)
- [roycho87 / clown_context_suite](https://github.com/roycho87/clown_context_suite)
- [Kijai / ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) (ColorMatch)

**From the Banodoco Discord** (quotes from the community discussion)
- **Kijai** (29 Oct 2026), on context windows: "by default zero degradation for length, but only really useful for 2nd stage or with some control signal".
- **Kijai** (29 Oct 2026): "out of 8 steps, first 6 just extend with latent inpainting, then last 2 steps full context windows to stabilize and remove degradation".
- **Ablejones** (15 Sep 2026), after chaining 6 six-second clips with plain ref2va: "I honestly didn't know it was this bad". On the VAE theory: "H3 is natively just continuing from its own generations. I honestly expected it to just work, I don't have a mechanism for *how* it's degrading."
- **toyxyz** (25 Aug 2026), extending a 5-second clip with R2V and Add Guide: "there is noticeable image quality degradation with each extension."
- **RetroMidget** (23 Aug 2026), comparing it to WAN's last-frame chaining: "It's more subtle."
- **KingGore2023** (7 Sep 2026), describing the look as "Dark. Blury. Error accumulated." His fix: "You rectify the end part when the degradation appeared. Then lock start and end. Then regenerate."
- **Miku** (20 Aug 2026): re-using a finished video as input again and again degrades it because of the mp4 output, and because the default VHS loader uses OpenCV. Switching the output to ffv1 and loading with the VHS FFmpeg path loader avoids it.
- **JalenBrunson** (8 Sep 2026): a "fix" for degradation in later loops based on the Rolling Diffusion paper, as the `h3-hybrid-resume` branch of ComfyUI-MMH3Tools: a sampler that runs 2 passes.
- **RuneX** (2 Nov 2026): the simplest degradation "fix" is to "change camera (aka new fresh video with same subject/scene)". He tried "tail latent with mask, tail latent with mask + latent guide, tail latent with mask + image guide, but none were perfect".
- **Ryko** (2 Nov 2026): "pushing any kind of guide seems to inspire the degradation". He first suspected the VAE decode, but "using saved latents that aren't vae decoded will also produce degradation. So anything that serves as a guide latent seems to be the factor."
- **Lumifel** (20 Aug 2026): all his methods suffered "burn in" at 60 s or more, with "the color get warmer and warmer when the scene doesn't change with a cut". His fix "recalculate[s] how much of the sampler's current noise to apply to the context every sampling steps, while fading that treatment to zero near the end of the context so the join stays clean."

**From GitHub issues**
- **NikoDemon80** (Nov 2026): three months of H3 drift tests, including what failed. Kept in this repo as is: [community/NikoDemon80_H3_drift_notes.md](community/NikoDemon80_H3_drift_notes.md).

**From Reddit**
- **ucost4** (r/StableDiffusion, Nov 2026): tested a last-frame PNG, a similar prompt with the same sigmas and seed, the previous clip's video latent, and prompting an angle cut by cinema rules; found the video latent "looking promising", using angle cuts until a solid solution turns up.

**Papers**
- [FramePack: Frame Context Packing and Drift Prevention in Next-Frame-Prediction Video Diffusion Models](https://arxiv.org/abs/2504.12626)
- [Self Forcing: Bridging the Train-Test Gap in Autoregressive Video Diffusion](https://arxiv.org/abs/2506.08009)
- [Stable Video Infinity: Infinite-Length Video Generation with Error Recycling](https://arxiv.org/abs/2510.09212)
- [Rolling Diffusion Models](https://arxiv.org/abs/2402.09470)

**About H3**
- [What Is MiniMax H3 (Hailuo 3.0)? (Hugging Face blog)](https://huggingface.co/blog/ResterChed/minimax-h3-hailuo-3-0)
