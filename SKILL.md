---
name: canvas-editor
description: >
  Transform any content module into a free-form canvas editor where items can be
  dragged, resized, snapped to alignment guides, and synced to a preview view.
  Use this skill whenever the user wants to build a "drag and drop canvas",
  "free layout editor", "canvas editing mode", "PPT-style layout", "visual
  page builder", "movable card grid", or any kind of WYSIWYG spatial editor.
  Also trigger when the user asks to convert a static grid/list into an
  editable canvas, or mentions "drag to rearrange" / "resize handles" /
  "alignment guides" / "画布编辑" / "自由拖拽布局".
---

# Canvas Editor — Universal Guide

Build a free-form canvas editor where items can be dragged, resized, snapped,
and previewed. This skill provides the architecture and algorithms; the agent
must adapt them to the target project's framework and styling approach.

---

## Philosophy

This is a **pattern document**, not a copy-paste library. Every project has its
own tech stack, styling system, and data layer. The agent implementing this
skill must:

1. **Understand the WHY** behind each pattern — then translate to the project's idiom
2. **Choose the right storage** — key-value config, REST API, database, localStorage
3. **Adapt the DOM/CSS** — Tailwind, CSS Modules, styled-components, vanilla CSS
4. **Respect the project's conventions** — component structure, state management, naming

All code examples are **pseudocode/React-flavored** — the agent must translate.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Canvas Container                   │
│  ┌───────────────────────────────────────────────┐  │
│  │              Edit Mode (isEditMode=true)       │  │
│  │  • Absolute-positioned cards                  │  │
│  │  • Drag: onPointerDown → onPointerMove        │  │
│  │  • Resize: corner handles with pointer events │  │
│  │  • Snap: alignment guides (5px threshold)     │  │
│  │  • Draggable text elements (label, title)     │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │            Preview Mode (isEditMode=false)     │  │
│  │  • Percentage-based absolute positioning      │  │
│  │  • Individual scroll-reveal per card          │  │
│  │  • Matches edit layout perfectly              │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Data flow:**
```
Edit drag/resize → positions state → Save → Storage
Storage → Load → positions state → Preview rendering
```

Refer to `references/architecture.md` for the detailed component tree and data
flow diagram.

---

## Phase 1: Data Model

Define the position type and defaults. This is the foundation — everything else
builds on it.

```ts
type CardPosition = {
  left: number   // px from canvas left edge
  top: number    // px from canvas top edge
  w: number      // card width in px
  h: number      // card height in px
}
```

**Storage format:** Serialize to JSON string for persistence.
```json
[
  { "left": 28, "top": 100, "w": 460, "h": 345 },
  { "left": 512, "top": 80, "w": 330, "h": 440 }
]
```

**Default positions:** Provide sensible initial values so the preview doesn't
look broken before the user customizes. The agent should compute defaults based
on the actual card count and desired layout (e.g., a staggered editorial grid).

**State initialization pattern:**
```
try to load saved positions from storage
  → if found, parse JSON
  → if not found or parse error, use defaults
```

**CRITICAL — Storage abstraction:**
Do NOT hardcode a specific storage backend. Provide an interface the agent
adapts:

```ts
// The agent decides how to implement these based on the project:
function loadPositions(key: string): CardPosition[] | null   // read from wherever
function savePositions(key: string, data: CardPosition[]): Promise<void>  // write to wherever
```

Common patterns the agent should consider:
- **Supabase/Postgres:** `site_config` table with key-value rows, save as JSON text
- **REST API:** `PUT /api/config/:key` with JSON body
- **localStorage:** `localStorage.setItem(key, JSON.stringify(data))`
- **URL hash:** For shareable layouts, encode in URL

Also store positions for any draggable text elements (section title, labels):
```
curated_title_pos → { left, top }
curated_label_pos → { left, top }
```

---

## Phase 2: Edit Mode — Drag

### Core Interaction

