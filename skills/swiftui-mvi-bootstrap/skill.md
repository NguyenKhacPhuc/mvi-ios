---
name: swiftui-mvi-bootstrap
description: "Bootstrap a brand-new SwiftUI iOS app project from scratch with the MVI (Model-View-Intent) architecture pre-wired: generic Store, design-system tokens, a starter Home feature, reducer test, signing scoped to simulator, and a CLAUDE.md describing the conventions. Uses xcodegen so the project file is regenerable from a declarative `project.yml`. Use when the user says 'start a new project', 'scaffold an app', 'create a new SwiftUI app', 'new iOS project', or 'bootstrap an iOS MVI app from zero'. SKIP when an existing `.xcodeproj` is in the working directory — use `swiftui-mvi-feature` to add to it instead."
---

# SwiftUI MVI Project Bootstrap

Creates a complete, buildable iOS SwiftUI app from zero using the MVI conventions developed across the sibling skills. Output is a regenerable [xcodegen](https://github.com/yonaskolb/XcodeGen)-driven project, not a hand-baked `.pbxproj`.

## Phase 1 — Gather inputs (MANDATORY before any file is written)

Ask the user the following, in order. If they answer multiple at once, parse them. If unsure or skipped, use the listed default. Keep the question short and conversational — don't dump the whole list at once unless the user requested it.

| # | Question | Default | Notes |
|---|---|---|---|
| 1 | **App name?** (display + product name) | _required_ | e.g. "StoryWorld" — used as scheme, target, display name |
| 2 | **Bundle ID prefix?** | `com.<reverse-username>` | Final bundle ID = `<prefix>.<slug(app name)>` |
| 3 | **Apple Development team ID?** | empty | 10-char alphanumeric. Empty = signing left automatic-no-team for now; user fills in Xcode later. |
| 4 | **Minimum iOS version?** | `16.0` | |
| 5 | **Output directory?** | `~/Desktop/<AppName>` | Must not already exist. |
| 6 | **Initialize git + initial commit?** | yes | |

Echo the parsed values back as a confirmation before proceeding:

```
About to bootstrap:
  App name:       StoryWorld
  Bundle ID:      com.phucnk.storyworld
  Team ID:        YC7G67SKRA  (or "(none — set later in Xcode)")
  Min iOS:        16.0
  Output:         /Users/steve/Desktop/StoryWorld
  Git init:       yes

Proceed? (yes/no)
```

Wait for confirmation. If they correct any field, re-echo and ask again.

## Phase 2 — Preflight

```bash
command -v xcodegen >/dev/null 2>&1 || {
  echo "xcodegen not found."
  read -p "Install via 'brew install xcodegen'? (y/n) " ans
  [ "$ans" = "y" ] && brew install xcodegen || exit 1
}
```

If `command -v brew` also fails, tell the user to install Homebrew from https://brew.sh first; do not attempt to install brew automatically.

## Phase 3 — Variable substitution

Throughout the templates below, replace these placeholders with the user's inputs:

| Placeholder | Source |
|---|---|
| `{{APP_NAME}}` | answer 1 verbatim |
| `{{APP_SLUG}}` | answer 1 lowercased, spaces stripped (e.g. "StoryWorld" → "storyworld") |
| `{{BUNDLE_ID}}` | `<prefix>.<slug>` |
| `{{TEAM_ID}}` | answer 3 (empty string if skipped) |
| `{{MIN_IOS}}` | answer 4 |
| `{{OUTPUT_DIR}}` | answer 5 |

## Phase 4 — Directory layout

```
{{OUTPUT_DIR}}/
├── project.yml
├── README.md
├── CLAUDE.md
├── .gitignore
├── {{APP_NAME}}/
│   ├── App/
│   │   └── {{APP_NAME}}App.swift
│   ├── Core/
│   │   ├── DesignSystem/
│   │   │   ├── AppIcon.swift
│   │   │   ├── Colors.swift
│   │   │   ├── Spacing.swift
│   │   │   └── Typography.swift
│   │   └── MVI/
│   │       ├── MVIProtocols.swift
│   │       └── Store.swift
│   ├── Features/
│   │   └── Home/
│   │       ├── Intent/HomeIntent.swift
│   │       ├── Model/HomeEffectHandler.swift
│   │       ├── State/HomeReducer.swift
│   │       ├── State/HomeState.swift
│   │       └── View/HomeView.swift
│   └── Resources/
│       └── Assets.xcassets/
│           └── Contents.json
└── {{APP_NAME}}Tests/
    └── HomeReducerTests.swift
```

`mkdir -p` everything, then write each file. Do NOT create `Assets.xcassets/AppIcon.appiconset/` here — leave it for the user to run `ios-appicon-from-art` once they have artwork.

## Phase 5 — Files (literal contents)

All file bodies live as code-block templates in this skill. Substitute the placeholders, then `Write` each one to the target path.

### `project.yml`

```yaml
name: {{APP_NAME}}
options:
  bundleIdPrefix: {{BUNDLE_ID}}
  deploymentTarget:
    iOS: "{{MIN_IOS}}"
  createIntermediateGroups: true
  generateEmptyDirectories: true

settings:
  base:
    SWIFT_VERSION: "5.9"
    DEVELOPMENT_TEAM: "{{TEAM_ID}}"
    CODE_SIGN_STYLE: Automatic
    CODE_SIGNING_ALLOWED[sdk=iphonesimulator*]: NO
    CODE_SIGNING_REQUIRED[sdk=iphonesimulator*]: NO
    CODE_SIGN_IDENTITY[sdk=iphonesimulator*]: ""
    CODE_SIGN_IDENTITY[sdk=iphoneos*]: "Apple Development"

targets:
  {{APP_NAME}}:
    type: application
    platform: iOS
    sources:
      - path: {{APP_NAME}}
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: {{BUNDLE_ID}}
        GENERATE_INFOPLIST_FILE: YES
        INFOPLIST_KEY_CFBundleDisplayName: "{{APP_NAME}}"
        INFOPLIST_KEY_UIApplicationSceneManifest_Generation: YES
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
        INFOPLIST_KEY_UISupportedInterfaceOrientations: "UIInterfaceOrientationPortrait UIInterfaceOrientationPortraitUpsideDown UIInterfaceOrientationLandscapeLeft UIInterfaceOrientationLandscapeRight"
        TARGETED_DEVICE_FAMILY: "1,2"

  {{APP_NAME}}Tests:
    type: bundle.unit-test
    platform: iOS
    sources:
      - path: {{APP_NAME}}Tests
    dependencies:
      - target: {{APP_NAME}}
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: {{BUNDLE_ID}}.tests
        GENERATE_INFOPLIST_FILE: YES
```

### `{{APP_NAME}}/Core/MVI/MVIProtocols.swift`

```swift
import Foundation

/// Immutable value describing a feature's UI at a moment in time.
public protocol MVIState {}

/// Action a user (or the system) wants to perform.
public protocol MVIIntent {}

/// Async work that should happen as a result of an intent.
public protocol MVIEffect {}
```

### `{{APP_NAME}}/Core/MVI/Store.swift`

```swift
import Foundation
import SwiftUI

public typealias Reducer<State, Intent, Effect> = (State, Intent) -> (State, Effect?)
public typealias EffectHandler<Intent, Effect> = (Effect) async -> Intent?

@MainActor
public final class Store<State, Intent, Effect>: ObservableObject {

    @Published public private(set) var state: State

    private let reducer: Reducer<State, Intent, Effect>
    private let effectHandler: EffectHandler<Intent, Effect>

    public init(
        initialState: State,
        reducer: @escaping Reducer<State, Intent, Effect>,
        effectHandler: @escaping EffectHandler<Intent, Effect> = { _ in nil }
    ) {
        self.state = initialState
        self.reducer = reducer
        self.effectHandler = effectHandler
    }

    public func send(_ intent: Intent) {
        let (nextState, effect) = reducer(state, intent)
        state = nextState

        guard let effect else { return }
        Task { [weak self] in
            guard let self else { return }
            if let nextIntent = await self.effectHandler(effect) {
                self.send(nextIntent)
            }
        }
    }
}
```

### `{{APP_NAME}}/Core/DesignSystem/Colors.swift`

```swift
import SwiftUI

public enum AppColor {
    public static let background    = Color(hex: 0x0E0B14)
    public static let surface       = Color(hex: 0x1B1626)
    public static let textPrimary   = Color.white
    public static let textSecondary = Color(hex: 0xC9C2D6)
    public static let textTertiary  = Color(hex: 0x8A8398)
    public static let primary       = Color(hex: 0xFFB0CA)
    public static let onPrimary     = Color(hex: 0x2A1320)
    public static let divider       = Color.white.opacity(0.08)
}

extension Color {
    init(hex: UInt32, alpha: Double = 1.0) {
        let r = Double((hex >> 16) & 0xFF) / 255.0
        let g = Double((hex >> 8)  & 0xFF) / 255.0
        let b = Double(hex & 0xFF) / 255.0
        self.init(.sRGB, red: r, green: g, blue: b, opacity: alpha)
    }
}
```

### `{{APP_NAME}}/Core/DesignSystem/Typography.swift`

```swift
import SwiftUI

public enum AppFont {
    public static let displayTitle = Font.system(size: 28, weight: .bold)
    public static let largeTitle   = Font.system(size: 24, weight: .bold)
    public static let title        = Font.system(size: 20, weight: .semibold)
    public static let headline     = Font.system(size: 17, weight: .semibold)
    public static let bodyEmph     = Font.system(size: 15, weight: .semibold)
    public static let body         = Font.system(size: 15, weight: .regular)
    public static let caption      = Font.system(size: 13, weight: .regular)
    public static let captionEmph  = Font.system(size: 13, weight: .semibold)
}
```

### `{{APP_NAME}}/Core/DesignSystem/Spacing.swift`

```swift
import CoreGraphics

public enum AppSpacing {
    public static let xxs: CGFloat = 2
    public static let xs:  CGFloat = 4
    public static let sm:  CGFloat = 8
    public static let md:  CGFloat = 12
    public static let lg:  CGFloat = 16
    public static let xl:  CGFloat = 24
    public static let xxl: CGFloat = 32
}
```

### `{{APP_NAME}}/Core/DesignSystem/AppIcon.swift`

```swift
import SwiftUI

/// Renders a template SVG/PNG asset from Assets.xcassets at a given size.
/// Tints via `.foregroundStyle()` when the imageset has template-rendering-intent.
struct AppIcon: View {
    let name: String
    var size: CGFloat = 24

    var body: some View {
        Image(name)
            .resizable()
            .scaledToFit()
            .frame(width: size, height: size)
    }
}
```

### `{{APP_NAME}}/App/{{APP_NAME}}App.swift`

```swift
import SwiftUI

@main
struct {{APP_NAME}}App: App {
    var body: some Scene {
        WindowGroup {
            HomeView(
                store: Store(
                    initialState: HomeState(),
                    reducer: HomeReducer.reduce,
                    effectHandler: HomeEffectHandler.handle
                )
            )
        }
    }
}
```

### `{{APP_NAME}}/Features/Home/State/HomeState.swift`

```swift
import Foundation

struct HomeState: MVIState, Equatable {
    var isLoading: Bool = false
    var greeting: String = "Welcome to {{APP_NAME}}"
    var errorMessage: String? = nil
}
```

### `{{APP_NAME}}/Features/Home/Intent/HomeIntent.swift`

```swift
import Foundation

enum HomeIntent: MVIIntent {
    case onAppear
    case greetingLoaded(String)
    case loadFailed(String)
}

enum HomeEffect: MVIEffect {
    case loadGreeting
}
```

### `{{APP_NAME}}/Features/Home/State/HomeReducer.swift`

```swift
import Foundation

enum HomeReducer {
    static func reduce(_ state: HomeState, _ intent: HomeIntent) -> (HomeState, HomeEffect?) {
        var s = state
        switch intent {
        case .onAppear:
            s.isLoading = true
            s.errorMessage = nil
            return (s, .loadGreeting)
        case .greetingLoaded(let greeting):
            s.isLoading = false
            s.greeting = greeting
            return (s, nil)
        case .loadFailed(let message):
            s.isLoading = false
            s.errorMessage = message
            return (s, nil)
        }
    }
}
```

### `{{APP_NAME}}/Features/Home/Model/HomeEffectHandler.swift`

```swift
import Foundation

enum HomeEffectHandler {
    static func handle(_ effect: HomeEffect) async -> HomeIntent? {
        switch effect {
        case .loadGreeting:
            try? await Task.sleep(nanoseconds: 200_000_000)
            return .greetingLoaded("Hello from {{APP_NAME}}!")
        }
    }
}
```

### `{{APP_NAME}}/Features/Home/View/HomeView.swift`

```swift
import SwiftUI

struct HomeView: View {
    @ObservedObject var store: Store<HomeState, HomeIntent, HomeEffect>

    var body: some View {
        ZStack {
            AppColor.background.ignoresSafeArea()
            VStack(spacing: AppSpacing.lg) {
                if store.state.isLoading {
                    ProgressView().tint(AppColor.primary)
                } else if let error = store.state.errorMessage {
                    Text(error)
                        .font(AppFont.body)
                        .foregroundStyle(AppColor.textSecondary)
                } else {
                    Text(store.state.greeting)
                        .font(AppFont.displayTitle)
                        .foregroundStyle(AppColor.textPrimary)
                }
            }
            .padding(AppSpacing.xl)
        }
        .preferredColorScheme(.dark)
        .onAppear { store.send(.onAppear) }
    }
}

#Preview {
    HomeView(
        store: Store(
            initialState: HomeState(),
            reducer: HomeReducer.reduce,
            effectHandler: HomeEffectHandler.handle
        )
    )
}
```

### `{{APP_NAME}}/Resources/Assets.xcassets/Contents.json`

```json
{
  "info" : { "author" : "xcode", "version" : 1 }
}
```

### `{{APP_NAME}}Tests/HomeReducerTests.swift`

```swift
import XCTest
@testable import {{APP_NAME}}

final class HomeReducerTests: XCTestCase {

    func test_onAppear_setsLoadingAndEmitsLoadEffect() {
        let (next, effect) = HomeReducer.reduce(HomeState(), .onAppear)
        XCTAssertTrue(next.isLoading)
        XCTAssertNil(next.errorMessage)
        if case .loadGreeting = effect {} else { XCTFail("Expected .loadGreeting") }
    }

    func test_greetingLoaded_clearsLoadingAndStoresGreeting() {
        var s = HomeState()
        s.isLoading = true
        let (next, effect) = HomeReducer.reduce(s, .greetingLoaded("hi"))
        XCTAssertFalse(next.isLoading)
        XCTAssertEqual(next.greeting, "hi")
        XCTAssertNil(effect)
    }

    func test_loadFailed_setsErrorMessage() {
        var s = HomeState()
        s.isLoading = true
        let (next, _) = HomeReducer.reduce(s, .loadFailed("oops"))
        XCTAssertFalse(next.isLoading)
        XCTAssertEqual(next.errorMessage, "oops")
    }
}
```

### `CLAUDE.md`

```markdown
# {{APP_NAME}} — Claude instructions

## Architecture

This is a SwiftUI iOS app using **MVI** (Model–View–Intent). When generating or modifying feature code:

- **State** is a value type (`struct`), `Equatable`, and immutable from the view's perspective.
- **Intent** is an `enum` of all actions the user/system can trigger.
- **Effect** is an `enum` of async work the reducer can request.
- **Reducer** is a pure static function `reduce(State, Intent) -> (State, Effect?)`. No I/O, no `Task`, no `Date.now()`, nothing impure.
- **EffectHandler** is a static async function that runs the effect and returns an optional follow-up Intent.
- Views observe `Store<State, Intent, Effect>` via `@ObservedObject` and dispatch through `store.send(_:)`.

## Folder convention

One folder per feature under `{{APP_NAME}}/Features/<FeatureName>/`:
- `Model/` — domain types, repositories, effect handlers
- `Intent/` — `Intent` and `Effect` enums
- `State/` — `State` struct and pure `Reducer`
- `View/` — SwiftUI views

## Design system

All colors, fonts, and spacing must come from `Core/DesignSystem/`. No raw hex values, font sizes, or magic-number paddings in feature code.

## Project file regeneration

The `.xcodeproj` is generated from `project.yml` via [xcodegen](https://github.com/yonaskolb/XcodeGen). To regenerate after structural changes:

    xcodegen generate

Don't hand-edit the `.pbxproj` — edits will be lost on next regenerate.

## Tests

Reducers are pure → **every reducer should have unit tests** in `{{APP_NAME}}Tests/`. See `HomeReducerTests.swift` for the pattern.
```

### `README.md`

```markdown
# {{APP_NAME}}

SwiftUI iOS app built on the MVI (Model-View-Intent) pattern.

## Setup

    brew install xcodegen
    xcodegen generate
    open {{APP_NAME}}.xcodeproj

## Architecture

See [`CLAUDE.md`](CLAUDE.md) for the full conventions. Quick tour:

- `Core/MVI/` — the generic `Store<State, Intent, Effect>`.
- `Core/DesignSystem/` — `AppColor`, `AppFont`, `AppSpacing` tokens. All visual constants live here.
- `Features/<Name>/{Intent,Model,State,View}/` — one folder per feature, MVI parts kept separate.

## Tests

    xcodebuild test -project {{APP_NAME}}.xcodeproj -scheme {{APP_NAME}} -destination 'platform=iOS Simulator,name=iPhone 15'
```

### `.gitignore`

```
.DS_Store
.build/
DerivedData/
xcuserdata/
*.xcuserstate
*.xcodeproj/         # regenerated from project.yml
!*.xcodeproj/project.xcworkspace
```

## Phase 6 — Generate the project

```bash
cd {{OUTPUT_DIR}}
xcodegen generate
```

If xcodegen reports missing files, the layout in Phase 4 wasn't fully written — re-run and double-check.

## Phase 7 — Validate

Compile-check non-View files against macOS SDK (no full Xcode required):

```bash
SDK=$(xcrun --show-sdk-path --sdk macosx)
find {{APP_NAME}} -name "*.swift" ! -path "*/View/*" ! -name "*App.swift" -print0 \
  | xargs -0 xcrun swiftc -typecheck -sdk "$SDK" -target arm64-apple-macos13.0
```

Empty output = clean. If errors mention `MVIState`, `MVIIntent`, `MVIEffect`, `Store`, etc. — verify the Core/MVI files were written correctly.

## Phase 8 — Git init (if user opted in)

```bash
cd {{OUTPUT_DIR}}
git init -q -b main
git add -A
git -c user.email=<from-git-config> -c user.name=<from-git-config> \
  commit -q -m "Initial commit: {{APP_NAME}} scaffolded via swiftui-mvi-bootstrap"
```

Don't push to a remote unless the user explicitly asks. Print the path and remind them they can `gh repo create` themselves.

## Phase 9 — Final report

End with a concise summary:

```
✓ Scaffolded {{APP_NAME}} at {{OUTPUT_DIR}}
✓ project.yml + xcodegen → {{APP_NAME}}.xcodeproj
✓ Core/MVI Store + protocols
✓ Core/DesignSystem tokens (Colors, Typography, Spacing, AppIcon)
✓ Features/Home/{Intent,Model,State,View} starter feature
✓ HomeReducerTests with 3 cases
✓ CLAUDE.md + README.md + .gitignore

Open in Xcode:
  open {{OUTPUT_DIR}}/{{APP_NAME}}.xcodeproj

Next steps:
  • Add a real AppIcon (run skill `ios-appicon-from-art`).
  • Pull design assets if you have a Figma file (skill `figma-asset-pull`).
  • Add a new feature (skill `swiftui-mvi-feature`).
  • Set DEVELOPMENT_TEAM in project.yml when ready to archive.
```

## Caveats

- **xcodegen rewrites `.pbxproj` on every run.** If the user later switches to hand-editing, they need to delete `project.yml` or stop running `xcodegen generate`. Document this in `CLAUDE.md` — already covered.
- **App name with spaces** ("My App") becomes a Swift module name with no spaces ("MyApp"). The slug is more aggressive — lowercase, no spaces. Test names import the module, so `@testable import MyApp`, not `import My App`.
- **Empty team ID** is fine for simulator development. Don't fabricate a team ID — leave the build setting blank and tell the user to fill it in Xcode → Signing & Capabilities when they're ready to run on device.
- **Bundle ID collisions** can't be detected up front (the App Store registry isn't queryable without auth). If the user's prefix collides, the `ios-archive-validation` skill covers the fix.
- **xcodegen vs Xcode 14+ synchronized folders**: the latter requires Xcode to manage the file system. xcodegen doesn't use synchronized folders, so adding new sources outside the spec means re-running `xcodegen generate`. Document this in CLAUDE.md (covered).

## When not to use this skill

- If `<Output>/.xcodeproj` already exists. Use `swiftui-mvi-feature` to add to it instead.
- If the user wants UIKit, AppKit, or non-MVI architecture — this skill is opinionated SwiftUI + MVI.
- If the user wants a Swift Package (library) rather than an iOS app — use `swift package init` directly.
