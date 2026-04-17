> **HISTORICAL DOCUMENT — Skedulo-specific origin spec.**
> This is the original design doc written for the Skedulo Android project.
> The current generic public plugin spec is at: `docs/specs/2026-04-17-figma-to-compose-public-plugin-design.md`

---

# Figma-to-Compose Pipeline — Design Spec (Skedulo Origin)

**Date:** 2026-04-17
**Status:** Draft
**Author:** bahung

---

## 0. Core Philosophy

**This is NOT a UI generator.**

It is a **Design → System → Code pipeline.**

```
Figma             → input, NOT source of truth
Design System     → source of truth (component-mapping.json + token-mapping.json)
Generated Code    → output constrained by the system, not by Figma directly
```

Implication: if Figma uses a color that does not exist in the design system → FAIL.
If Figma uses a component that does not exist in the mapping → FAIL (or Box fallback).
Figma is never allowed to "override" the design system. The system wins.

This shapes every decision in the pipeline: tokens, mapping, fallback, confidence.

---

## 1. Problem

Designers push Figma frames. Devs manually translate to Compose. Translation is slow, inconsistent, and error-prone — especially for spacing, tokens, and component variants.

Goal: a Claude Code skill that automates Figma → Kotlin/Compose generation for the Skedulo Android app.

---

## 2. Solution Overview

A single Claude Code skill: `/figma-to-compose <url>`

- Detects whether input is a **component** or a **full screen**
- Generates appropriate output: single `.kt` composable OR Fragment + ViewModel + Composable
- Adapts interaction mode based on mapping confidence:
  - **Staged** (default): checkpoint after each pipeline stage
  - **One-shot**: skips checkpoints when `coverage ≥ 90% AND avg_confidence ≥ 0.9`
- Mapping table grows with every run → system matures automatically

---

## 3. Architecture

```
/figma-to-compose <url>
        ↓
   Figma API (get_design_context)
        ↓
   Node Normalizer
        ↓
   Token Extractor ─────────────→ token-mapping.json
        ↓
   Component Detector ──────────→ component-mapping.json
   (variant-aware, confidence-gated)
        ↓
   Layout IR Builder
   (LayoutNode tree)
        ↓
   Compose Generator
   (reads IR → emits Kotlin)
        ↓
   Preview + Validate
        ↓
   Append to mapping files (if confidence qualifies)
```

---

## 4. File Structure

```
.claude/skills/figma-to-compose/
  SKILL.md                      ← skill logic + hard rules
  component-mapping.json        ← grows over time
  token-mapping.json            ← seeded from Figma variables
```

Output files → `skedulo-presentation/src/main/java/` (auto-detected package from screen/component type)

---

## 5. Bootstrap (One-Time Setup)

Run once before first use:

```
Step 1: grep @Composable in skedulo-presentation
        → extract: name + full function signature + param types

Step 2: get_code_connect_suggestions from Figma MCP

Step 3: AI matches Figma component names → Compose function signatures
        → stored with source=bootstrap, confidence=0.85

Step 4: Show full mapping table to user for one-time review
        → user confirms each entry → confidence=1.0, source=manual
        → unconfirmed entries stay at 0.85
```

Bootstrap entries are **not** auto-applied at `confidence=1.0` until user confirms. This prevents permanently baking in wrong mappings.

---

## 6. Layout IR

The intermediate representation between Figma nodes and Compose output. Generator never reads Figma data directly — only the IR.

### Token Types

All token keys use **lowercase dot notation** — enforced strictly:

```kotlin
// Single unified type — namespace enforced by convention
typealias Token = String
// spacing.*    → "spacing.small", "spacing.medium", "spacing.large"
// color.*      → "color.primary", "color.surface", "color.on-surface"
// typography.* → "typography.body", "typography.headline", "typography.label"

// Aliases for IR readability (all resolve via token-mapping.json)
typealias SpacingToken = Token
typealias ColorToken = Token
typealias TypographyToken = Token
```