1. `onPointerDown` on card → record offset (`mouseX - cardLeft`, `mouseY - cardTop`)
2. Canvas-level `onPointerMove` → update card position to `(mouseX - offsetX, mouseY - offsetY)`
3. `onPointerUp` → clear dragging state

**Why canvas-level onPointerMove?**
Individual card `onPointerMove` only fires when the pointer is over that
card. Fast drags cause the pointer to leave the card → movement stops. Always
put `onPointerMove` and `onPointerUp` on the **canvas container**, not on
individual cards.

### Pointer Event Traps (learned the hard way)

These bugs will silently break your implementation. Read carefully.

**Trap 1: setPointerCapture kills all other interactions.**
Calling `element.setPointerCapture(pointerId)` makes that element receive ALL
subsequent pointer events — including `pointerup`. The canvas-level
`onPointerUp` never fires → all drag state cleanup is skipped → next
interaction is broken. **Do NOT use setPointerCapture for card/object drag.**

Instead, to prevent other elements from stealing the drag:
- Check drag state in conflicting elements' `onPointerDown` and bail out
- Use `e.stopPropagation()` on the dragged element's `onPointerDown`

```tsx
// Card onPointerDown — bail if text is being dragged
onPointerDown={(e) => {
  if (labelDragging || titleDragging) return  // ← don't steal the drag
  if (e.target.dataset.handle) return         // ← resize handles get their own handler
  // ... start card drag
}}
```

**Trap 2: Resize drift from mutable state reference.**
When resizing, the card's position must remain anchored. If you read
`positions[idx].left` on every frame (which was modified by the previous
frame), the reference point drifts. **Save the anchor position when resize
starts, and use that throughout:**

```tsx
// ✅ Correct: use saved anchor
onResizeStart: save { sx: cardLeft, sy: cardTop, sw: cardWidth, sh: cardHeight }
onResizeMove:  compute new size from (mousePos - sx, mousePos - sy), not from positions[idx]
onResizeEnd:   write final { left, top, w, h } to positions
```

**Trap 3: Rounded corners clip resize handles.**
If the card has `rounded-2xl overflow-hidden`, the corner handles are inside
the border-radius clipping zone → unclickable. Solution: two-layer card
structure.

```
┌─ Outer div (overflow: visible) ← handles live here, not clipped
│  ├─ Inner div (rounded-2xl overflow-hidden) ← clips image/overlay only
│  ├─ Handle NW ─┐
│  ├─ Handle NE  │ ← outside clipping zone
│  └─ …          │
└────────────────┘
```

```tsx
// Outer: no overflow clipping, handles drag events
<div style={{ position: "absolute", left, top, width, height }}
     onPointerDown={dragHandler}>
  {/* Inner: clips content, pointer-events-none so events pass through */}
  <div className="rounded-2xl overflow-hidden pointer-events-none"
       style={{ position: "absolute", inset: 0 }}>
    <img ... />
    <div className="overlay ...">...</div>
  </div>
  {/* Handles: z-30 so they're above everything */}
  {corners.map(corner => (
    <div data-handle={corner}
         style={{ position: "absolute", [corner]: "-4px", zIndex: 30 }}
         onPointerDown={resizeStart} />
  ))}
</div>
```

Make handles visible on hover: `opacity-0 group-hover:opacity-100`
Use semi-transparent white (`rgba(255,255,255,0.12)`) for subtle visibility.

Refer to `references/interactions.md` for the complete drag and resize
algorithm with edge cases.

---

## Phase 3: Edit Mode — Snap Alignment

When dragging a card, compare its 6 reference lines (left, centerX, right,
top, centerY, bottom) against the same 6 lines of every other card. If distance
< 5px, snap to alignment.

**Priority order** (center alignment beats edge alignment):
```
Horizontal: centerX > left > right > left↔right (edge-to-edge)
Vertical:   centerY > top > bottom > top↔bottom (edge-to-edge)
```

