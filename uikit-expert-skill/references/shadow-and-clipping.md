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

## Diagnostic Checklist

| Symptom | Likely cause |
|---------|-------------|
| Shadow invisible, card background visible | An ancestor has `clipsToBounds = true` |
| Both shadow and background invisible | `backgroundColor` never set (deferred theme not called) |
| Shadow visible on sides but clipped at bottom | Insufficient bottom margin for shadow extent |
| Shadow flickers during scroll | Shadow is recalculated per frame — set `shadowPath` for performance |
