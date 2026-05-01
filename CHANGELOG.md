# Changelog

## [Unreleased]

## [0.3.0] — 2026-05-01

- Added self-sizing reference — `systemLayoutSizeFitting`, `preferredContentSize`, cell auto-dimension.
- Added image-resizing reference — API decision matrix, export gotcha, template icon pattern.
- Added view-wrapping reference — `wrapped(insets:)`, section decoration, decision matrix.
- Added alignment and UIButton.Configuration references.
- Added shadow-and-clipping reference with parent/child split and iOS 26 sheet gotcha.
- Added keyboard-avoidance reference.
- Added overflow detection and expand/collapse animation references.
- Added menu-vs-popover decision reference.
- Added UIButton icon badge reference.
- Added navigation-bar-appearance reference (iOS 26 `backgroundColor` breaking change).
- Added autolayout-spacing reference.
- Added staggered entrance animation pattern for collection view cells.
- Added CALayer frame ownership, transform-safe positioning, and custom layout animation pitfalls.
- Added associated-objects reference — eliminating `objc_setAssociatedObject`.
- Added callback shape rule — avoid closure-of-closures in navigation wiring.
- Added UIViewRepresentable bridge patterns reference.
- Expanded list-composition with dispatch contract, generation checklist, pitfalls, section-level dispatch, stateful sections, and supplementary delegation.
- Added CellRegistration reference — prefer over legacy register/dequeue.
- Added compound-cell-row-animation reference (UISwitch distortion, `performBatchUpdates`, `isHidden` pattern).
- Added project-local release skill.

## [0.2.0] — 2026-04-02

- Relaxed UI property guidance: use direct initialization for one-line setup; reserve `lazy var` closures for multi-statement inline configuration.
- Expanded list composition guidance with a concrete `CellController` identity shape, a fuller `ListViewController` forwarding skeleton, and prefetch forwarding.
- Added visible load-more row guidance with `LoadMoreViewModel`, `LoadMoreCellController`, and a minimal `Paginated<Item>` seam for UIKit pagination consumption.
- Expanded UIKit testing guidance with semantic list helpers, load-more test helpers, pagination integration test flow, and a note on fake refresh controls for deterministic tests.
- Added composer guidance for programmatic screen factories, dependency wiring, navigation closures, and scene root composition.

## [0.1.0] — 2026-03-31

- Added file structure patterns reference for ViewController / UIView organization (extension separation, access level ordering, lazy var UI properties, layout placement).
- Added animation patterns reference (0.1s linear fade default, case-dependent transition durations, no spring unless requested).
- Cleaned up `SKILL.md` for AI-friendliness: operating rules with explicit trigger conditions, single topic router, removed empty placeholders.
- Removed "No Code Formatting or Linting" restriction from `AGENTS.md`.
