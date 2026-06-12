# Canvas Editor — Architecture Reference

通用画布编辑器架构，ZoMble 项目的文件路径作为示例标注。

---

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

**ZoMble files:**
- Canvas container (edit + preview): `src/components/sections/EditorSelection.tsx`
- CanvasCard: `src/components/home/CanvasCard.tsx`
- PreviewCard: `src/components/home/PreviewCard.tsx`
- EditorCard (card content): `src/components/home/EditorCard.tsx`
- Utility functions: `src/lib/curatedCanvas.ts`

---

## State Shape

```ts
// All canvas state lives in the canvas container component

positions:    CardPosition[]   // [{left, top, w, h}, ...]
dragging:     { idx: number; ox: number; oy: number } | null
resizing:     { idx: number; dir: string; sx: number; sy: number; sw: number; sh: number } | null
guides:       { x?: number; y?: number } | null
titlePos:     { left: number; top: number }
titleDragging:{ ox: number; oy: number; moved: boolean } | null
labelPos:     { left: number; top: number }
labelDragging:{ ox: number; oy: number; moved: boolean } | null
saving:       boolean
```

**CardPosition type** (defined in utils):
```ts
type CardPosition = {
  left: number   // px from canvas left edge
  top: number    // px from canvas top edge
  w: number      // card width in px
  h: number      // card height in px
}
```

---

## Computed Values

```ts
// 公式完全统一，编辑和预览使用同一套计算
previewBounds = {
  maxRight:  max(cardRights..., labelRight, titleRight, 100)
  maxBottom: max(cardBottoms..., titleBottom, labelBottom, 100)
}
canvasBottom = max(previewBounds.maxBottom, 700)  // 700px 下限
```

**Denominator rule:**
- 水平方向一律用 `maxRight`
- 垂直方向一律用 `canvasBottom`（不是 maxBottom）
- 编辑模式 canvas minHeight = `canvasBottom`
- 预览模式 aspect ratio = `canvasBottom / maxRight`

---

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
  → if resizing → compute from anchor → setPositions

pointerup on canvas
  → clear all drag/resize states + guides
```

### Card Resize
```
pointerdown on handle
  → stopPropagation() (don't trigger card drag)
  → preventDefault()
  → read card rect relative to canvas → setResizing({idx, dir, sx, sy, sw, sh})

pointermove on canvas
  → compute new w/h from mouse position relative to anchor (sx, sy)
  → minimum 100px per side
  → setPositions(update card at idx)
```

### Text Drag (label/title)
```
pointerdown on text element
  → stopPropagation()
  → setTextDragging({ox, oy, moved: false})

pointermove on canvas
  → applyTextDrag(drag, pos, setDrag, setPos, mx, my)
     → if > 3px movement: mark moved=true
     → if moved: update pos

pointerup on text element
  → clear textDragging (no save — manual Save button only)
```

---

## Config Sync Pattern

```ts
// Load from external config on mount / config change
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

---

## File Organization

通用结构，括号内为 ZoMble 示例路径：

```
src/
├── components/
│   └── canvas/
│       ├── CanvasContainer.tsx    — 编辑模式画布容器 (sections/EditorSelection.tsx)
│       ├── CanvasCard.tsx         — 可拖拽/缩放卡片 (home/CanvasCard.tsx)
│       ├── PreviewCard.tsx        — 逐卡滚动动画 (home/PreviewCard.tsx)
│       ├── EditorCard.tsx         — 卡片内容渲染 (home/EditorCard.tsx)
│       ├── DraggableText.tsx      — 可复用文字拖拽 (内联在 CanvasContainer)
│       ├── AlignmentGuides.tsx    — 吸附对齐线 (内联在 CanvasContainer)
│       └── SaveButton.tsx         — 批量保存按钮 (内联在 CanvasContainer)
├── hooks/
│   └── useCanvasState.ts         — 画布状态管理 (内联在 CanvasContainer)
└── lib/
    └── canvas-utils.ts           — calcSnap, applyTextDrag, 坐标辅助 (lib/curatedCanvas.ts)
```
