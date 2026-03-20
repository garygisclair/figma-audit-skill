---
name: figma-audit
description: Audit a Figma component library and generate an interactive inventory + per-component HTML reference pages. Use when the user invokes /figma-audit, asks to "audit a Figma library", "what components are in this Figma file", "inventory this Figma file", or wants to extract component specs from Figma for developer handoff. Works with any Figma design library file.
---

# Figma Audit — Component Library Extraction & Analysis

Crawls a Figma component library via MCP, classifies components by atomic design
tier, and generates developer-facing reference pages with complete token manifests,
variant grids, and auto-layout specs.

This is the reverse of `/code-audit`: instead of extracting design tokens from code,
it extracts component specifications from Figma.

---

## When This Skill Activates

- `/figma-audit <figma-url>` — explicit invocation
- "audit this Figma library"
- "what components are in this Figma file?"
- "inventory the Figma components"
- "extract component specs from Figma"
- "generate developer handoff from Figma"

---

## Step 0 — Parse Figma URL

Extract `fileKey` (and optionally `nodeId`) from the Figma URL:

| URL pattern | Extraction |
|-------------|------------|
| `figma.com/design/:fileKey/:fileName` | Use `fileKey` |
| `figma.com/design/:fileKey/:fileName?node-id=:nodeId` | Use `fileKey`, convert `-` to `:` in `nodeId` |
| `figma.com/design/:fileKey/branch/:branchKey/:fileName` | Use `branchKey` as `fileKey` |
| `figma.com/file/:fileKey/...` | Use `fileKey` |

If only a `fileKey` is provided (no URL), use it directly.

---

## Step 1 — Discover Pages

Call `get_metadata(fileKey, nodeId="0:1")` to get the first page's structure.

**IMPORTANT: Multi-page limitation.** The Figma MCP API does not provide an
endpoint to list all pages in a file. `get_metadata("0:1")` only returns the
first page. Page IDs in Figma are arbitrary (not sequential from `0:1`), so
there is no reliable way to probe for additional pages.

**If `0:1` contains component sets (XML `<frame>` nodes with `<symbol>` children
named like `Property=Value`):**
- Great — parse them and proceed to Step 2. This is the single-page library case.

**If `0:1` contains NO component sets (e.g., it's a cover page, playground, or
documentation page):**
- The file is likely multi-page with components on other pages.
- Tell the user:

> This appears to be a multi-page Figma file. The first page (`<page-name>`)
> contains no component sets. The MCP API cannot enumerate pages in a
> multi-page file automatically.
>
> To discover components, please provide:
> 1. **Page URLs** — navigate to a component page in Figma and share the URL
>    (the `node-id` parameter will identify the page)
> 2. **Component URLs** — click any component set in Figma and share its URL
> 3. **Node IDs** — if you know specific component node IDs
>
> If all components were on a single page, discovery would be automatic.

- If the user provides node IDs or URLs, call `get_metadata` on each to discover
  the component sets on those pages/nodes.
- Include a "Partial Discovery" warning in the inventory.html output.

---

## Step 2 — Enumerate Components

For each component page, call `get_metadata(fileKey, nodeId=pageId)` to list all
top-level nodes.

**Identify component types from the XML:**

| Node type | What it is | How to handle |
|-----------|------------|---------------|
| **COMPONENT_SET** | Container with variants | Primary target — this IS a design system component |
| **COMPONENT** (top-level) | Standalone component, no variants | Include — single-variant component |
| **FRAME** | Grouping frame, section divider | Skip unless it contains components |
| **INSTANCE** | Instance of another component | Skip — not a library definition |

For each COMPONENT_SET, record:
- Node ID
- Name (this is the component name)
- Child count (number of variants)
- Bounding box (width × height from XML)

For standalone COMPONENTs (not inside a COMPONENT_SET), record the same.

---

## Step 3 — Classify Components

Assign each component to an atomic design tier. Use **multiple signals**, not just one:

### Classification heuristics

