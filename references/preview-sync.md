# Canvas Editor — Preview Sync Reference

编辑模式与预览模式的坐标同步原理。

---

## The Fundamental Problem

- **编辑模式**使用绝对像素坐标 (`left: 512`, `top: 80`)
- **预览模式**必须保持相同布局，但需要响应式缩放（视口宽度不同）

## Why Simple Approaches Fail

### ❌ 像素坐标直接沿用
```tsx
style={{ position: "absolute", left: "512px", top: "80px" }}
```
容器宽度与编辑时不同 → 布局偏移。

### ❌ CSS Grid 近似
```tsx
grid-column: span 2
```
丢失了用户精细调整的 X/Y 位置。

### ❌ 编辑和预览使用不同坐标系
编辑用 `canvasBottom`，预览用 `maxBottom` → 分母差 1px 就导致全部元素偏移。

---

## The Correct Solution

### Step 1: 统一坐标基准（ONCE）

编辑模式和预览模式使用**完全相同的公式**。任何差异——哪怕 1px——都会级联到所有元素。

```ts
const previewBounds = useMemo(() => {
  const maxRight = Math.max(
    ...cards.map(p => p.left + p.w),
    labelPos.left + 300,    // label 宽度估算
    titlePos.left + 200,    // title 宽度估算
    100                     // 最小 fallback
  )
  const maxBottom = Math.max(
    ...cards.map(p => p.top + p.h),
    titlePos.top + 60,      // title 高度估算
    labelPos.top + 30,      // label 高度估算
    100                     // 最小 fallback
  )
  return { maxRight, maxBottom }
}, [cards, labelPos, titlePos])

const canvasBottom = Math.max(previewBounds.maxBottom, 700)  // 700px 下限
```

`canvasBottom` 的 700px floor 保证画布不会因内容太少而缩得太小。

### Step 2: 预览容器——同宽高比

```
┌──────────────────────────────────────────┐
│  Wrapper: max-width = maxRight           │  ← 宽屏时精确匹配 maxRight px
│  ┌────────────────────────────────────┐  │
│  │  Aspect box: padding-top 按比例    │  │  ← 高度 = 宽度 × (canvasBottom/maxRight)
│  │  ┌──────────────────────────────┐ │  │
│  │  │  Content: position absolute   │ │  │  ← 所有元素用百分比定位
│  │  └──────────────────────────────┘ │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

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

**为什么 `max-width: maxRight`？** 当屏幕宽度 ≥ maxRight 时，容器宽度恰好 = maxRight → 百分比数值直接还原为像素值。窄屏时等比例缩小。

**为什么 `padding-top`？** CSS `padding-top: X%` 是相对于容器宽度的。容器宽度为 maxRight 时，padding-top = (canvasBottom / maxRight) × maxRight = canvasBottom → 精确还原画布高度。

### Step 3: 元素百分比定位

所有元素使用同一对分母：

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
  left: `${(labelPos.left / maxRight) * 100}%`,
  top:  `${(labelPos.top / canvasBottom) * 100}%`,
}}

// Title
style={{
  left: `${(titlePos.left / maxRight) * 100}%`,
  top:  `${(titlePos.top / canvasBottom) * 100}%`,
}}
```

---

## Why This Works — 数学证明

当视口宽度 W 恰好等于 `maxRight` 时：

```
容器高度 = W × (canvasBottom / maxRight)
         = maxRight × (canvasBottom / maxRight)
         = canvasBottom ✓

卡片 left = W × (pos.left / maxRight)
           = maxRight × (pos.left / maxRight)
           = pos.left ✓

卡片 top = canvasBottom × (pos.top / canvasBottom)
          = pos.top ✓
```

**当 W = maxRight 时，像素级精确匹配。** 在其它宽度下等比例缩放。

---

## Common Pitfalls

### 1. 预览宽高比用了 maxBottom 而非 canvasBottom

```tsx
// ❌ 错误
padding-top: ${(maxBottom / maxRight) * 100}%

// ✅ 正确
padding-top: ${(canvasBottom / maxRight) * 100}%
```

当 `maxBottom < 700`（内容较少）时，预览容器比编辑画布矮 → 所有垂直位置偏下。

### 2. 编辑和预览使用不同公式

```tsx
// ❌ 错误：两套不同计算
editMinHeight   = max(titleTop + 60, labelTop + 30, ..., 700)
previewRatio    = max(titleTop + 50, labelTop + 20, ..., 100) / maxRight

// ✅ 正确：单一数据源
bounds = computeBounds(cards, label, title)   // 一次计算
canvasBottom = max(bounds.maxBottom, 700)
previewRatio = canvasBottom / bounds.maxRight
```

### 3. 垂直分母未统一

```tsx
// ❌ 错误：标题用 maxBottom 但卡片用 canvasBottom
title.top = titlePos.top / maxBottom * 100
card.top  = pos.top / canvasBottom * 100

// → 标题和卡片位置不同步

// ✅ 正确：全部用 canvasBottom
title.top = titlePos.top / canvasBottom * 100
card.top  = pos.top / canvasBottom * 100
```

### 4. 编辑和预览的缺省位置不一致

如果编辑模式初始化位置用了自己写的公式，预览模式用了另一套 → 用户打开页面看到的就是两样。**defaultPositions 必须共享给两个模式**。

---

## Scroll Animation — 逐卡独立入场

预览模式中每张卡片独立使用 IntersectionObserver 触发入场动画。

**为什么不能包一个大的 RevealOnScroll？** 因为卡片的容器（百分比绝对定位）本身 height = 0 → IntersectionObserver 永远看不到 → 动画不触发。

**解决方案：** 每张卡片一个独立 observer，且卡片 div 有明确尺寸（从百分比计算得出）。

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

CSS：
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
