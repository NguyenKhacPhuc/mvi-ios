---
name: xcode-pbxproj-edit
description: "Programmatically add Swift files, groups, build settings, and signing config to a non-synchronized Xcode project file. Use when adding new source files to an iOS/macOS Xcode project where files won't appear automatically (no PBXFileSystemSynchronizedRootGroup), when fixing 'No such file' build errors after creating sources from the CLI, or when changing bundle ID / display name / signing / orientations / app icon settings."
---

# Editing project.pbxproj Without Xcode

The `.pbxproj` is a structured plist. Manual edits are dangerous if the file uses synchronized folders, but routine when it doesn't. Always check first.

## Step 1 — Detect synchronization mode

```bash
grep -c "PBXFileSystemSynchronized" <Project>.xcodeproj/project.pbxproj
```

- **Non-zero** → folders sync automatically. New files on disk appear in Xcode without pbxproj edits. **Do not hand-edit; just write the files.**
- **Zero** → manual wiring required. Continue.

## Step 2 — The four sections to update for each new Swift file

To add `Foo.swift` (in some group) to a target, add entries in:

1. **`PBXBuildFile`** — links a file ref into a build phase
   ```
   <BUILD_ID> /* Foo.swift in Sources */ = {isa = PBXBuildFile; fileRef = <FILE_ID> /* Foo.swift */; };
   ```

2. **`PBXFileReference`** — declares the file on disk
   ```
   <FILE_ID> /* Foo.swift */ = {isa = PBXFileReference; lastKnownFileType = sourcecode.swift; path = Foo.swift; sourceTree = "<group>"; };
   ```

3. **`PBXGroup`** (parent group's `children` list) — adds to the navigator tree
   ```
   <FILE_ID> /* Foo.swift */,
   ```

4. **`PBXSourcesBuildPhase`** of the target (e.g. `96DD790F341B1338023689D1` for the app target) — adds to the compile list
   ```
   <BUILD_ID> /* Foo.swift in Sources */,
   ```

Resource files (xib, xcassets, plist) go into `PBXResourcesBuildPhase` instead, with `PBXBuildFile` referencing the resource.

## Step 3 — Stable, unique IDs

Each ID is 24 hex chars, uppercase. Never reuse one. Prefix scheme that's worked well:

```
A0010000000000000000A0XX  - PBXFileReference  (file XX)
A0020000000000000000B0XX  - PBXBuildFile      (build XX)
A0030000000000000000C0XX  - PBXGroup          (group XX)
```

Increment the trailing two hex digits per file. Avoid clashing with existing IDs (grep first if unsure).

## Step 4 — Adding a new feature folder (nested groups)

Each subfolder needs its own `PBXGroup`. For `Features/MyFeature/{Intent,Model,State,View}/`:

```
<MyFeature_GROUP> /* MyFeature */ = { isa = PBXGroup; children = (
    <Intent_GROUP> /* Intent */,
    <Model_GROUP>  /* Model */,
    <State_GROUP>  /* State */,
    <View_GROUP>   /* View */,
); path = MyFeature; sourceTree = "<group>"; };
<Intent_GROUP> /* Intent */ = { isa = PBXGroup; children = (
    <FILE_ID> /* MyFeatureIntent.swift */,
); path = Intent; sourceTree = "<group>"; };
... (same for Model, State, View)
```

Then add `<MyFeature_GROUP>` to the parent `Features` group's `children`.

## Step 5 — Common build setting edits

These all live inside `XCBuildConfiguration` blocks (one Debug + one Release per target). Find them by `grep -n "buildSettings = {"` and the surrounding configuration name.

### Bundle ID
```
PRODUCT_BUNDLE_IDENTIFIER = com.<unique>.<app>;
```

### Display name (App Store / home screen)
```
INFOPLIST_KEY_CFBundleDisplayName = "Your Display Name";
```

### Generated Info.plist (Xcode 13+)
```
GENERATE_INFOPLIST_FILE = YES;
INFOPLIST_KEY_UIApplicationSceneManifest_Generation = YES;
INFOPLIST_KEY_UILaunchScreen_Generation = YES;
INFOPLIST_KEY_UISupportedInterfaceOrientations =
    "UIInterfaceOrientationPortrait UIInterfaceOrientationPortraitUpsideDown UIInterfaceOrientationLandscapeLeft UIInterfaceOrientationLandscapeRight";
```

For universal apps, all four orientations are required for App Store validation (iPad multitasking).

### App icon (asset catalog)
```
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon;
INFOPLIST_KEY_CFBundleIconName = AppIcon;
```

### Code signing (split simulator vs device)

Signing for simulator is unnecessary friction. Scope no-sign to simulator only:
```
"CODE_SIGNING_ALLOWED[sdk=iphonesimulator*]" = NO;
"CODE_SIGNING_REQUIRED[sdk=iphonesimulator*]" = NO;
"CODE_SIGN_IDENTITY[sdk=iphonesimulator*]" = "";
"CODE_SIGN_IDENTITY[sdk=iphoneos*]" = "Apple Development";
CODE_SIGN_STYLE = Automatic;
DEVELOPMENT_TEAM = <TEAM_ID>;
```

For device/archive builds the team-scoped settings take effect; simulator skips signing entirely.

## Step 6 — Asset catalogs are auto-included

Don't add individual `.imageset/` or `.appiconset/` folders to pbxproj. The `Assets.xcassets` reference (a single `PBXFileReference` with `lastKnownFileType = folder.assetcatalog`) covers everything inside.

## Step 7 — Verify

```bash
plutil -lint <Project>.xcodeproj/project.pbxproj
```

If syntactically valid, open in Xcode. If a hand-edited `.pbxproj` produces "Project corrupted" errors, the most common cause is dangling references — a `PBXBuildFile` whose `fileRef` ID doesn't exist, or a `children` list referencing a missing group.

## Bogus group cleanup

If you see `PBXGroup` entries with literal-brace names like `{App,Core` — those came from a `mkdir -p '{App,Core/...}'` where quotes prevented brace expansion. Their `children` are usually empty. Safe to remove the orphan groups AND remove their references from any parent `children` list.