**Enforcement rule:** Any token key that does not match `^(spacing|color|typography)\.[a-z][a-z0-9-]*$` → FAIL with parse error. No exceptions.

### SizeMode

```kotlin
sealed class SizeMode {
    object Fill : SizeMode()
    object Wrap : SizeMode()
    data class Fixed(val size: SpacingToken) : SizeMode()
}
```

### AlignmentProps (unified for all node types)

```kotlin
data class AlignmentProps(
    val horizontal: Alignment.Horizontal? = null,
    val vertical: Alignment.Vertical? = null
)
```

### LayoutNode

```kotlin
sealed class LayoutNode {
    data class Vertical(
        val children: List<LayoutNode>,
        val spacing: SpacingToken,
        val padding: EdgeInsets,
        val width: SizeMode,
        val height: SizeMode,
        val alignment: AlignmentProps,
        val weight: Float = 0f          // 0f = no weight
    ) : LayoutNode()

    data class Horizontal(
        val children: List<LayoutNode>,
        val spacing: SpacingToken,
        val padding: EdgeInsets,
        val width: SizeMode,
        val height: SizeMode,
        val alignment: AlignmentProps,
        val weight: Float = 0f
    ) : LayoutNode()

    data class Component(
        val mappedComponent: String,
        val props: Map<String, PropValue>,
        val slots: Map<String, Slot>,
        val action: Action?
    ) : LayoutNode()

    data class Text(
        val content: String,
        val typography: TypographyToken,
        val color: ColorToken,
        val action: Action?
    ) : LayoutNode()

    data class Stack(
        val children: List<PositionedChild>, // rendered bottom → top (last = topmost)
        val width: SizeMode,
        val height: SizeMode
    ) : LayoutNode()
    // Maps from: Figma absolute positioning, overlay layers, badge overlays
    // Generates: Box { children.forEach { Box(Modifier.align(it.alignment).offset(it.offset)) { it.node } } }

    data class Fallback(
        val description: String,
        val reason: String,
        val level: FallbackLevel        // determines generation behavior
    ) : LayoutNode()
}

enum class FallbackLevel {
    Minor,  // icon, illustration, decorative element → WARN, continue generation
    Major   // Card, Button, Form, List item, or core layout component
}
// FallbackLevel rules:
//
//   Minor → log WARN, continue generation, include Box placeholder
//
//   Major + node is ROOT-LEVEL component (direct child of screen/top frame)
//     → BLOCK: no code emitted for entire tree
//     → log ERROR: "Blocked: root-level component '<name>' has no mapping.
//                   Add to component-mapping.json before generating."
//
//   Major + node is NON-ROOT (nested inside another component)
//     → downgrade to STAGED mode (even if one-shot was active)
//     → show at checkpoint: "Major fallback: <name> will generate as Box.
//                            Confirm to continue or abort?"
//     → require explicit user confirm before proceeding
}
```

### Supporting Types

