# figma-to-compose

> **Design → System → Code.** Not a UI generator — a pipeline where your Design System is the source of truth.

A Claude Code plugin that converts Figma designs into production-ready Jetpack Compose Kotlin code, constrained by your project's design tokens and component library.

## Core Principle

Figma = input. Your design system = source of truth. Generated code = output constrained by the system.

If Figma uses a color not in your token mapping → pipeline fails (never emits hardcoded values).
If Figma uses a component not in your component mapping → FallbackLevel determines behavior.

## Requirements

- Claude Code ≥ 1.5.0
- Figma API access (claude.ai Figma MCP)
- Android project with Jetpack Compose

## Install

Clone into your Claude Code plugins directory:

```bash
git clone https://github.com/nguyenbahung94/FigmaToCompose.git \
  ~/.claude/plugins/figma-to-compose
```

Or clone anywhere and add the path to your Claude Code settings. Restart Claude Code after install so the `/figma-to-compose` command and skills are discovered.

## Quick Start

```bash
# 1. First-time setup (run once per project)
/figma-to-compose init

# 2. Seed component mapping from your codebase
/figma-to-compose bootstrap

# 3. Verify everything is configured correctly
/figma-to-compose validate

# 4. Explore a Figma frame before generating
/figma-to-compose inspect https://figma.com/design/...

# 5. Generate code
/figma-to-compose run https://figma.com/design/...
```

## Commands

### Setup
| Command | Description |
|---|---|
| `/figma-to-compose init` | First-time setup: scan design system → generate config + token-mapping |
| `/figma-to-compose bootstrap` | Scan @Composable functions → seed component-mapping (requires init) |
| `/figma-to-compose validate` | Verify config + mappings are valid before use |

### Debug
| Command | Description |
|---|---|
| `/figma-to-compose inspect <url>` | Show IR tree + token resolution + mapping decisions (no codegen) |
| `/figma-to-compose <url> --layout-map` | Spatial ASCII tree of the Figma frame |
| `/figma-to-compose review-mappings` | Review + confirm pending component mappings |
| `/figma-to-compose suggest-mapping` | List unmapped Figma components with suggestions |
| `/figma-to-compose cleanup-orphans` | List generated blocks whose Figma node no longer exists |

### Execution
| Command | Description |
|---|---|
| `/figma-to-compose run <url>` | Smart: respects confidence gate, may auto dry-run |
| `/figma-to-compose generate <url>` | Deterministic: always generates |
| `/figma-to-compose ci <url>` | CI mode: fail-fast, machine-readable JSON output |

**Flags (run/generate):** `--dry-run`, `--allow-write-low-confidence`, `--verbose`

## Project Config

After `init`, your project gets:

```
.claude/figma-to-compose/
  config.json                  ← platform, typography class, colors path
  mappings/
    token-mapping.json         ← color/spacing/typography tokens
    component-mapping.json     ← Figma component → Compose component mappings
```

**config.json example:**
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

## How It Works

```
Figma URL
  → Stage 1: Fetch + Normalize (strip hidden layers, flatten redundant wrappers)
  → Stage 2: Token Extract (hex → token-mapping, fallback: colors.xml grep)
  → Stage 3: Screen vs Component detection
  → Stage 4: Component mapping lookup (component-mapping → FallbackLevel)
  → Stage 5: Layout IR build (Intent tree: direction, mainAxis, crossAxis)
  → Stage 6: Platform code generation (android: IR → Jetpack Compose)
```

Generated code uses markers for incremental updates:
```kotlin
// <figma-generated id="ProductCard.header" hash="abc123">
Text(text = data.title, style = AppTypography.HeadingBold, ...)
// </figma-generated>
```

Re-running updates inside markers only. Your edits outside are safe.

## Self-Improving System

The plugin learns over time:
- **Confidence decay:** unused mappings fade → flagged for re-review
- **Negative learning:** rejected candidates are remembered, never re-suggested
- **Fallback escalation:** Minor fallback ≥ 3 → auto-suggest mapping; ≥ 5 → force fix
- **Autonomy phases:** system progresses from manual → assisted → autonomous as confidence builds

## Adding a New Platform

See [docs/contributing-platform.md](docs/contributing-platform.md).

## License

MIT
