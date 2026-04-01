# What Colour is Hope?

A participatory art installation that invites people to define hope through colour rather than words.

---

## What it does

Visitors stand in front of a screen and move their hand through the air. As they move, the entire scene shifts colour in real time — the room, themselves, everything rendered as shades of a single hue. When the colour feels like hope to them, they pinch their fingers together to select it.

Each chosen colour joins a growing collective display. By the end of an event, the grid becomes a portrait of how differently people imagine and experience the same emotional concept.

---

## The Model — MediaPipe Hands

We use **MediaPipe Hands**, a pre-trained machine learning model built by Google. It runs entirely inside the browser via WebAssembly — no API calls, no internet required after load, nothing leaves your computer. It was trained on millions of hand images to detect and track hands in real time at ~30 frames per second.

### What it outputs — 21 landmarks

Every single frame from the webcam, the model outputs **21 landmark points** on the hand. Each point is an `{x, y, z}` coordinate:

```
x → horizontal position  (0.0 = left edge of frame,  1.0 = right edge)
y → vertical position    (0.0 = top of frame,         1.0 = bottom)
z → depth                (how close the hand is to the camera — we don't use this)
```

The 21 points map to every joint and tip on the hand:

```
        8   12  16  20       ← fingertips
        |   |   |   |
    4   7   11  15  19       ← middle joints
    |   |   |   |   |
    3   6   10  14  18       ← base joints
    |   |   |   |   |
    2   5   9   13  17       ← knuckles
     \  |   |   |   /
           0                 ← wrist
```

The landmarks we use:

| Landmark | Body part | Role |
|---|---|---|
| **9** | Middle finger MCP (middle knuckle) | Palm centre — drives colour |
| **4** | Thumb tip | Pinch detection |
| **8** | Index fingertip | Pinch detection |

Landmark 9 is used as the palm centre because it sits in the most stable part of the hand — it barely moves when fingers open and close, so the colour doesn't jump when you pinch.

---

## Hand Position → Colour

### Why HSL, not RGB

Colour is stored and calculated in **HSL (Hue, Saturation, Lightness)** rather than RGB. This is because HSL maps directly to how humans perceive colour:

- **Hue** — which colour on the spectrum (0° = red, 120° = green, 240° = blue, 360° = back to red)
- **Saturation** — how vivid vs grey (100% = pure colour, 0% = completely grey)
- **Lightness** — how bright vs dark (0% = black, 50% = true colour, 100% = white)

Moving your hand in RGB would feel arbitrary. Moving in HSL feels like moving through the rainbow — intuitive and continuous.

### The mapping

```
hand x position (0.0 → 1.0)  maps to  hue (0° → 360°)
hand y position (0.0 → 1.0)  maps to  saturation + lightness combined
```

In code:

```javascript
const palm = landmarks[9]

handX = 1 - palm.x   // mirrored — so moving left feels like going left
handY = palm.y

targetHue        = handX * 360                  // full spectrum left to right
targetSaturation = 45 + (1 - handY) * 50        // top = vivid (95%), bottom = muted (45%)
targetLightness  = 28 + handY * 42              // top = dark (28%), bottom = light (70%)
```

The x is mirrored (`1 - palm.x`) because webcam feeds are naturally mirrored — without this, moving your hand left would shift colour in the opposite direction to what feels natural.

---

## Smooth Colour Transitions — Lerp

We do **not** detect motion or velocity. There is no "direction the hand is moving" in the code. Instead, the colour smoothly chases the hand's current position using **linear interpolation (lerp)** applied every frame:

```javascript
currentHue = lerp(currentHue, targetHue, 0.08)

function lerp(a, b, t) {
  return a + (b - a) * t
}
```

What this means: each frame, the current colour moves 8% of the remaining distance toward the target. So if the target hue is 200° and you're currently at 100°:

