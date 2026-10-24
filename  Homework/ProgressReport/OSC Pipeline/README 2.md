# Sandplay Progress Log: OSC Pipeline

Oct 24, 2026 · Wanlin Huang

## Overview

This phase connected Max to Lyria RealTime end to end: OSC messages sent from Max now change parameters on the web app in real time, and the music changes with them. The camera is not connected yet, so manual sliders stand in for the sand tray data.

- **Goal**: let the sand tray installation (Max/cv.jit) control last semester's Lyria RealTime web app live, with no typing
- **Done**: OSC input, automatic control of the web app, a simulation test script, and a Max test patch; all four links work
- **Not yet**: feeding camera data from cv.jit in place of the manual sliders

## System architecture

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐       ┌──────────────────────┐  OSC  ┌──────────────────────┐
  Camera under glass   - - ->│ Max + cv.jit         │──────>│ server.js            │
│ not connected yet    │       │ reads the sand tray  │       │ UDP port 7400 in     │
  test sliders stand in        │ udpsend 127.0.0.1 7400│       │ forwards to the page │
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘       └──────────────────────┘       └──────────┬───────────┘
                                                                         │ WebSocket
                                                                         v
                     ┌──────────────────────┐  prompts + config  ┌──────────────────────┐
                     │ Lyria RealTime       │<───────────────────│ Web app (app.js)     │
                     │ one session per track│                    │ moves sliders smoothly│
                     │ music changes live   │───────────────────>│ plays audio in browser│
                     └──────────────────────┘    audio stream    └──────────────────────┘
```

Max only reports what happens in the tray; server.js relays it; the web app turns it into slider moves that Lyria hears. Max and the browser cannot talk directly, which is why server.js sits in between.

## OSC message protocol

Max sends these messages through `[udpsend 127.0.0.1 7400]`. All values are 0 to 1, and each message repeats every 100 ms.

| OSC address | Argument | Meaning in the tray | Web app control |
| --- | --- | --- | --- |
| `/figure tree` | float 0–1 | Tree present, and how close to center | Track 2 Volume |
| `/figure house` | float 0–1 | House | Track 3 Volume |
| `/figure person` | float 0–1 | Person | Track 4 Volume |
| `/sand/motion` | float 0–1 | How fast the hand pushes sand | Density, all tracks |
| `/sand/exposed` | float 0–1 | Glass area uncovered by pushed-away sand | Brightness, all tracks |
| `/idle` | int 0 or 1 | No hand movement for a while | Mute drums, all tracks |

Track 1 is the always-on background and ignores `/figure`. BPM and key are never changed over OSC, because changing them makes Lyria cut hard.

## Changes to the web app

All changes are additions to last semester's Lyria web app; Add track, Stop all and recording work as before. The code changes were made with Cursor.

**server.js**

- New UDP port 7400 receives OSC from Max, using the `osc` library
- New WebSocket channel `/api/osc-control` forwards those messages to the page
- The existing audio channel `/api/lyria-stream` is unchanged

**public/app.js**

- A "figure → track" mapping table sits at the top of the file and can be edited directly
- Incoming messages move the matching slider or checkbox and fire input / change events, so the original Lyria logic needed no changes
- Values glide smoothly instead of jumping
- A figure that reads 0, or sends nothing for 1 second, waits 1 second and then fades out, so detection jitter does not make the sound stutter
- Config updates to Lyria are throttled: at most 4 per second per track, only when a value moves more than 0.02, always as a complete config
- Density and Brightness used to send only on slider release; they now update live during playback
- A status box in the bottom-right corner shows whether OSC is connected and the last 5 messages

**Other**

- New dependency `osc` and a new script `npm run test:osc`
- The project is now under git; `.env` (the API key) is excluded from commits

## Testing

The pipeline was tested in two stages, so each failure could be traced to one half of the system.

**Stage 1: test-osc.js (no Max)**

`npm run test:osc` pretends to be Max and plays a script: empty tray → place tree → push sand → uncover glass → place house → remove tree → idle. The status box received every message, and Density, Brightness and Mute drums moved on their own (for example Density 0.50 → 0.80, Brightness 0.50 → 0.85). Sound only changed once every track had a prompt and was playing; a stopped track moves its sliders but sends nothing to Lyria.

Test prompts used:

| Track | Role | Prompt |
| --- | --- | --- |
| Track 1 | Background | calm ambient pads, spacious, soft and warm, slow evolving texture |
| Track 2 | Tree | soft woodwinds, flute and clarinet, gentle forest ambience, organic |
| Track 3 | House | warm felt piano, intimate and close, gentle chords, safe and cozy |
| Track 4 | Person | soft wordless humming voice, human warmth, gentle and tender |

**Stage 2: Max test patch (no camera)**

`sandplay_osc_test.maxpat` replaces the camera with six controls: tree, house, person, motion, exposed (number boxes, 0–1) and idle (toggle). A `[qmetro 100]` re-sends every value through `[udpsend 127.0.0.1 7400]` ten times a second. The web app received the messages and the music followed the controls.

Demo video: [Max test patch controlling the web app](https://drive.google.com/file/d/12dLKjfsjU5tn9nf5Atrhg5sZ2XuSY9GB/view?usp=sharing)

## Problems and fixes

Four problems came up; all are fixed.

| Problem | Cause | Fix |
| --- | --- | --- |
| Only one track could play at a time, though several worked last semester | The new code broke multi-session playback. Confirmed by stashing the changes: the old version played several tracks at once, so the API was not the limit | Cursor compared old and new versions with git diff and fixed the conflicting change |
| A figure's track volume did not rise during the test, so only Track 1 was audible | test-osc.js sent each `/figure` once, while the page fades out a figure after 1 second of silence | The test script now re-sends while a figure is present, as Max does |
| test-osc steps were too short to hear Lyria change | Lyria changes gradually, over several seconds | Longer steps requested, plus a manual keyboard mode |
| Could not save a checkpoint (`not a git repository`) | The project folder had no git repository | Ran `git init`, confirmed `.env` is ignored, committed the working version |

Lesson: test one half of the chain at a time, and commit a working version before each change.

## Status and next steps

The pipeline Max → server.js → web app → Lyria is working and saved in git. The next phase replaces the manual controls in the test patch with values computed by cv.jit from the camera under the glass.

| Test patch control | cv.jit source to connect |
| --- | --- |
| motion | `cv.jit.framesub` → `cv.jit.mean` (frame difference) |
| exposed | brightness threshold (dark pixels) → `cv.jit.mean` |
| tree / house / person | `cv.jit.blobs.centroids` per base color; blob weight when present, 0 when absent |
| idle | motion below a threshold for several seconds → 1 |

- [ ] Set up the camera under the glass with lighting from below and a dark space above
- [ ] Connect motion first, since it is the simplest and most audible
- [ ] Connect exposed, then the figure colors one at a time
- [ ] Scale every value to 0–1 with `[scale]` or `[clip 0. 1.]` before sending
- [ ] Revise the figure prompts after guidance from the Music Therapy faculty on sandplay symbolism