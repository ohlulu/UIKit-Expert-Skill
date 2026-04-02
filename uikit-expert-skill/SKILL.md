---
name: uikit-expert-skill
description: Write, review, or improve UIKit code following best practices for view lifecycle, table/collection views, animation, and UIKit testability. Use when building new UIKit features, refactoring existing views, reviewing code quality, or modernizing legacy UIKit code.
---

# UIKit Expert Skill

## Precedence Rule

When a project already has an established convention that conflicts with guidance in this skill, ask the user which to follow. Do not silently override project conventions.

## Topic Router

Consult the reference file for each topic relevant to the current task. Apply all rules from the matched references — do not skip them for convenience.

| Topic | Reference |
|-------|-----------|
| File structure (property ordering, extensions, layout placement for UIViewController, UIView, UITableViewCell, and other UIKit subclasses) | [file-structure](references/file-structure.md) |
| Animation (duration, curve, fade defaults) | [animation](references/animation.md) |
| List composition (heterogeneous cells, row/item controllers, section controllers, diffable, compositional layout, default shapes) | [list-composition](references/list-composition.md) |
| Testing principles (test levels, async spies, assertion strategy, memory leak tracking) | [testing-principles](references/testing-principles.md) |
| UIKit testing (lifecycle simulation, interaction helpers, reuse/visibility tests, screen integration tests) | [testing](references/testing.md) |
