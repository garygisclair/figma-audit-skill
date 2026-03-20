# Figma Audit Skill for Claude Code

Audit Figma component libraries and generate developer-facing reference pages. Auto-discovers components, extracts design tokens, and produces interactive inventories with complete token manifests.

This is the reverse of [code-audit-skill](https://github.com/garygisclair/code-audit-skill) — instead of extracting tokens from code, it extracts component specifications from Figma.

## What It Does

Point it at a Figma library file. It will:

1. **Discover all components** — crawls pages via Figma MCP, identifies component sets and standalone components, classifies them as Atom / Molecule / Organism
2. **Generate an interactive `inventory.html`** — filterable by tier and search, with variant counts, property summaries, and links back to Figma
3. **Show detected design tokens** — color, spacing, radius, font, and text style variables bound to the components
4. **Ask what to audit** — pick one, several (comma-separated), or batch by tier ("all atoms")
5. **Deep-dive each component** — produces per-component HTML reference pages with:
   - Visual variant grid (all styles × sizes × states rendered with exact Figma values)
   - "With Optional Elements" section for BOOLEAN/INSTANCE_SWAP properties (icons, touch targets)
   - 9 reference tables documenting every token a developer needs
   - **Figma Import Cheat Sheet** — auto-layout settings, variables needed (COLOR/FLOAT/STRING), variant property axes

## Tested On

| File | Components Found | Variables | Complexity |
|------|-----------------|-----------|------------|
| Outline Import Test | 2 (Button 36v, Input 12v) | None (hardcoded) | SIMPLE — TEXT only |
| ITSS Design System | 6 discovered (CTA Button 60v, alerts, spinner, state layer, logos) | 13 COLOR, 8 FLOAT, 3 STRING, 4 TEXT STYLE | COMPLEX — BOOLEAN + INSTANCE_SWAP |

## Install

Copy the `figma-audit` folder into your Claude Code skills directory:

```
~/.claude/skills/figma-audit/SKILL.md
```

On Windows:
```
C:\Users\<you>\.claude\skills\figma-audit\SKILL.md
```

That's it. Claude Code will auto-discover the skill.

## Usage

```
/figma-audit https://www.figma.com/design/<fileKey>/<fileName>
```

Or just ask naturally:

- "audit this Figma library"
- "what components are in this Figma file?"
- "generate developer handoff from Figma"

### Batch Mode

After the inventory is displayed, pick multiple components:

```
> CTA button, Section notice
> all atoms
> all molecules
```

Each component runs in parallel via subagents.

## What You Get

### inventory.html (always first)

Interactive component inventory with:
- Summary strip (total count per tier)
- Library tags (variable collections detected, font families)
- Design token preview (all bound COLOR, FLOAT, STRING variables)
- Filterable tables by tier and text search
- Variant count bars for visual complexity scanning
- Links to Figma nodes
- Figma capture script for import

### Component HTML Reference Pages

Every variant rendered with exact Figma values — sizes, styles, all interaction states (hover, pressed, disabled, loading). Includes:

- Figma screenshot (embedded)
- Component properties table (VARIANT, TEXT, BOOLEAN, INSTANCE_SWAP)
- Visual variant grid
- "With Optional Elements" section (icons enabled, touch targets visible)
- Size geometry table with variable refs
- Color tokens with inline swatches and Figma variable names
- Interaction state tokens (state layer mechanism documented)
- Disabled state tokens
- Dark mode color mapping (when variables have modes)
- **Figma Import Cheat Sheet** — auto-layout settings, complete variable list (COLOR + FLOAT + STRING + TEXT STYLE), variant property axes

## Extraction Rules

The skill follows 6 extraction rules from the [Code Bridge pipeline](https://github.com/garygisclair/bootstrap-figma-to-code-and-back-again):

1. **Read properties first** — `componentPropertyDefinitions` tell you everything before you inspect children
2. **Property-guided child inspection** — don't speculate; find what properties declared
3. **Variable binding audit** — `get_variable_defs` returns all COLOR, FLOAT, STRING tokens bound to the component
4. **Layout capture** — layoutMode, padding (all 4 sides), itemSpacing, sizing modes, cornerRadius
5. **Font capture** — fontFamily, fontStyle, fontSize, lineHeight, letterSpacing, text style bindings
6. **Effects capture** — Figma-native DROP_SHADOW/INNER_SHADOW format

## How It Reads Figma

Uses the Figma MCP server tools:

| Tool | Purpose |
|------|---------|
| `get_metadata` | XML tree of node IDs, names, types, sizes — for page/component enumeration |
| `get_design_context` | Full design details as React+Tailwind code + screenshot — primary extraction source |
| `get_variable_defs` | All design token variables bound to a node (colors, spacing, fonts) |
| `get_screenshot` | Screenshot of a node — embedded in HTML pages |

### Known Limitation: Multi-Page Files

The Figma MCP API does not support listing all pages in a file. `get_metadata("0:1")` returns only the first page. For multi-page library files, the skill will ask you to provide page URLs or component node IDs so it can discover components on other pages.

Single-page files work automatically — all components are discovered in one call.

## Output Format

JSON component specs use the same schema as the [Code Bridge Figma Plugin](https://github.com/garygisclair/bootstrap-figma-to-code-and-back-again), enabling:

- **Drift detection** — compare Figma-extracted specs against code-extracted specs using `compare.mjs`
- **Round-trip** — extract from Figma → compare with code → identify divergences
- **Plugin import** — re-import modified specs back into Figma

## Related Projects

- [code-audit-skill](https://github.com/garygisclair/code-audit-skill) — the reverse: extract design tokens from code

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- Figma MCP server configured (included with Claude Code's Figma integration)