**Algorithm** (refer to `references/interactions.md` for full pseudocode):
```ts
function calcSnap(rx, ry, cardW, cardH, selfIdx, allCards):
  guides = {}
  for each other card:
    check horizontal alignments → snap rx, record guides.x
    check vertical alignments   → snap ry, record guides.y
  return { snappedX, snappedY, guides }
```

**Rendering guides:** Two absolutely positioned divs with `z-index: 40`,
`pointer-events-none`, pink glow (`box-shadow: 0 0 6px rgba(255,107,157,0.6)`).
Vertical guide: full height, 1px wide at `guides.x`. Horizontal guide: full
width, 1px tall at `guides.y`.

---

## Phase 4: Edit Mode — Draggable Text Elements

Section titles and labels should be movable within the canvas too. These are
simpler than cards (no resize, just drag).

**Pattern:**
```
State: textPos = { left, top }
State: textDragging = { offsetX, offsetY, moved: boolean } | null
```

The `moved` flag implements a **3px dead zone**: if the pointer moves less than
3px before release, treat it as a click (for inline text editing). If ≥3px,
treat as a drag.

```tsx
onPointerMove:
  if !textDragging: return
  distance = abs(mouseX - offsetX - textPos.left) + abs(mouseY - offsetY - textPos.top)
  if distance > 3: mark moved = true
  if moved: update textPos to (mouseX - offsetX, mouseY - offsetY)

onPointerUp:
  clear textDragging  // no save here — user must click Save button explicitly
```

**All saves happen through a single explicit "Save" button.**

**Also include in the main Save button** so batch save covers everything.

---

## Phase 5: Coordinate System — The Key to Preview Sync

This is the most subtle part. Edit mode uses absolute pixel positions.
Preview mode must reproduce the exact same layout, but responsively.

### The Problem

Edit canvas width ≠ preview canvas width (responsive). If you save `left: 512`
and the edit canvas was 1200px wide, what does 512 mean on a 900px screen?

### The Solution: Percentage-Based Preview

**Step 1: Compute the coordinate bounds.**
These represent the tight-fitting bounding box of all positioned elements.
Use the SAME formula for both edit min-height and preview ratio — if they
differ by even 1px, positions will shift.

```ts
maxRight  = max(allCards.left + allCards.w,  labelPos.left + 300,  titlePos.left + 200)
maxBottom = max(allCards.top  + allCards.h,  titlePos.top  + 60,   labelPos.top + 30)

// Edit mode canvas minHeight:
canvasBottom = max(maxBottom, 700)  // floor prevents tiny canvas
```

**Step 2: Preview container.**
The preview must have the SAME aspect ratio as the edit canvas.

```html
<!-- Outer wrapper: caps width to maxRight for exact match -->
<div style="max-width: {maxRight}px; margin: 0 auto;">
  <!-- Inner: aspect ratio = canvasBottom / maxRight -->
  <div style="width: 100%; padding-top: {(canvasBottom / maxRight) * 100}%; position: relative;">
    <!-- Content space -->
    <div style="position: absolute; inset: 0;">
      <!-- All elements positioned with percentages -->
    </div>
  </div>
</div>
```

**Why `max-width: maxRight`?** It makes the container exactly `maxRight` px
wide when the screen is wide enough → pixel-perfect layout match. On narrow
screens it scales down proportionally.

**Why `padding-top` percentage?** CSS `padding-top: X%` is relative to the
containing block's width. With the outer wrapper capping width at `maxRight`,
`padding-top: (canvasBottom/maxRight)%` creates exactly the right height.

**Step 3: Position elements with percentages.**
All elements (cards, label, title) use the SAME coordinate basis:

```
Horizontal: left%  = (element.left  / maxRight)  * 100
            width% = (element.w     / maxRight)  * 100
Vertical:   top%   = (element.top  / canvasBottom) * 100
            height% = (element.h   / canvasBottom) * 100
```

