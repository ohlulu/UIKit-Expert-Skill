# List Composition Patterns for UITableView / UICollectionView

## Overview

Use row/item controllers and section controllers only when list complexity earns the extra indirection.

Goal:
- keep the list container thin
- keep row/section behavior local
- tie async work to UIKit lifecycle
- preserve stable identity during diffable updates

For simple lists, prefer direct `UITableViewDiffableDataSource`, `UICollectionViewDiffableDataSource`, `CellRegistration`, or a focused view controller.

For concrete default shapes used by this skill, also read [list-composition-shapes.md](list-composition-shapes.md).

## Decision Rule

### Consider row/item controllers when the list has:
- multiple cell types
- row-specific selection behavior
- async image or data loading per row
- prefetching and cancellation
- load-more / pagination rows
- a view controller growing around `switch indexPath.section` or `switch indexPath.row`

### Prefer simpler list code when the list has:
- one cell type
- trivial cell configuration
- no async row work
- little row-specific interaction
- one or two sections with minimal behavior

## Row / Item Controllers

A row/item controller may own:
- cell creation and configuration
- selection handling
- `willDisplay` / `didEndDisplaying`
- prefetch start / cancel
- async request start / cancel
- stable item identity

### Guidance
- Keep the list container responsible for snapshot application and callback forwarding.
- Keep row-specific behavior in the row/item controller, not in a large screen-level switch.
- Use a lightweight wrapper when needed to carry:
  - stable identity
  - cell data source object
  - optional delegate hooks
  - optional prefetch hooks

## Reuse and Async Lifecycle

For rows that load images or other async resources:
- reset visible state before starting a new request
- cancel work in `didEndDisplaying`, `cancelPrefetchingForItemsAt`, or equivalent lifecycle callbacks
- if the cell exposes a reuse callback, use it to release stale references or cancel work
- never assume a previously captured cell still represents the same model after reuse

## Identity and Diffable

Stable identity matters more than the abstraction itself.

- Use domain identity when the same logical item survives refreshes.
- Avoid generating a new random ID on every update unless the row is intentionally transient.
- Consider preserving an existing row/item controller when the same model remains on screen and the controller owns meaningful UI state or in-flight work.

## Incremental Pagination

A paginated UIKit list may be driven by a value that carries:
- the current items
- an optional next-page loader or continuation

This idea is useful, but the exact type belongs to the app/shared layer rather than UIKit-specific guidance.

When generating UIKit code, depend only on the UI-facing behavior the screen needs:
- current items to render
- whether more content can be loaded
- a way to request the next page

Do not prescribe a concrete app-level pagination type unless the project already has one.

## Section Controllers

Section abstractions are optional. Use them only when a section owns real behavior beyond grouping rows/items.

### UITableView sections

Consider a section controller when a section owns:
- header or footer view behavior
- header or footer sizing rules
- distinct row composition with clear semantic meaning

Guidance:
- keep the API narrow
- prefer section-specific capabilities over exposing the full `UITableViewDelegate` surface
- if a section only wraps an array of rows and adds no behavior, the abstraction may not be earning its keep

### UICollectionView + compositional layout

Section controllers are more justified here.

A collection section may reasonably own:
- item controllers for that section
- supplementary view creation
- `NSCollectionLayoutSection` creation

This fits the native compositional layout model.

If a layout concern is global to the whole collection view, keep that concern at the screen/container level instead of forcing one section to own global configuration.

## API Shape Guidance

When designing row/item/section abstractions:
- prefer precise capabilities over oversized protocols
- if many conformers return `nil` or implement no-op methods, the protocol is too broad
- property-style access is often clearer than methods for stored outputs like `cells`
- keep UIKit responsibilities aligned with UIKit boundaries:
  - row/item behavior at item level
  - section layout at section level
  - snapshot orchestration at container level

## Good Signs

The pattern is helping when:
- the screen controller mostly forwards events instead of branching on cell type
- async row work starts and stops in predictable lifecycle hooks
- new cell types can be added without rewriting a large switch
- section-specific layout or supplementary behavior stays near the section that owns it

## Warning Signs

The pattern is too heavy when:
- most controllers are one-line wrappers with no behavior
- the screen would be clearer with direct registrations and a small diffable data source
- protocols expand just to satisfy a few advanced sections
- trivial screens require jumping through many layers to debug basic behavior
