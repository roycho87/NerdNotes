# Issue title

H3 chain degradation: NikoDemon80's test notes (what I measured, what failed, what changed my mind)

# Issue body

These are my notes from about three months of testing H3 chain degradation while building [ComfyUI-H3-Motion-Context](https://github.com/NikoDemon80/ComfyUI-H3-Motion-Context). They include the failures, because those took the most time to learn from.

## Short version

- **Handover:** H3 returns the pinned picture almost exactly (0.9975 correlation). It also raises the level slightly (about 1.027x).
- **Where drift happens:** it depends on the shot.
  - With movement, most contrast and color drift happens inside each clip.
  - On locked-off shots, a lot of it is a step at each join.
  - My earlier "it all happens inside the clip" line was too broad. See section 3.
- **VAE:** it isn't the main cause. Latent passthrough removes visible seams, but drift continues.
- **When contrast and color are set:** they are already in the model's first guess at step 0, before there is a picture. Fine detail comes later.
- **Pin corrections:** most of them failed. The model copies what you hand it, and it returns to its own level.
- **Best stack so far:** Join Match, one-step Refine Blend, Sigmas Tail, and careful prompts. It slows drift. It doesn't stop it.
- **Audio:** this is the least solved part. How well the pinned sound carries through depends heavily on the material, and treble still fades.

## How I tested

- **Model:** FastH3 8-step int8 model, int8 text encoder and video VAE, full-precision audio VAE.
- **Sampling:** stock sparse attention on unless noted.
- **Chain:** 124-frame clips, 22-frame pin, 18-frame audio pin, latent passthrough between clips.
- **Size:** 0.4 to 0.8 MP in the early tests, 1024x768 in the later ones.
- **Volume:** usually 5 or 6 clips and one chain per setting. Treat most single numbers as a guide.
- **Machine:** one Windows machine, so everything here is one person's renders.
- **Seeds:** unlocked on chains, because a fixed seed with an unchanged prompt on a locked-off scene gave me the same clip five times in a row.
- **Metrics:**
  - Detail is what survives a small blur.
  - Texture is detail divided by contrast, so it shows fine structure changing separately from overall contrast.
  - An "x" figure is the end of the chain against the start.

**Seed noise is large.** Four chains of the same prompt, with small correction settings, gave these contrast results:

| Chain | Contrast over the chain |
|---|---|
| Node off | 1.049x |
| Strength 0.025 | 1.132x |
| Strength 0.03, run 1 | 0.979x |
| Strength 0.03, run 2 | 1.002x |

The node's effect at those strengths was smaller than the difference between seeds. Any small gain from one chain against one chain can be luck, and that includes mine.

## 1. What the pin does

Measured by comparing the pinned latent rows with the same rows the model returned, across 7 joins.

- **Picture structure:** 0.9975 correlation.
- **Level:** scaled up about 1.027x, with fine detail growing a further 1.4% or so.
- **Depends on the frame cycle:** 1-frame latents about 1.013x, 4-frame latents about 1.032x. A single correction number would be the wrong shape.
- **Doesn't feed forward the way I assumed:** each clip's tail comes back settled, and the pinned latent is only about 1.003x hotter link to link. The rows that come back hot are the ones that get trimmed.
- **The gain isn't a fixed multiplier:** when I scaled the pin down by 1.022, the gain at the join only halved (spread 1.0220 to 1.0124). The model pushed back.
- **Latent to picture:** a 0.9% latent difference showed up as about 2.5% on screen, roughly 3x.

## 2. Ruled out for the picture

| Suspect | Result |
|---|---|
| Lossy codec | Same drift on lossless |
| int8 vs fp16 video VAE | Identical to 3 decimals on the same seed |
| VAE decode position (head vs tail) | Decoding with prior latents prepended gave 1.0008x, so no effect |
| Clip length | Not the cause |
| Audio grid alignment | Not the cause |
| FastH3 distillation | 20 steps with no LoRA drifted more (table below) |
| Resolution, on a calm scene | 0.4 and 0.8 MP both stayed flat on contrast |
| Reference image, for contrast | Contrast stayed flat with or without one |

| Config | In-clip contrast | At joins | Over the chain |
|---|---|---|---|
| 8 steps, FastH3 | +6.10 | -0.30 | 1.128x |
| 20 steps, no LoRA | +7.62 | +1.39 | 1.196x |

On resolution, I never chained above about 0.8 MP because of memory. That result says nothing about higher sizes. joeygambino reported drift growing with canvas size, so that question is still open.

## 3. Where drift happens depends on the shot

**Moving content**

- On those chains, 85 to 95% of contrast and 88 to 97% of saturation movement happened inside the clips.
- Brightness behaved differently. On one chain it was -17.7 inside the clips and +17.7 at the joins, ending where it started. So the model has a brightness level it returns to.
- Not every chain split that cleanly. One lossless-style chain came out 60/40 on contrast.

**Locked-off shots with a subject doing something**

- Over 6 clips with nothing on: 5% darker, 2.4% less contrast, 7% less color, 26% more texture.
- That is about 1% brightness and 1.5% color lost per join.
- The first new frames sat about 2.8x further from the pin than normal in-clip motion.
- Correcting only that step held the levels (section 5), which wouldn't work if most of this drift were inside the clips.

**Still subject on a locked-off camera, calm scene**

- Contrast stayed flat (0.98x to 1.00x over 5 clips).
- Saturation still rose, mostly at the joins.
- This matches other reports only partly. "Static shots are worst" depends on what is static. A still subject, a static camera, and a static background are different test cases.

**Same scene with constant motion added**

- Contrast went from 0.996x to 1.122x, and up to 1.21x on other chains. About 1.029x per join.

**Note on the summary in `h3_latents_and_degradation.md`:** section 4.1 quotes me saying most color drift builds inside each clip. That holds for moving content, but not for locked-off shots. It's why Join Match works on locked-off shots and does little on moving ones.

## 4. When drift gets decided

I logged the model's guess at the finished clip after every sampling step. The picture values come from the latent preview, so they're good for trends, not exact levels.

- **Contrast:** the climb across the clip is +7.7% at step 0 and +5.4% at the end.
- **Brightness:** +3.1 levels at step 0, +4.1 at the end.
- **Saturation:** settled by step 1.
- **Fine detail:** added around steps 2 to 4.

At step 0 there is only noise, the prompt, and the references. So contrast and color are part of the model's plan for the clip, not something added while finishing. This was also true in a clip with nothing pinned. On one set of chains, clip 1 alone gained 8 to 16 brightness levels over its 124 frames. So a single clip has a drift of its own, and chaining stacks it.

The last-step tricks only reach detail. That's why Sigmas Tail fixed part of the problem and not the rest.

`h3_latents_and_degradation.md` says the picture only forms in the last two steps. That's true for what you can see. These trends were already in the model's early guesses.

## 5. What worked

| Setup | Result | Limits |
|---|---|---|
| **Join Match** on a locked-off shot, 6 clips | Darkening 5% to 0.3%, contrast loss 2.4% to 0.5%, color loss 7% to 2.4% | Texture unchanged. Stands down on moving shots |
| **Sigmas Tail** added (`tail_steps` 3, strength 0.5), 6 clips | Texture build-up 18%, against 23 to 26% without it. Faces stopped aging on calm backgrounds | Busy patterned backgrounds still sharpen slowly. Moved from the joins to inside the clips |
| **One-step Refine Blend, Join Match after it**, 6 clips | Detail +14.9% (was +23.5% with Join Match first), texture +11.4% (was +23.4%), contrast 1.03x, saturation and brightness flat | Slow detail climb remains inside the clips |
| **Full stack** on a scene built to show drift, 5 clips | Saturation +3.6%, contrast -3.2%, brightness -0.8%, detail +7.8%, texture +11% | One chain |

- **Node order matters.** Join Match has to run after Refine Blend, so it can catch the detail the refine adds.
- **Sigmas Tail is specific to my schedule.** FastH3 ends with one big jump from 0.63 to zero, so that was the lever. The schedule in the repo notes ends at 0.42. I haven't tested how much this carries over.
- **Refine strength:** one step (split at step 7 of 8) held face and background over 5 clips. Step 4, about 0.5 denoise, made the subject read 5 to 10 years older by clip 5, and the background became a different street. The seeds differed, but the difference was obvious by eye.
  - This doesn't mean xyzDist's 0.5 to 0.6 is wrong for his setup. It didn't work for mine.
- **Reference images:** character sheets on a solid background help pull the subject back. A frame from clip 1 as the reference repeats the background on dynamic shots. Sheet lighting can leak into the scene if the prompt doesn't state the scene's light.

### Prompt levers

These help and cost nothing. The details are in my example workflow and README.

- State the color, lighting, contrast, saturation, and level of detail as constant.
- Describe fine detail as natural and steady. Removing "sharp" and "crisp" cut build-up by about a third.
- Avoid words like "saturated" and "bright highlights."
- Name hair color and wardrobe in every clip. On a night city walk this kept the hair from lightening, and contrast on the subject went from +19% to +8.6%. The measured area changed slightly between runs, so the hair holding is the cleaner evidence.
- H3 reads hex color codes. They work well on backgrounds and major colors, and poorly on hair and near-black.
- Without a look and detail recipe, moving chains ran 1.12x to 1.21x on contrast. With one they ran 0.94x to 0.97x, and contrast eased down about 1% per join instead.
- A camera arc can snap back to frontal when the prompt states where the eyes are looking. The model has to pick, and it picks the face.

## 6. What failed

| Test | Idea | Result |
|---|---|---|
| **Pin Gain** | Scale the pin down to cancel the 1.022x gain | Fixed the rows that get trimmed. Worth about 1.8% of chain contrast, not the 70% the latent numbers suggested. Over-corrected once the prompts improved |
| **Pin Shift** | Move each latent channel the opposite way | Helped coat hue (14.5 degrees to 1.9, about 50 to 10 on a walk). On a locked-off desk shot it doubled the darkening (5% to 11.5%). Tuned on rows you never see |
| **Noise Aug** | Pin label 0.999 vs 1.0, tested on audio | No change |
| **Level lock** | Anchor each clip's levels to the pin | Ratcheted brightness to 1.22x, against 1.003x without it. Seeds differed. Removed |
| **Detail Match** | Soften decoded frames and re-encode the pin | The VAE round trip moved the latent 3 to 5x more than the problem |
| **Pin Noise** | SG0102's idea, adapted to the latent | Saturation +23 to +35%, texture +32 to +53%. Lost to the full stack with it off |
| **Chain Memory** | Clip 1 frames outside the clip's timeline, for look only | Before the clip: no color gain, joins 1.9 to 2.4x. After the clip: clean joins, but clip 4 ended on clip 1's opening frame |
| **Motion-aware Join Match** | Correct moving shots too | Two versions, both measured worse than leaving those shots alone |
| **Masking vs pinning** | drozbay's mask path against my pin | Picture matched. Audio dipped 3 to 30 dB for 0.35 to 1 second, then swelled back. A soft edge reduced the dip but didn't remove it. Adding a guide removed the dip and brought the muffling back |

**Pin Noise vs context noise.** These aren't the same test, and my failure doesn't argue against roycho's result.

- Mine wrote a fixed noise pattern into the pinned latent, and the model copied it.
- Lumifel's method and roycho's Context Noise use a noise level that changes each step, and the model is told about it.
- SG0102's original worked in pixels. I didn't test that version.

**Per-channel note:** two unrelated scenes moved the same latent channels in the same direction (85% match, 20 of 24 channels). That suggests this comes from the model, not the scene. ethanfel's Color Carry worked on join color before I did.

## 7. Not settled

**Habit Correction** is an unreleased test node. It pulls contrast down during sampling without looking at any frame.

| Strength | Detail | Texture |
|---|---|---|
| 0 | 1.10x | 1.05x |
| 0.025 | 1.20x | 1.06x |
| 0.03, run 1 | 1.09x | 1.11x |
| 0.03, run 2 | 1.13x | 1.12x |
| 0.10, run 1 | 0.98x | 1.00x |
| 0.10, run 2 | 1.00x | 0.97x |

- **Contrast:** it failed. The model rebuilt it on the final steps, and four times the strength had no extra effect.
- **Detail:** both 0.10 chains stayed flat, which is the only result here that repeated.
- **Caveats:** I haven't judged these by eye for a flat look, one saturation result dropped to 0.92x, and the clip order on one chain needs confirming. Treat this as a lead.

**Keystone and capstone**

- My read matches roycho's data for a locked camera, and his walk results are strong.
- All of his prompts hold the camera still. My moving tests used a camera arc.
- Any real frame from clip 1 carried composition in my off-clip tests. I don't know yet how his best stack behaves when the camera itself moves.

## 8. Audio

- **Timing:** putting the pinned audio on the clip's own timeline removed the join tick and locked the offset.
- **Continuation depends on the material.** Measured as correlation between the pinned audio and what came back:

| Chain | Audio correlation |
|---|---|
| Most chains | 0.45 to 0.63 |
| 124 frames, FastH3, one prompt | 0.49, 0.52, 0.45 |
| 124 frames, FastH3, new prompt, seed and size | 0.94, 0.96, 0.95 |
| A dialogue chain | 0.87 |
| 7-join chain at 141 frames | about 0.61, with about 66% of the audio unexplained by the pin |

  - The same settings gave both the 0.49 and the 0.95 results, so settings aren't what separates them.
  - I chased clip length, grid alignment, and the pin's timestep label off a single 0.88 reading. None of them held up. The 124-frame result above is what killed the length theory.
  - Dialogue held together. Thin ambience didn't. On one sparse chain the model grabbed a stray sound from the pin and repeated it in the same spot in the next two clips.
  - Turning sparse attention off moved one pair from 0.46 to 0.54, so it's a small effect at best.
  - The picture on these same joins always came back around 0.997.
- **Pin length, by ear:**
  - 24: music muffles and drops in level across joins.
  - 12: level holds, but the music changes clip to clip.
  - 18: best so far, with some treble and level loss that levels off around clip 4.
- **Decode test:** the audio arm of the prepend test gave 0.239x. I think that was an alignment error, so it proves nothing.
- **Speed tricks:** turbo LoRAs and Spectrum both cost audio quality. I found that by ear on early chains.
- **What I do for music:** use a real audio file when music is the main thing. A track that was never generated can't drift.

## 9. Numbers I no longer stand behind

- **"VAE head vs tail decoding causes the drift."** Refuted for picture.
- **"The drift feeds forward in the latent."** Measured false, about 1.003x per hop.
- **"Audio returns at 0.95+."** True on some chains, not most. See section 8.
- **"Sparse attention causes plastic skin."** Turning it off raised texture in two chains (0.766x to 1.339x in one), so it does change texture. It isn't the cause of the build-up, and texture climbed with it off.
- **"Drift comes from motion, not chaining."** Too simple. Locked-off shots degrade too.
- **"Spectrum and the turbo LoRA explain the audio loss."** They add to it, but they aren't the whole story. See section 8.
- **"Audio continuation depends on clip length."** It came from one reading that never repeated.

## Credits

- ChimeraWerks: audio grid fix and amplification analysis
- ethanfel: Ref2VA multi-reference work and Color Carry
- drozbay: upstream mask and layout work, and MaskVidExperiments
- xyzDist: refine and blend approach
- SG0102: chroma noise idea
- joeygambino: published drift measurements
- roycho87: Keystone, capstone, and this repo