```kotlin
data class EdgeInsets(
    val top: SpacingToken,
    val bottom: SpacingToken,
    val start: SpacingToken,
    val end: SpacingToken
)

sealed class PropValue {
    data class Literal(val value: String) : PropValue()

    // NodeRef path format — formal grammar, FAIL on violation:
    //
    //   data namespace:
    //     data.<identifier>(.<identifier>)*
    //     valid:   "data.shiftId", "data.job.title", "data.shift.status.label"
    //     invalid: "data.shift[0]", "data.shiftId.0"
    //
    //   node namespace:
    //     node.<identifier>(.<identifier>)* | node.<identifier>.children[<int>](.<identifier>)*
    //     valid:   "node.header.title", "node.children[0].text", "node.footer.children[1]"
    //     invalid: "node.children", "node.[0].text"
    //
    //   FORBIDDEN: mixing namespaces in single path, any other prefix
    //   Parser: regex ^(data|node)(\.[a-zA-Z][a-zA-Z0-9]*|\\.children\[\d+\])+$
    data class NodeRef(val path: String) : PropValue()

    data class TokenRef(val token: String) : PropValue()   // design token reference

    // Expression — whitelist only, no free-form scripting
    data class Expression(
        val type: ExpressionType,
        val args: List<PropValue> = emptyList()
    ) : PropValue()
}

enum class ExpressionType {
    Uppercase,    // args[0].uppercase()
    Lowercase,    // args[0].lowercase()
    FormatDate,   // DateFormatter.format(args[0], pattern=args[1])
    Concat,       // args.joinToString(separator=args.last as Literal)
}

data class PositionedChild(
    val node: LayoutNode,
    val zIndex: Int = 0,                    // higher = rendered on top
    val alignment: AlignmentProps? = null,  // null = default (top-start)
    val offset: EdgeInsets? = null          // null = no offset
)
// Render algorithm (unambiguous):
//   children.sortedBy { it.zIndex }.forEach { render(it) }
//   Later items in sorted list = drawn on top (same as Compose natural draw order)
//   zIndex=0 → background, zIndex=1 → overlay, zIndex=2 → badge (topmost)
//   Tie-breaking: original list order (first declared = lower layer)
// Example: badge top-end with 4dp offset, on top of card
// PositionedChild(BadgeNode, zIndex=1, AlignmentProps(horizontal=End, vertical=Top), EdgeInsets(top=-4.dp, end=-4.dp))

data class Slot(
    val path: String,
    val required: Boolean = false,
    val multiple: Boolean = false
)

data class Action(
    val type: ActionType,
    val target: String?,
    val handler: ActionHandler,
    val params: Map<String, PropValue> = emptyMap()
)
// params examples:
// Navigate("ShiftDetail", params = { "id" → NodeRef("shift.id") })
// ViewModelEvent("OnShiftClick", params = { "shiftId" → NodeRef("shift.uid") })

enum class ActionType { Click, Navigate, Dismiss }

enum class ActionHandler {
    NavController,    // → findNavController().navigate(R.id.target, args)
    ViewModelEvent,   // → viewModel.onEvent(Event.OnShiftClick(shiftId))
    Callback          // → onClick: (params) -> Unit passed as composable param
}
```

---

## 7. component-mapping.json Schema

```json
{
  "Button/Primary/Large": {
    "component": "AppButton",
    "signature": "AppButton(text: String, variant: ButtonVariant, size: ButtonSize)",
    "props": {
      "variant": "ButtonVariant.Primary",
      "size": "ButtonSize.Large"
    },
    "slots": {
      "icon": { "path": "icon node path", "required": false, "multiple": false }
    },
    "confidence": 0.92,
    "source": "bootstrap | manual | ai"
  },
  "Card/Shift": {
    "component": "ShiftCard",
    "signature": "ShiftCard(header: @Composable () -> Unit, content: @Composable () -> Unit)",
    "props": {},
    "slots": {
      "header":  { "path": "Header", "required": true,  "multiple": false },
      "content": { "path": "Body",   "required": true,  "multiple": false },
      "actions": { "path": "Footer", "required": false, "multiple": true  }
    },
    "confidence": 1.0,
    "source": "manual"
  }
}
```

**Auto-save rules:**
- `source=manual` or `source=bootstrap` (confirmed) → always saved
- `source=ai` → only saved if `confidence ≥ 0.85` OR user explicitly confirms

**Confidence decay:**
```
After each run:
  IF entry was used → no decay
  IF entry was NOT used → confidence *= 0.98

last_used: ISO timestamp of last successful mapping application

When confidence < 0.5 → flag entry as "stale", require re-confirmation before use
When confidence < 0.3 → remove entry, log removal to user
```

Full schema with decay + usage fields:
```json
{
  "Button/Primary/Large": {
    "component": "AppButton",
    "confidence": 0.92,
    "source": "manual",
    "last_used": "2026-04-17T10:30:00Z",
    "usage_count": 14
  }
}
```

**Decay modifier:** entries with `usage_count ≥ 10` decay at `0.995` (slower) instead of `0.98`. Frequently-used mappings earn stability.

