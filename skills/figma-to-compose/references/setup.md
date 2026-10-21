# figma-to-compose: setup procedures

Loaded by `SKILL.md` for `init`, `bootstrap`, `validate`, `inspect` and `review-mappings`.

## Bootstrap Procedure

Run once: `/figma-to-compose bootstrap`

### Step 1 — Extract Compose components from codebase

```
Run: grep -r "@Composable" <compose_source_dir> --include="*.kt" -l
     (compose_source_dir = resolved from config.json "compose_package" → e.g. src/main/java/)
For each file found:
  Extract: function name, full parameter list, parameter types
  Store: { name, signature, file_path }
```

**Component filter rules (applied after extraction):**
```
INCLUDE:
  - public or internal visibility
  - Has @Composable annotation
  - Has ≥ 2 parameters
  - Name does NOT end with "Preview"

EXCLUDE:
  - @Preview annotated functions
  - private functions
  - Functions with < 2 params (likely helpers or wrappers)
  - Functions in files named *Preview*.kt
```

Extract signatures by reading each function from `fun` to the matching `)` before `{` or `=`, so multi-line parameter lists, defaults and lambda types count. param count = top-level commas + 1.

### Step 2 — Get Figma component names

```
Input: `/figma-to-compose bootstrap <figma-url>` (design system file or any screen that uses it)
1. Call get_code_connect_suggestions with the file key and the URL's nodeId (no node in the URL → the page id from get_metadata, usually "0:1").
2. Result empty or error (Code Connect not set up is the common case):
   → call search_design_system(fileKey, queries=[one per distinct concept, max 20 per call])
     (concept = the Compose name minus feature prefixes and suffixes like Screen/Section/Card, e.g. CheckoutSummaryCard → "summary card")
   → plus the component instance names in get_design_context of the given node
3. No URL given → skip Figma names; seed entries from the Compose side only
   (Figma name = Compose name in "Group/Name" form), all needs_review: true
Log which source was used.
```

### Step 3 — AI match Figma names → Compose signatures

```
Non-semantic properties: Mode, Theme, Dark, Platform, Device, OS (case-insensitive, also as a prefix).
  Ignore them everywhere below: in keys and in param_similarity.

Key format: Figma component set path joined by "/", plus each semantic variant as "Prop=Value",
  sorted by prop name: "Checkout/Section card/Type=Order summary". No variants → the path only.
  Variants become separate keys only when different variant values match different Compose functions;
  otherwise one key for the set path, and the variant is a prop.
  Two sets with the same path → prefix the nearest parent frame or section that differs, else "#<node id>".

Normalize the Figma name before comparing: drop spaces and "/", PascalCase each part
  ("Product / Detail card" → "ProductDetailCard"; compare also against the last part alone, "DetailCard").

name_similarity:
  1.00 exact match of the normalized name (full or last part)
  0.85 one name contains the other, or edit distance ≤ 2
  0.75 semantic match (AI judgment: same role, different words)
param_similarity = share of semantic Figma properties covered by the Compose function (no semantic properties → 0.5)
  covered = a parameter with the same meaning, OR a field with that meaning inside a parameter's model class
  (ProductCardState.title covers a Figma "Title" text property)
  A variant already fixed in the key is not counted (neither covered nor missing).
usage_frequency  = min(1.0, call sites in src/main, previews excluded / 3)

confidence = 0.7 * name_similarity + 0.2 * param_similarity + 0.1 * usage_frequency
Compare the unrounded value; store it rounded to 2 decimals.
Accept ONLY if confidence ≥ 0.70. Never 1.0 here — 1.0 means a human confirmed it.
NEVER accept a candidate present in the entry's rejected_candidates list.

Role check before saving: the Figma component and the Compose function must play the same role on screen
  (one chip vs a chip row, a month calendar vs a date-range picker = different roles → unmapped, reason "role mismatch").

Several Compose candidates for one Figma component:
  keep the highest confidence; a tie within 0.05 → unmapped, list every tied candidate.

props: one entry per Figma component property that maps to a parameter, { "from": "<figma prop>", "type": "<kotlin type>" }.
slots: one entry per lambda parameter returning Unit that takes composable content, { "required": <no default> }.
Unknown → leave {} and keep needs_review=true.
```

