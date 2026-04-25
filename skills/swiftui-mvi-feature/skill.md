---
name: swiftui-mvi-feature
description: "Scaffold a SwiftUI feature folder following the MVI (Model-View-Intent) pattern with State/Intent/Reducer/EffectHandler/View, plus pbxproj wiring. Use when implementing a new screen or feature in a SwiftUI iOS app that uses MVI architecture (look for Store<State,Intent,Effect>, MVIState/MVIIntent/MVIEffect protocols, pure reducers). Also use when adding routes/navigation between MVI features."
---

# SwiftUI MVI Feature Scaffolding

A SwiftUI MVI feature is **5 files per feature** in a strict folder layout. The reducer is pure, effects are async, and views observe a `Store` and dispatch intents. Everything else is consequence.

## Folder layout (mandatory)

```
Features/<FeatureName>/
├── Intent/
│   └── <Feature>Intent.swift          // enum Intent + enum Effect
├── Model/
│   └── <Feature>EffectHandler.swift   // async run of effects → optional follow-up Intent
├── State/
│   ├── <Feature>State.swift           // immutable Equatable struct
│   └── <Feature>Reducer.swift         // pure (State, Intent) -> (State, Effect?)
└── View/
    ├── <Feature>View.swift            // observes Store, dispatches intents
    └── <Feature>Components.swift      // (optional) split when View > 200 lines
```

Keep individual files under 200 lines. Split components into `<Feature>Components.swift`.

## State + Reducer pattern

State is a value type, conforms to `MVIState, Equatable`. All fields default-valued where possible so callers can do `MyState()` at startup.

```swift
struct MyState: MVIState, Equatable {
    var isLoading: Bool = false
    var errorMessage: String? = nil
    var items: [Item] = []
    var path: [Route] = []   // navigation, if MVI-driven
}
```

Reducer is a `static func` on an enum (not a class). **No I/O, no Date.now(), no Task.** State transitions only. Returns `(NextState, Effect?)`.

```swift
enum MyReducer {
    static func reduce(_ s: MyState, _ i: MyIntent) -> (MyState, MyEffect?) {
        var next = s
        switch i {
        case .onAppear, .refresh:
            next.isLoading = true
            return (next, .loadItems)
        case .loaded(let items):
            next.isLoading = false
            next.items = items
            return (next, nil)
        ...
        }
    }
}
```

## Effect handler (the only impure layer)

```swift
enum MyEffectHandler {
    static func handle(_ effect: MyEffect) async -> MyIntent? {
        switch effect {
        case .loadItems:
            try? await Task.sleep(nanoseconds: 250_000_000)
            return .loaded(MockData.items)   // emits a follow-up intent
        }
    }
}
```

The Store auto-dispatches the returned intent back through the reducer. **For typewriter / polling / re-scheduling effects**, return the same effect from the reducer on each tick:

```swift
case .tickReveal:
    let next = n + 1
    if next >= total { state.phase = .done; return (state, nil) }
    state.phase = .typing(next)
    return (state, .scheduleTick(afterMs: 30))   // re-schedules itself
```

## View

```swift
struct MyView: View {
    @ObservedObject var store: Store<MyState, MyIntent, MyEffect>

    var body: some View {
        content.onAppear { store.send(.onAppear) }
    }
}
```

For features that own a sub-feature's store, use `@StateObject` so it survives recompositions:

```swift
@StateObject private var subStore = Store(
    initialState: SubState(),
    reducer: SubReducer.reduce,
    effectHandler: SubEffectHandler.handle
)
```

## Navigation between MVI features

Use a typed route enum stored in state:

```swift
enum HomeRoute: Hashable {
    case detail(Story.ID)
    case player(PlaySession)
}
struct PlaySession: Hashable { let storyId: String; let chapterIndex: Int }

// State: var path: [HomeRoute] = []
// Intents: case openDetail(Story.ID), case openPlayer(...), case setPath([HomeRoute])
```

In the view:

```swift
NavigationStack(path: pathBinding) {
    rootContent
    .navigationDestination(for: HomeRoute.self) { route in
        switch route {
        case .detail(let id): DetailContainer(...)
        case .player(let s):  PlayerContainer(...)
        }
    }
}

private var pathBinding: Binding<[HomeRoute]> {
    Binding(get: { store.state.path },
            set: { store.send(.setPath($0)) })
}
```

**Avoid `navigationDestination(item:)`** — iOS 17+ only. Path-based works on iOS 16.

## Container view pattern (owns its store)

When a screen is reachable via navigation and needs construction parameters, wrap it:

```swift
struct DetailContainerView: View {
    let item: Item
    let onPlay: (Int) -> Void   // bubble actions back up the nav stack
    let onClose: () -> Void

    var body: some View {
        DetailView(
            store: Store(
                initialState: DetailState(item: item),
                reducer: DetailReducer.reduce,
                effectHandler: DetailEffectHandler.handle
            ),
            onPlay: onPlay,
            onClose: onClose
        )
    }
}
```

Pass `onClose` / `onPlay` callbacks from the parent so child features don't need to know about the parent's navigation.

## Tests

Reducers are pure → **every reducer should have unit tests** in `<Project>Tests/`. Pattern:

```swift
func test_onAppear_setsLoadingAndEmitsLoadEffect() {
    let (next, effect) = MyReducer.reduce(MyState(), .onAppear)
    XCTAssertTrue(next.isLoading)
    if case .loadItems = effect {} else { XCTFail("Expected .loadItems") }
}
```

## After scaffolding: pbxproj wiring

If the project does NOT use synchronized folders (`PBXFileSystemSynchronizedRootGroup`), each new `.swift` file needs entries in `.xcodeproj/project.pbxproj`. See the `xcode-pbxproj-edit` skill — it has stable-ID generation patterns and the four sections that must be updated.

## Compile check without Xcode

If only Command Line Tools are installed (no full Xcode), `xcodebuild` is unavailable. Catch most type errors with:

```bash
SDK=$(xcrun --show-sdk-path --sdk macosx)
find <ProjectDir> -name "*.swift" ! -path "*/View/*" ! -name "*App.swift" -print0 \
  | xargs -0 xcrun swiftc -typecheck -sdk "$SDK" -target arm64-apple-macos13.0
```

This typechecks all non-View, non-App files together (they're platform-agnostic). View files reference iOS-only APIs (`navigationBarHidden`, `toolbar(.hidden, for:)`, `topBarLeading`, `#Preview`) — filter those errors:

```bash
... 2>&1 | grep "error:" | grep -vE "is unavailable in macOS|external macro|PreviewsMacros|toolbar.*for:|topBarLeading|topBarTrailing|navigationBarBackButtonHidden|toolbarBackground|navigationBarHidden|refreshable|UIKit"
```

Empty output = clean.

## SourceKit cross-file diagnostics are noise

When editing a single MVI file, SourceKit reports "Cannot find type 'MVIState'", "Cannot find 'AppColor' in scope", etc. — because it analyzes files in isolation. These are not real errors as long as all files are in the same Swift module/target. Confirm with the typecheck-all command above before claiming a fix.