**Consistency boost:**
```
ONLY applies when source = bootstrap OR source = manual
source = ai → NO boost until user manually confirms the mapping (sets source = manual)

IF source qualifies AND mapping used in same context across ≥ 3 runs
  context key = parentComponentName + slotName + componentVariant
  // e.g. "ShiftCard:actions:PrimaryButton" ≠ "JobCard:actions:SecondaryButton"
  // variant must be included — same slot in different components = different context
→ confidence += 0.01 per consistent use
→ capped at 1.0
→ log INFO: "Confidence boost: Button/Primary/Large → 0.93 (consistent in ShiftCard.actions)"
```
AI-sourced mappings cannot self-reinforce. Only human-confirmed mappings earn consistency boosts — preventing wrong mappings from becoming more confident over time.

---

## 8. Token Enforcement (Hard Fail)

```
ALL colors     → MUST resolve to MaterialTheme.colorScheme.* or colorResource(R.color.*)
ALL spacing    → MUST resolve to SpacingToken (e.g., 8.dp → Spacing.Small)
ALL typography → MUST resolve to MaterialTheme.typography.*

IF any token is unresolved → FAIL generation
  + log: "Color #3B82F6 → closest match: color.primary (distance 0.02)"
  + generate draft mapping entry (NOT applied automatically)
  → user reviews draft → confirms → appended to token-mapping.json

**Closest match algorithm (per token type):**

```
Color distance:
  Euclidean distance in CIELAB color space
  ΔE = sqrt((ΔL)² + (Δa)² + (Δb)²)
  ΔE < 2.0 → acceptable match (human eye barely perceives difference)
  ΔE ≥ 2.0 → warn user, require explicit confirm

Spacing distance:
  abs(figma_dp - token_dp)
  Δ ≤ 4dp → acceptable match
  Δ > 4dp → FAIL, require user to add token

  Multiple candidates within threshold:
    → choose closest (smallest Δ)
    → if tie (equal Δ to 2+ tokens): log WARN with all candidates, require user selection
    → example: "14dp → candidates: spacing.small=12dp (Δ=2), spacing.medium=16dp (Δ=2)
                Please confirm which token to use."
    → never auto-resolve ties silently

Typography match (ordered priority):
  1. fontSize exact match
  2. fontWeight exact match
  3. lineHeight within 10%
  All 3 must pass → match accepted. Partial match → FAIL.
```
```

Generator does not emit any code if unresolved tokens remain. No silent fallbacks.

---

## 9. SKILL.md Hard Rules (Generator Constraints)

```
1. IF component exists in component-mapping.json → MUST use it, no exception
2. IF component NOT in mapping → generate Layout IR only, NO new component created
3. NEVER hardcode Color(0x...), raw dp values, or font sizes
4. NEVER emit raw Box() except as LayoutNode.Fallback — Fallback MUST log reason
5. Fallback usage triggers a warning to user: "X nodes fell back to Box — consider adding mapping"
6. NEVER create new @Composable functions for components already in mapping
7. ALL slot rules apply:
   - required=true + node missing → FAIL
   - required=false + node missing → skip
   - multiple=true → generate list/forEach
```

---

## 10. Confidence Gate (One-Shot vs Staged)

```
coverage      = matched_components / total_detected_components
avg_confidence = average confidence of all matched entries

IF coverage ≥ 0.90 AND avg_confidence ≥ 0.90 → ONE-SHOT mode
ELSE → STAGED mode (checkpoint per pipeline stage)

Staged checkpoints:
  [1] After Node Normalize  → show detected node tree, confirm
  [2] After Component Detect → show mapping decisions, confirm unknowns
  [3] After IR Build        → show IR summary, confirm before gen
  [4] After Gen             → show generated files, confirm before write
```

---

## 11. Output Structure

**Component detected:**
```
skedulo-presentation/src/main/java/com/skedulo/presentation/components/
  <ComponentName>.kt
```

