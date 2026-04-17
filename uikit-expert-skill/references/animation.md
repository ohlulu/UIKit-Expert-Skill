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

## Summary

| Scenario | Duration | Curve |
|----------|----------|-------|
| Alpha fade (show/hide) | 0.1s | `.curveLinear` |
| Transitions (push, present, layout) | Case-dependent | `.curveLinear` |
| Expand height | 0.4s | Spring (damping 0.85) |
| Expand content (stagger) | 0.3s each, 0.04s gap | Spring (damping 0.8) |
| Collapse content | 0.15s | `.curveEaseIn` |
| Collapse height | 0.3s | Spring (damping 0.9) |
| Spring | Only when explicitly requested | Spring |
