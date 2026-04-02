# List Composition Patterns

## Overview

Use row/item controllers and section controllers only when list complexity earns the extra indirection.

Goal:
- keep the list container thin
- keep row/section behavior local
- tie async work to UIKit lifecycle
- preserve stable identity during diffable updates

For simple lists, prefer direct `UITableViewDiffableDataSource`, `UICollectionViewDiffableDataSource`, `CellRegistration`, or a focused view controller.

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

A row/item controller owns:
- cell creation and configuration
- selection handling
- `willDisplay` / `didEndDisplaying`
- prefetch start / cancel
- async request start / cancel
- stable item identity

### Default Row Wrapper (UITableView)

```swift
struct CellController: Hashable {
    let id: AnyHashable
    let dataSource: UITableViewDataSource
    let delegate: UITableViewDelegate?
    let prefetching: UITableViewDataSourcePrefetching?

    init(id: AnyHashable, _ dataSource: UITableViewDataSource) {
        self.id = id
        self.dataSource = dataSource
        self.delegate = dataSource as? UITableViewDelegate
        self.prefetching = dataSource as? UITableViewDataSourcePrefetching
    }
}
```

### Default Item Wrapper (UICollectionView)

```swift
struct CellController: Hashable {
    let id: AnyHashable
    let dataSource: UICollectionViewDataSource
    let delegate: UICollectionViewDelegate

    init(id: AnyHashable, _ dataSource: UICollectionViewDataSource & UICollectionViewDelegate) {
        self.id = id
        self.dataSource = dataSource
        self.delegate = dataSource
    }
}
```

### Row Controller Skeleton

```swift
final class AvatarRowController: NSObject, UITableViewDataSource, UITableViewDelegate, UITableViewDataSourcePrefetching {
    private let model: AvatarViewModel
    private weak var cell: AvatarCell?

    init(model: AvatarViewModel) {
        self.model = model
    }

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { 1 }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell: AvatarCell = tableView.dequeueReusableCell()
        self.cell = cell
        cell.nameLabel.text = model.name
        return cell
    }

    func tableView(_ tableView: UITableView, didEndDisplaying cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        cancelWork()
        self.cell = nil
    }
}
```

Hard rules:
- one row controller per logical row type
- store the model the row needs
- release stale cell references on reuse / end-display
- guard duplicate async starts in the controller or loader

## Section Controllers

Section abstractions are optional. Add them only when a section owns real behavior beyond grouping rows/items.

### UITableView Sections

Generate a section controller only when the section owns:
- header or footer view behavior
- header or footer sizing rules
- distinct row composition with clear semantic meaning

Keep the API narrow. Do not expose the full `UITableViewDelegate` surface unless the project already uses that pattern consistently.

### UICollectionView + Compositional Layout

Section controllers are more justified here. A collection section may own:
- item controllers for that section
- supplementary view creation
- `NSCollectionLayoutSection` creation

#### Default Section Wrapper

```swift
protocol SectionControlling {
    var cells: [CellController] { get }
    func layoutSection(at section: Int, environment: NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection
    func supplementaryView(for collectionView: UICollectionView, kind: String, at indexPath: IndexPath) -> UICollectionReusableView?
}

struct SectionController: Hashable {
    let id: AnyHashable
    let dataSource: SectionControlling
}
```

Generate this by default for compositional layout when sections differ meaningfully in layout, supplementary views, or item composition.

## Containers

The container owns:
- the table/collection view
- diffable data source and snapshots
- screen-level refresh / loading / error UI
- callback forwarding to row/section objects

Keep the container as a forwarding shell. Do not rebuild row behavior inside a large `switch indexPath`.

### UITableView Container Skeleton

```swift
final class ListViewController: UITableViewController, UITableViewDataSourcePrefetching {
    private var dataSource: UITableViewDiffableDataSource<Int, CellController>!

    func display(_ sections: [CellController]...) {
        var snapshot = NSDiffableDataSourceSnapshot<Int, CellController>()
        sections.enumerated().forEach { section, cells in
            snapshot.appendSections([section])
            snapshot.appendItems(cells, toSection: section)
        }
        dataSource.apply(snapshot)
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        cellController(at: indexPath)?.delegate?.tableView?(tableView, didSelectRowAt: indexPath)
    }

    override func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        cellController(at: indexPath)?.delegate?.tableView?(tableView, willDisplay: cell, forRowAt: indexPath)
    }

    override func tableView(_ tableView: UITableView, didEndDisplaying cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        cellController(at: indexPath)?.delegate?.tableView?(tableView, didEndDisplaying: cell, forRowAt: indexPath)
    }
}
```

### UICollectionView Container Skeleton

```swift
final class ProductListViewController: UIViewController {
    private lazy var dataSource = UICollectionViewDiffableDataSource<SectionController, CellController>(collectionView: collectionView) {
        collectionView, indexPath, controller in
        controller.dataSource.collectionView(collectionView, cellForItemAt: indexPath)
    }

    private func makeLayout() -> UICollectionViewLayout {
        UICollectionViewCompositionalLayout { [weak self] sectionIndex, environment in
            self?.dataSource.sectionIdentifier(for: sectionIndex)?
                .dataSource
                .layoutSection(at: sectionIndex, environment: environment)
        }
    }
}
```

Hard rules:
- keep global layout configuration at the screen/container level
- do not move section-specific layout decisions back into a large screen-level switch once section controllers exist

## Incremental Pagination

UIKit code should depend only on:
- current items
- whether more content can be loaded
- a way to request the next page

Do not invent a new app-level pagination type inside UIKit-only code. If the project already has one, reuse it.

### Load-More Row / Item

For visible incremental pagination, generate a dedicated load-more row/item by default.

Ownership:
- the container decides whether the load-more row/item exists
- the load-more row/item owns loading UI, error UI, retry UI, and next-page triggering

Trigger priority:
1. use `willDisplay` for simple end-of-list loading
2. add retry interaction for failure states
3. use prefetch or near-end detection only when earlier loading materially improves UX

Rules:
- guard duplicate requests
- hide or remove the load-more row/item when no next page exists
- do not generate a load-more row/item when pagination is invisible or unnecessary

## Identity

Generate stable identity by default.

- Use domain identity when the same logical item survives refreshes.
- Do not generate fresh random identifiers on every update unless the row is intentionally transient.
- Preserve an existing controller for the same logical model when it owns meaningful UI state or in-flight work.

## Async Lifecycle

Generate lifecycle-aware async cleanup.

- Reset visible state before starting a request.
- Cancel work on end-display / cancel-prefetch / reuse boundaries.
- Do not keep stale cell references after reuse.
- Do not update a reused cell for an old model.

## Warning Signs

The pattern is too heavy when:
- most controllers are one-line pass-through wrappers with no behavior
- the screen would be clearer with direct registrations and a small diffable data source
- protocols expand just to satisfy a few advanced sections
- trivial screens require jumping through many layers to debug basic behavior
- the container still contains a large `switch indexPath`
