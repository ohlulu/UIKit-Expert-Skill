# Changelog

## [Unreleased]

- Relaxed UI property guidance: use direct initialization for one-line setup; reserve `lazy var` closures for multi-statement inline configuration.
- Expanded list composition guidance with a concrete `CellController` identity shape, a fuller `ListViewController` forwarding skeleton, and prefetch forwarding.
- Added visible load-more row guidance with `LoadMoreViewModel`, `LoadMoreCellController`, and a minimal `Paginated<Item>` seam for UIKit pagination consumption.
- Expanded UIKit testing guidance with semantic list helpers, load-more test helpers, pagination integration test flow, and a note on fake refresh controls for deterministic tests.

## [0.1.0] — 2026-03-31

- Added file structure patterns reference for ViewController / UIView organization (extension separation, access level ordering, lazy var UI properties, layout placement).
- Added animation patterns reference (0.1s linear fade default, case-dependent transition durations, no spring unless requested).
- Cleaned up `SKILL.md` for AI-friendliness: operating rules with explicit trigger conditions, single topic router, removed empty placeholders.
- Removed "No Code Formatting or Linting" restriction from `AGENTS.md`.
