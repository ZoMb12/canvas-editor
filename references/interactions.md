# Canvas Editor — Interaction Algorithms

通用交互算法，Agent 需根据项目技术栈适配。

---

## Card Drag

### Core Interaction

1. `onPointerDown` on card → record offset (`mouseX - cardLeft`, `mouseY - cardTop`)
2. Canvas-level `onPointerMove` → update position to `(mouseX - offsetX, mouseY - offsetY)`
3. Canvas-level `onPointerUp` → clear dragging state

### Algorithm

```
onPointerDown(cardElement, event):
  if event.target.dataset.handle: return           // resize handle, skip
  if labelDragging or titleDragging: return        // text is being dragged, skip
  event.preventDefault()                           // prevent text selection
  cardRect = cardElement.getBoundingClientRect()
  ox = event.clientX - cardRect.left               // offset within card
  oy = event.clientY - cardRect.top
  setDragging({ idx: cardIndex, ox, oy })

onPointerMove(canvasElement, event):
  canvasRect = canvasElement.getBoundingClientRect()
  mx = event.clientX - canvasRect.left
  my = event.clientY - canvasRect.top

  if textDragging: applyTextDrag(...)
  if labelDragging: applyTextDrag(...)

  if dragging:
    rawX = mx - dragging.ox
    rawY = my - dragging.oy
    { rx, ry, guides } = calcSnap(rawX, rawY, cardW, cardH, dragging.idx, allCards)
    setGuides(guides)
    updatePositions(dragging.idx, { left: rx, top: ry })

  if resizing:
    compute new w/h relative to resize anchor (sx, sy)
    updatePositions(resizing.idx, { left, top, w, h })

onPointerUp():
  clear dragging, resizing, guides, textDragging, labelDragging
```

> **ZoMble 实现位置:** `src/components/sections/EditorSelection.tsx` 中 `onPointerMove` / `onPointerUp` / `onPointerDown`

### Why canvas-level onPointerMove?

**Wrong approach:** `onPointerMove` on each card.
❌ When the user drags fast, the pointer leaves the card → event stops firing → drag freezes.

**Correct approach:** `onPointerMove` on the canvas container.
✅ The canvas covers all cards, so events fire continuously regardless of pointer position.

Same for `onPointerUp` — put it on the canvas, not on cards. Otherwise releasing
the mouse outside a card doesn't trigger cleanup.

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
    sx: cardRect.left - canvasRect.left,    // anchor X (relative to canvas)
    sy: cardRect.top - canvasRect.top,      // anchor Y (relative to canvas)
    sw: cardRect.width,                     // start width
    sh: cardRect.height,                    // start height
  })
```

> **ZoMble 实现位置:** `src/components/home/CanvasCard.tsx` resize handles' `onPointerDown`

### Resize Move — 关键陷阱：锚点漂移

**始终从锚点 (sx, sy) 计算，不要从当前 positions state 读取。**
读取 `positions[idx].left` 会拿到上一次被修改过的值，导致累积漂移。

```
onPointerMove:
  if resizing:
    r = resizing
    w = r.sw, h = r.sh
    x = r.sx, y = r.sy          // ← START from anchor, not positions[idx]

    if r.dir includes "e":  w = max(100, mouseX - x)     // right edge moves
    if r.dir includes "w":  w = max(100, x + r.sw - mouseX); x = mouseX
    if r.dir includes "s":  h = max(100, mouseY - y)
    if r.dir includes "n":  h = max(100, y + r.sh - mouseY); y = mouseY

    updatePositions(r.idx, { left: x, top: y, w, h })
```

Minimum `100px` prevents cards from becoming invisibly small.

> **ZoMble 实现位置:** `src/components/sections/EditorSelection.tsx` 中 resizing 分支

### 双层 DOM 结构（防圆角裁剪）

卡片有 `rounded-2xl overflow-hidden` 时，角落的 resize handle 会被裁掉 → 不可点击。
解决方案：外层 div 不 clip，内层 div 负责 clip 图片和 overlay。

```
┌─ Outer div (overflow: visible) ← handles 放这里，不被裁剪
│  ├─ Inner div (rounded-2xl overflow-hidden) ← clip 图片/overlay
│  │   pointer-events: none (让事件透传给外层)
│  ├─ Handle NW ─┐
│  ├─ Handle NE  │ ← 在裁剪区之外
│  └─ …          │
└────────────────┘
```

```tsx
// Outer
<div style={{ position: "absolute", left, top, width, height }}
     onPointerDown={dragHandler}>
  {/* Inner: clips content only */}
  <div className="rounded-2xl overflow-hidden pointer-events-none"
       style={{ position: "absolute", inset: 0 }}>
    <img ... />
    <div className="overlay">...</div>
  </div>
  {/* Handles: z-30, outside clipping zone */}
  {corners.map(corner => (
    <div data-handle={corner}
         style={{ position: "absolute", [corner]: "-4px", zIndex: 30 }}
         onPointerDown={resizeStart} />
  ))}
</div>
```

> **ZoMble 实现位置:** `src/components/home/CanvasCard.tsx`

---

## Snap Alignment

### Reference Lines

每张卡片 6 条参考线:
- 水平: left edge, centerX, right edge
- 垂直: top edge, centerY, bottom edge

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

  return { rx, ry, guides: (guides.x != null || guides.y != null) ? guides : {} }
```

> **ZoMble 实现位置:** `src/lib/curatedCanvas.ts` → `calcSnap()`

### Guide Rendering

粉色发光对齐线：

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

### Shared Helper

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

3px dead zone 允许内联文字编辑：短点击（<3px）不会触发拖拽，保留点击事件用于文字编辑。

> **ZoMble 实现位置:** `src/lib/curatedCanvas.ts` → `applyTextDrag()`

### Element Setup

```tsx
<div
  style={{ position: "absolute", left: textPos.left, top: textPos.top, zIndex: 20 }}
  onPointerDown={(e) => {
    e.stopPropagation()  // prevent canvas/card drag
    setTextDragging({ ox, oy, moved: false })
  }}
  onPointerUp={() => setTextDragging(null)}
>
  {/* 可编辑文字内容 */}
</div>
```

**Do NOT use `setPointerCapture`.** It prevents canvas-level `pointermove` from
firing. Instead use: (a) `stopPropagation` to prevent card drag, (b) card's
`onPointerDown` checking `if (textDragging) return`.

---

## Resize Handle Visibility

Handles 默认隐藏，hover 时显示：

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

Handle 定位在四角，-4px 偏移伸出卡片边界，便于抓取：

```
nw: top: -4px,  left: -4px,  cursor: nw-resize
ne: top: -4px,  right: -4px, cursor: ne-resize
sw: bottom: -4px, left: -4px, cursor: sw-resize
se: bottom: -4px, right: -4px, cursor: se-resize
```