```
Frame 1:  100 + (200 - 100) × 0.08  =  108°
Frame 2:  108 + (200 - 108) × 0.08  =  115.4°
Frame 3:  115.4 + (200 - 115.4) × 0.08  =  122.1°
...and so on until it reaches 200°
```

Because this runs 30 times per second, it feels fluid and alive — not jumpy. The 8% rate is fast enough to feel responsive, slow enough to feel graceful.

When no hand is detected, the rate drops to 1.3% so the colour drifts very slowly rather than freezing.

---

## Pinch Detection

Pinch is detected purely with distance maths — no gesture classifier, no ML model for this part:

```javascript
const thumbTip = landmarks[4]
const indexTip = landmarks[8]

const distance = Math.hypot(
  thumbTip.x - indexTip.x,
  thumbTip.y - indexTip.y
)

if (distance < 0.05) → pinch = true
```

`Math.hypot` computes the straight-line (Euclidean) distance between the two points:

```
distance = √( (x2 - x1)² + (y2 - y1)² )
```

Since the coordinates are normalised (0.0 to 1.0), a distance of 0.05 means the fingertips are within 5% of the frame width of each other — roughly touching.

To avoid saving the colour repeatedly while held, we only trigger on the **first frame** the pinch crosses the threshold:

```javascript
if (pinching && !lastPinch) selectColour()   // fires once on onset
lastPinch = pinching                          // remember state for next frame
```

---

## The Duotone / X-ray Effect

Every video frame is composited in three steps using the Canvas 2D API:

**Step 1** — Draw the full webcam feed (mirrored) in greyscale with a contrast boost:
```javascript
ctx.filter = 'grayscale(1) contrast(1.15) brightness(0.78)'
ctx.drawImage(video, ...)
```

**Step 2** — Draw a solid rectangle filled with the current hope colour on top, using the `color` blend mode:
```javascript
ctx.globalCompositeOperation = 'color'
ctx.fillStyle = `hsl(${hue}, ${sat}%, 52%)`
ctx.fillRect(0, 0, width, height)
```

The `color` blend mode works like this: it takes the **hue and saturation** from the source (the solid colour fill) and the **luminance** from the destination (the greyscale video). The result is that every pixel in the scene takes on the hope colour while keeping its original brightness.

```
dark pixel in video   →  dark shade of hope colour
bright pixel in video →  bright shade of hope colour
```

**Step 3** — Reset composite mode and apply a vignette (radial darkening toward the edges) for atmosphere.

---

## The Collective Grid

Each selected colour is pushed as an HSL string into an array:

```javascript
savedColors.push(`hsl(${hue}, ${sat}%, ${light}%)`)
```

After each save, the grid redraws. The number of columns is calculated to keep cells roughly square given the canvas proportions:

```javascript
const cols = Math.ceil(Math.sqrt(n * (canvasWidth / canvasHeight)))
const rows = Math.ceil(n / cols)
```

The collection resets when the page is refreshed — intentional for installation use, giving each event a clean start.

---

## Stack

- Vanilla HTML / CSS / JavaScript — single file, no framework, no build step
- [MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker) — hand tracking (runs in-browser via WebAssembly)
- Canvas 2D API — real-time rendering and colour compositing
- Fonts: Playfair Display, Cormorant Garamond, Space Mono (Google Fonts)

No server. No database. No install. Open in a browser with a webcam.

---

## Running it

Open `index.html` in Chrome. Allow camera access when prompted. Requires HTTPS or localhost for camera access (Vercel deployment handles this automatically).

**Controls:**
| Input | Action |
|---|---|
| Move hand left / right | Shift hue across the spectrum |
| Move hand up / down | Shift tone (vivid ↔ soft) |
| Pinch thumb + index together | Save colour |
| Spacebar | Save colour (keyboard fallback) |
| Click canvas | Save colour (mouse fallback) |

---

## Built for

Howler Bar event — part of a larger wall-based display showing how the colours of hope changed across the exhibition as new responses were added each day.
