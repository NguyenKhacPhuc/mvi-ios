---
name: domain-data-layer
description: "Add a new repository (Domain protocol + Data implementation + DI wiring) to an existing SwiftUI MVI app, OR refactor a feature that has data access inline in its EffectHandler. Use when the user says 'add a repository', 'wire up a backend', 'introduce a domain layer', 'extract this from the effect handler', 'make this testable', or when an effect handler is calling URLSession / Firebase / GRDB directly. SKIP when bootstrapping a brand-new project — use `swiftui-mvi-bootstrap` (it already creates the layers)."
---

# Add a Repository to an MVI App

This skill adds a single capability — fetching, persisting, or syncing one kind of data — across the four-layer architecture (Domain, Data, Presentation, Composition root). It's both the "new feature needs network" path and the "refactor inline mock out of the effect handler" path.

## Phase 1 — Detect the lay of the land

Before writing anything, confirm the project supports the pattern:

```bash
ls <App>/Core/Domain/ <App>/Core/Data/ <App>/App/AppContainer.swift 2>/dev/null
```

- All three exist → proceed.
- One or more missing → the project hasn't been bootstrapped to four layers. Either run `swiftui-mvi-bootstrap` first (new project), or pause and ask the user if they want to introduce the layers (each missing folder + the container file is one Write).
- `Core/Domain/DomainError.swift` should exist; if not, scaffold it (template in `swiftui-mvi-bootstrap`).

## Phase 2 — Gather inputs

Ask in one short message:

1. **What entity?** (e.g. `Story`, `User`, `Order`) — singular noun, PascalCase.
2. **What operations?** Pick from: `fetch`, `fetchByID`, `list`, `save`, `delete`, `observe`. Multiple OK.
3. **Backend?** `mock` (default), `URLSession`, `Firebase`, `Supabase`, `GRDB`, `CoreData`, `UserDefaults`, or `other` (the user names the impl). The mock always exists; pick one real backend if needed.
4. **Which feature consumes this?** Existing feature folder name, or "none yet".

Echo back, get confirmation, then proceed.

## Phase 3 — Domain layer

Add two files to `Core/Domain/`:

### `<Entity>.swift`

```swift
import Foundation

public struct <Entity>: Equatable, Hashable, Identifiable {
    public let id: String
    // ... fields driven by the user's spec ...

    public init(id: String /*, ... */) {
        self.id = id
        // ...
    }
}
```

Stick to value semantics. No `class`. No `URL` references unless the entity *is* a hyperlink. Conform to `Identifiable` only if there's a real ID — don't fabricate one to satisfy the protocol.

### `<Entity>Repository.swift`

```swift
import Foundation

public protocol <Entity>Repository {
    func fetch<Entities>() async throws -> [<Entity>]                    // if .list
    func fetch(id: <Entity>.ID) async throws -> <Entity>                 // if .fetchByID
    func save(_ item: <Entity>) async throws                             // if .save
    func delete(id: <Entity>.ID) async throws                            // if .delete
    func observe() -> AsyncStream<[<Entity>]>                            // if .observe
}
```

Include only the operations the user picked. Methods throw `DomainError` — never `URLError`, `DecodingError`, or framework-specific types. If a method can't fail, drop the `throws`.

