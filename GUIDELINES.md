# Skill Authoring Guidelines

Conventions used across the skills in this repo. Keep new skills consistent so Claude triggers them reliably and reads them quickly.

## 1. Scope: one task, one playbook

A skill should answer **one** question that recurs across sessions. If a task naturally splits ("pull assets" vs. "scaffold feature" vs. "fix archive"), make it three skills and a fourth meta-skill that orders them. The end-to-end skill links to the primitives — it does not inline them.

Bad: "iOS Helper Skill" covering pbxproj, signing, icons, screenshots, fastlane.
Good: `xcode-pbxproj-edit` + `ios-appicon-from-art` + `ios-archive-validation`.

## 2. Frontmatter

```yaml
---
name: <kebab-case>
description: <one paragraph>
---
```

The description is the **only** thing Claude sees when choosing whether to fire the skill. Make it count:

- **First sentence: WHAT.** "Pull images and icons from a Figma file via REST API."
- **Second sentence onward: WHEN.** "Use when the user shares a Figma URL and asks to pull/fetch/export/download assets…"
- **List actual trigger phrases** ("archive validation failed", "missing CFBundleIconName") — these are how the model detects relevance.
- **Note skip conditions** if the skill might fire wrongly. ("SKIP when filename like `*-openai.py`")

Length: ~80–250 words. Long enough to be specific, short enough to scan.

## 3. Body structure

```markdown
# <Skill name>

<one-paragraph elevator pitch — what this does and what it does NOT do>

## Inputs / preconditions
## Steps (numbered, with concrete commands)
## Caveats / pitfalls
## Cross-references to sibling skills
```

Adapt as needed. The recurring sections that pay off:

- **Inputs** the model must collect first (file keys, node IDs, tokens, project paths). State where to look (env var? keychain? user prompt?).
- **Caveats** are gold. Document the things you wasted time on: zsh's lack of `declare -A`, alpha-rejection on 1024 icons, `navigationDestination(item:)` being iOS 17 only, brace-expansion-in-quotes creating literal `{App,Core` folders.
- **Cross-references** save tokens. "See `xcode-pbxproj-edit` for the four-section pattern" beats restating it.

## 4. Concrete commands beat prose

Skills are reference material the model rereads at runtime. Show the literal command, not "use a tool to fetch the resource":

```bash
curl -sS -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/images/$FILE_KEY?ids=$IDS&format=png&scale=2"
```

When a command depends on shell quirks (zsh on macOS), say so:

> zsh on macOS doesn't accept `declare -A`. Wrap in `bash <<'BASH' ... BASH` or use parallel indexed arrays.

## 5. Don't reinvent — reference existing tools

If `sips`, `xcrun`, `gh`, `plutil`, `python3 -c`, or `curl` already does the job, use them. Skills are not the place to introduce new dependencies. If a step truly needs Python, prefer one-liners (`python3 -c "import json,sys; ..."`) over a separate `.py` file.

## 6. Validate before claiming done

Skills that produce code should describe how to verify the output:

- `swiftc -typecheck` against macOS SDK to catch Swift errors without Xcode
- `plutil -lint` for `.pbxproj`
- The exact filter regex for ignoring expected platform-mismatch errors

Empty output = clean. State that.

## 7. Token / secret hygiene

If a skill uses a personal access token (Figma, GitHub, etc.):

1. Probe env first (`FIGMA_TOKEN`, `GITHUB_TOKEN`).
2. Fall back to `~/.<service>_token` or keychain.
3. Only ask the user as a last resort.
4. **Remind the user to rotate** if the token appears in a transcript or shell history.

## 8. Review checklist before merging a skill

- [ ] Folder name matches `name:` in frontmatter
- [ ] Description leads with WHAT, lists WHEN with concrete trigger phrases
- [ ] All commands are runnable as-is on macOS / zsh
- [ ] Caveats section names ≥ 1 real gotcha you hit
- [ ] Cross-references link to sibling skills, not paste their content
- [ ] No secrets, no machine-specific paths, no project-specific names baked in (those go in examples, not the canonical commands)
