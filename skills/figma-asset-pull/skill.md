---
name: figma-asset-pull
description: "Pull images and icons from a Figma file into a project via the Figma REST API. Fetches PNG raster covers at 1x/2x/3x for iOS asset catalogs and SVG icons for template/tintable use. Use when the user shares a Figma file/URL and asks to pull, fetch, export, or download assets, covers, or icons. Also when implementing a Figma design and the visual assets aren't yet in the codebase."
---

# Figma Asset Pull

Fetches design assets directly from a Figma file via the REST API and writes them into the project (typically `Assets.xcassets/<Name>.imageset/` for iOS, or `assets/` for web/Flutter/etc.). Bypasses Figma MCP — the MCP `get_screenshot` only renders inline visuals, you can't extract bytes from it.

## Inputs you must collect first

1. **Figma file key** — from the URL `https://www.figma.com/design/<FILE_KEY>/...`
2. **Node IDs** — from the URL fragment `?node-id=12-1184` → `12:1184`. Either user provides them or extract from `mcp__Figma__get_design_context` / `get_metadata` on a parent frame.
3. **Personal access token** — `figd_...`. Check env first (`FIGMA_TOKEN`, `FIGMA_ACCESS_TOKEN`), `~/.figma_token`, keychain. If missing, ask the user to paste one from https://www.figma.com/settings → Personal access tokens. **After use, remind them to revoke if it appeared in the transcript.**

## Resolve URLs (one round-trip per scale/format)

```bash
curl -sS -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/images/$FILE_KEY?ids=12:1184,12:1231&format=png&scale=2"
# → {"err":null,"images":{"12:1184":"https://figma-alpha-api.s3...","12:1231":"..."}}
```

The response gives short-lived S3 URLs. Download each with a second `curl`.

`format` options: `png`, `jpg`, `svg`, `pdf`. `scale` is a multiplier (1, 2, 3, 4) — applies to raster only, ignored for SVG.

## Raster covers (iOS imageset)

For each cover, create `Assets.xcassets/<Name>.imageset/` with:

```
<Name>.png        (1x)
<Name>@2x.png     (2x)
<Name>@3x.png     (3x)
Contents.json
```

`Contents.json`:
```json
{
  "images" : [
    { "idiom" : "universal", "filename" : "<Name>.png",    "scale" : "1x" },
    { "idiom" : "universal", "filename" : "<Name>@2x.png", "scale" : "2x" },
    { "idiom" : "universal", "filename" : "<Name>@3x.png", "scale" : "3x" }
  ],
  "info" : { "author" : "xcode", "version" : 1 }
}
```

Loop scales 1/2/3, call the API once per scale, parse JSON with `python3 -c`, download each URL.

## SVG icons (iOS, tintable)

Save as a single `.svg` file in `<Name>.imageset/<Name>.svg` with:

```json
{
  "images" : [ { "filename" : "<Name>.svg", "idiom" : "universal" } ],
  "info" : { "author" : "xcode", "version" : 1 },
  "properties" : {
    "preserves-vector-representation" : true,
    "template-rendering-intent" : "template"
  }
}
```

The `template-rendering-intent: template` makes the icon tint via `.foregroundStyle()` like an SF Symbol. Use it from SwiftUI as `Image("Name").resizable().scaledToFit().frame(width: s, height: s).foregroundStyle(...)` — wrap that in a small `AppIcon(name:size:)` helper.

### Strip Figma chip backgrounds

Figma icon components often include a faint rounded-rect background (the "chip" wrapping the glyph). Under template rendering this background tints together with the glyph and looks wrong. Strip them with regex:

```python
import re
s = re.sub(r'<rect[^>]*fill-opacity="0\.0[0-9]+"[^>]*/>\s*', '', s)
s = re.sub(r'<path[^>]*fill-opacity="0\.0[0-9]+"[^>]*/>\s*', '', s)
s = re.sub(r'<g[^>]*>\s*</g>\s*', '', s)
```

Only paths/rects with very low fill-opacity (`0.0X`) get removed — actual content is preserved.

## bash array gotcha

zsh on macOS doesn't accept `declare -A`. If you need an associative map, wrap the script in `bash <<'BASH' ... BASH` to force bash, or use parallel indexed arrays:

```bash
NODES=("12:1184" "12:1231")
NAMES=("CherryBlossomCover" "HenUocDuoiMuaCover")
for i in "${!NODES[@]}"; do ...; done
```

## Things to verify after writing

- Asset catalog folders don't need `.pbxproj` entries — Xcode picks up nested `.imageset/` automatically.
- 1024×1024 marketing icon must have NO alpha channel. If you also export an app icon, round-trip through JPEG to flatten: `sips -s format jpeg in.png --out tmp.jpg && sips -s format png tmp.jpg --out flat.png`.
- For SVG icons in iOS, `scaledToFit()` with a `frame(width:height:)` is required — bare `Image(name)` won't size correctly.

## Token hygiene

Once an access token has appeared in a transcript or shell history, treat it as compromised. Tell the user to revoke at Figma → Settings → Personal access tokens, then issue a fresh one if more pulls are needed.