**CRITICAL:** Always use `canvasBottom` (not `maxBottom`) for vertical
denominators. `canvasBottom` includes the 700px floor, matching the edit mode
canvas exactly. Using `maxBottom` directly introduces a mismatch when the
layout is small.

Refer to `references/preview-sync.md` for the derivation and edge cases.

---

## Phase 6: Preview Mode — Individual Scroll Animation

Instead of revealing the entire section at once, each card should slide in
independently as the user scrolls — this creates a layered, editorial feel.

### Pattern: Per-Card IntersectionObserver

Each card gets its own observer. The wrapper div has explicit dimensions so
the observer can compute intersection correctly.

```tsx
function PreviewCard({ position, bounds, canvasBottom, children }) {
  const ref = useRef(null)
  useEffect(() => {
    const el = ref.current
    if (!el) return
    const obs = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          el.classList.add("visible")  // triggers CSS transition
          obs.disconnect()             // fire once only
        }
      },
      { threshold: 0.15 }  // trigger when 15% visible
    )
    obs.observe(el)
    return () => obs.disconnect()
  }, [])

  return (
    <div ref={ref} class="reveal" style={{
      position: "absolute",
      left: `${(pos.left / bounds.maxRight) * 100}%`,
      top: `${(pos.top / canvasBottom) * 100}%`,
      width: `${(pos.w / bounds.maxRight) * 100}%`,
      height: `${(pos.h / canvasBottom) * 100}%`,
    }}>
      {children}
    </div>
  )
}
```

### CSS Animation

```css
.reveal {
  opacity: 0;
  transform: translateY(60px);
  transition: opacity 1s cubic-bezier(.165,.84,.44,1),
              transform 1s cubic-bezier(.165,.84,.44,1);
}
.reveal.visible {
  opacity: 1;
  transform: translateY(0);
}
```

⚠️ Do NOT wrap the preview canvas in a single RevealOnScroll — if each card's
container collapses to 0 height (because its content is absolutely positioned),
the IntersectionObserver never fires and nothing appears.

---

## Phase 7: Save & Load

### Save Trigger

**Manual only.** A single "Save" button batch-saves all positions.
No auto-save on drag end — the user decides when changes are final.

### Save Implementation

```ts
async function saveAll() {
  await Promise.all([
    saveToStorage("canvas_positions", JSON.stringify(positions)),
    saveToStorage("canvas_title_pos", JSON.stringify(titlePos)),
    saveToStorage("canvas_label_pos", JSON.stringify(labelPos)),
  ])
  // Trigger any global refresh / cache invalidation
  dispatchCustomEvent("config-updated")
}
```

### Load Implementation

On component mount, load from storage into state. Also sync when storage
changes externally (e.g., another tab updates it).

```ts
useEffect(() => {
  const saved = loadFromStorage("canvas_positions")
  if (saved) setPositions(JSON.parse(saved))
}, [storageChangeSignal])
```

---

## Implementation Checklist

When implementing this skill in a project, work through these phases in order:

- [ ] **P1:** Data model + state (positions, defaultPositions)
- [ ] **P2:** Edit canvas — drag (pointer events on canvas, not cards)
- [ ] **P3:** Edit canvas — resize (anchored, no drift, handles outside clip zone)
- [ ] **P4:** Edit canvas — snap alignment (5px threshold, center-priority)
- [ ] **P5:** Draggable text elements (label, title; 3px dead zone)
- [ ] **P6:** Coordinate system (maxRight, canvasBottom, unified formula)
- [ ] **P7:** Preview mode (percentage absolute positioning, per-card animation)
- [ ] **P8:** Save/Load (storage abstraction, manual save button only)
- [ ] **P9:** Storage implementation (choose based on project: DB, API, localStorage)
- [ ] **P10:** Polish (handle hover states, mobile gestures, loading states)

---

## Reference Files

- `references/architecture.md` — Component tree, state diagram, event flow
- `references/interactions.md` — Complete drag/resize/snap algorithms with pseudocode
- `references/preview-sync.md` — Coordinate system derivation and common pitfalls
