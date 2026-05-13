# UIButton.Configuration API

## Core Principle

`UIButton.Configuration` and legacy UIButton API are **mutually exclusive rendering paths**. Once you set `button.configuration`, the legacy properties (`titleLabel?.font`, `setTitle(_:for:)`, `setTitleColor(_:for:)`, `imageView?.image`) are ignored for layout and rendering. Pick one API surface and stay on it.

## The Silent Override Trap

```swift
// ❌ Font is silently ignored — Configuration overrides legacy properties
button.setTitle("Save", for: .normal)
button.titleLabel?.font = .systemFont(ofSize: 13, weight: .semibold)

var config = UIButton.Configuration.plain()
config.contentInsets = NSDirectionalEdgeInsets(top: 4, leading: 6, bottom: 4, trailing: 6)
button.configuration = config  // ← everything above is now dead code for rendering
```

Setting `configuration` switches the button into Configuration mode. The title renders with the Configuration's default font (~17pt body), not 13pt. The button's intrinsic height grows, but nothing crashes and nothing warns — it just looks wrong.

## Correct Configuration Usage

Set **all** appearance through the Configuration object:

```swift
// ✅ All properties on the Configuration — single source of truth
var config = UIButton.Configuration.plain()
config.title = text
config.baseForegroundColor = theme.textPrimary
config.contentInsets = NSDirectionalEdgeInsets(top: 4, leading: 6, bottom: 4, trailing: 6)
config.titleTextAttributesTransformer = UIConfigurationTextAttributesTransformer { incoming in
    var attrs = incoming
    attrs.font = .systemFont(ofSize: 13, weight: .semibold)
    return attrs
}
button.configuration = config
```

### Property Mapping

| Legacy API | Configuration API |
|-----------|-------------------|
| `setTitle(_:for:)` | `config.title` / `config.attributedTitle` |
| `titleLabel?.font` | `config.titleTextAttributesTransformer` |
| `setTitleColor(_:for:)` | `config.baseForegroundColor` |
| `setImage(_:for:)` | `config.image` |
| `imageView?.tintColor` | `config.baseForegroundColor` (shared) |
| `contentEdgeInsets` | `config.contentInsets` |
| `titleEdgeInsets` / `imageEdgeInsets` | `config.imagePadding` / `config.titlePadding` |

## Detecting the Problem

If a UIButton using Configuration renders at unexpected size:

1. Search for `titleLabel?.font`, `setTitle(`, `setTitleColor(` near any `configuration =` assignment.
2. Any legacy setter **before or after** `configuration =` is suspect.
3. Measure: `button.systemLayoutSizeFitting(.layoutFittingCompressedSize).height` — if it's ~34pt when you expect ~24pt, the font isn't what you think.

## Dynamic Font Changes

To change font per state (e.g., preferred vs non-preferred):

```swift
func configure(text: String, isPreferred: Bool, theme: AppTheme) {
    let font: UIFont = isPreferred
        ? .systemFont(ofSize: 13, weight: .semibold)
        : .systemFont(ofSize: 12, weight: .regular)

    var config = UIButton.Configuration.plain()
    config.title = text
    config.baseForegroundColor = isPreferred ? theme.textPrimary : theme.textSecondary
    config.titleTextAttributesTransformer = UIConfigurationTextAttributesTransformer { incoming in
        var attrs = incoming
        attrs.font = font
        return attrs
    }
    configuration = config
}
```

Rebuild the entire Configuration each time. Don't try to patch individual properties on an existing configuration — it's cheap and avoids partial-update bugs.

## `configurationUpdateHandler` for State-Driven Updates

For buttons that change appearance based on `isHighlighted`, `isSelected`, etc.:

```swift
button.configurationUpdateHandler = { button in
    var config = button.configuration ?? .plain()
    config.baseForegroundColor = button.isHighlighted ? .gray : .label
    button.configuration = config
}
```

This replaces the old `setTitleColor(.gray, for: .highlighted)` pattern.
