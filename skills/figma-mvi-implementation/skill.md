---
name: figma-mvi-implementation
description: "End-to-end workflow for implementing a Figma design as an MVI SwiftUI feature: pull design context via Figma MCP, map design tokens to existing design system, scaffold the MVI feature, render the layout, then pull asset bytes via REST. Use when the user shares a Figma URL/node-id and asks to implement, build, or recreate the design as a SwiftUI feature."
---

# Figma → MVI SwiftUI Implementation Pipeline

Composes 3 sub-skills (`figma-asset-pull`, `swiftui-mvi-feature`, `xcode-pbxproj-edit`) into a repeatable order of operations. Use this when starting a new feature from a Figma URL.

## Step 1 — Identify file key + node ID

URL format: `https://www.figma.com/design/<FILE_KEY>/...?node-id=12-1163`. Convert hyphen to colon → `12:1163`.

## Step 2 — Pull design context (Figma MCP)

```
mcp__Figma__get_design_context(nodeId: "12:1163", forceCode: true)
mcp__Figma__get_screenshot(nodeId: "12:1163")    // visual reference
mcp__Figma__get_variable_defs(nodeId: "12:1163") // design tokens
```

`get_design_context` may prompt about Code Connect mappings — if the user has no paid Figma seat, skipping is fine.

If the response says "the user has selected a section node, so you have received a sparse metadata response" — still useful as an outline. Drill into specific child frames with another `get_design_context` call only when you need the deeper layout details.

## Step 3 — Map design tokens to existing design system

The codebase already has `Core/DesignSystem/{Colors,Typography,Spacing}.swift`. From `get_variable_defs`:
- `"Primary Color":"#FFB0CA"` → extend `AppColor.primary`
- Font references → extend `AppFont`

**Don't introduce raw hex / magic-numbers in feature code.** If Figma has a color you don't have a token for yet, add it to `Colors.swift` first.

## Step 4 — Decide feature folder

One folder per feature: `Features/<FeatureName>/{Intent,Model,State,View}/`. See `swiftui-mvi-feature` skill for the per-file scaffolding.

For multi-screen flows (welcome → player, list → detail, etc.), each screen is usually its own MVI feature. Wire them via a parent feature's `path: [Route]` enum and `NavigationStack(path:)` + `.navigationDestination(for:)`.

## Step 5 — Implement the layout with placeholder visuals first

Render the structure using gradient blocks + emoji + SF Symbols as fallbacks:
- `StoryCover { gradient + emoji }` over `LinearGradient(colors: palette)`
- `Image(systemName: "gearshape.fill")` etc. for icons
- Real text from the design (Vietnamese / non-Latin characters preserve fine in `.swift` UTF-8 source)

Watch for design strings with emoji — Vietnamese on macOS+CLT may render some emoji as tofu in iOS simulators with sparse fonts. Prefer SF Symbols over emoji for category chips and badges. (The `figma-asset-pull` skill explains the SVG icon path if you need exact Figma glyphs.)

## Step 6 — Wire navigation BEFORE pulling assets

Get tap-throughs working with placeholders. This validates the route enum, container views, and back-stack semantics before you sink time into pixel polish.

## Step 7 — Pull real assets via REST

Once layout is solid, run `figma-asset-pull` to fetch:
- Cover artwork as multi-scale PNG imagesets
- Icons as template-rendered SVG imagesets

Wire each into the model layer:
```swift
struct Story {
    ...
    let coverAssetName: String?   // "CherryBlossomCover" → falls back to gradient when nil
}
```

In the cover view:
```swift
if let name = assetName, UIImage(named: name) != nil {
    Image(name).resizable().scaledToFill()
} else {
    LinearGradient(...) + emoji  // designer-friendly fallback
}
```

## Step 8 — pbxproj wiring + compile check

Add every new `.swift` file to the project per the `xcode-pbxproj-edit` skill (asset catalogs and nested `.imageset/` folders are auto-included).

Compile-check without Xcode using `swiftc -typecheck` against the macOS SDK. The non-View files (state/intent/reducer/effects) are platform-agnostic and will type-check cleanly. View files reference iOS-only APIs — filter expected errors:

```bash
... 2>&1 | grep "error:" \
  | grep -vE "is unavailable in macOS|external macro|toolbar.*for:|topBarLeading|topBarTrailing|navigationBarBackButtonHidden|toolbarBackground|navigationBarHidden|refreshable|UIKit"
```

Empty output = clean.

## Step 9 — Reducer tests

Pure reducers should always have unit tests. One test per intent variant minimum. See `swiftui-mvi-feature` skill.

## Pitfalls observed in real sessions

- `navigationDestination(item:)` is iOS 17+. Project targets iOS 16 → use path-based `[Route]` instead.
- `NavigationPath` (untyped) is more flexible but less type-safe than `[Route]`. For 2–3 destination types, prefer the typed array of an enum.
- iOS 17 `onChange` has 2-arg closure; iOS 16 has 1-arg. The same code can be ambiguous on macOS SDK type-checks but works on both iOS targets.
- Vietnamese text + emoji: prefer SF Symbols for chip glyphs to avoid font fallback tofu on minimal simulator runtimes.
- Brace expansion gone wrong: `mkdir -p '{App,Core/{X,Y}}'` with single quotes creates a literal-named folder. If you see folders with `{` or `,` in their names, brace expansion was suppressed by quoting.
