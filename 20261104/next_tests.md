# Next tests (in order)

## 1. Anti-Burn LoRA: quality and best strength

A few 30-clip tests to gauge exactly how much the [H3 Anti-Burn LoRA](https://huggingface.co/Imagineer13/H3-Anti-Burn-LoRA) affects quality, what the best strength is, and whether the slight softening some people see comes from the LoRA or from degradation showing up in a different form. Details: [antiburn_results.md](antiburn_results.md). First test: [static_chair_lora_test.md](static_chair_lora_test.md) (no LoRA, 0.2, 0.4, 0.6, 30 clips each): done, no visible difference on a still scene. Second test: [the same four strengths on a moving beach run, 50 clips](../20261105/beach_lora_strength_test.md) (running).

## 2. Continuations with Geiru Hybrid Windows (after the LoRA testing)

A single 2-minute render already holds up with no degradation ([hybrid_windows_2min.md](hybrid_windows_2min.md)), so the next step is extending one.

The idea: extend a finished Hybrid Windows video without re-rendering it.

1. Load the finished video's saved latent.
2. Make the video longer, with prompts for the new windows.
3. Lock all the old frames as done.
4. Render only the new windows, each one locked to the overlap before it (the same as inside one run), with the original window 1's keystone and capstone.

Hybrid Windows already does most of this: its crash protection picks a run back up by locking the windows that are done and rendering the rest. The open question is the blended finish, which smooths over every window: it would touch the end of the old part again, so the old part may need to stay fully locked, with only the new windows and the join blended.

Related work found so far:
- JalenBrunson mentioned unpublished resume/restart nodes for his [ComfyUI-HybridWindows](https://github.com/Jalen-Brunson/ComfyUI-HybridWindows) (Banodoco, 2026-10-17).
- kijai described the same two-stage idea on Banodoco (2026-10-29): extend with latent inpainting for the first steps, then full context windows for the last steps to remove degradation. That's what Hybrid Windows does inside one run.
- [Herrgott's H3 Infinite Continuation Suite](https://github.com/HerrgottMargott/Herrgotts-H3-Infinite-Continuation-Suite/releases/tag/1.3.0) and [ComfyUI-MiniMaxH3-Contex-Loop](https://github.com/salikshah/ComfyUI-MiniMaxH3-Contex-Loop) do continuations with clip chaining, not context windows.

## 3. A better keystone with H3's image model (idea)

Clean up clip 1's first frame with an H3 image model and use the cleaned frame as the keystone for the whole chain. If the image model shares the video model's latent space (same VAE), the single-frame keystone latent could be edited and plugged straight back in, with no decode and re-encode. That's the first thing to check.

Related: on Banodoco, Hevi, VK and QuintForms clean up keyframes in an outside image model (Z-Image at 0.25–0.5) and run H3 again with them ([notes](../20261031/h3_long_video_experiments.md)). Same goal, but done outside H3 and with a decode in between.
