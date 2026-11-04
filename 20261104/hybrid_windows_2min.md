# Geiru Hybrid Windows: a 2-minute render in one go

A 2-minute, 16-window Geiru Hybrid Windows render (the space roller coaster, extended through a wormhole, a jungle planet, a cave, a waterfall, a whale, rockets, a meteor storm and a second black hole).

## Setup

- 2929 frames (2:02) at 960 × 544 (0.52 MP), 16 windows of 226 frames, overlap 39.
- One prompt per window, each picking up where the one before ends.
- Keystone and capstone (`capstone_step` 4) on, context noise on (linear, strength 1), `freenoise` off.
- **`switch_step` 8:** every window renders all 8 steps on its own, so there's no blended pass at the end. Each window is a straight continuation of the finished window before it.
- Crash protection on (it saves after every window; it picked the run back up several times after out-of-memory crashes).

## Result

**By eye (roycho): no visible degradation over the full 2 minutes, even with every window rendered at full steps.** Before the latest Hybrid Windows changes (context noise on the overlap, the fade selector, freeing memory between windows), full-step windows without the blended finish were expected to drift.

## Next

- The same ride with the overlaps prompted (each window's first 1.6 s worded exactly like the end of the window before), keystone and capstone on, `switch_step` 6, 3031 frames so all 16 windows line up evenly.