### Step 4 — Save, then show the table (never wait for confirmation)

```
Write every entry with confidence ≥ 0.70 to component-mapping.json NOW:
  source=bootstrap, needs_review=true, confidence = the Step 3 value
Entries < 0.70 → saved under "unmapped":
  "unmapped": { "<figma key>": { "candidates": [{ "component": "...", "confidence": 0.55 }], "reason": "..." } }

Then print the table once:
  | Figma Name     | → | Compose Component | Confidence |
  |----------------|---|------------------|------------|
  | Card/Product   | → | ProductCard       | 0.85       |

Log: "Bootstrap complete. X saved, Y unmapped. Fix any row with review-mappings or by editing the file."
Do NOT ask y/n per entry. The run must end with the file written.
```

---

## Init Procedure

Run once per project: `/figma-to-compose init`

**Pre-check:** If `.claude/figma-to-compose/config.json` already exists → ask user:
"Config found. Re-run init? This will overwrite config.json but preserve existing mappings. (y/n)"
If n → abort.

### Step 1 — Detect platform

```
Search strategy (in priority order):
  1. Find ANY file with @Composable annotation:
     grep -r "androidx.compose\|@Composable" . --include="*.kt" -l
  2. If multiple modules found → count @Composable functions per module:
     grep -r "@Composable" <module>/src --include="*.kt" -c
     → pick module with highest count
  3. Find ANY file with SwiftUI:
     grep -r "import SwiftUI" . --include="*.swift" -l
  4. If both Android + iOS found → ask user which platform to target
  5. Nothing found → ask user: "Which platform? (android/ios)"

Store detected platform + source module path for Steps 2-5.
```

### Step 2 — Scan color tokens (Android)

```
Source priority (try each, merge results):
  1. Parse colors_source (ask user for path, default: src/main/res/values/colors.xml):
     grep '<color name="' <colors_source>
     Extract: name="xxx" → #HEX
     → entry: { "hex": "#HEX", "resource": "xxx", "compose": "colorResource(R.color.xxx)", "needs_review": false }

  2. Search for Color() definitions in Kotlin:
     grep -r 'Color(0x' . --include="*.kt" | head -50
     For each unique hex NOT already in step 1 results:
       → add entry with "needs_review": true

  3. If < 3 colors found total → ask user:
     "Found only X colors. Please point me to your colors file or theme class."

Output: list of color token candidates
```

### Step 3 — Scan typography tokens (Android)

```
Source priority:
  1. Search for TextStyle definitions:
     grep -r 'TextStyle(' . --include="*.kt" -l
     For each file found → extract fontSize, fontWeight, lineHeight
     → map to typography.* key by size+weight heuristic:
       fontWeight=700 + fontSize≥20 → "typography.display-bold"
       fontWeight=700 + fontSize=18 → "typography.heading-bold"
       fontWeight=400 + fontSize=16 → "typography.body-regular"
       fontWeight=500 + fontSize=16 → "typography.body-medium"
       fontWeight=700 + fontSize=16 → "typography.body-bold"
       fontWeight=400 + fontSize=14 → "typography.subhead-regular"
       fontWeight=700 + fontSize=14 → "typography.subhead-bold"
       fontWeight=400 + fontSize=12 → "typography.caption-regular"
       Unrecognized combo → "typography.unknown-NNpx-WWw", needs_review: true

  2. Ask user: "Found X typography styles. Is there a typography class/object to use?
     Enter class name (e.g. AppTypography) or press Enter to skip:"
     If provided → store as typography_class in config

  3. If no TextStyle found → ask user to select typography file manually

Output: list of typography token candidates
```

### Step 4 — Scan spacing tokens (Android)

