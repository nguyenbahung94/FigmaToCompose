---
description: Figma → Jetpack Compose pipeline. Subcommands: init, bootstrap, validate, inspect, run, generate, review-mappings, suggest-mapping, cleanup-orphans, ci. Flags: --layout-map, --dry-run, --allow-write-low-confidence, --verbose.
---

Invoke the `figma-to-compose` skill to handle this request.

**Arguments received:** `$ARGUMENTS`

## Dispatch

Parse `$ARGUMENTS` and route to the skill stages as follows:

| Args start with | Skill action |
|---|---|
| `init` | Stage 0: scan design system, generate `.claude/figma-to-compose/config.json` + `mappings/token-mapping.json` |
| `bootstrap` | Stage 0.5: scan `@Composable` functions in the project, seed `mappings/component-mapping.json` |
| `validate` | Verify `config.json` + mappings are well-formed and reference real files |
| `inspect <url>` | Stages 1-5: fetch Figma node, show IR tree + token resolution + mapping decisions (no codegen) |
| `<url> --layout-map` | Spatial ASCII tree of the Figma frame (no codegen) |
| `review-mappings` | Walk pending component mappings, prompt for confirm/reject |
| `suggest-mapping` | List unmapped Figma components with suggested Compose component matches |
| `cleanup-orphans` | List generated blocks whose Figma node no longer exists upstream |
| `run <url>` | Full pipeline, smart mode: respects confidence gate, may auto dry-run |
| `generate <url>` | Full pipeline, deterministic: always emits code |
| `ci <url>` | CI mode: fail-fast, machine-readable JSON output to stdout |

**Flags (run/generate only):** `--dry-run`, `--allow-write-low-confidence`, `--verbose`.

## Required: read the skill

Before executing ANY subcommand, load `skills/figma-to-compose/SKILL.md` (core pipeline) and, for Android codegen, `skills/figma-to-compose-android/SKILL.md`. The skills contain the authoritative stage definitions, hard rules, and edge-case handling.

## Reject

If `$ARGUMENTS` is empty or matches no subcommand above: print this command's description and exit. Do NOT guess.