**Screen detected:**
```
skedulo-presentation/src/main/java/com/skedulo/presentation/<feature>/
  <FeatureName>Fragment.kt
  <FeatureName>ViewModel.kt
  <FeatureName>Screen.kt
```

Detection logic (semantic > size — priority order):

```
1. IF node has navigation intent (contains nav action, deep link, back button)
   OR node contains distinct sections (header + content + footer at top level)
   → Screen

2. ELSE IF node is a Figma component instance (used multiple times in file)
   → Component

3. ELSE IF node width ≥ 90% of device frame AND depth ≥ 3 section levels
   → Screen (size heuristic, last resort)

4. ELSE
   → Component
```

**Known false-positive cases (must handle):**
- Modal/bottom sheet full-screen → detected by nav intent (Dismiss action), not size
- Reusable full-width layout → detected as Component by instance check
- Nested large frame → fails size heuristic, falls to Component correctly

---

## 12. Compose Generator Output Style (mandatory)

All generated Kotlin must look like code the team wrote by hand. Deviations destroy trust and adoption.

```
Rules:
  1. Modifier chaining only — never nested modifiers
  2. Trailing commas on all multi-line call sites
  3. Named parameters always — no positional args
  4. Long lambdas → extracted to slot composables (never inline > 10 lines)
  5. Max line length: 120 characters
  6. @Suppress("FunctionNaming") on every PascalCase @Composable
  7. @Suppress("UnusedPrivateMember") on every @Preview
  8. private const val for all magic numbers
  9. No // generated by AI or // auto-generated comments
```

**Example — correct output:**
```kotlin
@Suppress("FunctionNaming")
@Composable
fun ShiftCard(
    title: String,
    status: ShiftStatus,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .padding(horizontal = Spacing.Medium),
        onClick = onClick,
    ) {
        ShiftCardContent(title = title, status = status)
    }
}
```

---

## 13. Mapping Conflict Resolution

When 2+ entries in `component-mapping.json` match the same Figma node name:

```
IF confidence difference > 0.1
  → use highest confidence entry automatically

IF confidence difference ≤ 0.1
  → require user selection (staged mode shows both options)
  → selected entry gets confidence bump +0.05, source=manual

NEVER resolve conflicts silently or randomly.
```

---

## 14. Incremental Generation (do not overwrite existing code)

Generator MUST NOT overwrite code the developer has edited manually.

**Strategy: marked regions**

Generated files use region markers with IR hash:
```kotlin
// <figma-generated id="ShiftCardHeader" hash="a3f9c2">
@Composable
private fun ShiftCardHeader(...) { ... }
// </figma-generated>
```

**Hash:** SHA-6 of the IR node subtree. Changes when Figma node structure changes.

**Rules:**
- On re-run: match by `id` first, then verify `hash`
- `id` matches + `hash` same → skip (no Figma change)
- `id` matches + `hash` differs → regenerate block only
- `id` not found → append new block (dev may have renamed — do NOT delete old)
- Code outside markers → NEVER touched
- If file does not exist → create full file with all regions marked

**If file exists but has no markers:**
```
→ STOP, warn user: "File exists without generation markers.
  Overwrite will lose manual edits. Add markers manually or use --force."
→ Do NOT proceed without explicit user confirmation
```

---

## 15. Compose Generator Spec (output quality detail)

### Modifier Pipeline

Modifiers emitted in canonical order:
```
1. size      (fillMaxWidth, size, width, height)
2. weight    (if inside Row/Column with weight)
3. padding   (outer padding first, inner last)
4. background / border / clip
5. clickable / semantics
6. test tags (always last)
```

### Slot Rendering

Multi-slot components → each slot extracted to private composable:
```kotlin
// [DON'T] — inline lambda soup
ShiftCard(
    header = {
        Row { Text("Title"); Spacer(); StatusChip(status) }
    },
    content = { ... 15 lines ... }
)

// [DO] — extracted slots
ShiftCard(
    header = { ShiftCardHeader(title = title, status = status) },
    content = { ShiftCardBody(description = description) },
)

@Composable
private fun ShiftCardHeader(title: String, status: ShiftStatus) { ... }

@Composable
private fun ShiftCardBody(description: String) { ... }
```