| Signal | Atom | Molecule | Organism |
|--------|------|----------|----------|
| **Variant count** | 1–6 | 7–30 | 30+ |
| **Nested instances** | 0 | 1–3 | 4+ |
| **Child node depth** | 1–2 levels | 2–3 levels | 4+ levels |
| **Component properties** | 0–2 (TEXT only) | 2–4 (TEXT + BOOLEAN) | 5+ (TEXT + BOOLEAN + INSTANCE_SWAP) |

### Page name override

If the designer organized components into pages named after tiers, respect that:
- Pages named "Atoms", "Primitives", "Foundation" → classify contents as Atoms
- Pages named "Molecules", "Components" → classify as Molecules
- Pages named "Organisms", "Patterns", "Modules" → classify as Organisms
- Pages named "Templates", "Pages", "Layouts" → classify as Templates

### Component name heuristics (tiebreaker)

| Pattern | Likely tier |
|---------|-------------|
| Icon, Badge, Spinner, Divider, Avatar, Dot, Tag | Atom |
| Button, Input, Chip, Toggle, Switch, Checkbox, Radio | Atom or Molecule |
| Card, Alert, Toast, Tooltip, Dropdown, Modal, Dialog | Molecule or Organism |
| Navbar, Sidebar, Header, Footer, Table, Form, Panel | Organism |

When signals conflict, prefer the page name → component properties count → variant count.

---

## Step 4 — Generate inventory.html

Generate an **`inventory.html`** file in the output directory. This is the first
deliverable — produced before any deep-dive work begins.

### inventory.html structure

Match the visual style of component HTML pages (same font, spacing, color palette). Include:

1. **Title & subtitle** — `Component Inventory: <file-name>` + Figma file URL + date
2. **Library tags** — design system name, file key, page count, variable collections detected
3. **Summary strip** — total components + count per tier (Atoms / Molecules / Organisms / Templates), displayed as large numbers with labels
4. **Filter bar** — text search input + tier toggle buttons (All / Atoms / Molecules / Organisms / Templates) + live match count. Filtering is client-side JavaScript.
5. **Tier sections** — one `<div>` per tier, each with:
   - Tier header: name + count badge + one-line description
   - Table columns: `#` | Component | Node ID | Variants | Properties | Description
   - Variants column shows count with a proportional bar for visual scanning
   - Rows are filterable by the search input and tier buttons
6. **Footer** — generator attribution (`/figma-audit — Component Discovery`) + date + Figma file URL
7. **Figma capture script** in `<head>`:
   `<script src="https://mcp.figma.com/mcp/html-to-design/capture.js" async></script>`

### Tier descriptions

| Tier | Description |
|------|-------------|
| Atoms | Self-contained primitives — icons, badges, spinners, dividers |
| Molecules | Small composites — buttons, inputs, chips, toggles |
| Organisms | Complex composites — cards, modals, navbars, data tables |
| Templates / Pages | Full page layouts and high-level wrappers |

### Filtering behavior

- Text search matches against component name and description (case-insensitive)
- Tier buttons show/hide entire tier sections
- Match count updates live (e.g., "12 matches")
- "All" button is active by default

### Column definitions

| Column | Source | Notes |
|--------|--------|-------|
| `#` | Row index | Sequential within tier |
| Component | Component set name from Figma | Link to Figma node: `https://figma.com/design/{fileKey}?node-id={nodeId}` |
| Node ID | Figma node ID | `123:456` format, useful for MCP tool calls |
| Variants | Count of children in component set | 1 for standalone components |
| Properties | Summary of component property types | e.g., "2 VARIANT, 1 TEXT, 2 BOOLEAN" |
| Description | Component description from Figma (if set) | May be empty |

**After generating `inventory.html` and opening it in the browser**, present a brief markdown summary in the conversation and ask the user:
> Which component(s) would you like to audit first? You can pick one, several (comma-separated), or say "all atoms" / "all molecules" to batch them.

---

## Step 5 — Extract Component Data

For each selected component, follow the extraction rules in order. These rules come
from the Code Bridge pipeline's `extraction-rules.md` and are the proven method for
reading Figma components.

