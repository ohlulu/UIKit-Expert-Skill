# Cell Registration

## Core Principle

**Prefer `UICollectionView.CellRegistration` / `UITableView.CellRegistration` over legacy string-based `register` + `dequeueReusableCell`.** The modern API provides compile-time type safety, eliminates reuse-identifier typos, and centralizes cell configuration in a single handler closure.

- `UICollectionView.CellRegistration` — iOS 14+
- `UITableView.CellRegistration` — iOS 16+

## Basic Usage

```swift
// ✅ Modern — type-safe, no string identifier, no manual cast
let registration = UICollectionView.CellRegistration<MyCell, MyItem> { cell, indexPath, item in
    var content = cell.defaultContentConfiguration()
    content.text = item.title
    content.secondaryText = item.subtitle
    cell.contentConfiguration = content
}

// In diffable data source cell provider:
collectionView.dequeueConfiguredReusableCell(using: registration, for: indexPath, item: item)
```

```swift
// ❌ Legacy — string typo = runtime crash, manual cast
collectionView.register(MyCell.self, forCellWithReuseIdentifier: "MyCell")
let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "MyCell", for: indexPath) as! MyCell
```

## Pitfalls

### 1. Never Create Registration Inside Cell Provider (iOS 15+ crash)

iOS 15 added a runtime assertion. Creating `CellRegistration` inside `cellForItemAt` or a diffable data source `cellProvider` closure crashes immediately.

```swift
// ❌ CRASH on iOS 15+
dataSource = .init(collectionView: collectionView) { cv, indexPath, item in
    let reg = UICollectionView.CellRegistration<MyCell, MyItem> { cell, _, item in
        cell.configure(with: item)
    }
    return cv.dequeueConfiguredReusableCell(using: reg, for: indexPath, item: item)
}
```

Even "lazy but only created once" patterns are forbidden if the creation site is inside the provider.

**Fix**: store registrations as properties or create them during initialization.

```swift
// ✅ Registration as stored property
private let cellRegistration = UICollectionView.CellRegistration<MyCell, MyItem> { cell, _, item in
    cell.configure(with: item)
}
```

### 2. One Registration per Content Configuration Type

Mixing different `UIContentConfiguration` types on the same registration triggers a UIKit warning: *"A cell's existing content view must be replaced with a new one, which is expensive."*

Each registration manages its own reuse pool. Different configuration types → different registrations.

```swift
// ✅ Separate registrations for different cell configurations
let textRegistration = UICollectionView.CellRegistration<UICollectionViewListCell, TextItem> { cell, _, item in
    var content = cell.defaultContentConfiguration()
    content.text = item.text
    cell.contentConfiguration = content
}

let imageRegistration = UICollectionView.CellRegistration<ImageCell, ImageItem> { cell, _, item in
    cell.imageView.image = item.image
}
```

### 3. Closure Capture — Watch for Retain Cycles

The handler closure is held by the registration object. If the registration is a stored property on a view controller and the handler captures `self`, a retain cycle forms: `VC → registration → handler → VC`.

```swift
// ❌ Retain cycle
class MyVC: UIViewController {
    private lazy var registration = UICollectionView.CellRegistration<MyCell, MyItem> { [self] cell, _, item in
        self.trackAnalytics(item) // strong capture of self
    }
}

// ✅ Option A: weak self
private lazy var registration = UICollectionView.CellRegistration<MyCell, MyItem> { [weak self] cell, _, item in
    self?.trackAnalytics(item)
}

// ✅ Option B: configure using only the closure parameters (no external capture)
private let registration = UICollectionView.CellRegistration<MyCell, MyItem> { cell, _, item in
    var content = cell.defaultContentConfiguration()
    content.text = item.title
    cell.contentConfiguration = content
}
```

Option B is preferred — if the handler only uses `(cell, indexPath, item)`, no capture concern exists.

### 4. Dynamic / Server-Driven Cell Types

CellRegistration requires all registrations to be created before dequeue. For truly dynamic cell types unknown at startup (server-driven UI where the cell type set cannot be enumerated), **fall back to legacy `register` + `dequeueReusableCell`**.

Apple engineer confirmation: *"If your use case truly requires lazily registering cell types, you can use the traditional class/nib registration + reuse identifiers."*

Registration objects are lightweight — pre-creating many is fine. Only fall back when the type set is genuinely unbounded at compile time.

## Handler Lifecycle

The registration handler runs on every `dequeueConfiguredReusableCell` call, **after** `prepareForReuse()`. This is an improvement over the legacy pattern: configuration always runs after reset, reducing stale-state bugs.

Follow Apple's guidance: start from a fresh configuration each update (e.g., `defaultContentConfiguration()`), set all fields, assign to `cell.contentConfiguration`. Do not read-modify-write on existing configuration — build from scratch.

## Supplementary Views

`UICollectionView.SupplementaryRegistration` follows the same pattern for headers, footers, and other supplementary views:

```swift
let headerRegistration = UICollectionView.SupplementaryRegistration<MyHeader>(
    elementKind: UICollectionView.elementKindSectionHeader
) { header, _, indexPath in
    header.titleLabel.text = sections[indexPath.section].title
}

dataSource.supplementaryViewProvider = { collectionView, kind, indexPath in
    collectionView.dequeueConfiguredReusableSupplementary(using: headerRegistration, for: indexPath)
}
```

## Integration with List Composition Pattern

When using the [list-composition](list-composition.md) row/item controller pattern, CellRegistration can live inside individual row controllers:

```swift
final class AvatarItemController: ItemControllerDataSource {
    private let model: AvatarViewModel

    private let registration = UICollectionView.CellRegistration<AvatarCell, AvatarViewModel> { cell, _, model in
        cell.configure(with: model)
    }

    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        collectionView.dequeueConfiguredReusableCell(using: registration, for: indexPath, item: model)
    }
}
```

This keeps cell type knowledge inside the row controller while using type-safe registration. Each row controller owns its own registration — no shared string identifiers to coordinate.
