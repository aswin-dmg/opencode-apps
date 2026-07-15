# हस्तलेख · Hindi Handwriting Lab

An **AI-style, on-device Hindi handwriting evaluation system** for children (ages 3–8), built as a
self-contained web module that plugs into the BhashaLab app. The child draws a Devanagari
character on a canvas; a **hybrid evaluation engine** judges *how well* they wrote it (not just
what it is) and returns playful, constructive feedback.

> Design note: the spec asked for React + FastAPI + OpenCV. That stack cannot run or be verified
> in this environment and would break the single-file app model. This implementation delivers the
> **same functional architecture** (vector capture → normalize → rasterize → compare → score →
> feedback) purely in the browser, with a clean swap-in point for a future backend.

---

## Features

| Requirement | Implementation |
|---|---|
| 4 practice modes | **Watch** (animated stroke reveal) · **Trace** (draw over faint letter) · **Copy** (beside reference) · **Memory** (from recall) |
| Vector capture | Every stroke stored as `{x, y, time, pressure}` via Pointer Events (mouse / touch / Apple Pencil / stylus) |
| Shape similarity | Rasterize child + reference glyph to a 256×256 mask, **IoU with dilation tolerance** (contour + overlap) |
| Stroke coverage | % of reference pixels reached by the child's drawing |
| Stroke order | Compared to `acceptedStrokeOrders` in `characters.json`; never fails, only suggests |
| Stroke direction | Detects Top→Bottom / Bottom→Top / Left→Right / Right→Left per stroke |
| Alignment | Centering, size, and tilt (PCA) vs. the writing guide |
| Completeness | Missing / extra / broken / disconnected strokes via mask analysis |
| Scoring | Shape 40% · Coverage 25% · Alignment 15% · Direction 10% · Order 10% → 0–100 |
| Feedback engine | Child-friendly messages ("Excellent!", "Try making this line a little longer.") — never "Correct/Wrong" |
| Visual feedback | Green = good · Orange = close · Red = off/missing · red dotted ideal outline |
| Progressive difficulty | Easy (big + dotted + wide tolerance) → Medium → Hard (no outline, tight tolerance) |
| Gamification | ⭐ Stars · 🪙 Coins · ✨ XP · 🏅 Badges (Perfect Stroke / Letter / Daily Streak) |
| Analytics | Per-character attempts, average score, common mistakes → **Parent Dashboard** |
| Accessibility | Mouse, touch, pen, pressure sensitivity |
| Performance | Fully local; evaluation runs in well under 300 ms |

---

## Architecture

```
Child draws ─▶ PointerEvents ─▶ strokes[] (vectors)
                                    │
                        normalize + rasterize (256² masks)
                                    │
        ┌───────────────┬───────────┼───────────────┬──────────────┐
     Shape (IoU)   Coverage      Alignment      Direction       Order
       40%           25%           15%             10%            10%
        └───────────────┴───────────┼───────────────┴──────────────┘
                                    ▼
                              Final score 0–100
                                    ▼
                    Feedback engine ─▶ on-screen result + visual colors
```

**Swap-in point for a real backend (FastAPI + OpenCV / TF / ONNX):** replace the body of
`evaluateWithReference()` (currently the in-browser raster logic) with a `fetch('/api/evaluate')`
call. The frontend contract — input `strokes[]`, output `{shape, coverage, alignment, direction,
order, score, feedback}` — stays identical, so no UI changes are required. The future CNN / ViT /
MediaPipe pipeline can live entirely server-side.

---

## Folder structure

```
hindi-handwriting/
├── index.html          # the full module (UI + canvas + engine + gamification + analytics)
├── characters.json     # normalized reference DB (schema below)
└── README.md
```

The module is added to the app via a link in the BhashaLab header/footer
(`index.html` → "✍️ Handwriting" → `hindi-handwriting/index.html`).

---

## characters.json schema

```json
{
  "schemaVersion": "1.0",
  "language": "hi",
  "defaultTolerance": 0.10,
  "characters": [
    {
      "character": "क",
      "name": "ka",
      "example": "क से कमल",
      "strokeCount": 3,
      "referenceStrokes": [ [[x,y],...], ... ],   // normalized 0..1, per stroke
      "acceptedStrokeOrders": [ [0,1,2] ],
      "tolerance": 0.12                          // 0..1 of the writing box
    }
  ]
}
```

Coordinates are **normalized 0–1** (top-left origin) so they scale to any screen. The raster
evaluator works for *every* Devanagari character (and numbers, English, Tamil, etc. in future)
even without a curated entry — the JSON entries only enhance stroke-order / direction feedback
and Trace tolerance.

---

## Installation

No build step. Serve the folder over HTTP (needed for `fetch` of `characters.json`):

```bash
cd hindi-handwriting
python3 -m http.server 8080
# open http://localhost:8080/index.html
```

Or drop it into any static host (Netlify, GitHub Pages, etc.). The page also degrades gracefully
to an embedded DB fallback if `characters.json` can't be fetched.

## Deployment

- **Netlify / static:** point the build at this folder; `index.html` is the entry.
- **BhashaLab integration:** ensure `hindi-handwriting/` sits beside `index.html`; the
  "✍️ Handwriting" link resolves relatively.

---

## Usage

1. Pick a letter (or let it start on **क**).
2. Choose a **mode** (Watch / Trace / Copy / Memory) and **difficulty**.
3. Draw on the canvas.
4. Tap **✅ Check my writing** for score + feedback + colored strokes.
5. Earn stars/coins/badges; parents review progress in **📊 Parents**.

---

## Testing

Open `index.html`, draw **क** and verify:
- Watch animates the letter top→bottom.
- Trace shows a faint guide; drawing over it scores high.
- Copy shows the letter beside an empty canvas.
- Memory hides the reference.
- "Check my writing" returns a 0–100 score, colored strokes, and friendly feedback.
- Stars/coins update in the header; the Parent Dashboard lists per-character progress.

## Future AI version

The modular `evaluateWithReference()` seam is where a CNN / Vision Transformer / ONNX Runtime /
MediaPipe model would attach — running server-side and returning the same metric shape, so the
frontend, gamification, and analytics require no changes.
