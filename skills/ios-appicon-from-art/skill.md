---
name: ios-appicon-from-art
description: "Generate a complete iOS AppIcon.appiconset (all required sizes + Contents.json) from a single source PNG using sips. Strips alpha for App Store marketing icon. Use when archive validation fails with 'Missing required icon file' / 'CFBundleIconName missing', when adding an app icon for the first time, or when replacing the placeholder icon with real artwork."
---

# Generate AppIcon.appiconset from a Single Source

App Store validation requires explicit PNGs in the bundle for every iPhone + iPad icon size. Xcode does NOT generate them from a single 1024 source — you must produce all 13 PNGs yourself. `sips` (built into macOS) does it cleanly.

## Required sizes

| Size | Idiom × Scale | Purpose |
|---|---|---|
| 20, 40, 60      | iPhone 20pt @1/2/3x | Notification |
| 29, 58, 87      | iPhone 29pt @1/2/3x | Settings |
| 40, 80, 120     | iPhone 40pt @1/2/3x | Spotlight |
| 80, 120, 180    | iPhone 60pt @2/3x | Home screen |
| 20, 40          | iPad 20pt @1/2x | Notification |
| 29, 58          | iPad 29pt @1/2x | Settings |
| 40, 80          | iPad 40pt @1/2x | Spotlight |
| 76, 152         | iPad 76pt @1/2x | Home screen |
| 167             | iPad Pro 83.5pt @2x | Home screen |
| 1024            | App Store marketing | NO alpha |

Many sizes appear in multiple slots → only **13 unique PNGs** needed: 20, 29, 40, 58, 60, 76, 80, 87, 120, 152, 167, 180, 1024.

## Step 1 — Prepare a 1024×1024 master (no alpha)

Apple rejects marketing icons with alpha channels. Round-trip through JPEG to flatten:

```bash
SRC=path/to/source-art.png   # any size, ideally ≥ 1024 in both dims
TMP=$(mktemp -d)

# Strip alpha
sips -s format jpeg "$SRC" --out "$TMP/flat.jpg" >/dev/null
sips -s format png  "$TMP/flat.jpg" --out "$TMP/flat.png" >/dev/null

# Scale shorter side to 1024, then center-crop to 1024×1024
W=$(sips -g pixelWidth  "$TMP/flat.png" | awk '/pixelWidth/{print $2}')
H=$(sips -g pixelHeight "$TMP/flat.png" | awk '/pixelHeight/{print $2}')
if [ "$W" -lt "$H" ]; then
  sips --resampleWidth  1024 "$TMP/flat.png" --out "$TMP/sq.png" >/dev/null
else
  sips --resampleHeight 1024 "$TMP/flat.png" --out "$TMP/sq.png" >/dev/null
fi
sips -c 1024 1024 "$TMP/sq.png" --out master-1024.png >/dev/null
```

## Step 2 — Generate all 13 sizes

```bash
ICONSET=Assets.xcassets/AppIcon.appiconset
mkdir -p "$ICONSET"
cp master-1024.png "$ICONSET/AppIcon-1024.png"

for S in 20 29 40 58 60 76 80 87 120 152 167 180; do
  sips -z $S $S "$ICONSET/AppIcon-1024.png" --out "$ICONSET/AppIcon-${S}.png" >/dev/null
done
```

## Step 3 — Write Contents.json

```json
{
  "images" : [
    { "idiom" : "iphone",        "size" : "20x20",    "scale" : "2x", "filename" : "AppIcon-40.png"   },
    { "idiom" : "iphone",        "size" : "20x20",    "scale" : "3x", "filename" : "AppIcon-60.png"   },
    { "idiom" : "iphone",        "size" : "29x29",    "scale" : "2x", "filename" : "AppIcon-58.png"   },
    { "idiom" : "iphone",        "size" : "29x29",    "scale" : "3x", "filename" : "AppIcon-87.png"   },
    { "idiom" : "iphone",        "size" : "40x40",    "scale" : "2x", "filename" : "AppIcon-80.png"   },
    { "idiom" : "iphone",        "size" : "40x40",    "scale" : "3x", "filename" : "AppIcon-120.png"  },
    { "idiom" : "iphone",        "size" : "60x60",    "scale" : "2x", "filename" : "AppIcon-120.png"  },
    { "idiom" : "iphone",        "size" : "60x60",    "scale" : "3x", "filename" : "AppIcon-180.png"  },
    { "idiom" : "ipad",          "size" : "20x20",    "scale" : "1x", "filename" : "AppIcon-20.png"   },
    { "idiom" : "ipad",          "size" : "20x20",    "scale" : "2x", "filename" : "AppIcon-40.png"   },
    { "idiom" : "ipad",          "size" : "29x29",    "scale" : "1x", "filename" : "AppIcon-29.png"   },
    { "idiom" : "ipad",          "size" : "29x29",    "scale" : "2x", "filename" : "AppIcon-58.png"   },
    { "idiom" : "ipad",          "size" : "40x40",    "scale" : "1x", "filename" : "AppIcon-40.png"   },
    { "idiom" : "ipad",          "size" : "40x40",    "scale" : "2x", "filename" : "AppIcon-80.png"   },
    { "idiom" : "ipad",          "size" : "76x76",    "scale" : "1x", "filename" : "AppIcon-76.png"   },
    { "idiom" : "ipad",          "size" : "76x76",    "scale" : "2x", "filename" : "AppIcon-152.png"  },
    { "idiom" : "ipad",          "size" : "83.5x83.5","scale" : "2x", "filename" : "AppIcon-167.png"  },
    { "idiom" : "ios-marketing", "size" : "1024x1024","scale" : "1x", "filename" : "AppIcon-1024.png" }
  ],
  "info" : { "author" : "xcode", "version" : 1 }
}
```

## Step 4 — Build settings

In each app target's Debug + Release configs:

```
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon;
INFOPLIST_KEY_CFBundleIconName = AppIcon;
```

The asset catalog itself is auto-included if `Assets.xcassets` is already in the project; the new `.appiconset/` is picked up automatically.

## Caveat: cropping non-square art

When the source is portrait (e.g., a book cover), center-cropping to square cuts off head or feet. Acceptable for fast validation, ugly for production. Note this and offer the user a path to a real designed icon — single bold mark, no thin lines, no text, full bleed.

## Caveat: alpha rejection

If validation still complains "icon has alpha channel," the JPEG round-trip didn't fully strip. Fallback: use Python with `Pillow` (`im.convert("RGB").save(...)`) or `magick in.png -background white -alpha remove out.png` if ImageMagick is installed.
