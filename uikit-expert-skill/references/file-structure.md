# File Structure Patterns for UIViewController / UIView

## Overview

UIKit view controllers and views follow a consistent file structure that separates concerns into the main declaration and purpose-specific extensions. The main declaration answers "what is this object?", and extensions answer "what does it do?"

## Main Declaration

The main declaration contains only:
1. **Protocol conformances** — all listed on the class declaration line
2. **Properties** — ordered by access level: public → internal → private. UI properties use the simplest form that fits: direct initialization for one-line setup, `lazy var` with a closure only when inline configuration needs multiple statements (see [UI Properties](#ui-properties)).
3. **Lifecycle methods** — after all properties

```swift
class ProfileViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {

    lazy var titleLabel: UILabel = {
        let label = UILabel()
        label.font = .systemFont(ofSize: 16)
        return label
    }()

    lazy var tableView: UITableView = {
        let tableView = UITableView()
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "Cell")
        return tableView
    }()

    private let viewModel: ProfileViewModel

    // MARK: - Lifecycle

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        viewModel.refresh()
    }
}
```

## Extensions by Responsibility

Everything outside properties and lifecycle goes into extensions, each with a single responsibility.

### Protocol Conformance Extensions

Each protocol gets its own extension. The conformance is already declared on the class line — extensions only organize the implementation.

```swift
// MARK: - UITableViewDataSource
extension ProfileViewController {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        viewModel.items.count
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        cell.textLabel?.text = viewModel.items[indexPath.row].title
        return cell
    }
}

// MARK: - UITableViewDelegate
extension ProfileViewController {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
    }
}
```

### Action Handler Extension

All `@objc` action methods live in a dedicated private extension, separate from layout and protocol conformances.

```swift
// MARK: - Actions
private extension ProfileViewController {
    @objc func didTapSaveButton() {
        viewModel.save()
    }

    @objc func didPullToRefresh() {
        viewModel.refresh()
    }
}
```

### Layout Extension (Always Last)

Layout code is always the **last section** in the file. A single `private extension` contains `setupUI()`, which handles both view hierarchy assembly (`addSubview`) and constraint setup.

```swift
// MARK: - Layout
private extension ProfileViewController {
    func setupUI() {
        view.addSubview(titleLabel)
        view.addSubview(tableView)

        titleLabel.translatesAutoresizingMaskIntoConstraints = false
        tableView.translatesAutoresizingMaskIntoConstraints = false

        NSLayoutConstraint.activate([
            titleLabel.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 16),
            titleLabel.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
            titleLabel.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),

            tableView.topAnchor.constraint(equalTo: titleLabel.bottomAnchor, constant: 8),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])
    }
}
```

## UI Properties

UI properties should use the **simplest initializer form that keeps the code obvious**. Default access level is `internal`.

Use direct initialization when a single expression is enough:

```swift
lazy var subtotalRow: SummaryRow = SummaryRow(label: L10n.tr("cashier.subtotal"))
```

Use **`lazy var` with a closure** only when inline configuration needs multiple statements:

```swift
lazy var avatarImageView: UIImageView = {
    let imageView = UIImageView()
    imageView.contentMode = .scaleAspectFill
    imageView.clipsToBounds = true
    imageView.layer.cornerRadius = 24
    return imageView
}()
```

Do **not** add `[unowned self]` or `[weak self]` to `lazy var` closures — the closure executes after `self` is fully initialized and is not retained beyond that point.

## Access Level Ordering

Within any scope (main declaration or extension), order members by access level:

1. `public` / `open`
2. `internal` (implicit)
3. `private`

**Exception**: when two members have a strong logical relationship (e.g., a public method and its private helper), keep them together regardless of access level.

## Complete File Structure Summary

```
┌─────────────────────────────────────────┐
│ class Foo: UIViewController, Protocols  │
│                                         │
│   internal properties (direct init / lazy)│
│   private properties                    │
│   lifecycle methods                     │
│                                         │
├─────────────────────────────────────────┤
│ extension Foo  // Protocol A methods    │
├─────────────────────────────────────────┤
│ extension Foo  // Protocol B methods    │
├─────────────────────────────────────────┤
│ private extension Foo  // Actions       │
├─────────────────────────────────────────┤
│ private extension Foo  // Layout (LAST) │
└─────────────────────────────────────────┘
```
