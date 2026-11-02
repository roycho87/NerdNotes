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

So **the picture only really forms in the last two steps.** For six of eight steps, everything is mostly noise.

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

NikoDemon80 measured that **most colour drift builds up inside each clip while it generates**, not at the handover. The handover is just how it gets passed forward.

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

But Ablejones showed that **chaining purely in latents still degrades**, faster than expected. So the VAE isn't the main cause.

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

The simplest fix. A hard cut or a new camera angle every ~30 s resets the drift. Ethanfel plans a camera cut about every 30 seconds and uses "semantic anchors" in the prompts so details carry over.

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
- Niko's caveat: a frame carries pose, camera and background, not just colour. On moving shots, a clip-1 anchor drags the shot back toward clip 1.

### Context windows

- Several overlapping windows are denoised each step and blended. By default this gives no length degradation, but Kijai notes it's mainly useful as a second stage or with a control signal.
- **Extend first, window last** (Kijai, Oct 2026): of 8 steps, the first 6 extend with latent inpainting, then the last 2 run full context windows over everything to stabilise it and remove degradation.
- **Hybrid windows / Rolling Diffusion** (JalenBrunson, `h3-hybrid-resume` branch): split the steps between windows and a refining pass.

### Noise tricks on the held frames

- **seitanism:** noise masking plus a linear seam blend for the fried look.
- **Lumifel (Aug 2026):** at each step, add some of the sampler's current noise to the held context, fading it out toward the join. He reported it removed burn-in and artifacts past 60 s, but didn't fully fix colour drift.
- **Step Echo** (our test): use the last clip's real in-between step latents for the held frames instead of fresh noise. Results are in our [conclusions, section 7](../20261101/frame0_conclusions.md).

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
- Watch static shots most closely

---

## 10. Sources

**Code**
- ComfyUI native H3 implementation: `comfy/ldm/minimax/vae.py`, `comfy/ldm/minimax/model.py`, `comfy/model_base.py` (MiniMaxH3), `comfy/samplers.py` (KSamplerX0Inpaint)
- [xyzDist / H3-LongTakeNoCuts](https://github.com/xyzDist/H3-LongTakeNoCuts)
- [NikoDemon80 / ComfyUI-H3-Motion-Context](https://github.com/NikoDemon80/ComfyUI-H3-Motion-Context)
- [roycho87 / clown_context_suite](https://github.com/roycho87/clown_context_suite)
- [Kijai / ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes) (ColorMatch)

**Banodoco Discord** (community discussion; links need a Banodoco account)
- [Kijai: extend first, context windows for the last 2 of 8 steps](https://discord.com/channels/1076117621407223829/1532625331960152124/1565313136737656853)
- [Kijai: context windows give no length degradation, best as a second stage](https://discord.com/channels/1076117621407223829/1532625331960152124/1565304604256636979)
- [Ablejones: latent-only native extends still degrade](https://discord.com/channels/1076117621407223829/1532625331960152124/1549259373048500465)
- [seitanism's degradation example](https://discord.com/channels/1076117621407223829/1532625331960152124/1548725476942422036)
- [Rectify the degraded tail](https://discord.com/channels/1076117621407223829/1532625331960152124/1546541049281913005)
- [toyxyz: R2V extends degrade every pass](https://discord.com/channels/1076117621407223829/1532625331960152124/1541719603644993576)
- [Washed-out tails on long chains](https://discord.com/channels/1076117621407223829/1535700117452226560/1541209912028307537)
- [Save/reload losses; ffv1](https://discord.com/channels/1076117621407223829/1535700117452226560/1539825508928192604)
- [JalenBrunson: Rolling Diffusion hybrid windows](https://discord.com/channels/1076117621407223829/1540391549731471461/1547017719214243851)
- [Lumifel: noise on the held context, fading toward the join](https://discord.com/channels/1076117621407223829/1535700117452226560/1540092434946723900)

**Papers**
- [FramePack: Frame Context Packing and Drift Prevention in Next-Frame-Prediction Video Diffusion Models](https://arxiv.org/abs/2504.12626)
- [Self Forcing: Bridging the Train-Test Gap in Autoregressive Video Diffusion](https://arxiv.org/abs/2506.08009)
- [Stable Video Infinity: Infinite-Length Video Generation with Error Recycling](https://arxiv.org/abs/2510.09212)
- [Rolling Diffusion Models](https://arxiv.org/abs/2402.09470)

**About H3**
- [What Is MiniMax H3 (Hailuo 3.0)? (Hugging Face blog)](https://huggingface.co/blog/ResterChed/minimax-h3-hailuo-3-0)