```
Source priority:
  1. Parse dimens.xml if exists:
     grep '<dimen' src/main/res/values/dimens.xml 2>/dev/null
     Extract dp values
  2. Search for .dp constants:
     grep -r '= [0-9]*\.dp\b' . --include="*.kt" | grep -o '[0-9]*\.dp' | sort -u | head -20
     Extract unique integer dp values

  Map to spacing.* keys:
    4dp  → spacing.xsmall
    8dp  → spacing.small
    16dp → spacing.medium
    24dp → spacing.large
    32dp → spacing.xlarge
    48dp → spacing.xxlarge
    Other → spacing.custom-Ndp, needs_review: true

Output: list of spacing token candidates
```

### Step 5 — Generate files

```
Write .claude/figma-to-compose/config.json:
{
  "platform": "<detected>",
  "schema_version": "1.0",
  "mapping_version": "1.0",
  "plugin_version": "1.x",
  "autonomy_phase": 1,
  "colors_source": "<detected_or_asked>",
  "typography_class": "<detected_or_asked_or_empty>",
  "compose_package": "<detected_or_asked>"
}

Write .claude/figma-to-compose/mappings/token-mapping.json:
  Use templates/android/token-mapping.template.json as base schema.
  Merge all color/typography/spacing candidates from Steps 2-4.
  Mark uncertain entries with "needs_review": true.

DO NOT write component-mapping.json — that is bootstrap's job.
```

### Step 6 — Summary

```
Print:
✓ Config written to .claude/figma-to-compose/config.json
✓ X color tokens extracted (Y need review)
✓ Z typography styles mapped (W need review)
✓ S spacing tokens extracted

⚠ Next steps (run in order):
  1. /figma-to-compose bootstrap   — scan @Composable functions → component mapping
  2. /figma-to-compose validate    — verify everything before first use
  3. /figma-to-compose review-mappings  — confirm pending entries
```

---

## Validate Procedure

Run: `/figma-to-compose validate`

```
Checks:
  ✓ config.json exists + valid against schemas/config.schema.json
  ✓ mappings/token-mapping.json exists + all required fields present
  ✓ mappings/component-mapping.json exists + schema valid
  ✓ colors_source path resolves (file exists)
  ✓ No token entries with "needs_review": true remaining
  ✓ plugin_version in config is compatible with this plugin version

Output:
  PASS → "Config valid. Ready to use."
  FAIL → list each failing check with fix instructions
```

---

## Inspect Procedure

Run: `/figma-to-compose inspect <figma-url>`

```
Runs Stages 1–5 (Fetch → Normalize → Token Extract → Screen Detection → Existing UI Check → IR Build)
Does NOT run Stage 6 (Generator) — no files written.

Output:
  Stage 3.5 block: Existing UI, Mode, covered / new sections, Change list
  Layout Map (spatial ASCII tree from Stage 1)
  Token Resolution log:
    ✓ #0066CC → color.brand-primary (token-mapping.json exact match)
    ✓ #16px → spacing.medium (exact match)
    ⚠ #D9D9D9 → not found in token-mapping.json, not in colors_source → UNRESOLVED
  Component Mapping log:
    ✓ Button/Primary/Large → AppButton (confidence=1.0)
    ⚠ Badge/Status → Fallback(Minor) — no mapping found
  IR Tree (LayoutNode structure)
  Summary: X resolved, Y unresolved tokens, Z fallbacks
```

---

## Review Mappings Procedure

Run: `/figma-to-compose review-mappings`

```
Shows all entries in component-mapping.json where:
  needs_review = true  OR  confidence < 0.7  OR  source = "bootstrap"

For each entry:
  | Figma Name     | → | Compose Component | Confidence | Source    | Action   |
  |----------------|---|------------------|------------|-----------|----------|
  | Card/Product     | → | ProductCard        | 0.85       | bootstrap | confirm? |

User action per entry:
  y       → confidence=1.0, source=manual, needs_review=false
  n       → entry dropped (marked unmapped)
  skip    → keep as-is (stays at bootstrap confidence)
  <name>  → remap to different Compose component name
```
