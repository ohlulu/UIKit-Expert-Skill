---
name: uikit-expert-skill
description: >-
  UIKit best practices for view lifecycle, Auto Layout, table/collection views,
  navigation, animation, and testability. Load whenever working in a UIKit
  codebase — building features, fixing bugs, debugging layout issues, implementing
  designs or mockups, adjusting animations, resolving visual glitches, writing
  tests, or doing SDD/Ralph tasks that produce UIKit code. If the project uses
  UIKit and you will read or edit .swift view/controller files, load this skill.
---

# UIKit Expert Skill

## Precedence Rule

When a project already has a clear, established local convention, follow it by default.

Ask the user only when local conventions are ambiguous, conflicting, or likely harmful.
Do not silently replace a strong project convention with this skill's defaults.

## Topic Router

Consult the reference file for each topic relevant to the current task. Apply all rules from the matched references — do not skip them for convenience.

| Topic | Reference |
|-------|-----------|
| File structure (property ordering, extensions, layout placement for UIViewController, UIView, UITableViewCell, and other UIKit subclasses) | [file-structure](references/file-structure.md) |
| Animation (duration, curve, fade defaults, expand/collapse choreography, stagger reveal, custom layout view height animation pitfalls) | [animation](references/animation.md) |
| List composition (heterogeneous cells, row/item controllers, section controllers, load-more controller, pagination seam, diffable, compositional layout, default shapes) | [list-composition](references/list-composition.md) |
| Screen composition / composer (programmatic controller instantiation, dependency wiring, navigation closures, scene root composition) | [composer](references/composer.md) |
| Image resizing (UIGraphicsImageRenderer, preparingThumbnail, CGImageSource downsampling, template icon sizing, display vs encode) | [image-resizing](references/image-resizing.md) |
| Self-sizing (systemLayoutSizeFitting, preferredContentSize, self-sizing cells, complete constraint chains) | [self-sizing](references/self-sizing.md) |
| Vertical alignment (centerY vs baseline, mixed-size text, manual-frame containers, measuring real button height) | [alignment](references/alignment.md) |
| UIButton.Configuration (Configuration vs legacy API, titleTextAttributesTransformer, silent override trap) | [uibutton-configuration](references/uibutton-configuration.md) |
| UIButton icon badge (small circular icon overlay, .custom type, icon as subview not setImage, highlight via touchDown/Up) | [uibutton-icon-badge](references/uibutton-icon-badge.md) |
| View wrapping (wrapped(insets:), container padding, section decoration helpers, when to use vs manual wrapper) | [view-wrapping](references/view-wrapping.md) |
| Content overflow detection (two-pass layout, actual vs estimated measurement, dynamic collapse/expand triggers) | [overflow-detection](references/overflow-detection.md) |
| Shadow and clipping (CALayer shadow visibility, contentView.clipsToBounds default, ancestor chain check, deferred visual initialization) | [shadow-and-clipping](references/shadow-and-clipping.md) |
| Keyboard avoidance (scroll view inset adjustment, keyboardWillChangeFrame, animation sync, inputView handling, first responder scrolling) | [keyboard-avoidance](references/keyboard-avoidance.md) |
| Menu vs popover (UIMenu for flat option selection, UIPopover for custom UI, decision rule, sizing pitfalls, migration guide) | [menu-vs-popover](references/menu-vs-popover.md) |
| Auto Layout spacing and distribution (setCustomSpacing, CSS flex vs AL, spacer view pitfalls, independent top/bottom pinning, flexible gap strategies) | [autolayout-spacing](references/autolayout-spacing.md) |
| Navigation bar appearance (UINavigationBarAppearance slots, iOS 26 backgroundColor breaking change, Liquid Glass compatibility, three-slot consistency) | [navigation-bar-appearance](references/navigation-bar-appearance.md) |
| Testing principles (test levels, async spies, assertion strategy, memory leak tracking) | [testing-principles](references/testing-principles.md) |
| UIKit testing (lifecycle simulation, semantic list helpers, reuse/visibility tests, pagination integration tests, screen integration tests) | [testing](references/testing.md) |