### Rule 1: Read the Property List First

Call `get_design_context(fileKey, componentSetNodeId)` to get the component's full
design context.

**IMPORTANT: Understanding the response format.** `get_design_context` returns
generated React+Tailwind code, NOT raw Figma JSON properties. You must parse
design values from the generated code:

- **Variant axes and values**: extracted from the TypeScript props type definition
  (e.g., `size?: "Large" | "Medium" | "Small"`)
- **Colors**: extracted from Tailwind classes (e.g., `bg-[#0366d6]`, `text-white`,
  `text-[#111319]`, `text-[rgba(255,255,255,0.7)]`)
- **Padding**: from Tailwind classes (e.g., `px-[16px]`, `py-[10px]`)
- **Borders**: from Tailwind classes (e.g., `border-2 border-[#0366d6] border-solid`)
- **Shadows**: from Tailwind classes (e.g., `shadow-[0px_1px_2px_0px_rgba(0,0,0,0.2)]`)
- **Corner radius**: from Tailwind classes (e.g., `rounded-[6px]`)
- **Font**: from Tailwind classes (e.g., `font-['Inter:Regular',sans-serif]`, `text-[14px]`)
- **Opacity**: from Tailwind classes (e.g., `opacity-50`)
- **Overflow**: from Tailwind classes (e.g., `overflow-clip` → clipsContent: true)
- **Component properties**: from the function parameter defaults
  (e.g., `label = "Button"` → TEXT property with default "Button")

The response also includes a **screenshot** of the component set — embed this in
the HTML reference page as visual reference.

Parse the conditional logic in the generated code to map each variant combination
(e.g., `isPrimaryAndLargeAndEnabled`) to its specific CSS values. Build a lookup
table of Style × Size × State → { fill, text, border, shadow, padding, fontSize }.

From the props type, extract component property definitions:

| Property Type | What It Means |
|---------------|---------------|
| **VARIANT** | Axis of the variant matrix (e.g., Size, Color, State) |
| **TEXT** | An editable text label exists somewhere inside |
| **BOOLEAN** | A hidden toggleable element exists (icon slot, touch target, etc.) |
| **INSTANCE_SWAP** | A swappable nested component exists (icon, avatar, etc.) |

**If there are no BOOLEAN or INSTANCE_SWAP properties, the component has no
hidden inner parts.** What you see in the default variant is what you get.

Record the verdict:
- **SIMPLE** — TEXT only, no hidden structure
- **COMPLEX** — has BOOLEAN or INSTANCE_SWAP properties (hidden layers, swappable parts)

### Rule 2: Property-Guided Child Inspection

Once you know the property list, inspect children **only to map what the properties
already told you exists:**

- For each **BOOLEAN** property: find the child whose `visible` is wired to it
- For each **INSTANCE_SWAP** property: find the child instance whose `mainComponent` is wired to it
- For each **TEXT** property: find the text node whose `characters` is wired to it

This is not a speculative search — the property list declared everything.

### Rule 2b: Rendering Hidden Properties in HTML Output

BOOLEAN and INSTANCE_SWAP properties are toggleable — they exist in every variant
but may be hidden by default. Handle them in the HTML reference page:

**If the generated code shows the hidden elements** (conditional renders gated by
the BOOLEAN prop, with CSS/layout values visible):
- Add a **"With Optional Elements"** section below the main variant grid
- Show a representative subset (e.g., one size per style, default state only)
  with the BOOLEAN toggled ON — e.g., buttons with leading + trailing icons
- Label clearly: "With leading icon + trailing icon enabled"
- This tells developers what the component looks like with all slots populated

