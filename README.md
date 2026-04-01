# What Colour is Hope?

A participatory art installation that invites people to define hope through colour rather than words.

---

## What it does

Visitors stand in front of a screen and move their hand through the air. As they move, the entire scene shifts colour in real time — the room, themselves, everything rendered as shades of a single hue. When the colour feels like hope to them, they pinch their fingers together to select it.

Each chosen colour joins a growing collective display. By the end of an event, the grid becomes a portrait of how differently people imagine and experience the same emotional concept.

---

## How it works

### Hand tracking — MediaPipe Hands

The webcam feed is processed by a machine learning model (running entirely in the browser — nothing is uploaded) that detects 21 landmark points on the hand every frame.

- **Hand position** is read from landmark 9 (middle knuckle — the stable centre of the palm)
- **Pinch** is detected by measuring the distance between landmark 4 (thumb tip) and landmark 8 (index fingertip)

```
distance < 0.05  →  pinch triggered  →  colour saved
```

### Colour mapping — HSL

The hand's position in space maps directly to the HSL colour model:

| Hand movement | Colour effect |
|---|---|
| Left ↔ Right | Hue (which colour: red → orange → yellow → green → blue → violet) |
| Up ↕ Down | Saturation + Lightness (vivid and dark → muted and soft) |

HSL was chosen because it maps naturally to how humans perceive colour — moving your hand feels like moving through the spectrum, not adjusting numbers.

### The duotone / x-ray effect

Every video frame is processed through three steps:

1. Draw the full webcam feed in greyscale with a slight contrast boost
2. Draw a solid fill of the current hope colour on top
3. Apply the `color` canvas blend mode — this keeps the light/dark structure of the video and replaces all hue and saturation with the hope colour

Result: the whole scene (person, background, room) rendered as shades of the chosen colour.

### The collective grid

Each selected colour is stored as an HSL string in an array. The grid redraws itself after every selection, dividing the canvas equally between all saved colours. The most recent entry is outlined in white.

The collection resets when the page is refreshed — intentional for installation use, giving each session a clean start.

---

## Stack

- Vanilla HTML / CSS / JavaScript — single file, no framework, no build step
- [MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker) — hand tracking and gesture detection
- Canvas 2D API — real-time rendering and colour compositing
- Fonts: Playfair Display, Cormorant Garamond, Space Mono (Google Fonts)

No server. No database. No install. Open in a browser with a webcam.

---

## Running it

Open `index.html` in Chrome. Allow camera access when prompted.

**Controls:**
- Move hand left / right — shift hue
- Move hand up / down — shift tone
- Pinch (bring thumb and index finger together) — save colour
- Spacebar or click — save colour (fallback)

---

## Built for

Howler Bar event — part of a larger wall-based display showing how the colours of hope changed across the exhibition as new responses were added each day.
