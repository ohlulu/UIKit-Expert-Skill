---
name: uikit-expert-skill
description: Write, review, or improve UIKit code following best practices for view lifecycle, Auto Layout, table/collection views, navigation, animation, and modern UIKit patterns. Use when building new UIKit features, refactoring existing views, reviewing code quality, or modernizing legacy UIKit code.
---

# UIKit Expert Skill

## Operating Rules

- When writing or reviewing a **UIViewController or UIView** → read [file-structure.md](references/file-structure.md) first.
- When writing or reviewing **any animation** (`UIView.animate`, transitions, alpha changes) → read [animation.md](references/animation.md) first.
- When writing or reviewing a **complex table/collection list** (multiple cell types, diffable data source, prefetching, compositional layout, row/item controllers, or section controllers) → read [list-composition.md](references/list-composition.md) and [list-composition-shapes.md](references/list-composition-shapes.md) first. If the project has no stronger established local pattern, generate code using the defaults in `list-composition-shapes.md`.
- Apply all rules from the matched references. Do not skip them for convenience.

## Topic Router

Consult the reference file for each topic relevant to the current task:

| Topic | Reference |
|-------|-----------|
| File structure (VC/View organization, extensions, layout placement) | [file-structure](references/file-structure.md) |
| Animation (duration, curve, fade defaults) | [animation](references/animation.md) |
| List composition (heterogeneous cells, row/item controllers, section controllers, diffable, compositional layout) | [list-composition](references/list-composition.md) |
| List composition shapes (default wrappers, containers, row/item/section ownership) | [list-composition-shapes](references/list-composition-shapes.md) |