**If the generated code does NOT show the hidden elements** (e.g., the MCP
response was truncated, or the elements aren't in the default variant code):
- In the Component Properties table, add a note column or footnote:
  "Not visible in default variants — toggle on in Figma to inspect"
- Still list the property name, type, and default value — the property
  definition itself is valuable even without the visual

**Always list all properties in the table regardless** — the properties table
is the component's public API. If you can render them, show them. If you can't,
say so explicitly. Never silently omit a detected property.

### Rule 3: Variable Binding Audit

Call `get_variable_defs(fileKey, componentSetNodeId)` to extract all bound variables.

**`get_variable_defs` returns ALL variable types, not just colors.** Capture every
type the component uses:

| Variable Type | Examples | What to record |
|---------------|----------|----------------|
| **COLOR** | `Background/Accent`, `Foreground/Primary`, `Border/Medium` | Variable name + resolved hex (light and dark if modes exist) |
| **FLOAT** | `Spacing/Component`, `Radius/Md`, `Spacing/Inline-sm` | Variable name + resolved number (px) |
| **STRING** | `font/family-primary` | Variable name + resolved string value |

The response format is a flat object: `{ "variable/path": resolvedValue }` where
resolvedValue is a hex string for colors or a number for floats.

**Recording format by type:**

For colors — use ColorRef: `{ "var": "Background/Accent", "fallback": "#0968f6" }`

For numbers — use numeric ref: `{ "var": "Spacing/Component", "fallback": 12 }`

For strings — use string ref: `{ "var": "font/family-primary", "fallback": "Market Sans" }`

**In the "Figma Variables Needed" table**, include ALL bound variables organized
by type. Columns:

| Variable Name | Type | Light Value | Dark Value | Used By |
|---------------|------|-------------|------------|---------|

This table should list every variable the component references, including:
- Color variables for fills, strokes, text colors (COLOR type)
- Spacing/padding variables (FLOAT type)
- Radius variables (FLOAT type)
- Font family variables (STRING type)
- Any other bound variables

**When `get_variable_defs` returns `{}`** (no variables bound), all values are
hardcoded. In this case:
- Use `{ "fallback": "#hex" }` format for all ColorRefs (no `var` key)
- In the "Figma Variables Needed" table, suggest variable names based on the
  hardcoded values found. Label these as suggestions, not existing bindings.
- Note "No Figma variables bound" at the top of the Variables table

### Rule 4: Layout Capture

For the root component and every FRAME child, record:
- `layoutMode` (HORIZONTAL / VERTICAL / NONE)
- Padding: all 4 sides separately (never collapse to shorthand)
- `itemSpacing`
- Sizing modes: `primaryAxisSizingMode`, `counterAxisSizingMode`
- `cornerRadius`
- `clipsContent`
- Min/max width/height constraints

### Rule 5: Font Capture

For every text node, record:
- `fontFamily`, `fontStyle` (Regular, Bold, Semi Bold, etc.)
- `fontSize`, `lineHeight`, `letterSpacing`
- Text style binding (if bound to a Figma text style, record the style name)

### Rule 6: Effects Capture

For every node with effects, record in Figma-native format:
```json
{
  "type": "DROP_SHADOW",
  "color": { "r": 0, "g": 0, "b": 0, "a": 0.2 },
  "offset": { "x": 0, "y": 1 },
  "radius": 2,
  "spread": 0,
  "visible": true
}
```

Also capture effect styles if bound (e.g., `Shadow/Elevation-1`).

### Extraction output format

Assemble the extracted data into the Bridge JSON format. This is the same schema used
by the Code Bridge Figma plugin (`bootstrap-figma-to-code-and-back-again/plugin/src/code.ts`):

```json
{
  "name": "<component-name>",
  "source": "figma:<fileKey>/<nodeId>",
  "renderType": "declarative",
  "variantProperties": {
    "Style": ["Primary", "Secondary"],
    "Size": ["Large", "Medium", "Small"],
    "State": ["Enabled", "Hovered", "Disabled"]
  },
  "componentProperties": {
    "label": { "type": "TEXT", "defaultValue": "Button" },
    "leading icon?": { "type": "BOOLEAN", "defaultValue": false },
    "leading icon": { "type": "INSTANCE_SWAP", "defaultName": "edit-16" }
  },
  "base": {
    "layoutMode": "HORIZONTAL",
    "primaryAxisAlignItems": "CENTER",
    "counterAxisAlignItems": "CENTER",
    "primaryAxisSizingMode": "AUTO",
    "counterAxisSizingMode": "AUTO",
    "itemSpacing": 8,
    "cornerRadius": 48,
    "clipsContent": true,
    "fontFamily": "Market Sans",
    "fontWeight": 700
  },
  "sizes": {
    "Large": { "height": 48, "paddingH": 24, "fontSize": 16, "lineHeight": 24, "letterSpacing": -0.32 },
    "Medium": { "height": 40, "paddingH": 20, "fontSize": 14, "lineHeight": 20, "letterSpacing": 0 },
    "Small": { "height": 32, "paddingH": 16, "fontSize": 14, "lineHeight": 20, "letterSpacing": 0 }
  },
  "styles": {
    "Primary": {
      "fill": { "var": "Background/Accent", "fallback": "#0968f6" },
      "textColor": { "var": "Foreground/On-accent", "fallback": "#ffffff" },
      "border": null,
      "stateLayerColor": "#ffffff",
      "stateOpacity": { "hover": 0.16, "pressed": 0.24 },
      "disabledFill": { "var": "Background/Disabled", "fallback": "#c7c7c7" },
      "disabledTextColor": { "var": "Foreground/On-disabled", "fallback": "#ffffff" }
    }
  },
  "children": [
    {
      "type": "TEXT",
      "name": "label",
      "characters": "Button",
      "fontFamily": "Market Sans",
      "fontStyle": "Bold",
      "propertyRef": { "characters": "label" }
    }
  ],
  "variants": [
    {
      "name": "Style=Primary, Size=Large, State=Enabled",
      "fillColor": { "var": "Background/Accent", "fallback": "#0968f6" },
      "textColor": { "var": "Foreground/On-accent", "fallback": "#ffffff" },
      "paddingLeft": 24, "paddingRight": 24,
      "paddingTop": 12, "paddingBottom": 12,
      "fontSize": 16, "lineHeight": 24,
      "effects": []
    }
  ],
  "_metadata": {
    "figmaFileKey": "<fileKey>",
    "figmaNodeId": "<nodeId>",
    "extractedAt": "ISO-8601 timestamp",
    "verdict": "COMPLEX — 3 hidden layers, 2 instance swaps"
  }
}
```

### Structured vs flat format choice

Use the **structured format** (with `sizes` and `styles` objects, like CtaButton.json)
when the component has a clean matrix of Style × Size × State. This is more readable
and shows the design intent.

Use the **flat variant format** (with a `variants` array listing every combination,
like Button.json) when:
- Properties differ per-variant in non-systematic ways
- The component doesn't have a clean style/size/state matrix
- You need plugin-compatible output for direct import

Generate both when the user requests JSON output — the structured format for reading,
the flat format for import.

---

## Step 6 — Generate HTML Reference Page

For each audited component, create `<component>.html` in the output directory.

### Page structure

```
┌─────────────────────────────────────────────────────┐
│ Component Name                                       │
│ Source: Figma file / Node ID / Extraction date        │
├─────────────────────────────────────────────────────┤
│                                                      │
│ ┌─────────────────────────────────────────────────┐  │
│ │ Screenshot from Figma (embedded image)           │  │
│ └─────────────────────────────────────────────────┘  │
│                                                      │
│ Component Properties                                 │
│ ┌─────────────────────────────────────────────────┐  │
│ │ Property | Type | Default | Wired To             │  │
│ └─────────────────────────────────────────────────┘  │
│                                                      │
│ Visual Variant Grid                                  │
│ ┌─────────────────────────────────────────────────┐  │
│ │ Rendered variants using extracted CSS values      │  │
│ │ One section per style, grid of size × state       │  │
│ └─────────────────────────────────────────────────┘  │
│                                                      │
│ ═══════════════════════════════════════════════════  │
│ REFERENCE TABLES                                     │
│ ═══════════════════════════════════════════════════  │
│                                                      │
│ 1. Size Geometry                                     │
│ 2. Color Tokens — Enabled State                      │
│ 3. Interaction State Tokens                          │
│ 4. Disabled State Tokens                             │
│ 5. Dark Mode Color Tokens (if applicable)            │
│ 6. Figma Auto-Layout Settings                        │
│ 7. Figma Variables Needed                            │
│ 8. Variant Property Axes                             │
│                                                      │
│ ═══════════════════════════════════════════════════  │
│ FIGMA IMPORT CHEAT SHEET                             │
│ ═══════════════════════════════════════════════════  │
│                                                      │
│ Footer: /figma-audit + date + source                 │
└─────────────────────────────────────────────────────┘
```

### Rules for the HTML

- **Use the exact values from Figma** — same colors, padding, border-radius, fonts, shadows
- **Render all states** as static elements using CSS classes (`.is-hovered`, `.is-pressed`, `.is-disabled`)
- **Include dark mode** if the component has dark-mode variable bindings (render in a dark container)
- **Include the Figma capture script** in `<head>`:
  `<script src="https://mcp.figma.com/mcp/html-to-design/capture.js" async></script>`
- **Embed the Figma screenshot** at the top as visual reference (use the image URL from `get_design_context`)

### MANDATORY CHECKLIST — every component HTML file must include ALL of these sections

- [ ] Figma screenshot (embedded at top)
- [ ] Component properties table (VARIANT, TEXT, BOOLEAN, INSTANCE_SWAP)
- [ ] Visual variant grid (all states rendered with extracted values)
- [ ] Size Geometry table
- [ ] Color Tokens — Enabled State table
- [ ] Interaction State Tokens table
- [ ] Disabled State Tokens table
- [ ] Dark Mode Color Tokens table (if component uses mode-aware variables)
- [ ] **Figma Auto-Layout Settings table** ← often skipped, always required
- [ ] **Figma Variables Needed table** ← often skipped, always required
- [ ] **Variant Property Axes table (Figma format)** ← often skipped, always required

The last three tables are the **Figma Import Cheat Sheet** — they are the most
important deliverable for developers implementing from this reference. Without them,
a developer cannot build the component from the reference page alone. Never omit them.

### Reference table specifications

#### 1. Component Properties

| Property | Type | Default Value | Wired To |
|----------|------|---------------|----------|

One row per component property definition. This is the component's public API.

#### 2. Size Geometry — one row per size variant

| Size | Height | Padding V | Padding H | Font Size | Line Height | Font Weight | Letter Spacing | Radius |
|------|--------|-----------|-----------|-----------|-------------|-------------|----------------|--------|

Note any size exceptions (e.g., a style that overrides padding at a specific size,
like borderless buttons with tighter padding).

#### 3. Color Tokens — Enabled State — one row per style

| Style | [swatch] | Fill | [swatch] | Text | [swatch] | Stroke | Stroke Weight | Variable Refs |
|-------|----------|------|----------|------|--------|---------------|---------------|

- Use inline color swatches (`<span>` with background color + 1px border)
- Use "transparent" (not blank) for transparent fills; use dashed-border swatch
- **Variable Refs** column: list the Figma variable names used (e.g., `Background/Accent`)

#### 4. Interaction State Tokens — how each state modifies the enabled appearance

| Style | State | Mechanism | Fill Change | Text Change | Opacity/Overlay |
|-------|-------|-----------|-------------|-------------|-----------------|

- For state-layer systems: show the overlay color + opacity per state
- Document the *rule*: "16% white overlay on primary" is more useful than just the rgba value

#### 5. Disabled State Tokens — one row per style

| Style | [swatch] | Fill (Disabled) | [swatch] | Text (Disabled) | Stroke (Disabled) | Opacity |
|-------|----------|-----------------|----------|-----------------|-------------------|---------|

#### 6. Dark Mode Color Tokens (if applicable)

| Token Name | [swatch] | Light | [swatch] | Dark | Variable Name |
|------------|----------|-------|----------|------|---------------|

- Include all variables that have both light and dark values
- Flag tokens that are static across modes

#### 7. Figma Auto-Layout Settings — base frame properties

| Property | Value |
|----------|-------|

Include: Layout Mode, Primary Axis Align, Counter Axis Align, Primary Axis Sizing,
Counter Axis Sizing, Item Spacing, Padding (all 4 sides), Corner Radius, Clips Content,
Font Family, Font Weight, Min Width, Max Width.

#### 8. Figma Variables Needed — complete variable list for the component

| Variable Name | Type | Light Value | Dark Value | Collection | Used By |
|---------------|------|-------------|------------|------------|---------|

- Include both COLOR and FLOAT variables
- This table is the single source of truth for what variables must exist before
  building this component

#### 9. Variant Property Axes — the cartesian product definition

| Property | Values | Count |
|----------|--------|-------|
| Style | Primary, Secondary, Tertiary, Borderless | 4 |
| Size | Large, Medium, Small | 3 |
| State | Default, Hover, Pressed, Disabled, Loading | 5 |
| **Total Variants** | | **60** |

---

## Step 7 — Write Outputs

Save all artifacts to the output directory (default: `audit/` relative to the
working directory, or a user-specified path).

| File | Content |
|------|---------|
| `inventory.html` | Interactive component inventory — first deliverable |
| `<component>.html` | HTML reference page with screenshot, variant grid, and all reference tables |
| `<component>.json` | Bridge JSON spec (structured format) — generated on request |
| `<component>-flat.json` | Bridge JSON spec (flat variant format, plugin-compatible) — generated on request |

### Batch mode

When the user selects multiple components, use subagents to extract and generate
pages in parallel. Each subagent handles one component end-to-end (extract → HTML → JSON).

Always include the extraction rules (Step 5, Rules 1-6) in the subagent prompt —
subagents don't have access to this skill file.

Always include the mandatory checklist in the subagent prompt — subagents have been
observed skipping the Figma Import Cheat Sheet tables unless explicitly mandated.

---

## Autonomy Rules

**Run automatically (no confirmation needed):**
- Calling Figma MCP tools on the file the user provided
- Creating audit output files in the output directory
- Taking screenshots of components
- Serving HTML locally for visual verification

**Ask before:**
- Accessing Figma files the user hasn't provided
- Writing to directories outside the output directory
- Making any modifications to the Figma file (Code Connect mappings, etc.)

---

## Ground Rules

- **Extract, don't guess.** Every value must come from the Figma MCP response. If a value isn't in the response, say "not available from MCP" — don't infer from component names or conventions.
- **Properties first.** Always read `componentPropertyDefinitions` before inspecting children (Rule 1). The property list tells you exactly what exists.
- **Variable bindings are gold.** Always call `get_variable_defs` — these map Figma variables to resolved values and give you the token architecture for free.
- **Parallelize extraction.** Use subagents to audit multiple components concurrently.
- **Keep reports scannable.** Lead with the screenshot and properties table. Put detailed variant data in the reference tables.
- **Bridge JSON compatibility.** The JSON schema uses the same format as the Code Bridge Figma plugin (`bootstrap-figma-to-code-and-back-again/plugin/src/code.ts`). Specs extracted from Figma can be directly compared against specs extracted from code using `compare.mjs`.
- **Respect designer intent.** If components are organized into pages by tier, honor that classification. If component descriptions exist, include them in the inventory.
- **ColorRef format always.** All colors must use `{ "var": "name", "fallback": "#hex" }` for variable-bound colors, or `{ "fallback": "#hex" }` for hardcoded colors. Never use bare hex strings.
- **Effects as Figma-native.** Shadows and blurs must use Figma's `effects` array format with `{ type, color: {r,g,b,a}, offset: {x,y}, radius, spread, visible }`. Never use CSS box-shadow strings.
- **Separate component sets by geometry.** If a component has fundamentally different frame structures across variants (e.g., standard buttons vs FABs vs icon-only buttons), note this in `_metadata` and recommend splitting into separate component sets.