### IR Debug Output

Before generating code, skill prints IR tree in staged mode:
```
IR Tree:
  Vertical(spacing=Spacing.Medium, width=Fill, height=Wrap)
  ├── Component(ShiftCard) [confidence=0.92]
  │   ├── slot:header → ShiftCardHeader
  │   └── slot:content → ShiftCardBody
  └── Component(AppButton) [confidence=1.0]
      └── props: variant=Primary, size=Large
```

Gives developer full visibility into what the AI decided before code is emitted.

### Fallback Report

At end of every run:
```
Generation complete.
  Mapped:   12/14 components
  Fallback: 2 nodes → Box (see below)
  Unknown tokens: 0

Fallbacks:
  - "Badge/Status" [Major] → no mapping found → BLOCKED (root-level)
    Option A — create component first:
      @Composable fun StatusBadge(status: String, modifier: Modifier = Modifier) { ... }
    Option B — add mapping:
      "Badge/Status" → "StatusBadge" in component-mapping.json
    Then re-run.

  - "Illustration/Empty" [Minor] → no mapping found → Box placeholder
    Option A — create component: @Composable fun EmptyIllustration(modifier: Modifier = Modifier) { ... }
    Option B — add mapping: "Illustration/Empty" → "EmptyIllustration"
```

---

## 16. Evolution Path

| Phase | Trigger | Behavior |
|---|---|---|
| Phase 1 (now) | 0 runs | Always staged, manual confirmation |
| Phase 2 | mapping table covers team design system | Share via git (component-mapping.json committed) |
| Phase 3 | coverage ≥ 90% AND avg_confidence ≥ 0.9 | Auto one-shot, staged only for unknowns |

Plugin packaging (future): extract skill + mapping files into standalone Claude Code plugin for cross-project sharing.

---

## 17. What We Explicitly Do NOT Build Now

- Full Figma Code Connect integration (deferred to Phase 3)
- CI/CD automation (deferred, requires stable one-shot first)
- Gradle task integration (rejected — anti-pattern)
- Auto-PR creation (deferred)

---

## 18. Future Enhancements (post-stable)

- **"Why this mapping?" debug** — print name similarity score, variant match count, usage history per mapping decision
- **Dry-run mode** — `/figma-to-compose --dry-run` shows IR + mapping without writing files
- **Suggest-mapping CLI** — `/figma-to-compose suggest-mapping` lists all Fallback nodes with closest component candidates
- **Orphan cleanup** — `/figma-to-compose cleanup-orphans` lists all `<figma-generated>` blocks whose Figma node ID no longer exists; user selects which to delete
- **Snapshot test** — render Compose preview → screenshot → diff against Figma (prerequisite for CI automation)

---

## 19. End-to-End Run Example

**Input:** `/figma-to-compose figma.com/design/abc123?node-id=ShiftCard`

---

**Stage 1 — Fetch + Normalize**
```
Figma node: "Card/Shift" (component instance, used 8× in file)
→ Detection: Component (rule 2 — multiple instances)
→ Node children: Header frame, Body frame, Footer frame
```

**Stage 2 — Token Extract**
```
Colors found:
  fill #1A56DB → token lookup → "color.primary" ✓
  text #111928 → token lookup → "color.on-surface" ✓

Spacing found:
  gap 16px → "spacing.medium" ✓
  padding 12px → "spacing.small" ✓ (closest match, Δ=4px — warn user)
```

**Stage 3 — Component Detect**
```
"Card/Shift" → component-mapping.json lookup
  → ShiftCard [confidence=0.92, source=manual] ✓

"Button/Primary" inside footer → AppButton [confidence=1.0, source=manual] ✓
"Badge/Status" → NOT FOUND → Fallback (Box)

Coverage: 2/3 = 67% → STAGED mode (below 90%)
```

**[Checkpoint] Show to user:**
```
Mapped:   ShiftCard, AppButton
Fallback: Badge/Status → no mapping
Proceed? (y/n/add-mapping)
```

