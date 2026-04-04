# Composer / Screen Composition Patterns

## Overview

Use a dedicated composer or factory when a UIKit screen needs more wiring than a plain initializer.

Goal:
- instantiate the screen in one place
- keep `UIViewController` focused on rendering and user events
- wire loaders, callbacks, and navigation handlers at the edge
- return a ready-to-display controller
- keep scene/root composition thin

## Decision Rule

### Consider a composer when the screen has:
- storyboard or nib loading plus dependency injection
- two or more external collaborators
- callback wiring such as `onRefresh`, retry, or selection
- UI-specific adapters or row controllers to create
- scene-level root composition in `SceneDelegate` or `AppDelegate`

### Prefer direct initialization when the screen has:
- one simple dependency
- no storyboard or nib
- no callback wiring
- no scene/root setup beyond `UIViewController()` or `UINavigationController(rootViewController:)`

## What the Composer Owns

A composer typically owns:
- instantiating the screen
- setting title or navigation items that belong to composition time
- creating UI-specific adapters or bridges
- wiring controller callbacks to those adapters
- injecting navigation closures
- returning the fully wired controller

A composer typically does **not** own:
- business rules
- persistence or network policy
- view rendering logic
- starting side effects during composition

## Screen Composer Skeleton

Consider this shape when a UIKit screen is storyboard-backed and needs callback wiring.

```swift
@MainActor
public final class FeedUIComposer {
    private init() {}

    public static func feedComposedWith(
        feedLoader: @escaping () async throws -> Paginated<FeedImage>,
        imageLoader: @escaping (URL) async throws -> Data,
        selection: @escaping (FeedImage) -> Void = { _ in }
    ) -> ListViewController {
        let controller = makeFeedViewController(title: "Feed")
        let adapter = FeedLoaderAdapter(
            controller: controller,
            feedLoader: feedLoader,
            imageLoader: imageLoader,
            selection: selection
        )

        controller.onRefresh = adapter.loadResource
        return controller
    }

    private static func makeFeedViewController(title: String) -> ListViewController {
        let bundle = Bundle(for: ListViewController.self)
        let storyboard = UIStoryboard(name: "Feed", bundle: bundle)
        let controller = storyboard.instantiateInitialViewController() as! ListViewController
        controller.title = title
        return controller
    }
}
```

`FeedLoaderAdapter` stands for the UI-specific bridge between controller callbacks and the injected loaders. The composer creates that bridge, but the bridge owns the loading flow.

## Navigation Wiring

Selection-driven navigation is often easiest to compose as a closure injected into the screen.

```swift
@MainActor
public static func feedComposedWith(
    feedLoader: @escaping () async throws -> Paginated<FeedImage>,
    imageLoader: @escaping (URL) async throws -> Data,
    selection: @escaping (FeedImage) -> Void
) -> ListViewController {
    let controller = makeFeedViewController(title: "Feed")
    let adapter = FeedLoaderAdapter(
        controller: controller,
        feedLoader: feedLoader,
        imageLoader: imageLoader,
        selection: selection
    )

    controller.onRefresh = adapter.loadResource
    return controller
}
```

This keeps push/present decisions outside the view controller while still making the wiring explicit at composition time.

## Scene Root Composition

When a scene needs a root navigation stack plus child-screen composition, keep that wiring in the scene layer and delegate each screen to its composer.

```swift
final class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    private lazy var navigationController = UINavigationController(
        rootViewController: FeedUIComposer.feedComposedWith(
            feedLoader: feedService.loadFeed,
            imageLoader: feedService.loadImageData,
            selection: showComments
        )
    )

    func configureWindow() {
        window?.rootViewController = navigationController
        window?.makeKeyAndVisible()
    }

    private func showComments(for image: FeedImage) {
        let controller = CommentsUIComposer.commentsComposedWith(
            commentsLoader: { try await feedService.loadComments(for: image) }
        )

        navigationController.pushViewController(controller, animated: true)
    }
}
```

The scene layer decides navigation structure. Each screen composer decides how its own screen is instantiated and wired.

## Hard Rules

- Do not trigger loaders during composition. Wait for lifecycle or user action.
- Avoid retain cycles when wiring closures back into navigation or loaders.
- Keep storyboard or nib lookup private to the composer unless local convention says otherwise.
- Return a controller that is ready to display without more external mutation.

## Testability

Composers are easiest to verify through wiring tests:
- assert the returned controller type
- assert the title or root navigation structure
- simulate user actions on the composed screen instead of calling adapter internals
- keep scene composition tests at the scene/root level

## Warning Signs

The composition boundary is drifting when:
- `SceneDelegate` manually wires row controllers or per-cell logic
- a view controller fetches app dependencies directly
- the composer starts rendering or mapping view models itself
- the same storyboard lookup and callback wiring is duplicated across call sites
