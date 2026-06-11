# Canvas Editor — Interaction Algorithms

## Card Drag

```
onPointerDown(cardElement, event):
  if event.target.dataset.handle: return           // resize handle, skip
  if labelDragging or titleDragging: return        // text is being dragged
  event.preventDefault()                           // prevent text selection
  cardRect = cardElement.getBoundingClientRect()
  canvasRect = canvasElement.getBoundingClientRect()
  ox = event.clientX - cardRect.left               // offset within card
  oy = event.clientY - cardRect.top
  setDragging({ idx: cardIndex, ox, oy })

onPointerMove(canvasElement, event):
  canvasRect = canvasElement.getBoundingClientRect()
  mx = event.clientX - canvasRect.left
  my = event.clientY - canvasRect.top

  if dragging:
    rawX = mx - dragging.ox
    rawY = my - dragging.oy
    { rx, ry, guides } = calcSnap(rawX, rawY, cardW, cardH, dragging.idx, allCards)
    setGuides(guides)
    updatePositions(dragging.idx, { left: rx, top: ry })

onPointerUp():
  setDragging(null)
  setGuides(null)
```

### Why canvas-level onPointerMove?

**Wrong approach:** `onPointerMove` on each card.
Problem: when the user drags fast, the pointer leaves the card element. The
event stops firing → drag freezes. Only fires again when the pointer re-enters
the card.

**Correct approach:** `onPointerMove` on the canvas container.
The canvas covers all cards, so events fire continuously regardless of where
the pointer is within the canvas.

Same for `onPointerUp` — put it on the canvas, not on cards. Otherwise
releasing the mouse outside a card doesn't trigger cleanup.

---

## Card Resize

### Resize Start

```
onPointerDown(handleElement, event):
  event.stopPropagation()      // don't trigger card drag
  event.preventDefault()
  card = handleElement.parentElement
  canvasRect = card.parentElement.getBoundingClientRect()
  cardRect = card.getBoundingClientRect()
  setResizing({
    idx: cardIndex,
    dir: "nw" | "ne" | "sw" | "se",
    sx: cardRect.left - canvasRect.left,    // anchor X
    sy: cardRect.top - canvasRect.top,      // anchor Y
    sw: cardRect.width,                     // start width
    sh: cardRect.height,                    // start height
  })
```

### Resize Move

The key insight: **always compute from the anchor point (sx, sy), never from
the current positions state.** Reading `positions[idx].left` in the move
handler gives the value from the PREVIOUS frame (already modified), causing
cumulative drift.

```
onPointerMove:
  if resizing:
    r = resizing
    w = r.sw, h = r.sh
    x = r.sx, y = r.sy          // ← START from anchor, not positions[idx]

    if r.dir includes "e":  w = max(100, mouseX - x)     // right edge moves
    if r.dir includes "w":  w = max(100, x + r.sw - mouseX); x = mouseX  // left edge moves
    if r.dir includes "s":  h = max(100, mouseY - y)     // bottom edge moves
    if r.dir includes "n":  h = max(100, y + r.sh - mouseY); y = mouseY  // top edge moves

    // OPTIONAL: lock aspect ratio
    // if (r.dir is horizontal) h = round(w * r.startH / r.startW)
    // if (r.dir is vertical)   w = round(h * r.startW / r.startH)

    updatePositions(r.idx, { left: x, top: y, w, h })
```

The `100` minimum prevents cards from becoming invisibly small.

---

## Snap Alignment

### Reference Lines

Each card has 6 reference lines:
- Horizontal: left edge, centerX, right edge
- Vertical: top edge, centerY, bottom edge

### Algorithm

