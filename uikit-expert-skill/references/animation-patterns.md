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

## Summary

| Scenario | Duration | Curve |
|----------|----------|-------|
| Alpha fade (show/hide) | 0.1s | `.curveLinear` |
| Transitions (push, present, layout) | Case-dependent | `.curveLinear` |
| Spring | Only when explicitly requested | Spring |
