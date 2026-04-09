---
name: figma-design-md
version: 2.1.0
description: |
  Generate a precision DESIGN.md from Figma as the single source of truth for front-end implementation.
  Pulls design context via Figma MCP, translates design values into Tailwind-native syntax,
  and optionally cross-references with the existing codebase for class mapping.
  - MANDATORY TRIGGERS: figma-design-md, design-md, design md
  - AUTO-DETECT: When the user pastes a Figma URL (figma.com/design/*, figma.com/board/*, figma.com/make/*) and asks to implement, build, or code a section — this skill MUST activate automatically BEFORE any implementation begins. The DESIGN.md is generated first, then used as the reference for coding.
  - Also trigger when: user says "generate design md", "create design md from figma", "extract design system from figma"
  - WORKFLOW: When auto-triggered by a Figma URL, the skill generates the DESIGN.md, saves the raw Figma source, outputs both to the user for review, and THEN proceeds to implementation using the DESIGN.md as the source of truth.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - mcp__claude_ai_Figma__get_design_context
  - mcp__claude_ai_Figma__get_screenshot
  - mcp__claude_ai_Figma__get_metadata
---

# Figma to DESIGN.md Skill (v2.1 — Figma-First Edition)

> *Inspired by [awesome-design-md](https://github.com/VoltAgent/awesome-design-md) — bridging the gap between static DESIGN.md files and live Figma MCP extraction.*

## Preamble (run first)

When this skill is loaded, **immediately** output:

> **[figma-design-md] Design system extraction skill active (v2 — precision mode).**

Then proceed with the workflow below.

---

## Input

This skill accepts arguments in these forms:

1. **A Figma URL** — e.g. `https://www.figma.com/design/ABC123/MyFile?node-id=1-284`
2. **A Figma URL + output path** — e.g. `https://figma.com/design/ABC123/MyFile?node-id=1-284 DESIGN.md`
3. **No arguments** — prompt the user for a Figma URL

If no output path is given, default to `DESIGN.md` in the current working directory.

---

## Workflow

### Step 1 — Parse the URL

Extract `fileKey` and `nodeId` from the Figma URL:
- `figma.com/design/:fileKey/:fileName?node-id=:nodeId` — convert `-` to `:` in nodeId
- `figma.com/design/:fileKey/branch/:branchKey/:fileName` — use branchKey as fileKey

If no URL was provided, ask the user for one using AskUserQuestion.

### Step 2 — Pull design context + Save raw source

Call `mcp__claude_ai_Figma__get_design_context` with:
- `fileKey` and `nodeId` extracted from Step 1
- `clientLanguages`: detect from the current project (check for common files), or default to `"html,css,javascript"`
- `clientFrameworks`: detect from the current project, or default to `"unknown"`

This returns:
- A screenshot of the design
- Generated reference code with colors, fonts, spacing, absolute positions
- Asset URLs (valid for 7 days)

**IMMEDIATELY save the raw Figma output** to `figma-sources/` directory:

```
figma-sources/
  node-{nodeId}.md          # Raw source for this node
```

Each `node-{nodeId}.md` file must contain:

```markdown
---
figma_url: {original URL}
file_key: {fileKey}
node_id: {nodeId}
extracted_at: {ISO date}
asset_expiry: {7 days from now}
---

## Screenshot
{Reference to screenshot — note it was displayed inline during extraction}

## Raw Reference Code
```jsx
{The COMPLETE reference code returned by get_design_context — every line, unchanged}
```

## Asset URLs
| Asset Name | URL | Expires |
|---|---|---|
| {name} | {url} | {date} |
```

This raw source is the **ground truth**. The DESIGN.md is a Tailwind-translated interpretation of this Figma source.

### Step 3 — Codebase context scan (optional, for Tailwind class mapping)

If an existing codebase is present, scan it to find the closest Tailwind class equivalents for Figma values. This helps translate Figma pixel values into the project's Tailwind vocabulary — but **Figma values always win when there's a conflict**. Run these searches **in parallel**:

#### 3a. Color audit
Search ALL project files for every hex color, rgba value, and opacity modifier actually in use:
- Every `#xxxxxx` and `#xxx` hex color
- Every `rgba(...)` value
- Every Tailwind opacity modifier pattern: `bg-[#hex]/NN`, `border-[#hex]/NN`, `text-[#hex]/NN`
- Every `shadow-[...]` definition
- Record which files each color appears in and how many times

#### 3b. Spacing & dimension audit
Search ALL project files for actual Tailwind spacing classes in use:
- All `px-*`, `py-*`, `p-*`, `pt-*`, `pb-*`, `pl-*`, `pr-*` (including arbitrary `p-[Npx]`)
- All `mx-*`, `my-*`, `m-*`, `mt-*`, `mb-*`, `ml-*`, `mr-*`
- All `gap-*`, `space-x-*`, `space-y-*`
- All `h-*`, `w-*`, `min-h-*`, `min-w-*`, `max-w-*` (especially for buttons, inputs, containers)
- Count frequency of each to identify the dominant patterns

#### 3c. Border radius audit
Search for all `rounded-*` classes and their frequency to identify the actual radius scale.

#### 3d. Typography audit
- Read the layout file for Google Fonts imports (exact families and weights loaded)
- Search for all `font-['*']`, `text-[*px]`, `font-*` (weight), `tracking-*`, `leading-*` classes
- Search for any `@font-face` declarations in CSS files

#### 3e. Shadow & elevation audit
- Search for all `shadow-*` classes (both Tailwind defaults and arbitrary)
- Search for any `box-shadow` in CSS files

#### 3f. Existing DESIGN.md check
- Check if `DESIGN.md` already exists — read it if so
- Check if `figma-sources/` directory exists

### Step 4 — Translate Figma values to Tailwind

The Figma design is the source of truth. Convert its values to Tailwind syntax:

1. **Colors**: Use the exact hex values from Figma. Note if they also exist in the codebase for context.
2. **Spacing**: Convert Figma pixel values to the nearest standard Tailwind class. Example:
   - Figma says `25px` → use `p-[25px]` (arbitrary) or `px-6` (24px) if the 1px difference is acceptable
   - Figma says `40px` → use `h-10` (40px)
   - Figma says `44px` → use `h-11` (44px)
3. **Radius**: Map Figma radius to standard Tailwind classes where they match. Example:
   - Figma says `28px` radius on a pill → `rounded-full`
   - Figma says `5px` radius → `rounded` (4px) or `rounded-[5px]` — prefer exact Figma value
4. **Typography**: Document the font families, weights, and sizes exactly as specified in the Figma design.
5. **Shadows**: Use the exact shadow values from the Figma design.

**ALWAYS prefer Figma's design values over what the codebase currently uses.** The DESIGN.md represents the intended design, not the current implementation state.

### Step 5 — Generate the DESIGN.md

Generate a DESIGN.md with these requirements:

#### PRECISION RULES (non-negotiable):

1. **Every color must include its opacity variants.** If `#121f46` is used in the Figma design, also document opacity variants specified in the design. If a codebase exists, note additional opacity variants found there. Format as a sub-list:
   ```
   - **Navy Text** (`#121f46`): Primary heading color
     - `/5` — subtle background tint
     - `/10` — light background
     - `/20` — form input borders
     - `/30` — medium borders
     - `/70` — semi-transparent overlay
   ```

2. **Every spacing value must be in Tailwind classes, not pixels.** Write `px-6` (24px) not "25px padding". The parenthetical pixel value is for reference only. If the project uses an arbitrary value like `p-[25px]`, write it as-is.

3. **Every dimension must match the Figma design.** If the design specifies different sizes for different button variants, document each with its context.

4. **Every border-radius must use the standard Tailwind class where it maps cleanly.** Write `rounded-full` not `rounded-[28px]`. Write `rounded-lg` for `8px`. Use arbitrary values like `rounded-[5px]` when no standard class matches the Figma value.

5. **Every shadow must match the Figma design.** Convert Figma shadow values to Tailwind syntax — either standard classes (`shadow-md`) or arbitrary values (`shadow-[0px_4px_4px_0px_#f3d4c6]`).

6. **Hover states must be documented.** If the Figma design specifies hover variants (e.g., `#c58600` for gold hover), include them.

7. **Font weights must match the Figma design.** Document all weights specified in the design — these are what should be loaded in the implementation.

8. **Responsive patterns must use actual breakpoint prefixes.** Write `px-6 md:px-12 lg:px-20` not "24px on mobile, 48px on tablet, 80px on desktop".

#### DESIGN.MD STRUCTURE:

```markdown
# Design System — {Page/Section Name}

## 1. Visual Theme & Atmosphere

{2-3 paragraphs describing the overall mood, density, and design philosophy.}

**Key Characteristics:**
- {Bullet list of 5-8 defining visual traits with specific values}

## 2. Color Palette & Roles

### Primary
- **{Semantic Name}** (`{hex}`): {Functional role}
  - `/{opacity}` — {where this opacity variant is used}
  - `/{opacity}` — {where this opacity variant is used}

### Accent / CTA
- **{Semantic Name}** (`{hex}`): {Role}
  - Hover: `{hover hex}` — {context}
  - `/{opacity}` — {usage}

### Neutral Scale
- **{Name}** (`{hex}`): {Role}

### Surface & Borders
- **{Name}** (`{hex}`): {Role}

### Shadow Colors
- **{Name}** (`{exact shadow value as used in code}`): {Role}

## 3. Typography Rules

### Font Families (as loaded in project)
- **{Role}**: `{font-family}`, weights: {exact weights from Google Fonts import}, {source URL}

### Hierarchy

| Role | Font | Size (Tailwind) | Weight | Line Height | Letter Spacing | Tailwind Classes |
|------|------|-----------------|--------|-------------|----------------|------------------|
| ... | ... | `text-[32px]` | 300 | normal | normal | `font-['Newsreader'] font-light text-[32px]` |

The "Tailwind Classes" column must contain the **exact class string** to copy-paste.

### Principles
- {Key typography rules}

## 4. Component Stylings

For each component, provide the **exact Tailwind class string** that recreates it:

### Buttons
**Primary CTA (Gold Pill)**
```html
<button class="{exact tailwind classes}">
  {Button text}
</button>
```
- Hover: `{hover classes or state}`
- Variants: {list size/color variants if any}

**Secondary / Outline**
```html
<button class="{exact tailwind classes}">
  {Button text}
</button>
```

### Cards & Containers
```html
<div class="{exact tailwind classes}">
  ...
</div>
```

### Inputs & Forms
```html
<input class="{exact tailwind classes}" />
<label class="{exact tailwind classes}">Label</label>
```
- Document BOTH `h-10` and `h-11` variants if both exist, with context for when each is used.

### {Other components as needed}

## 5. Layout Principles

### Spacing System (Tailwind classes, not pixels)
- **Container padding**: `{exact responsive classes}` (e.g., `px-6 md:px-12 lg:px-20`)
- **Max width**: `{class}` (e.g., `max-w-screen-xl mx-auto`)
- **Section gap**: `{classes used between major sections}`
- **Component gap**: `{classes used within sections}`
- **Form field gap**: `{classes used between form fields}`

### Spacing Scale (actually used in project)
| Tailwind Class | Pixel Value | Frequency | Primary Use |
|----------------|-------------|-----------|-------------|
| `gap-4` | 16px | 15+ | Flex layouts |
| `px-6` | 24px | 40+ | Mobile horizontal padding |
| ... | ... | ... | ... |

### Grid & Container
- {Actual grid patterns used}

## 6. Depth & Elevation

| Level | Tailwind Class | CSS Value | Use |
|-------|---------------|-----------|-----|
| Flat | (none) | — | Most surfaces |
| Subtle | `shadow-md` | {browser default} | {where used} |
| Branded | `shadow-[0px_4px_4px_0px_#f3d4c6]` | `box-shadow: 0px 4px 4px 0px #f3d4c6` | {where used} |

## 7. Do's and Don'ts

### Do
- {Rules with specific Tailwind values, not vague guidance}

### Don't
- {Anti-patterns with specific wrong values to avoid}

## 8. Responsive Behavior

### Breakpoints (as used in project)
| Prefix | Width | Padding | Key Changes |
|--------|-------|---------|-------------|
| (none) | <768px | `px-6` | {changes} |
| `md:` | 768px+ | `md:px-12` | {changes} |
| `lg:` | 1024px+ | `lg:px-20` | {changes} |

### Touch Targets
- Buttons: `{height class}` ({px value})
- Inputs: `{height class}` ({px value})
- {Other interactive elements}

## 9. Agent Prompt Guide

### Quick Color Reference (with opacity variants)
```
Primary CTA:      bg-[#d99400] text-white hover:bg-[#c58600]
Heading text:     text-[#121f46]
Body text:        text-[#3e4453]
Light bg:         bg-[#fdf7f7]
Warm bg:          bg-[#f8eeee]
Dark bg:          bg-[#0a132e]
Input border:     border-[#121f46]/20
Card shadow:      shadow-[0px_4px_4px_0px_#f3d4c6]
```

### Component Copy-Paste Snippets
{For each major component, provide a complete HTML snippet with exact Tailwind classes
that can be dropped directly into a template. These are generated from the Figma design
values, translated to Tailwind syntax.}

## 10. Figma Source Reference

- **Node**: `{nodeId}` | **File**: `{fileKey}`
- **Raw source**: `figma-sources/node-{nodeId}.md`
- **Screenshot**: captured during extraction (see raw source)
- **Asset URLs**: valid until `{expiry date}` (see raw source for full list)
```

### Step 6 — Self-audit before writing

Before writing the DESIGN.md, run a self-audit checklist:

- [ ] **Every hex color** in the DESIGN.md comes from the Figma source
- [ ] **Every opacity variant** specified in the Figma design is documented
- [ ] **Every spacing value** is expressed as a Tailwind class, not raw pixels (unless an arbitrary value is needed)
- [ ] **Every border-radius** uses the correct Tailwind class mapping
- [ ] **Every shadow** matches the Figma design values
- [ ] **Every font weight** matches what the Figma design specifies
- [ ] **Every hover state** from the Figma design is documented
- [ ] **Responsive patterns** follow the Figma design's breakpoint specifications
- [ ] **Button/input heights** list ALL variants from the design, not just one
- [ ] **The component snippets** faithfully represent the Figma design in Tailwind

If any check fails, fix it before writing.

### Step 7 — Write files

1. Create `figma-sources/` directory if it doesn't exist
2. Write `figma-sources/node-{nodeId}.md` with raw Figma source
3. Write `DESIGN.md` (or update if exists — ask user: replace, merge, or new path)

### Step 8 — Summary

Output:

> **[figma-design-md] Generated `DESIGN.md` + `figma-sources/node-{nodeId}.md`**
> - Colors: {count} base tokens + {count} opacity variants
> - Typography: {count} levels, {count} font families ({list weights loaded})
> - Components: {list with snippet count}
> - Spacing: {count} unique Tailwind spacing classes documented
> - Shadows: {count} shadow definitions
> - Self-audit: {PASS/FAIL — list any failures}

---

## Accuracy Standard

This skill targets **99% precision**. That means:

1. **Zero invented values.** Every number in the DESIGN.md must come from the Figma source. If a value can't be verified from the Figma design, flag it with `⚠️ unverified` rather than guessing.

2. **Tailwind-native output.** The DESIGN.md speaks Tailwind, not CSS. `px-6` not `padding: 24px`. `rounded-full` not `border-radius: 28px`. `h-10` not `height: 40px`. Exception: arbitrary values like `shadow-[...]` which are already Tailwind syntax.

3. **Figma is the source of truth, not the codebase.** The DESIGN.md represents the designer's intent. When Figma says 25px, document `p-[25px]` (or `px-6` if the 1px rounding is acceptable). The codebase may diverge from the design — that's a bug in the implementation, not in the DESIGN.md. If the codebase uses different values, note them as `🔧 current implementation differs` for reference.

4. **Complete opacity coverage.** If the Figma design uses `#121f46` at various opacities, ALL must be in the DESIGN.md. Additionally note any opacity variants found in the codebase for migration context.

5. **All shadow types.** Document all shadows specified in the Figma design, converted to Tailwind syntax.

6. **All interactive states.** Hover colors, focus rings, active states — if they're specified in the Figma design, they must be documented.

---

## Multi-node workflow

If the user provides multiple Figma URLs or asks to analyze an entire file:

1. Call `get_design_context` for each node
2. Save each to its own `figma-sources/node-{nodeId}.md`
3. Merge findings into a single DESIGN.md, deduplicating colors/fonts
4. Note which components appear in which sections

---

## Updating an existing DESIGN.md

If invoked with an existing DESIGN.md and a new Figma URL:

1. Read the existing DESIGN.md
2. Pull the new design context and save raw source
3. Optionally re-run the codebase context scan (Step 3) for Tailwind class mapping
4. Identify new colors, components, or patterns not already documented
5. Add them to the appropriate sections
6. Flag any conflicts (e.g., same role mapped to different colors) for user resolution
7. Update the Figma Source Reference section (Section 10)