**Domain layer rules (re-state in your message to the user if they're new to this):**
- Imports `Foundation` only.
- No `URL`, `URLRequest`, `URLSession`, `Firestore`, `NSManagedObjectContext`, etc.
- Pure data structures and protocols.
- Lives forever — public API for everyone above it.

## Phase 4 — Data layer

Add to `Core/Data/`:

### `Mock<Entity>Repository.swift` (always)

```swift
import Foundation

public struct Mock<Entity>Repository: <Entity>Repository {
    public init() {}

    public func fetch<Entities>() async throws -> [<Entity>] {
        try? await Task.sleep(nanoseconds: 200_000_000)
        return <Entity>.previewSet
    }
    // ... mirror the protocol; return seed data ...
}

private extension <Entity> {
    static let previewSet: [<Entity>] = [
        // 2-3 representative seed values
    ]
}
```

Mock impls are the **default in dev** — wired in `AppContainer.live` until the real backend is ready. They also power `#Preview` blocks and reducer tests.

### `<Backend><Entity>Repository.swift` (if a real backend was picked)

Example for URLSession:

```swift
import Foundation

public struct URLSession<Entity>Repository: <Entity>Repository {
    let base: URL
    let session: URLSession
    let decoder: JSONDecoder

    public init(base: URL, session: URLSession = .shared, decoder: JSONDecoder = JSONDecoder()) {
        self.base = base; self.session = session; self.decoder = decoder
    }

    public func fetch<Entities>() async throws -> [<Entity>] {
        do {
            let (data, response) = try await session.data(from: base.appending(path: "<entities>"))
            try Self.checkStatus(response)
            return try decoder.decode([<Entity>].self, from: data)
        } catch let e as DomainError { throw e }
        catch { throw Self.map(error) }
    }

    private static func checkStatus(_ response: URLResponse) throws {
        guard let http = response as? HTTPURLResponse else { throw DomainError.unknown }
        switch http.statusCode {
        case 200..<300: return
        case 401:       throw DomainError.unauthorized
        case 404:       throw DomainError.notFound
        case 500..<600: throw DomainError.server(message: "HTTP \(http.statusCode)")
        default:        throw DomainError.unknown
        }
    }

    private static func map(_ error: Error) -> DomainError {
        if (error as NSError).domain == NSURLErrorDomain { return .offline }
        return .unknown
    }
}
```

The Data layer is the **only** place that imports network/persistence frameworks AND the only place that maps their errors to `DomainError`. Never let `URLError` / `DecodingError` escape.

For Firebase / Supabase / GRDB / CoreData, mirror the same shape: real impl, error mapping in one place, no leakage.

## Phase 5 — Composition root

Open `App/AppContainer.swift` and add the new repo:

```swift
struct AppContainer {
    let greetings: GreetingRepository
    let <entities>: <Entity>Repository           // ← add this

    var homeEffectHandler: HomeEffectHandler { ... }
    var <feature>EffectHandler: <Feature>EffectHandler {
        <Feature>EffectHandler(<entities>: <entities>)
    }

    static let live = AppContainer(
        greetings: MockGreetingRepository(),
        <entities>: Mock<Entity>Repository()      // ← swap to real impl when ready
    )
}
```

If a real impl was created, also expose a `static let staging` / `static let production` variant or add an init param for the base URL — don't bake URLs into the type.

## Phase 6 — Wire into the feature

If the feature already exists, refactor its `EffectHandler` from `enum` (static) to `struct` (instance with the repo):

```swift
// before
enum <Feature>EffectHandler {
    static func handle(_ effect: <Feature>Effect) async -> <Feature>Intent? { ... }
}

// after
struct <Feature>EffectHandler {
    let <entities>: <Entity>Repository

    func handle(_ effect: <Feature>Effect) async -> <Feature>Intent? {
        switch effect {
        case .load:
            do {
                let items = try await <entities>.fetch<Entities>()
                return .loaded(items)
            } catch let e as DomainError {
                return .loadFailed(message(for: e))
            } catch {
                return .loadFailed("Something went wrong.")
            }
        }
    }

    private func message(for e: DomainError) -> String {
        switch e {
        case .offline:       return "You appear to be offline."
        case .unauthorized:  return "Please sign in again."
        case .notFound:      return "Not found."
        case .server(let m): return m
        case .unknown:       return "Something went wrong."
        }
    }
}
```

In the App entry, swap the static reference for the container:

```swift
// before
effectHandler: <Feature>EffectHandler.handle

// after
effectHandler: container.<feature>EffectHandler.handle
```

Update each `#Preview` to construct the handler with the mock directly — previews shouldn't go through the live container:

```swift
#Preview {
    let handler = <Feature>EffectHandler(<entities>: Mock<Entity>Repository())
    <Feature>View(store: Store(
        initialState: <Feature>State(),
        reducer: <Feature>Reducer.reduce,
        effectHandler: handler.handle
    ))
}
```

## Phase 7 — Tests

Add to `<App>Tests/`:

### `Mock<Entity>RepositoryTests.swift`

Pin the seed data so future changes are intentional:

```swift
import XCTest
@testable import <App>

final class Mock<Entity>RepositoryTests: XCTestCase {
    func test_fetch<Entities>_returnsSeed() async throws {
        let repo = Mock<Entity>Repository()
        let items = try await repo.fetch<Entities>()
        XCTAssertFalse(items.isEmpty)
    }
}
```

### Effect handler test using a stub repo (when error mapping matters)

Define stubs in the test target only:

```swift
struct Stub<Entity>Repository: <Entity>Repository {
    var result: Result<[<Entity>], Error> = .success([])
    func fetch<Entities>() async throws -> [<Entity>] { try result.get() }
    // ... default-implement the rest as fatalError ...
}

final class <Feature>EffectHandlerTests: XCTestCase {
    func test_load_offline_emitsFailWithOfflineMessage() async {
        let repo = Stub<Entity>Repository(result: .failure(DomainError.offline))
        let handler = <Feature>EffectHandler(<entities>: repo)
        let intent = await handler.handle(.load)
        guard case .loadFailed(let msg) = intent else { return XCTFail() }
        XCTAssertTrue(msg.contains("offline"))
    }
}
```

Stubs go **next to the tests**, never in `Core/Data/` — production code shouldn't ship test fakes.

## Phase 8 — Validate

```bash
SDK=$(xcrun --show-sdk-path --sdk macosx)
find <App> -name "*.swift" ! -path "*/View/*" ! -name "*App.swift" -print0 \
  | xargs -0 xcrun swiftc -typecheck -sdk "$SDK" -target arm64-apple-macos13.0
```

Empty output = clean. If errors mention `Cannot find type 'DomainError'`, the project doesn't have the layer yet — see Phase 1.

If the project is xcodegen-driven, regenerate so the new sources are in the target:

```bash
xcodegen generate
```

If not, the new files need pbxproj entries — see the `xcode-pbxproj-edit` skill.

## Caveats

- **Don't import a Data type from a feature.** Feature views and effect handlers see protocols only. Grep for `import.*Data` or `Mock<X>Repository` outside `Core/Data/` and `App/` — those are red flags.
- **Don't leak transport errors.** If you see `throw error` in the Data layer (re-throwing a `URLError` or `DecodingError`), wrap it in `DomainError`. The Domain boundary is the firewall.
- **Don't put repository instances in State.** `MVIState` is `Equatable`. Repositories aren't. Inject them into the EffectHandler, not the reducer.
- **Don't skip the protocol.** "I'll just call URLSession from the effect handler for now" is how you end up with the original problem. The protocol takes 8 lines and pays back forever.
- **Don't put live config in the type.** Base URLs, API keys, decoder customizations — pass them via init from `AppContainer`. The repo type is reusable across staging/prod.

## Cross-references

- `swiftui-mvi-bootstrap` — initial creation of Domain/Data/AppContainer if they don't exist.
- `swiftui-mvi-feature` — adding the *Presentation* side of a feature that consumes a repo.
- `xcode-pbxproj-edit` — adding new `.swift` files to a non-xcodegen project.
