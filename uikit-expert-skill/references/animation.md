# Animation Patterns

## Core Principle

Keep animations minimal and snappy. Most UI state changes need only a short linear fade. Avoid spring animations unless explicitly requested.

## Default: Linear Fade 0.1s

The standard animation for showing, hiding, or toggling UI elements is a **0.1s linear alpha change**. This covers the majority of cases.

```swift
UIView.animate(withDuration: 0.1, delay: 0, options: .curveLinear) {
    self.overlayView.alpha = 1
}
```

Use this for:
- Showing / hiding views
- Toggling element visibility
- Fading in loading indicators, toasts, or overlays

## Transitions: Duration Varies by Case

Transitions — view controller push/present/dismiss, layout constraint animations, or any multi-element choreography — are **not bound to 0.1s**. Choose duration based on the specific case and the amount of visual change involved.

```swift
// Layout constraint animation — duration depends on distance/complexity
self.topConstraint.constant = newValue
UIView.animate(withDuration: 0.25, delay: 0, options: .curveLinear) {
    self.view.layoutIfNeeded()
}
```

## Curve: Always Linear

Use `.curveLinear` by default. Do not use spring animations (`UIView.animate(withSpringDuration:)`, `usingSpringWithDamping`) unless the user explicitly requests spring behavior.

```swift
// ✅ Default
UIView.animate(withDuration: 0.1, delay: 0, options: .curveLinear) { ... }

// ❌ Don't add spring unless requested
UIView.animate(withDuration: 0.3, delay: 0, usingSpringWithDamping: 0.8, ...) { ... }
```

## Expand / Collapse Choreography

For content that expands to reveal more items (unit values, details, accordion sections). Two-phase structure makes the motion feel intentional, not mechanical.

### Expand: Spring Height + Staggered Reveal

Two animations fire simultaneously:

1. **Height** — spring grows the container.
2. **Content** — each item staggers in with fade + scale.

```swift
func animateExpand(row: ExpandableRow) {
    // Prep: unhide items, start invisible + scaled down
    row.prepareExpandAnimation()

    // 1. Spring height
    UIView.animate(
        withDuration: 0.4,
        delay: 0,
        usingSpringWithDamping: 0.85,
        initialSpringVelocity: 0,
        options: .curveEaseOut
    ) {
        self.layoutIfNeeded()
        collectionView.collectionViewLayout.invalidateLayout()
        collectionView.layoutIfNeeded()
    }

    // 2. Staggered values reveal
    row.animateValuesIn()
}
```

Prepare phase — set state before animation starts:
```swift
func prepareExpandAnimation() {
    for view in collapsibleViews {
        view.isHidden = false
        view.alpha = 0
        view.transform = CGAffineTransform(scaleX: 0.85, y: 0.85)
    }
    flowView.setNeedsLayout()
}
```

Stagger phase — each item fades in with slight delay:
```swift
func animateValuesIn() {
    let staggerDelay: TimeInterval = 0.04
    for (i, view) in collapsibleViews.enumerated() {
        UIView.animate(
            withDuration: 0.3,
            delay: staggerDelay * Double(i),
            usingSpringWithDamping: 0.8,
            initialSpringVelocity: 0,
            options: .curveEaseOut
        ) {
            view.alpha = 1
            view.transform = .identity
        }
    }
}
```

### Collapse: Fade Out → Then Shrink

Two-phase sequential — content disappears first, then container closes:

```swift
func animateCollapse(row: ExpandableRow) {
    // Phase 1: fast fade out + scale down
    UIView.animate(withDuration: 0.15, delay: 0, options: .curveEaseIn) {
        row.setCollapsedVisuals()  // alpha = 0, scale = 0.85
    } completion: { _ in
        // Phase 2: spring height shrink
        row.applyCollapsedLayout()  // isHidden = true → triggers re-layout
        UIView.animate(
            withDuration: 0.3,
            delay: 0,
            usingSpringWithDamping: 0.9,
            initialSpringVelocity: 0,
            options: .curveEaseOut
        ) {
            self.layoutIfNeeded()
            collectionView.collectionViewLayout.invalidateLayout()
            collectionView.layoutIfNeeded()
        }
    }
}
```

### Why This Structure Works

