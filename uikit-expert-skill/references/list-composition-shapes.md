# Default Shapes for Complex UIKit Lists

## Purpose

When a UIKit list is complex enough to justify composition, and the project does not already define a stronger local pattern, generate code with these shapes by default.

Use this file as a generation guide, not as a pattern catalog.
Keep the code close to these skeletons unless the project already has a clearer established shape.

## Default Rules

- For `UITableView`, default to row controllers first.
- Add table section controllers only when sections own real behavior.
- For `UICollectionView` with compositional layout, default to section-aware composition.
- For visible incremental pagination, default to a dedicated load-more row/item.
- Keep the container thin.
- Keep identity stable.
- Tie async work to UIKit lifecycle callbacks.

## Incremental Pagination Contract

Do not generate a specific pagination value type as part of the UIKit skill.

UIKit code should depend only on:
- current items
- whether more content can be loaded
- a way to request the next page

If the project already has a pagination type, reuse it.
Do not invent a new app-level pagination type inside UIKit-only code.

## Load-More Row / Item Default

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

## UITableView Default

### Ownership

- **Container**
  - owns table view
  - owns diffable data source and snapshots
  - owns screen-level refresh / loading / error UI
  - forwards callbacks to row or section objects
- **Row controller**
  - owns cell configuration
  - owns selection
  - owns display lifecycle
  - owns prefetch / cancel when needed
  - owns row-level async work
- **Section controller**
  - optional
  - add only for header/footer behavior or meaningful section-specific row composition

### Canonical row wrapper

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

Generate this by default for complex table lists.

### Canonical container skeleton

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

Generate the container as a forwarding shell.
Do not rebuild row behavior inside a large `switch indexPath`.

### Canonical row controller skeleton

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

### Table section controllers

Do not generate a table section controller by default.

Generate one only when the section owns:
- header/footer view behavior
- header/footer sizing rules
- clear section-specific row composition

Keep the API narrow.
Do not expose the full `UITableViewDelegate` surface unless the project already uses that pattern consistently.

## UICollectionView Default

### Ownership

- **Container**
  - owns collection view
  - owns diffable data source and snapshots
  - owns compositional layout section provider
  - owns collection-level configuration
- **Item controller**
  - owns cell configuration
  - owns selection
  - owns item-level async work
- **Section controller**
  - owns items for that section
  - may own supplementary creation
  - may own `NSCollectionLayoutSection`

### Canonical item wrapper

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

### Canonical section wrapper

```swift
protocol SectionControllerDataSource {
    var cells: [CellController] { get }
    func layoutSection(at section: Int, environment: NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection
    func collectionView(_ collectionView: UICollectionView, viewForSupplementaryElementOfKind kind: String, at indexPath: IndexPath) -> UICollectionReusableView?
}

struct SectionController: Hashable {
    let id: AnyHashable
    let dataSource: SectionControllerDataSource
}
```

Generate this by default for compositional layout when sections differ meaningfully in layout, supplementary views, or item composition.

### Canonical container skeleton

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
- keep the collection container as snapshot owner, layout owner, and callback forwarder
- do not move section-specific layout decisions back into a large screen-level switch once section controllers exist
- keep global layout configuration at the screen/container level unless the whole screen clearly derives from one section-level requirement

## Identity Rule

Generate stable identity by default.

- use domain identity when the same logical row/item survives updates
- do not generate fresh random identifiers on every refresh unless the UI element is intentionally transient
- preserve an existing controller for the same logical model when it owns meaningful UI state or in-flight work

## Async Lifecycle Rule

Generate lifecycle-aware async cleanup.

- reset visible state before starting a request
- cancel work on end-display / cancel-prefetch / reuse boundaries
- do not keep stale cell references after reuse
- do not update a reused cell for an old model

## Do Not Generate These Abstractions When

- the screen has one cell type and trivial configuration
- there is no row/item-level async work
- there is little row/item-specific interaction
- one or two simple sections are enough
- direct registrations plus a small diffable data source would be clearer

## Failure Modes

The generated abstraction is wrong when:
- most controllers are pass-through wrappers
- protocols are broad and full of no-op conformers
- trivial screens require tracing many layers
- the container still contains a large `switch indexPath`
