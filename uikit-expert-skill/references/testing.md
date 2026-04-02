# UIKit Testing and Testability Patterns

## Overview

For UIKit features, prefer tests that drive real view controllers and real views through UIKit lifecycle and user interactions.

Use mocks/spies at the boundaries that load data or perform side effects.
Do not replace UIKit itself with protocol-heavy test doubles unless the project already depends on that pattern.

## Test Levels

Use the smallest level that proves the behavior.

### 1. Composition / wiring tests

Use these to verify app wiring and navigation setup.

Examples:
- window becomes key and visible
- root view controller is the expected navigation stack
- selecting an item pushes the expected screen

### 2. Feature integration tests

Use these as the default for UIKit screens.

Test a real screen composed with:
- real UIKit views/controllers
- loader spies or stubs for data boundaries
- helper methods that simulate appearance, selection, pull-to-refresh, visibility, and reuse

Assert:
- title
- loading/error states
- rendered cells and visible content
- selection callbacks
- pagination behavior
- cancellation behavior

### 3. Acceptance tests

Use a small number of end-to-end tests to verify that real collaborators are correctly composed.

Prefer real infrastructure where the behavior matters, such as:
- real persistence store
- real cache logic
- network client stubs
- real scene/window composition

Keep acceptance coverage narrow and high-value.

## Default Testing Shape for UIKit Screens

When testing a UIKit screen, default to this shape:
- build the screen through its real composer/factory
- inject loader spies/stubs at the data boundary
- simulate lifecycle and user interactions through test helpers
- assert rendered UI state, not private implementation details

This keeps tests close to the user-visible behavior while still running fast and deterministically.

## Drive UIKit Through Real Lifecycle

Prefer simulating UIKit lifecycle over calling private methods directly.

Typical helpers:
- first appearance
- user-initiated refresh
- selection
- row becomes visible
- row stops being visible
- row becomes near-visible for prefetch
- row reuse

Minimal example:

```swift
extension UIViewController {
    func simulateAppearance() {
        loadViewIfNeeded()
        beginAppearanceTransition(true, animated: false)
        endAppearanceTransition()
    }
}
```

For list screens, add helpers on the concrete screen type so tests read in domain language rather than UIKit mechanics.

Example:

```swift
extension ListViewController {
    func simulateTapOnItem(at row: Int) {
        tableView.delegate?.tableView?(tableView, didSelectRowAt: IndexPath(row: row, section: 0))
    }
}
```

## Prefer Semantic Test Helpers

Test helpers should translate UIKit mechanics into feature language.

Prefer helpers like:
- `simulateUserInitiatedReload()`
- `simulateFeedImageViewVisible(at:)`
- `simulateTapOnFeedImage(at:)`
- `errorMessage`
- `numberOfRenderedComments()`

Avoid repeating raw index-path, control-event, and layout plumbing in every test.

## Assert Rendered State, Not Internals

For UIKit integration tests, assert what the user would observe:
- visible title
- labels/images rendered in cells
- loading indicator visibility
- error message visibility
- retry button visibility
- row count
- whether load-more is available

Avoid asserting:
- private stored properties
- exact internal call paths inside UIKit classes
- implementation details that do not affect user-visible behavior

## Async Testing Rule

Do not rely on sleeps or arbitrary delays.

Use async spies that:
- record requests
- let the test complete/fail each request explicitly
- report whether a request completed, failed, or was cancelled

This makes UI tests deterministic and lets you verify cancellation rules.

Typical behaviors worth testing:
- no duplicate request while a previous request is still running
- request starts when a screen first appears
- request starts again only after the previous one finishes or is cancelled
- image/data requests cancel when a row is no longer visible
- a reused cell does not render stale data from an older request

## Reuse, Visibility, and Cancellation

For reusable list cells with async work, test these transitions explicitly:
- not visible → visible
- near-visible → visible
- visible → not visible
- visible again after reuse
- request completes after the old cell has been reused

These tests are especially valuable for image loading, prefetching, and pagination rows.

## Screen-Specific Testability Hooks

It is acceptable to add small test-only helper extensions around UIKit types to make tests readable and deterministic.

Good examples:
- simulate `UIControl` events from tests
- force a layout cycle before reading visible state
- expose computed properties for visible UI state in test helpers
- replace platform-sensitive controls with fakes in tests when needed to keep behavior deterministic

Keep these helpers:
- tiny
- focused on behavior observation/simulation
- out of production code when possible

## Memory Leak Tracking

Track the screen under test and long-lived collaborators for leaks.

Minimal example:

```swift
extension XCTestCase {
    func trackForMemoryLeaks(_ instance: AnyObject, file: StaticString = #filePath, line: UInt = #line) {
        addTeardownBlock { [weak instance] in
            XCTAssertNil(instance, "Instance should have been deallocated.", file: file, line: line)
        }
    }
}
```

Use this for:
- the view controller under test
- loader spies/stubs retained by the screen
- adapters/presenters that should deallocate with the screen

## UIKit-Specific Guidance

### Pull-to-refresh
- test that refresh starts on first appearance if that is the feature behavior
- test that user-initiated refresh does not start a duplicate request while one is already running
- assert refresh indicator visibility while loading

### Navigation
- prefer asserting the pushed/presented screen type and visible state
- do not over-mock navigation controllers when a real navigation stack is easy to use in tests

### Pagination
- test whether load-more appears only when another page exists
- test loading/error/retry behavior for the load-more row/item
- test that load-more does not trigger duplicate requests

### Scene composition
- test scene/window setup with a real `UIWindow` when possible
- assert key-and-visible behavior and root controller composition

## Warning Signs

The test strategy is drifting when:
- most tests mock UIKit instead of driving real screens
- tests depend on sleeps or timing guesses
- the same UIKit interaction plumbing is repeated across many tests
- tests assert private implementation instead of rendered behavior
- async cell reuse bugs are left untested in screens that load data per row