| Aspect | Expand | Collapse |
|--------|--------|----------|
| Phases | Simultaneous (height + stagger) | Sequential (fade → height) |
| Height | Spring 0.4s, damping 0.85 | Spring 0.3s, damping 0.9 |
| Content | Stagger 0.04s gap, scale 0.85→1.0 | All simultaneous, 0.15s |
| Feel | "Items unfold one by one" | "Tuck away, then close" |

Expand is slower and playful (discovery). Collapse is faster and direct (dismissal). Asymmetric timing mirrors iOS system patterns.

### Prerequisites

- Use `isHidden` to toggle subview visibility, not add/remove. Container views like `UIStackView` and `FlowLayoutView` should skip hidden subviews in layout (`where !subview.isHidden`).
- Store references to collapsible views in an array at configure time.
- Never call `removeAllItems()` + re-add inside a toggle — that destroys views and can't be animated.
- Use `setNeedsLayout()` after changing `isHidden` so the container recalculates.

## Text Expand / Collapse (UILabel Height Animation)

Animating a UILabel between a truncated state (e.g. 3 lines) and full content requires a different strategy from the subview-based expand/collapse above. `numberOfLines` changes cause **instant text reflow** — the content snaps to its new state before the frame animation starts, producing a visual disconnect.

### The Problem with `numberOfLines`

```swift
// ❌ Not smooth — text reflows immediately, frame animates separately
label.numberOfLines = 0  // text snaps to full content
UIView.animate(withDuration: 0.35, ...) {
    self.layoutIfNeeded()  // frame grows, but text already changed
}
```

| Phase | What happens |
|-------|-------------|
| Frame 0 | `numberOfLines = 0` — label renders ALL text instantly |
| Frame 0 | Old frame still at 3-line height — extra text overflows or is clipped |
| Frame 1…N | Frame animates to new height — but text was already there |

Expand looks like a "sliding reveal" (acceptable). Collapse looks broken — text snaps to 3 lines while the frame is still tall, leaving empty space that then shrinks.

### Solution: Constraint-Driven Height with Clip Container

Keep `numberOfLines = 0` at all times. Control visible height with a constraint on a clipping container. The text never reflows — only the visible area changes.

```
┌─ NoteRowView ─────────────────────────┐
│  [bullet]  ┌─ labelContainer ───────┐ │
│            │  clipsToBounds = true   │ │
│            │  ┌─ contentLabel ────┐  │ │
│            │  │ numberOfLines = 0 │  │ │
│            │  │ (full text)       │  │ │
│            │  └───────────────────┘  │ │
│            │  ← height constraint → │ │
│            └────────────────────────┘ │
└───────────────────────────────────────┘
```

Setup:

```swift
// Label always renders full text
contentLabel.numberOfLines = 0
contentLabel.lineBreakMode = .byTruncatingTail

// Container clips overflow during animation
labelContainer.clipsToBounds = true

// Label pinned to container, bottom at lowered priority
contentLabel.topAnchor.constraint(equalTo: labelContainer.topAnchor),
contentLabel.leadingAnchor.constraint(equalTo: labelContainer.leadingAnchor),
contentLabel.trailingAnchor.constraint(equalTo: labelContainer.trailingAnchor),
{
    let c = contentLabel.bottomAnchor.constraint(equalTo: labelContainer.bottomAnchor)
    c.priority = .defaultHigh  // yields to height constraint when active
    return c
}()
```

Height constraint:

```swift
private var collapsedHeightConstraint: NSLayoutConstraint?

private var collapsedTextHeight: CGFloat {
    let font = contentLabel.font ?? .systemFont(ofSize: 15)
    return ceil(font.lineHeight * 3)  // 3 lines
}

private func ensureCollapsedConstraint() {
    let maxHeight = collapsedTextHeight
    if let existing = collapsedHeightConstraint {
        existing.constant = maxHeight
    } else {
        let c = labelContainer.heightAnchor.constraint(
            lessThanOrEqualToConstant: maxHeight
        )
        c.priority = .required
        collapsedHeightConstraint = c
    }
}
```

### Expand: Deactivate Constraint → Spring Height

