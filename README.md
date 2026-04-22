# figma-to-compose

> **Design → System → Code.** A Claude Code plugin that turns Figma frames into production-ready Jetpack Compose code — constrained by *your* design system, not the designer's hex codes.

Most "Figma-to-code" tools treat Figma as the source of truth and emit hardcoded values. This plugin does the opposite: **your design system is the contract**, Figma is just input. If a color or component isn't in your mapping, the pipeline fails loudly instead of guessing.

---

## Why this exists

- **No hardcoded values.** Every color, spacing, and typography value must resolve to a token. Missing token → pipeline fails, not a silent `Color(0xFFD9D9D9)` sneaking into prod.
- **Component reuse is enforced.** If `ProductCard` exists in your codebase, you get `ProductCard(...)` — never a generated `ProductCardV2`.
- **Fails safe, never half-right.** A root-level component with no mapping and no Material3 equivalent *blocks generation entirely* — the plugin refuses to write broken code. Fix the mapping, then re-run.
- **Incremental-safe.** Generated regions are wrapped in markers. Re-running updates only what Figma changed. Your manual edits outside the markers are never touched.
- **Self-improving.** Mappings gain confidence when used consistently, decay when stale, and escalate fallbacks automatically.

---

## Requirements

- Claude Code ≥ 1.5.0
- Figma MCP access (via `claude.ai Figma` connector)
- An Android project with Jetpack Compose

---

## Install

Clone into your Claude Code plugins directory:

```bash
git clone https://github.com/nguyenbahung94/FigmaToCompose.git \
  ~/.claude/plugins/figma-to-compose
```

Or clone anywhere and register the path in your Claude Code plugin settings. Restart Claude Code so the `/figma-to-compose` command and skills are discovered.

---

## Quick Start

```bash
# One-time project setup
/figma-to-compose init           # scan design system -> config + token mapping
/figma-to-compose bootstrap      # scan @Composable functions -> component mapping
/figma-to-compose validate       # sanity check before first generation

# Exploring a frame (read-only)
/figma-to-compose inspect https://figma.com/design/...
/figma-to-compose https://figma.com/design/... --layout-map

# Generating code
/figma-to-compose run https://figma.com/design/...        # smart, confidence-gated
/figma-to-compose generate https://figma.com/design/...   # always writes
```

---

## Commands

### Setup
| Command | Description |
|---|---|
| `/figma-to-compose init` | Scan design system, create `config.json` and `token-mapping.json` |
| `/figma-to-compose bootstrap` | Scan `@Composable` functions, seed `component-mapping.json` |
| `/figma-to-compose validate` | Verify config and mappings are well-formed |

### Debug
| Command | Description |
|---|---|
| `/figma-to-compose inspect <url>` | Full IR tree + token resolution + mapping decisions (no codegen) |
| `/figma-to-compose <url> --layout-map` | Spatial ASCII tree of the Figma frame |
| `/figma-to-compose review-mappings` | Walk pending component mappings, confirm or reject |
| `/figma-to-compose suggest-mapping` | List unmapped Figma components with suggested matches |
| `/figma-to-compose cleanup-orphans` | List generated blocks whose Figma node no longer exists |

### Execution
| Command | Description |
|---|---|
| `/figma-to-compose run <url>` | Smart mode — auto dry-run if confidence < 0.70 |
| `/figma-to-compose generate <url>` | Deterministic — always writes files |
| `/figma-to-compose ci <url>` | CI mode — fail-fast, machine-readable JSON |

**Flags** (`run` / `generate` only): `--dry-run`, `--allow-write-low-confidence`, `--verbose`.

---

## What you get

Generated code is wrapped in stable markers so re-runs are safe:

```kotlin
// <figma-generated id="ProductCard.header" hash="a3f9c2">
@Composable
private fun ProductCardHeader(title: String) {
    Text(
        text = title,
        style = AppTypography.HeadingBold,
        color = colorResource(R.color.on_surface),
        modifier = Modifier.padding(16.dp),
    )
}
// </figma-generated>
```

- Colors → `colorResource(R.color.*)` from your `colors.xml`
- Typography → `<YourTypographyClass>.*` from your config
- Spacing → `<N>.dp` from your spacing tokens
- Components → your existing `@Composable`, resolved via `component-mapping.json`

Never `Color(0xFF...)`. Never raw `fontSize = 16.sp`. Never a rogue new `ProductCardV2`.

---

## Project Config

After `init`, your project gets:

```
.claude/figma-to-compose/
  config.json                  # platform, typography class, colors path
  mappings/
    token-mapping.json         # color, spacing, typography tokens
    component-mapping.json     # Figma component -> Compose component
```

**config.json**
```json
{
  "platform": "android",
  "schema_version": "1.0",
  "mapping_version": "1.0",
  "plugin_version": "1.x",
  "autonomy_phase": 1,
  "colors_source": "src/main/res/values/colors.xml",
  "typography_class": "com.example.AppTypography",
  "compose_package": "com.example.presentation"
}
```

---

## Pipeline

```
Figma URL
  |
  |  Stage 1  Fetch + Normalize       strip hidden layers, flatten wrappers
  |  Stage 2  Token Extract           hex -> token-mapping, fallback to colors.xml
  |  Stage 3  Screen vs Component     semantic rules first, size last resort
  |  Stage 4  Component Lookup        component-mapping -> FallbackLevel
  |  Stage 5  Layout IR Build         platform-agnostic intent tree
  |  Stage 6  Codegen                 IR -> Jetpack Compose (or SwiftUI)
  |  Stage 7  Confidence Gate         auto dry-run if avg < 0.70
  |  Stage 8  Incremental Write       marker-based, safe on re-run
  v  Stage 9  Mapping Lifecycle       decay stale, boost consistent
```

Stages 1–5 are platform-agnostic. Only Stage 6 knows about Compose (or SwiftUI). Adding a new platform = writing one new skill file.

---

## Self-Improving System

- **Confidence decay** — unused mappings fade; <0.5 flagged for re-review, <0.3 removed
- **Negative learning** — rejected candidates stay rejected, never re-suggested
- **Consistency boost** — mappings used repeatedly in the same context gain confidence
- **Fallback escalation** — Minor fallback ≥ 3 uses → auto-suggest mapping; ≥ 5 → force fix
- **Autonomy phases** — system progresses from manual → assisted → autonomous as confidence builds

---

## Adding a New Platform

The IR (Stages 1–5) is platform-agnostic. A new target (iOS, Flutter, Web) = one new skill file implementing Stage 6.

See [docs/contributing-platform.md](docs/contributing-platform.md).

---

## License

MIT
