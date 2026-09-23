# Sandplay — MTEC-498 Capstone
Wanlin Huang · Weekly Update · Sep 23, 2026

## Overview
A sandplay-inspired sand tray: participants move miniature figures and push sand by hand to steer AI-generated music through Lyria RealTime. Inspired by sandplay therapy, framed as an art work.

## This Week
- Finalized the sand tray direction (replacing the building-blocks idea)
- Chose to read the tray **from below**: camera under glass, light from above, so the technology stays hidden
- Built a first working prototype in Max/Jitter (`sandplay_sand_reader.maxpat`):
  - iPhone as under-glass camera (Continuity Camera)
  - Image → brightness → 8×8 grid, compared to a stored flat-sand baseline
  - Outputs total change and four-quadrant change (0–1)
  - Test tone: total change controls pitch
- Used salt on glass as a temporary translucent material

## Demo Video
[Link to video]

## Challenges
- Regular sand is too opaque; need translucent material (glass beads)
- Continuity Camera connection is unstable
- Image is mirrored; lighting must be even

## Next Steps
- Glass beads + dedicated USB camera under the tray
- Figure tracking with reacTIVision
- Map sand data to Lyria RealTime
- Progress Report presentation (Sep 30)

https://drive.google.com/file/d/153h7hKz7p_MKbxjsL-vU-iC7Ej8AfrcC/view?usp=sharing