**Stage 4 — IR Build**
```
IR Tree:
  Stack(width=Fill, height=Wrap)          ← overlay for badge
  ├── Component(ShiftCard)
  │   ├── slot:header → ShiftCardHeader
  │   │   └── Text("Day Shift", typography.headline, color.on-surface)
  │   ├── slot:content → ShiftCardBody
  │   │   └── Text("9:00am – 5:00pm", typography.body, color.on-surface)
  │   └── slot:actions → ShiftCardFooter
  │       └── Component(AppButton, variant=Primary, size=Large)
  │           └── action: ViewModelEvent("OnShiftClick", { shiftId: NodeRef("shift.uid") })
  └── Fallback("Badge/Status", reason="no mapping found")
```

**Stage 5 — Generate**
```kotlin
// <figma-generated id="ShiftCard" hash="a3f9c2">
@Suppress("FunctionNaming")
@Composable
fun ShiftCard(
    title: String,
    timeRange: String,
    shiftId: String,
    onShiftClick: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    Box(modifier = modifier.fillMaxWidth()) {
        com.skedulo.presentation.components.ShiftCard(
            header = { ShiftCardHeader(title = title) },
            content = { ShiftCardBody(timeRange = timeRange) },
            actions = {
                AppButton(
                    text = stringResource(R.string.view_shift),
                    variant = ButtonVariant.Primary,
                    size = ButtonSize.Large,
                    onClick = { onShiftClick(shiftId) },
                )
            },
            modifier = Modifier.fillMaxWidth(),
        )
        // TODO: Badge/Status — no mapping found, add to component-mapping.json
    }
}
// </figma-generated>
```

**Stage 6 — Fallback Report**
```
Generation complete.
  Mapped:   2/3 components
  Fallback: 1 node → Box

Fallbacks:
  - "Badge/Status" → no mapping found
    Suggestion: add mapping for StatusBadge or create new component

Confidence decay applied: 0 unused entries this run.
```

---

## 20. Runtime Behavior Details

### Stable Node ID

Every `LayoutNode` carries a stable ID computed via priority chain:

```
Priority (highest → lowest):
  1. hash(figmaNodeId + semanticRole)   ← survives rename, survives reorder
  2. semantic path ("ShiftCard.header.title")  ← fallback if no Figma node ID
  3. index ("ShiftCard.children[2]")           ← last resort only

Rule:
  IF Figma node ID unchanged (designer renamed "Header" → "TopSection")
  → ID still matches → incremental block update, NOT duplicate

  IF Figma node ID changed (node was deleted + recreated)
  → treat as new node → append new block, old block becomes orphan
  → log WARN: "Orphan block detected: ShiftCard.header — Figma node ID changed.
               Review and delete manually if no longer needed."
```

Used for: incremental regen matching (alongside hash), IR debug tree, fallback report.

---

### Logging Levels

```
INFO  → pipeline stage transitions ("Stage 2: Component Detect — 3 nodes resolved")
WARN  → fallback used, approximate token match (Δ within threshold), stale mapping
ERROR → hard fail — unresolved token, missing required slot, parse error

Default output: WARN + ERROR only
Verbose mode (/figma-to-compose --verbose): all levels
```

---

### Auto Dry-Run on Low Confidence

```
IF avg_confidence < 0.70
  → switch to dry-run automatically
  → print IR + mapping decisions
  → do NOT write any files
  → log: "Auto dry-run: avg confidence 0.63 < 0.70 threshold.
           Review mappings above, then re-run to generate."

User can override: /figma-to-compose --force-write (bypasses dry-run gate)
```

---

## 21. Open Questions

- [ ] SpacingToken values: map Figma spacing values to Skedulo's spacing system (need design team input)
- [ ] TypographyToken names: confirm MaterialTheme.typography.* mapping against Figma styles
- [ ] Screen detection heuristic: viewport fill threshold (e.g., width ≥ 360dp = screen?)
