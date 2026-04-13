# Learn To Draw — Design Spec

**Date:** 2026-04-12
**Status:** Approved

## Overview

A two-page magic trick web app. The spectator visits the public URL and picks one of 12 simple subjects to draw. Their selection fires instantly to the magician's phone, which displays a convincing hand-drawn sketch of that exact subject inside a fake drawing app UI. The spectator can change their mind at any time — the phone updates instantly. When the trick is over the magician taps Erase and the app resets for the next performance.

## Files

```
LearnToDraw/
├── index.html       — spectator page (public URL)
├── receiver.html    — magician's phone (secret URL: /receiver)
└── vercel.json      — deploy config, no-cache header for /
```

No build system. Plain HTML/CSS/JS. Same architecture as AnySearch.

## Firebase

- **Project:** `https://magictrick-5fc12-default-rtdb.firebaseio.com` (existing)
- **Path:** `/learnToDraw/selection`
- **Values:** `"house"` | `"flower"` | `"tree"` | `"car"` | `"airplane"` | `"star"` | `"sun"` | `"heart"` | `"smiley"` | `"clock"` | `"stickman"` | `"rocket"` | `null` (erased/reset)
- **Write:** spectator page writes on every button tap (PATCH request to Firebase REST API)
- **Read:** receiver listens via `EventSource` SSE — same pattern as `/Users/carlandrews/Downloads/AnySearch/receiver.html`

## Data Flow

1. Spectator taps a picture button → `index.html` PATCHes `{ selection: "house" }` to Firebase
2. `receiver.html` SSE listener fires → hand-drawn sketch of a house appears instantly
3. Spectator taps a different button → Firebase updates to new value → phone swaps sketch instantly
4. Magician taps Erase → `receiver.html` PATCHes `{ selection: null }` → canvas goes blank

## Spectator Page (index.html)

**Visual design:** White background, minimal. No branding that suggests a magic trick.

**Header:** Simple centered heading — *"What will you draw?"*

**Picture grid:** 4 columns × 3 rows, 12 buttons total. Each button shows:
- A clean SVG line-art icon (not sketchy — simple and recognizable)
- Label beneath in small text

**Subjects and labels:**
House, Flower, Tree, Car, Airplane, Star, Sun, Heart, Smiley Face, Clock Face, Stickman, Rocket

**Interaction:**
- Tap a button → gold border highlights it, Firebase written, drawing canvas slides in below the grid
- Tap a different button → highlight moves, Firebase updates, canvas resets to blank
- Drawing canvas: real HTML5 `<canvas>` element, touch and mouse drawing in black. Prompt: *"Try to draw it here"*
- Nothing in this page references `receiver.html`

## Receiver Page (receiver.html) — Magician's Phone

**Overall look:** A believable drawing app. Dark chrome, paper-toned canvas, tool decorations.

**Top toolbar (dark background):**
- Left: "✏️ SketchPad" title in white
- Right: fake tool icon buttons (🖊 🖌 ◯) — styled but not interactive

**Canvas area:**
- Background: warm off-white (`#f5f0e8`) — looks like paper
- The hand-drawn sketch SVG is centered here
- On load: blank (no sketch shown until spectator picks)
- On selection received: sketch appears instantly (no animation)
- On selection change: old sketch replaced immediately with new one
- On erase: canvas goes blank

**Bottom toolbar (dark background):**
- Left: three decorative color dot swatches (black, brown, grey) — not interactive
- Right: **🗑 Erase** button — red, tappable. Writes `null` to Firebase and blanks the canvas locally

**SSE Listener:** Uses `EventSource` on `${FIREBASE_URL}/learnToDraw/selection.json`. Unlike AnySearch, the initial `put` event is NOT skipped — it is processed immediately so that if the magician's phone refreshes mid-trick (after spectator has already picked), the correct sketch reappears. On every `put` or `patch` event, reads `payload.data` (the selection string or null) and renders the matching sketch.

## The 12 Hand-Drawn Sketches

Each subject is an inline SVG rendered inside the canvas area. All sketches use:
- Wobbly, slightly imperfect paths (not geometric-perfect straight lines)
- Double/ghost strokes at low opacity to simulate pencil layering
- `stroke-linecap: round`, `stroke-linejoin: round`
- Stroke color: `#2c2c2c` (dark pencil grey)
- No fill — outline only

| Key | Subject |
|---|---|
| `house` | Peaked roof triangle + rectangular body + door arch + two cross-divided windows |
| `flower` | Circle center + 8 rounded petal loops + short stem + two leaves |
| `tree` | Rounded irregular cloud-blob canopy + short trunk |
| `car` | Rounded rectangular body + cab bump + 4 wheels (circles) + windows |
| `airplane` | Elongated fuselage + swept wings + tail fin + windows |
| `star` | 5-pointed star (hand-drawn, slightly uneven) |
| `sun` | Circle center + 8 wavy rays radiating out |
| `heart` | Classic heart shape, slightly asymmetric |
| `smiley` | Circle face + two dot eyes + curved smile |
| `clock` | Circle face + 12 tick marks + hour/minute hands pointing to ~10:10 |
| `stickman` | Circle head + straight body + two arms angled out + two legs angled down |
| `rocket` | Pointed nose cone + cylindrical body + two side fins + flame at base |

## Error Handling

- If Firebase write fails (no network): button still highlights locally, drawing pad still appears — spectator experience unaffected. Phone just won't update.
- If Firebase SSE drops: `receiver.html` attempts to reconnect (EventSource does this automatically).
- If `localStorage` is unavailable: not used in this app — no impact.

## Out of Scope

- User accounts or auth on the receiver page
- Sound effects
- The spectator's drawing being sent to the magician
- Multiple simultaneous performances
