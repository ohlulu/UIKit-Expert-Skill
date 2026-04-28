# Shadow and Clipping

## Core Rule

A `CALayer` shadow is drawn **outside** the layer's bounds. If **any ancestor** in the view hierarchy sets `clipsToBounds = true` (or `layer.masksToBounds = true`), the shadow is silently clipped — no warning, no error.

## UICollectionViewCell Default

`UICollectionViewCell.contentView.clipsToBounds` defaults to `true`. This means any shadow applied to a subview inside `contentView` is invisible unless you explicitly opt out:

```swift
override init(frame: CGRect) {
    super.init(frame: frame)
    clipsToBounds = false
    contentView.clipsToBounds = false
    // …
}
```

`UITableViewCell.contentView` has the same default behavior.

## Ancestor Chain Check

When a shadow doesn't appear, check the **entire** ancestor chain from the shadowed view up to the scroll view:

```
UICollectionView         clipsToBounds = true (expected — scroll clipping)
  └─ Cell                clipsToBounds = ?
       └─ contentView    clipsToBounds = ?   ← often the culprit
            └─ CardView  clipsToBounds = false, shadow configured
```

The collection/table view itself clips (for scrolling), but its cells and content views must not clip if their children need visible shadows.

## Shadow Needs Space

A shadow with `shadowOffset = (0, 4)` and `shadowRadius = 12` extends roughly 16pt below the layer and 12pt on each side. The shadowed view must have enough margin from its clipping ancestor's bounds for the shadow to be visible.

Two approaches:
1. **Inset the card** from `contentView` edges (e.g., 16pt), so the shadow falls within `contentView` bounds — no need to disable clipping.
2. **Disable clipping** on the cell and `contentView`, let the shadow extend beyond bounds.

Approach 2 is simpler when section `contentInsets` already provide the margin.

## Deferred Visual Initialization

When a view separates structural setup (`init`) from visual theming (`applyTheme`), the view starts with no background color. If `applyTheme` is not called before the first render, the view is transparent.

Always either:
- Set a default `backgroundColor` in `init`, or
- Call `applyTheme()` at the end of `init`

For reusable cells, also call `applyTheme()` at the end of `configure(with:)` to cover the reconfiguration path.

## Shadow + Corner Radius: Parent/Child Split

When a view needs **both** a shadow and rounded corners, they cannot coexist on the same layer. `clipsToBounds = true` (required for corner radius) clips the shadow. `clipsToBounds = false` (required for shadow) disables corner radius clipping.

**Solution: two layers.**

```swift
// Parent: owns the shadow, does NOT clip
let container = UIView()
container.clipsToBounds = false
container.layer.shadowColor = UIColor.black.cgColor
container.layer.shadowOpacity = 0.2
container.layer.shadowOffset = CGSize(width: 0, height: 4)
container.layer.shadowRadius = 8

// Child: owns the corner radius, DOES clip
let content = UIImageView(image: icon)
content.layer.cornerRadius = 16
content.clipsToBounds = true
container.addSubview(content)
// Pin content edges to container edges
```

This pattern applies to: app icon with shadow, card views, avatar images, any rounded element that also needs elevation.

## CALayer Frame Ownership — Each View Manages Its Own Layers

When adding `CAGradientLayer`, `CAShapeLayer`, or other sublayers to a view, **the owning view must update the layer's frame in its own `layoutSubviews`** — not a parent or grandparent.

### The bug

A cell's `layoutSubviews` reads a deeply nested subview's bounds and sets a sublayer frame:

```swift
// ❌ Cell's layoutSubviews — clipView is 3+ levels deep
override func layoutSubviews() {
    super.layoutSubviews()
    gradientLayer.frame = clipView.bounds  // zero on first display!
}
```

`super.layoutSubviews()` resolves constraints for the cell's direct subviews, but **deeply nested views may not have their bounds resolved yet**. The layer frame is set to `.zero`, and the gradient is invisible. Navigating away and back "fixes" it because the cell is reused with pre-existing bounds.

