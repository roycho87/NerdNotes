# Test 16: every combination of the three protections (30 beach clips)

Which of the three degradation protections does the work, and do they add up? Eight 30-clip chains of the same beach run, one for every combination of:

- **K: keystone + capstone** (always together, as one unit): Geiru Keystone (`keystone front + capstone back`, from clip 1) with Late Lock on the capstone at `swap_step` 6.
- **L: the protection LoRA**: the experimental [Geiru Protection LoRA](https://github.com/roycho87/Geiru-Protection-LoRa-Experimental) node with Imagineer13's H3 Anti-Burn LoRA at 0.6.
- **N: context noise**: Geiru Context Noise, linear, strength 1.

## Runs

| Run | Keystone + capstone | Protection LoRA | Context noise |
|---|---|---|---|
| 1. Context only | | | |
| 2. K | ✓ | | |
| 3. L | | ✓ | |
| 4. N | | | ✓ |
| 5. K + L | ✓ | ✓ | |
| 6. K + N | ✓ | | ✓ |
| 7. L + N | | ✓ | ✓ |
| 8. K + L + N | ✓ | ✓ | ✓ |

## Setup (the same for every run)

- **Scene:** the beach run from [test 14](../20261105/beach_lora_strength_test.md), clips 1–30 of the same prompts ([prompts/beach_lora50](../20261105/prompts/beach_lora50)), one continuous scene.
- **Chain:** 30 clips of 8 s, 480 × 864, seed 1234, 8 steps er_sde / beta, the 8-step turbo LoRA.
- **Geiru Context:** 39 frames, drop last 17, audio drop last 17, `degradation_protection` on (each protection is turned on or off on its own node).
- None of the protections touch clip 1, so all eight runs start from the same clip 1.

## What we're looking for

- How much each run drifts from clip 1 to clip 30 (darkening, colour, burn, detail).
- Which protection on its own cuts the drift the most, compared with context only.
- Whether the pairs and all three together do better than the best one alone.

*(running, started 2026-11-07)*