```swift
func applyExpanded() {
    collapsedHeightConstraint?.isActive = false
}

// In parent cell:
row.applyExpanded()
UIView.animate(
    withDuration: 0.4,
    delay: 0,
    usingSpringWithDamping: 0.85,
    initialSpringVelocity: 0,
    options: .curveEaseOut
) {
    self.layoutIfNeeded()
    collectionView.collectionViewLayout.invalidateLayout()
    collectionView.layoutIfNeeded()
}
```

Result: frame grows smoothly, text is revealed line by line as the container expands. Feels like a sliding reveal.

### Collapse: Activate Constraint → Spring Height

```swift
func applyCollapsed() {
    ensureCollapsedConstraint()
    collapsedHeightConstraint?.isActive = true
}

// In parent cell:
row.applyCollapsed()
UIView.animate(
    withDuration: 0.3,
    delay: 0,
    usingSpringWithDamping: 0.9,
    initialSpringVelocity: 0,
    options: .curveEaseOut
) {
    self.layoutIfNeeded()
    collectionView.collectionViewLayout.invalidateLayout()
    collectionView.layoutIfNeeded()
}
```

Result: frame shrinks smoothly, text is masked from bottom upward. No text reflow snap.

### Why This Works

| Aspect | `numberOfLines` toggle | Constraint + clip |
|--------|----------------------|-------------------|
| Text reflow | Instant snap — not animatable | Never happens — text always rendered |
| Frame change | Animatable via `layoutIfNeeded` | Same |
| Expand visual | Sliding reveal (ok) | Sliding reveal (same) |
| Collapse visual | Text snaps → empty space → shrink (bad) | Smooth mask from bottom (good) |
| Truncation "..." | Automatic from `numberOfLines` | Not shown (chevron indicates overflow instead) |

### Overflow Detection for Text

Use the same two-pass pattern from [overflow-detection](overflow-detection.md), adapted for text height:

```swift
override func layoutSubviews() {
    super.layoutSubviews()
    guard pendingOverflowCheck, labelContainer.bounds.width > 0 else { return }
    pendingOverflowCheck = false

    let fullHeight = textHeight(for: contentLabel)
    let maxCollapsed = collapsedTextHeight
    if fullHeight > maxCollapsed + 1 {
        DispatchQueue.main.async { [weak self] in
            self?.onOverflowDetected?()
        }
    }
}

private func textHeight(for label: UILabel) -> CGFloat {
    guard let text = label.text, let font = label.font else { return 0 }
    let width = label.bounds.width > 0 ? label.bounds.width : labelContainer.bounds.width
    guard width > 0 else { return 0 }
    return (text as NSString).boundingRect(
        with: CGSize(width: width, height: .greatestFiniteMagnitude),
        options: .usesLineFragmentOrigin,
        attributes: [.font: font],
        context: nil
    ).height
}
```

### Anti-Patterns

#### ❌ `performBatchUpdates(nil)` for Height Animation

```swift
// ❌ Causes visual glitches — the collection view recalculates cell
// sizes in its own animation context which conflicts with the
// constraint-driven height change
row.animateExpand()
collectionView.performBatchUpdates(nil)
```

Use `invalidateLayout()` + `layoutIfNeeded()` inside `UIView.animate` instead. This keeps the constraint change and the collection view layout in the same animation transaction.

#### ❌ Toggling `numberOfLines` for Animated Height Changes

```swift
// ❌ Text reflows instantly — frame and content are out of sync
label.numberOfLines = isExpanded ? 0 : 3
UIView.animate { self.layoutIfNeeded() }
```

Use a height constraint on a clipping container instead (see above).

## Summary

| Scenario | Duration | Curve |
|----------|----------|-------|
| Alpha fade (show/hide) | 0.1s | `.curveLinear` |
| Transitions (push, present, layout) | Case-dependent | `.curveLinear` |
| Expand height (subview-based) | 0.4s | Spring (damping 0.85) |
| Expand content (stagger) | 0.3s each, 0.04s gap | Spring (damping 0.8) |
| Collapse content (subview-based) | 0.15s | `.curveEaseIn` |
| Collapse height (subview-based) | 0.3s | Spring (damping 0.9) |
| Text expand (constraint + clip) | 0.4s | Spring (damping 0.85) |
| Text collapse (constraint + clip) | 0.3s | Spring (damping 0.9) |
| Spring | Only when explicitly requested | Spring |
