# Navigation Bar Appearance

## iOS 26 Breaking Change — backgroundColor Covers Large Title

iOS 26 (Tahoe) restructured `UINavigationBar`'s internal view hierarchy for Liquid Glass. The **background layer is now rendered in front of the large title label**, not behind it.

### Symptom

- Large title visible initially, disappears after scroll and never returns
- Inline (small) title unaffected — it lives in a different layer
- View Hierarchy debugger shows the title label exists behind the background view

### Root Cause

Setting `UINavigationBarAppearance.backgroundColor` (with or without `configureWithOpaqueBackground()`) places an opaque layer on top of the large title. This is a rendering-order change in iOS 26's Liquid Glass architecture. Tracked as FB20986869.

### Fix

```swift
let appearance = UINavigationBarAppearance()

if #available(iOS 26, *) {
    appearance.configureWithTransparentBackground()
    // Do NOT set appearance.backgroundColor — let view.backgroundColor show through
} else {
    appearance.configureWithOpaqueBackground()
    appearance.backgroundColor = theme.background
}

// These still work on all versions
appearance.shadowColor = .clear
appearance.titleTextAttributes = [.foregroundColor: theme.textPrimary]
appearance.largeTitleTextAttributes = [.foregroundColor: theme.textPrimary]

nav.navigationBar.standardAppearance = appearance
nav.navigationBar.scrollEdgeAppearance = appearance
nav.navigationBar.compactAppearance = appearance
```

The view controller's `view.backgroundColor` is already set to the theme color, so the transparent bar shows the correct background.

### Alternatives

| Approach | Effect |
|----------|--------|
| `configureWithTransparentBackground()` | Fully transparent bar; view's background shows through cleanly |
| `configureWithDefaultBackground()` | System Liquid Glass (frosted) effect; adapts to content behind |
| SwiftUI `.toolbarBackground(_:for:)` | SwiftUI-native; works on iOS 26; not available in pure UIKit |

### Key Constraint

Apple DTS explicitly recommends moving away from `UINavigationBarAppearance.backgroundColor` on iOS 26 (Developer Forums thread/807331). The API is not deprecated, but its behavior is broken for large titles.

## Three Appearance Slots

| Property | When active |
|----------|-------------|
| `scrollEdgeAppearance` | Content at top (large title visible, unscrolled) |
| `standardAppearance` | Content scrolled (title collapsed to inline) |
| `compactAppearance` | Compact height (landscape on non-Pro phones) |

Setting all three to the same object gives a consistent bar in all states. When they differ, the system animates between them during scroll transitions.

## Common Pitfalls

| Pitfall | Why it's wrong |
|---------|----------------|
| Setting `backgroundColor` on iOS 26 | Large title hidden behind background layer |
| Only setting `standardAppearance` | `scrollEdgeAppearance` defaults to translucent on iOS 15+, causing a flash when scrolling to top |
| Creating new appearance objects in `viewWillAppear` without guarding | Replaces appearance mid-transition on push/pop, can cause flicker |
| Using `UINavigationBar.appearance()` globally + per-instance overrides | Global proxy wins on first layout, then per-instance takes over — ordering is unpredictable |
