# figma-to-compose: CLI commands and flags

Loaded by `SKILL.md` for `suggest-mapping`, `cleanup-orphans`, `--layout-map`, `--dry-run`, `--allow-write-low-confidence` and `--verbose`.

## CLI Commands

### `/figma-to-compose suggest-mapping`

```
List all Fallback nodes from last run with actionable hints:

[Minor] "Icon/Chevron"
  Option A: @Composable fun ChevronIcon(modifier: Modifier = Modifier) { Icon(Icons.Default.ChevronRight, contentDescription = null) }
  Option B: add "Icon/Chevron" → "ChevronIcon" to component-mapping.json

[Major] "Card/Job"
  Option A: @Composable fun OrderCard(jobId: String, title: String, modifier: Modifier = Modifier) { ... }
  Option B: add "Card/Job" → "OrderCard" to component-mapping.json
  ⚠️ Major fallback — will block generation until resolved.
```

### `/figma-to-compose cleanup-orphans`

```
1. Read every stableNodeId in generated.json
2. Check each id against current Figma file via get_design_context
3. Show list of orphans (Figma node no longer exists):
   Orphan blocks:
   - id="ProductCard.header.oldBadge" in ProductCard.kt:142
     Delete? y/n
4. Delete selected → remove the function + its manifest entry
5. Leave non-selected untouched
```

### `--layout-map`

```
Run Stage 1 (Fetch + Normalize) only.
Output a spatial ASCII tree showing component hierarchy, position, and tokens.
No IR, no code generation, no file writes.
```

Format rules:
```
[ScreenName WxH]
│
├── [POSITION] ComponentName [size, bg=token, optional]
│   ├── "text content" [typography-token, color-token]
│   ├── ChildComponent [bg=token]
│   │   └── ...
│   └── Badge/Chip [bg=token, text color-token]
│
└── [POSITION] ComponentName
    └── ...
```

Position labels: TOP / HEADER / CENTER / BOTTOM / LEFT / RIGHT / OVERLAY
For each component show: resolved token name (NOT hex) for bg/text colors.

**N-column / N-day view detection (MANDATORY):**
```
IF multiple nodes have the same top Y (within ±2dp tolerance) AND the same height (within ±4dp tolerance) AND contain date/time header labels
→ This is an N-column parallel view (e.g. 2-day schedule view)
→ Count the number of column header labels = number of days shown simultaneously
→ DateRow selection circle ≠ number of grid columns — always count column headers, not circles
→ Label as: [GRID N-col] e.g. [GRID 2-col] Mon+Tue view

Detection signals:
  - N Card nodes with same top + same height → N parallel columns
  - N "DayName - Date" text labels at same Y → N day headers
  - Single circle in DateRow does NOT mean single column — verify by counting headers
```

Example output:
```
[CheckoutScreen 393×854]
│
├── [TOP] Header h=264dp bg=color.brand-primary
│   └── "Hi Alex" [typography.heading-bold, color.white]
│
├── [CENTER] TodayCard bg=color.card-background, rounded=11dp
│   ├── "TODAY" [typography.subhead-medium, color.text-label]
│   ├── "9:00am - 4:00pm" [typography.heading-bold, color.black-900]
│   └── "Ready in 2 min" [typography.body-medium, color.brand-primary]
│
└── [BOTTOM] Navbar h=50dp bg=color.white
    └── Tab×5: Summary | Schedule | Availability | Notifications | More
```

Unresolved token? Show WARN inline:
```
├── Container bg=#D9D9D9 ⚠️ NOT IN COLORS.XML
```

### `--dry-run`

```
Run full pipeline (stages 1–6). Print:
  - Detected node tree
  - Token resolutions (including WARN for approximate matches)
  - Component mapping decisions
  - IR tree
  - Generated code preview

Do NOT write files. Do NOT update component-mapping.json.
```

### `--allow-write-low-confidence`

```
Bypass auto dry-run gate (avg_confidence < 0.70) ONLY.
Does NOT bypass: unresolved tokens, Major fallback BLOCK rule.
```

### `--verbose`

```
Show INFO logs: pipeline stage transitions, confidence boost events, decay events.
```