```
const SNAP_THRESHOLD = 5  // pixels

function calcSnap(rx, ry, cardW, cardH, selfIdx, allCards):
  guides = {}
  myLines = {
    left: rx, centerX: rx + cardW/2, right: rx + cardW,
    top: ry,  centerY: ry + cardH/2, bottom: ry + cardH,
  }

  for each otherCard (skip selfIdx):
    oLines = {
      left: otherCard.left, centerX: otherCard.left + otherCard.w/2,
      right: otherCard.left + otherCard.w,
      top: otherCard.top, centerY: otherCard.top + otherCard.h/2,
      bottom: otherCard.top + otherCard.h,
    }

    // Horizontal (center priority > edge)
    if |myLines.centerX - oLines.centerX| < SNAP:
      rx = oLines.centerX - cardW/2; guides.x = oLines.centerX
    else if |myLines.left - oLines.left| < SNAP:
      rx = oLines.left; guides.x = oLines.left
    else if |myLines.right - oLines.right| < SNAP:
      rx = oLines.right - cardW; guides.x = oLines.right
    else if |myLines.left - oLines.right| < SNAP:
      rx = oLines.right; guides.x = oLines.right       // my-left to their-right
    else if |myLines.right - oLines.left| < SNAP:
      rx = oLines.left - cardW; guides.x = oLines.left  // my-right to their-left

    // Vertical (center priority > edge)
    if |myLines.centerY - oLines.centerY| < SNAP:
      ry = oLines.centerY - cardH/2; guides.y = oLines.centerY
    else if |myLines.top - oLines.top| < SNAP:
      ry = oLines.top; guides.y = oLines.top
    else if |myLines.bottom - oLines.bottom| < SNAP:
      ry = oLines.bottom - cardH; guides.y = oLines.bottom
    else if |myLines.top - oLines.bottom| < SNAP:
      ry = oLines.bottom; guides.y = oLines.bottom
    else if |myLines.bottom - oLines.top| < SNAP:
      ry = oLines.top - cardH; guides.y = oLines.top

  hasGuides = (guides.x != null || guides.y != null)
  return { rx, ry, guides: hasGuides ? guides : null }
```

### Guide Rendering

```tsx
{guides?.x != null && (
  <div style={{
    position: "absolute", top: 0, bottom: 0,
    left: guides.x, width: "1px",
    background: "#FF6B9D",
    boxShadow: "0 0 6px rgba(255,107,157,0.6)",
    zIndex: 40, pointerEvents: "none",
  }} />
)}
{guides?.y != null && (
  <div style={{
    position: "absolute", left: 0, right: 0,
    top: guides.y, height: "1px",
    background: "#FF6B9D",
    boxShadow: "0 0 6px rgba(255,107,157,0.6)",
    zIndex: 40, pointerEvents: "none",
  }} />
)}
```

---

## Text Drag (Label/Title)

### Shared helper function

```ts
function applyTextDrag(drag, pos, setDrag, setPos, mouseX, mouseY):
  if !drag: return

  dx = mouseX - drag.ox - pos.left
  dy = mouseY - drag.oy - pos.top
  moved = drag.moved || Math.abs(dx) > 3 || Math.abs(dy) > 3

  if moved and !drag.moved:
    setDrag(prev => prev ? { ...prev, moved: true } : null)

  if moved:
    setPos({ left: mouseX - drag.ox, top: mouseY - drag.oy })
```

The 3px dead zone allows inline text editing: a short click (< 3px movement)
won't trigger a position save, preserving the click for text editing.

### Element setup

```tsx
<div
  style={{ position: "absolute", left: textPos.left, top: textPos.top, zIndex: 20 }}
  onPointerDown={(e) => {
    e.stopPropagation()  // prevent canvas/card drag
    const rect = e.currentTarget.getBoundingClientRect()
    setTextDragging({ ox: e.clientX - rect.left, oy: e.clientY - rect.top, moved: false })
  }}
  onPointerUp={() => {
    if (textDragging?.moved) saveToStorage("text_pos", JSON.stringify(textPos))
    setTextDragging(null)
  }}
>
  <EditableText ... />
</div>
```

**Critical: Do NOT use `setPointerCapture`.** It prevents the canvas-level
`pointermove` from firing, which breaks everything. Instead, rely on: (a)
`stopPropagation` to prevent card drag, (b) card's `onPointerDown` checking
`if (textDragging) return` to bail out.

---

## Resize Handle Visibility

Handles should be invisible by default, appear on hover:

```css
.resize-handle {
  opacity: 0;
  transition: opacity 200ms;
  width: 16px; height: 16px;
  background: rgba(255, 255, 255, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 2px;
}
.card:hover .resize-handle { opacity: 1; }
```

Position handles at corners with -4px offset so they extend slightly outside
the card boundary (making them easier to grab):

```
nw: top: -4px,  left: -4px,  cursor: nw-resize
ne: top: -4px,  right: -4px, cursor: ne-resize
sw: bottom: -4px, left: -4px, cursor: sw-resize
se: bottom: -4px, right: -4px, cursor: se-resize
```
