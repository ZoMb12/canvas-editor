# Canvas Editor — Architecture Reference

## Component Tree

```
Page/Section
├── Header (save button — edit mode only)
├── Canvas Container
│   ├── Edit Mode (isEditMode=true)
│   │   ├── CanvasCard[] (draggable, resizable)
│   │   │   ├── Content layer (image + overlay, overflow-hidden)
│   │   │   ├── Resize handles ×4 (z-30, outside clip zone)
│   │   │   └── Size label (hover-visible)
│   │   ├── DraggableLabel (text + decorative line)
│   │   ├── DraggableTitle (text, editable)
│   │   └── Alignment guides (conditional, z-40)
│   └── Preview Mode (isEditMode=false)
│       ├── Label (static, positioned by saved coords)
│       ├── Title (static, positioned by saved coords)
│       └── PreviewCard[] (individual scroll animation)
└── Save Button (edit mode only, right-aligned)
```

## State Shape

```
positions:    CardPosition[]   // [{left, top, w, h}, ...] — 5 entries
dragging:     {idx, ox, oy} | null
resizing:     {idx, dir, sx, sy, sw, sh} | null
guides:       {x?, y?} | null
titlePos:     {left, top}
titleDragging:{ox, oy, moved} | null
labelPos:     {left, top}
labelDragging:{ox, oy, moved} | null
saving:       boolean
```

## Computed Values

```
previewBounds = {
  maxRight:  max(cardRights..., labelRight, titleRight, 100)
  maxBottom: max(cardBottoms..., titleBottom, labelBottom, 100)
}
canvasBottom = max(previewBounds.maxBottom, 700)
```

## Event Flow

### Card Drag
```
pointerdown on card
  → if handle: return (handled by resize)
  → if label/title dragging: return (don't steal)
  → preventDefault()
  → setDragging({idx, ox, oy})

pointermove on canvas
  → if titleDragging → applyTextDrag(...)
  → if labelDragging → applyTextDrag(...)
  → if dragging → calcSnap(...) → setPositions + setGuides
  → if resizing → compute new size → setPositions

pointerup on canvas
  → clear all drag/resize states + guides
```

### Card Resize
```
pointerdown on handle
  → stopPropagation() (don't trigger card drag)
  → preventDefault()
  → read card rect → setResizing({idx, dir, sx, sy, sw, sh})

pointermove on canvas
  → compute new w/h from mouse position relative to anchor (sx, sy)
  → setPositions(update card at idx)
```

### Text Drag (label/title)
```
pointerdown on text element
  → stopPropagation()
  → setTextDragging({ox, oy, moved: false})

pointermove on canvas
  → applyTextDrag(drag, pos, setDrag, setPos, mx, my)
     → if distance > 3px: mark moved=true
     → if moved: update pos

pointerup on text element
  → clear textDragging (no save — manual Save button only)
```

## Config Sync Pattern

```tsx
// Load from storage on mount and when config changes externally
useEffect(() => {
  const sync = (key, setter) => {
    if (config[key]) {
      try { setter(JSON.parse(config[key])); }
      catch { /* keep defaults */ }
    }
  }
  sync("canvas_positions", setPositions)
  sync("canvas_title_pos", setTitlePos)
  sync("canvas_label_pos", setLabelPos)
}, [config.canvas_positions, config.canvas_title_pos, config.canvas_label_pos])
```

## File Organization (suggested)

```
src/
├── components/
│   └── canvas/
│       ├── CanvasContainer.tsx    — edit mode canvas wrapper
│       ├── CanvasCard.tsx         — single draggable/resizable card
│       ├── PreviewCanvas.tsx      — preview mode canvas
│       ├── PreviewCard.tsx        — per-card scroll animation
│       ├── DraggableText.tsx      — reusable text drag component
│       ├── AlignmentGuides.tsx    — snap guides overlay
│       └── SaveButton.tsx         — batch save with loading state
├── hooks/
│   └── useCanvasState.ts         — all state + computed values
└── lib/
    └── canvas-utils.ts            — calcSnap, applyTextDrag, coordinate helpers
```
