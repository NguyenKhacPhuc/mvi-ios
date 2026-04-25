# Claude Code Skills — iOS / Figma / SwiftUI

A small collection of [Claude Code](https://claude.com/claude-code) skills harvested from real iOS / SwiftUI build sessions. Each skill is a single self-contained `skill.md` with YAML frontmatter — drop the folder under `~/.claude/skills/` and Claude will auto-trigger it on the matching keywords.

## Skills

| Skill | Purpose |
|---|---|
| [`swiftui-mvi-bootstrap`](skills/swiftui-mvi-bootstrap/skill.md) | Bootstrap a brand-new SwiftUI iOS app from zero with MVI pre-wired. Gathers app name / bundle ID / team / min iOS / output path, then generates `project.yml` + `Core/MVI` + `Core/DesignSystem` + `Core/Domain` + `Core/Data` + `App/AppContainer` + a starter Home feature + reducer test + `CLAUDE.md`. Uses xcodegen so the project is regenerable. |
| [`domain-data-layer`](skills/domain-data-layer/skill.md) | Add a new repository to an existing MVI app: Domain protocol + entity, Data mock (and optional URLSession/Firebase/etc. impl), `AppContainer` wiring, EffectHandler refactor from `enum` to `struct`, plus tests. Also handles introducing the layers into a project that doesn't have them yet. |
| [`figma-asset-pull`](skills/figma-asset-pull/skill.md) | Pull cover PNGs (1x/2x/3x) and tintable SVG icons from a Figma file into `Assets.xcassets` via the Figma REST API. Includes the chip-background regex for clean SVGs. |
| [`figma-mvi-implementation`](skills/figma-mvi-implementation/skill.md) | End-to-end pipeline: Figma MCP context → token mapping → MVI scaffold → placeholder layout → asset pull → typecheck. Composes the three skills below. |
| [`swiftui-mvi-feature`](skills/swiftui-mvi-feature/skill.md) | Scaffold a SwiftUI MVI feature folder (Intent / Model / State / View) with pure reducers, effect handlers, container views, typed `[Route]` navigation, and reducer tests. |
| [`xcode-pbxproj-edit`](skills/xcode-pbxproj-edit/skill.md) | Programmatically add Swift files, nested groups, build settings, and scoped signing config to a non-synchronized `.pbxproj`. The four sections, stable-ID scheme, common build-setting recipes. |
| [`ios-appicon-from-art`](skills/ios-appicon-from-art/skill.md) | Generate a complete `AppIcon.appiconset` with all 13 unique PNG sizes from a single source via `sips`, including alpha-stripping for the 1024 marketing icon. |
| [`ios-archive-validation`](skills/ios-archive-validation/skill.md) | Recipe book for the 7 most common archive / App Store Connect submission errors with their exact build-setting fixes. Includes a pre-archive checklist. |

## Suggested order for a new project

1. `swiftui-mvi-bootstrap` — create the project skeleton from zero
2. `figma-asset-pull` — drop in real artwork once Figma is ready
3. `swiftui-mvi-feature` — add additional screens beyond the Home starter
4. `ios-appicon-from-art` — generate the AppIcon set when you have a 1024 source
5. `ios-archive-validation` — only when something fails on the way to the App Store

## How to install

```bash
git clone https://github.com/<your-user>/claude-skills.git
mkdir -p ~/.claude/skills
cp -r claude-skills/skills/* ~/.claude/skills/
```

…or symlink each one:

```bash
for s in claude-skills/skills/*/; do
  ln -s "$(pwd)/$s" ~/.claude/skills/$(basename "$s")
done
```

Skills are loaded on next Claude Code startup. Confirm with the `find-skills` skill or by checking the system-reminder skill listing.

## How a skill is structured

```
skills/<skill-name>/
└── skill.md          # YAML frontmatter + markdown body
```

Frontmatter contract:

```yaml
---
name: <kebab-case-name>          # must match folder name
description: |
  One paragraph. Lead with WHAT it does, then WHEN to use it.
  Claude reads this to decide auto-triggering — be specific about
  trigger phrases ("Use when the user says X / asks for Y").
---
```

The body is freeform markdown the model reads when the skill activates. Treat it like a focused playbook, not a textbook chapter:

- **Concrete commands** beat prose. Show the `curl`, the `sips`, the regex.
- **Document the gotchas you actually hit** (zsh vs bash, iOS-version-gated APIs, alpha-rejected icons).
- **Cross-reference siblings** instead of duplicating. The end-to-end skill links to the three primitives it composes.

See [`GUIDELINES.md`](GUIDELINES.md) for the editorial conventions used across this set.

## License

MIT. Use, fork, copy into your own collection.
