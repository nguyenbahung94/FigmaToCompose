---
name: figma-to-compose-ios
description: Use when generating SwiftUI code from figma-to-compose IR. Experimental — v2 placeholder.
depends_on: figma-to-compose
platform: ios
status: experimental
---

# figma-to-compose-ios

SwiftUI code generator — Stage 6 of the figma-to-compose pipeline. **Experimental / v2.**

## Status

This skill is a placeholder for iOS/SwiftUI support. The core IR (Stages 1–5) is platform-agnostic and ready. This skill will translate `LayoutNode` IR → SwiftUI.

## Planned Token Resolution (iOS)

| IR token type | SwiftUI output |
|---|---|
| `TokenRef` (spacing) | `<dp_value>` (CGFloat) |
| `TokenRef` (color) | `Color("<resource>")` from Asset Catalog |
| `TokenRef` (typography) | `Font.<style>` or custom `AppFont.<style>` |

## Planned IR → SwiftUI Mapping

| LayoutNode | SwiftUI |
|---|---|
| `Vertical` | `VStack(alignment, spacing)` |
| `Horizontal` | `HStack(alignment, spacing)` |
| `Stack` | `ZStack` with `.offset()` per child |
| `Component` | mapped SwiftUI `View` from `component-mapping.json` |
| `Text` | `Text(...).font(...).foregroundColor(...)` |

## Contributing

To implement iOS support, fill in Stage 6 token resolution and IR → SwiftUI translation above. See `docs/contributing-platform.md`.
