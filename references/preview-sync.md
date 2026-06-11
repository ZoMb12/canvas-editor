# Canvas Editor — Preview Sync Reference

## The Fundamental Problem

Edit mode uses **absolute pixel positions** on a canvas with `width: 100%` (of
its parent section). Preview mode must render the **exact same layout** but
responsively — the viewport width during preview may differ from the viewport
width during editing.

## Why Simple Approaches Fail

### Approach 1: Pixel positions in preview
```
❌ style={{ position: "absolute", left: "512px", top: "80px" }}
```
Fails because the container width might not match the edit-time width → layout
looks wrong on different screen sizes.

### Approach 2: CSS Grid approximation
```
❌ Use grid-column-span based on card width/height ratio
```
Fails because grid placement ignores the saved X/Y positions — the user's
careful spatial arrangement is lost.

### Approach 3: Percentage-based with inconsistent coordinate systems
```
❌ Edit uses canvasBottom, Preview uses maxBottom for ratio calculation
```
Fails because a 10px difference in the denominator causes visible position
shifts across all elements.

## The Correct Solution

### Unified Coordinate Basis

Use the **same formula** for both edit min-height and preview aspect ratio.
Any difference — even 1px — cascades into visible misalignment.

```ts
// Step 1: Compute bounds ONCE, use everywhere
const previewBounds = useMemo(() => {
  const maxRight = Math.max(
    ...cards.map(p => p.left + p.w),
    labelPos.left + 300,    // estimated label width
    titlePos.left + 200,    // estimated title width
    100
  )
  const maxBottom = Math.max(
    ...cards.map(p => p.top + p.h),
    titlePos.top + 60,      // estimated title height
    labelPos.top + 30,      // estimated label height
    100
  )
  return { maxRight, maxBottom }
}, [cards, labelPos, titlePos])

// Step 2: Derive canvas height from bounds (with floor)
const canvasBottom = Math.max(previewBounds.maxBottom, 700)
```

The floor (700) prevents the edit canvas from being too small to work with.
The preview respects this floor through `canvasBottom`.

### Preview Container

```html
<!-- Wrapper: caps width to maxRight → pixel-perfect on wide screens -->
<div style="max-width: {maxRight}px; margin: 0 auto;">
  <!-- Aspect ratio box: height = width × (canvasBottom / maxRight) -->
  <div style="width: 100%; padding-top: {(canvasBottom / maxRight) * 100}%; position: relative;">
    <!-- Content: fills the aspect ratio box -->
    <div style="position: absolute; inset: 0;">
      <!-- All elements positioned with percentages -->
    </div>
  </div>
</div>
```

### Element Positioning

All elements use the same denominator pair:

```
Horizontal: percentage = (pixel_value / maxRight) × 100
Vertical:   percentage = (pixel_value / canvasBottom) × 100
```

```tsx
// Card
style={{
  position: "absolute",
  left:   `${(pos.left  / maxRight)     * 100}%`,
  top:    `${(pos.top   / canvasBottom)  * 100}%`,
  width:  `${(pos.w    / maxRight)      * 100}%`,
  height: `${(pos.h    / canvasBottom)   * 100}%`,
}}

// Label
style={{
  position: "absolute",
  left: `${(labelPos.left / maxRight) * 100}%`,
  top:  `${(labelPos.top / canvasBottom) * 100}%`,
}}

// Title
style={{
  position: "absolute",
  left: `${(titlePos.left / maxRight) * 100}%`,
  top:  `${(titlePos.top / canvasBottom) * 100}%`,
}}
```

### Why This Works

At a viewport where the container width W equals `maxRight`:
- Container height = W × (canvasBottom / maxRight) = maxRight × (canvasBottom / maxRight) = canvasBottom
- Card left = W × (pos.left / maxRight) = maxRight × (pos.left / maxRight) = pos.left ✓
- Card top = canvasBottom × (pos.top / canvasBottom) = pos.top ✓

**The layout is pixel-perfect at W = maxRight, and scales proportionally at
other widths.**

## Common Pitfall

### Using maxBottom instead of canvasBottom for the ratio

```tsx
// ❌ WRONG: preview ratio uses raw maxBottom
style={{ paddingTop: `${(maxBottom / maxRight) * 100}%` }}

// ✅ CORRECT: use canvasBottom (includes 700px floor)
style={{ paddingTop: `${(canvasBottom / maxRight) * 100}%` }}
```

When maxBottom < 700 (small layout), the preview would be shorter than the
edit canvas → all vertical positions shift → layout doesn't match.

### Using different formulas for edit vs preview

```tsx
// ❌ WRONG: two different calculations
canvasBottom = max(titlePos.top + 60, labelPos.top + 30, ...cardBottoms, 700)
previewRatio = max(titlePos.top + 50, labelPos.top + 20, ...cardBottoms, 100) / maxRight

// ✅ CORRECT: single source of truth
bounds = computeBounds(cards, label, title)  // once
canvasBottom = max(bounds.maxBottom, 700)
previewRatio = canvasBottom / bounds.maxRight
```

## Scroll Animation Setup

Each card should animate independently on scroll:

```tsx
function PreviewCard({ pos, maxRight, canvasBottom, children }) {
  const ref = useRef(null)

  useEffect(() => {
    const el = ref.current
    if (!el) return
    const obs = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          el.classList.add("visible")
          obs.disconnect()  // fire once
        }
      },
      { threshold: 0.15 }
    )
    obs.observe(el)
    return () => obs.disconnect()
  }, [])

  return (
    <div
      ref={ref}
      className="reveal"
      style={{
        position: "absolute",
        left:   `${(pos.left / maxRight) * 100}%`,
        top:    `${(pos.top  / canvasBottom) * 100}%`,
        width:  `${(pos.w   / maxRight) * 100}%`,
        height: `${(pos.h   / canvasBottom) * 100}%`,
      }}
    >
      {children}
    </div>
  )
}
```

CSS:
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

**Important:** Do NOT wrap the entire preview in a single RevealOnScroll
wrapper. If individual cards use absolute positioning, their wrapper divs
collapse to 0 height, and the IntersectionObserver never triggers → nothing
appears. Each card needs its own observer with a wrapper that has explicit
dimensions (from percentage calculations).
