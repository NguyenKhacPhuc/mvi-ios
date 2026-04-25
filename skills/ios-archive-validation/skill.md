---
name: ios-archive-validation
description: "Resolve common iOS archive / App Store Connect validation errors: missing icons, missing CFBundleIconName, wrong orientations for iPad multitasking, no team in archive, bundle ID conflict, app name conflict, and code signing scoped wrongly between simulator and device. Use when the user reports an Xcode archive validation failure or an App Store Connect submission error."
---

# Fixing iOS Archive & Submission Errors

The five errors below cover ~95% of first-time submission failures. Each has a specific build-setting or asset-catalog fix.

## Error: "No Team Found in Archive"

The archive embedded an empty team. Two likely causes:

1. **Signing globally disabled** (e.g. `CODE_SIGNING_ALLOWED = NO`) from earlier simulator-friendly setup. Scope it to simulator only:
   ```
   "CODE_SIGNING_ALLOWED[sdk=iphonesimulator*]" = NO;
   "CODE_SIGNING_REQUIRED[sdk=iphonesimulator*]" = NO;
   "CODE_SIGN_IDENTITY[sdk=iphonesimulator*]" = "";
   "CODE_SIGN_IDENTITY[sdk=iphoneos*]" = "Apple Development";
   CODE_SIGN_STYLE = Automatic;
   DEVELOPMENT_TEAM = <YOUR_TEAM_ID>;
   ```
2. **Team ID is empty string** in the app target's Debug/Release configs. Set `DEVELOPMENT_TEAM = <ID>` in both. Find the ID at https://developer.apple.com/account → Membership.

## Error: "App identifier ... cannot be registered ... not available"

Bundle ID is taken. Switch to a unique reverse-DNS that the team owns:

```
PRODUCT_BUNDLE_IDENTIFIER = com.<unique>.<app>;
```

Update tests too:
```
PRODUCT_BUNDLE_IDENTIFIER = com.<unique>.<app>.tests;
PRODUCT_BUNDLE_IDENTIFIER = com.<unique>.<app>.uitests;
```

Generic prefixes like `com.app.X`, `com.<commonword>.X` are nearly always taken. Suggest the user's reversed domain or email handle.

## Error: "App Name you entered is already being used"

This is the **App Store Connect listing** name, not the bundle ID. The display name is set at App Record creation time in App Store Connect. Suggest variants:
- Append a qualifier ("Pro", "AI", "App")
- Reorder words
- Rebrand entirely

You can also set the binary's display name (home-screen label) independently:
```
INFOPLIST_KEY_CFBundleDisplayName = "Your Visible Name";
```
But App Store Connect's name is set in the web UI when creating the app record.

## Error: "Missing required icon file ... 120x120 ... 152x152"

The bundle has no `AppIcon.appiconset` with required PNGs. See the `ios-appicon-from-art` skill for the full sips-based pipeline. Required sizes summary:
- iPhone home: 120, 180
- iPad home: 152, 167
- App Store marketing: 1024 (no alpha)
- Spotlight, settings, notification: 20/29/40 at @1/2/3x

## Error: "Missing Info.plist value ... CFBundleIconName"

Apps built with iOS 11+ SDK MUST declare which asset-catalog icon set is the app icon, even when using `GENERATE_INFOPLIST_FILE = YES`. Two settings, both required:

```
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon;
INFOPLIST_KEY_CFBundleIconName = AppIcon;
```

`AppIcon` is the catalog set name, not a file path.

## Error: "UISupportedInterfaceOrientations ... need to include all of ..."

iPad apps must support all four orientations for multitasking. Include the missing `PortraitUpsideDown`:

```
INFOPLIST_KEY_UISupportedInterfaceOrientations =
    "UIInterfaceOrientationPortrait UIInterfaceOrientationPortraitUpsideDown UIInterfaceOrientationLandscapeLeft UIInterfaceOrientationLandscapeRight";
```

If the app is iPhone-only, set `TARGETED_DEVICE_FAMILY = "1"` instead — the iPad rule no longer applies. (Universal is `"1,2"`.)

## Error: "Build input file cannot be found: 'Info.plist'"

Project expects an `Info.plist` file but you've deleted it. Either restore the file OR switch to Xcode-generated:

```
GENERATE_INFOPLIST_FILE = YES;
```

…and remove `INFOPLIST_FILE = ...` from build settings. Then add the per-key INFOPLIST_KEY_* values you actually need.

## Error: "asset catalog ... no matching ... AppIcon"

You've enabled `ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon` but `Assets.xcassets/AppIcon.appiconset/` doesn't exist. Either remove that build setting OR generate the appiconset (see the `ios-appicon-from-art` skill).

## Pre-flight checklist (run before every archive)

1. Bundle ID is unique to this team
2. Display name set (or accepting Xcode's target-name fallback)
3. AppIcon.appiconset exists with all sizes (no alpha on the 1024)
4. `CFBundleIconName` and `ASSETCATALOG_COMPILER_APPICON_NAME` both = `AppIcon`
5. All four orientations listed (or `TARGETED_DEVICE_FAMILY = "1"`)
6. `DEVELOPMENT_TEAM` set in app target Debug + Release
7. `CODE_SIGNING_ALLOWED` is YES (or scoped to simulator only)
8. `Info.plist` exists OR `GENERATE_INFOPLIST_FILE = YES`
