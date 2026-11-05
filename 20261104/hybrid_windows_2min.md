# Geiru Hybrid Windows: a 2-minute render in one go

A 2-minute, 16-window Geiru Hybrid Windows render (the space roller coaster, extended through a wormhole, a jungle planet, a cave, a waterfall, a whale, rockets, a meteor storm and a second black hole).

## Setup

- 2929 frames (2:02) at 960 × 544 (0.52 MP), 16 windows of 226 frames, overlap 39.
- One prompt per window, each picking up where the one before ends.
- Keystone and capstone (`capstone_step` 4) on, context noise on (linear, strength 1), `freenoise` off.
- **`switch_step` 8:** every window renders all 8 steps on its own, so there's no blended pass at the end. Each window is a straight continuation of the finished window before it.
- Crash protection on (it saves after every window; it picked the run back up several times after out-of-memory crashes).

## Result

**By eye (roycho): no visible degradation over the full 2 minutes, even with every window rendered at full steps.**

## Follow-up: prompted overlaps, a clean 2-minute ride (2026-11-05)

The same ride, rebuilt so the joins between windows hold together (versions v3 to v7):

- 3031 frames (2:06) at 960 × 544, 16 windows of 226 frames, overlap 39, so all 16 windows line up evenly.
- **The overlaps are prompted:** each window's first 1.6 s is worded exactly like the end of the window before, and each prompt only describes its own window (an earlier version described the whole ride in every window, so the ending showed up again and again).
- **Every window renders all 8 steps on its own** (`switch_step` 8), no blended pass.
- Keystone and capstone on, `capstone_step` 6 (the capstone comes in for steps 7 and 8), context noise linear 1, `freenoise` off, crash protection on, Motion Booster LoRA at 1.
- Prompt fixes along the way, all found by watching the renders. Each one was a cut or a jump at a window join:
  - A handoff that already showed the next scene (the waterfall at the end of the cave window) made the next window repeat it, so it cut back. Each window's ending now only shows what's in that window.
  - The cart stopping on the lake, then a different track appearing in the next window, read as a cut. The cart now keeps moving, and the track it climbs is the one that rose out of the water at the end of the window before.
  - A move the track couldn't make (a full upside-down loop) made the track change shape partway; a big banked turn works.
  - A planet that changed colour between windows (blue in one, red in the next) read as a cut. Now there's one red ringed planet, seen from the first window it appears in through the re-entry.
  - Things should arrive from where they're described: the phoenix rose out of the track until it was prompted to swoop down from the sky.

**Result (v7): a success.** By eye (roycho), the full 2 minutes hold up with no visible degradation, in one render with no cuts and no stitching. A compressed copy is on the [Geiru Context Suite README](https://github.com/roycho87/geiru_context_suite#experiment-2-minutes-in-one-render-16-windows); the full-size file is on roycho's PC (`output\h3_hybrid_degrade\rollercoaster_wormhole_overlap_v7_00001-audio.mp4`).
