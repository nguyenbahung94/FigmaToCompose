---
name: figma-to-compose-android
description: Use when generating Jetpack Compose code from figma-to-compose IR. Reads LayoutNode IR produced by the core skill and emits production-ready Kotlin/Compose code.
depends_on: figma-to-compose
platform: android
---

# figma-to-compose-android

Android/Jetpack Compose code generator — Stage 6 of the figma-to-compose pipeline.

## Responsibility

Consume `LayoutNode` IR from the core skill. Emit production-ready Kotlin.

**This skill does NOT fetch from Figma. It does NOT build IR. It only translates IR → Kotlin.**

## Token Resolution (Android)

| IR token type | Android output |
|---|---|
| `TokenRef` (spacing) | `<dp_value>.dp` from `token-mapping.json` |
| `TokenRef` (color) | `colorResource(R.color.<resource>)` |
| `TokenRef` (typography) | `<TypographyClass>.<style>` from `config.json` `typography_class` |

## IR → Compose Mapping

| LayoutNode | Compose |
|---|---|
| `Vertical` | `Column(verticalArrangement, horizontalAlignment)` |
| `Horizontal` | `Row(horizontalArrangement, verticalAlignment)` |
| `Stack` | `Box` with `Modifier.align()` per child |
| `Component` | mapped `@Composable` from `component-mapping.json` |
| `Text` | `Text(text, style, color, maxLines, overflow, textAlign)` |
| `Fallback(Minor)` | `Box { // TODO: [Minor] <description> }` |
| `Fallback(Major)` | BLOCK — do not generate |

## CommonProps → Modifier

Emit in canonical order:
1. size (`fillMaxWidth`, `width`, `height`, `size`)
2. padding
3. background / border / clip
4. clickable / semantics
5. testTag (always last)

## Output Style

- `@Suppress("FunctionNaming")` on every PascalCase `@Composable`
- `@Suppress("UnusedPrivateMember", "FunctionNaming")` on every `@Preview`
- `private const val` for numeric literals (except 0, 1, -1)
- Trailing commas on all multi-line call sites
- Max line length: 120 characters

## Config Required

Reads from project `.claude/figma-to-compose/config.json`:
- `typography_class` — fully qualified class for typography tokens
- `compose_package` — base package for generated files
- `colors_source` — path to colors.xml for token lookup
