# Adding a New Platform

This guide explains how to add a new code generation target (e.g. iOS/SwiftUI, Flutter, Web) to figma-to-compose.

## How It Works

The core plugin (Stages 1–5) produces a platform-agnostic IR (Intent tree). Platform skills implement Stage 6: IR → platform code.

Adding a platform = writing one new skill file. Zero changes to core.

## Steps

### 1. Create the skill file

```
skills/figma-to-compose-<platform>/
  SKILL.md
```

### 2. SKILL.md must implement

**Token resolution** — translate `TokenRef` keys to platform code:

| TokenRef type | Expected output |
|---|---|
| `color.*` | Platform color reference (e.g. `Color("primary")` for iOS) |
| `spacing.*` | Platform spacing value (e.g. `16` CGFloat for iOS) |
| `typography.*` | Platform typography style (e.g. `Font.system(size: 16)`) |

**IR → platform code** — translate each `LayoutNode` type:

| LayoutNode | Required output |
|---|---|
| `Layout(direction=Vertical)` | Vertical stack (e.g. `VStack` for iOS) |
| `Layout(direction=Horizontal)` | Horizontal stack (e.g. `HStack` for iOS) |
| `Layout(direction=Stack)` | Absolute positioning (e.g. `ZStack` for iOS) |
| `Component(mappingKey)` | Lookup in component-mapping.json → emit mapped view |
| `Text` | Platform text widget with token-resolved style/color |
| `Fallback(Minor)` | Placeholder with TODO comment |
| `Fallback(Major)` | BLOCK — do not generate |

**mainAxis → platform arrangement:**

| MainAxisArrangement | Android | iOS | Web |
|---|---|---|---|
| Start | `Arrangement.Top` / `Start` | default | `justify-content: flex-start` |
| Center | `Arrangement.Center` | `.center` | `justify-content: center` |
| SpaceBetween | `Arrangement.SpaceBetween` | `Spacer()` between items | `justify-content: space-between` |

**crossAxis → platform alignment:**

| CrossAxisAlignment | Android | iOS | Web |
|---|---|---|---|
| Start | `Alignment.Start` | `.leading` | `align-items: flex-start` |
| Center | `Alignment.CenterHorizontally` | `.center` | `align-items: center` |
| Stretch | `Alignment.Start` + `fillMaxWidth` | `.trailing` fill | `align-items: stretch` |

**CommonProps → platform modifiers:**

| CommonProps field | Platform equivalent |
|---|---|
| `padding` | Padding modifier / `.padding()` |
| `background` | Background fill / `.background()` |
| `cornerRadius` | Clip shape / `.clipShape(RoundedRectangle(...))` |
| `action` | Tap gesture / `.onTapGesture {}` |

### 3. Register in plugin.json

```json
"skills": [
  ...
  {
    "name": "figma-to-compose-<platform>",
    "path": "skills/figma-to-compose-<platform>/SKILL.md",
    "type": "platform",
    "depends_on": ["figma-to-compose"]
  }
],
"platforms": {
  "<platform>": { "generator": "figma-to-compose-<platform>" }
}
```

### 4. Add config support

Update `schemas/config.schema.json` `platform` enum to include the new platform name.

### 5. Create token-mapping template

Copy `templates/android/token-mapping.template.json` to `templates/<platform>/token-mapping.template.json`.
Update `compose` values to use platform-specific syntax.

## IR Reference

The IR is defined in spec `docs/specs/2026-04-17-figma-to-compose-public-plugin-design.md` §6.

Key rule: **IR describes intent, not implementation.** `direction=Vertical` means "stack children along the vertical axis" — not "use Column". Platform skill decides the render primitive.

## Questions

Open an issue at the GitHub repo.