This is especially deceptive: the second display always works, making the bug appear intermittent or timing-related.

### The fix

Create a UIView subclass that owns the layer and updates it in its own `layoutSubviews`:

```swift
// ✅ Each view manages its own layers
private final class GradientBackgroundView: UIView {
    let gradientLayer = CAGradientLayer()

    override init(frame: CGRect) {
        super.init(frame: frame)
        layer.addSublayer(gradientLayer)
        // Set constant properties here (colors, startPoint, etc.)
    }

    override func layoutSubviews() {
        super.layoutSubviews()
        gradientLayer.frame = bounds  // always correct — this view's own bounds
    }
}
```

When Auto Layout sets this view's frame, its `layoutSubviews` fires and the layer frame is immediately correct — regardless of how deep it sits in the hierarchy.

### Rule of thumb

- **Constant layer properties** (colors, startPoint, endPoint, cornerRadius): set in `init` or a `configure` method.
- **Frame-dependent properties** (frame, path, shadowPath): set in the **owning view's** `layoutSubviews`.
- **Never reach down** from a parent's `layoutSubviews` to read a child's bounds for layer sizing.

## Sheet Presentation: Double-Background Color Shift (iOS 26+)

When a view controller is presented as a sheet (`.automatic` / `.pageSheet`), iOS 26 inserts a `UIDropShadowView` with rounded corners and material compositing between the presenting and presented view controllers.

If a subview (e.g., `UICollectionView`) has the **same opaque backgroundColor** as the root view, two opaque layers of the same color are composited through the sheet's material pipeline. GPU floating-point precision and color space conversion (sRGB ↔ Display P3) produce a **subtle but visible color seam** at the boundary where the subview starts.

```
❌ Two opaque layers — visible seam
┌─ Sheet material ──────────────┐
│  ┌─ VC.view (theme.bg) ──────┐│
│  │  Search bar area           ││  ← one layer
│  │  ┌─ CollectionView ─────┐ ││
│  │  │  (theme.bg, opaque)  │ ││  ← two layers composited
│  │  └──────────────────────┘ ││
│  └────────────────────────────┘│
└────────────────────────────────┘

✅ Single background layer — no seam
│  │  ┌─ CollectionView ─────┐ ││
│  │  │  (.clear)             │ ││  ← root view shows through
│  │  └──────────────────────┘ ││
```

### Fix

Set scroll view / collection view / table view `backgroundColor = .clear` and let the VC's root view be the **sole background provider**:

```swift
private func applyTheme() {
    ThemeApplier.apply(to: self)          // sets view.backgroundColor
    collectionView.backgroundColor = .clear // ← single background source
}
```

### When this bites

- Sheet-presented screens with a search bar above a scroll view (the seam appears at the search bar / list boundary)
- Any modal with subviews that duplicate the root view's background color
- More visible on P3 displays and with non-neutral theme colors (warm grays, tinted backgrounds)

### Does NOT affect

- Full-screen presentations (no sheet material layer)
- Views where the scroll view intentionally has a different background color
- Pre-iOS 26 (no `UIDropShadowView` in the hierarchy)

## Diagnostic Checklist

| Symptom | Likely cause |
|---------|-------------|
| Shadow invisible, card background visible | An ancestor has `clipsToBounds = true` |
| Both shadow and background invisible | `backgroundColor` never set (deferred theme not called) |
| Shadow visible on sides but clipped at bottom | Insufficient bottom margin for shadow extent |
| Shadow flickers during scroll | Shadow is recalculated per frame — set `shadowPath` for performance |
| Gradient/shape layer invisible on first display, appears after navigate-back | Layer frame set from a parent's `layoutSubviews` reading a deep child's bounds (still zero). Move layer to a subclass that manages it in its own `layoutSubviews` |
| Subtle color seam between search bar area and scroll content (iOS 26 sheet) | Scroll view has same opaque backgroundColor as root view — set scroll view to `.clear` |
