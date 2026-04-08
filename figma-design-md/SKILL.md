---
name: figma-design-md
version: 2.0.0
description: |
  Generate a precision DESIGN.md + save raw Figma source for 1:1 front-end implementation.
  Pulls design context via Figma MCP, cross-references with the actual codebase, and outputs
  a pixel-accurate design document with Tailwind-native values.
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

# Figma to DESIGN.md Skill (v2 — Precision Edition)

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

This raw source is the **ground truth** for implementation. The DESIGN.md is an interpreted layer on top.

### Step 3 — Deep codebase audit (CRITICAL for accuracy)

This step is what makes the difference between 80% and 99% accuracy. Run these searches **in parallel**:

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

### Step 4 — Cross-reference Figma values with codebase

This is the precision step. For every value extracted from the Figma reference code:

1. **Colors**: Does this hex exist in the codebase? If yes, note how it's used. If not, flag it as new.
2. **Spacing**: Convert Figma absolute pixel values to the nearest Tailwind class the project actually uses. Example:
   - Figma says `left-[25px]` → codebase uses `px-6` (24px) → document as `px-6` (24px), NOT 25px
   - Figma says `h-[40px]` → codebase uses `h-10` → document as `h-10` (40px)
   - Figma says `h-[44px]` → codebase uses `h-11` → document as `h-11` (44px)
3. **Radius**: Map Figma `rounded-[Npx]` to the Tailwind class actually used. Example:
   - Figma says `rounded-[28px]` → codebase uses `rounded-full` → document as `rounded-full`
   - Figma says `rounded-[5px]` → check if codebase uses `rounded-[5px]` or `rounded-sm` or `rounded`
4. **Typography**: Verify font weights/sizes against what's actually loaded in Google Fonts imports.
5. **Shadows**: Check if the Figma shadow matches codebase shadows exactly.

**ALWAYS prefer the codebase's Tailwind class over raw pixel values.** The DESIGN.md must speak the project's language.

### Step 5 — Generate the DESIGN.md

Generate a DESIGN.md with these requirements:

#### PRECISION RULES (non-negotiable):

1. **Every color must include its opacity variants.** If `#121f46` is used, also document `#121f46/5`, `/10`, `/20`, `/30`, `/70` etc. that appear in the codebase. Format as a sub-list:
   ```
   - **Navy Text** (`#121f46`): Primary heading color
     - `/5` — subtle background tint
     - `/10` — light background
     - `/20` — form input borders
     - `/30` — medium borders
     - `/70` — semi-transparent overlay
   ```

2. **Every spacing value must be in Tailwind classes, not pixels.** Write `px-6` (24px) not "25px padding". The parenthetical pixel value is for reference only. If the project uses an arbitrary value like `p-[25px]`, write it as-is.

3. **Every dimension must match the actual codebase.** If buttons are `h-10` in some places and `h-11` in others, document both with their contexts. Do not simplify to one value.

4. **Every border-radius must use the project's actual Tailwind class.** Write `rounded-full` not `rounded-[28px]`. Write `rounded-lg` not `rounded-[8px]` (unless the codebase literally uses the arbitrary form).

5. **Every shadow must be listed exactly as it appears in code.** Both custom (`shadow-[0px_4px_4px_0px_#f3d4c6]`) AND standard Tailwind (`shadow-md`) if both are used.

6. **Hover states must be documented.** If a color has a hover variant (e.g., `#c58600` for gold hover), include it.

7. **Font weights must match Google Fonts imports.** If only weights 300 and 400 of Newsreader are loaded, don't document weight 500.

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
that can be dropped directly into an ERB template. These must be tested against the
codebase patterns — not generated from Figma pixel values.}

## 10. Figma Source Reference

- **Node**: `{nodeId}` | **File**: `{fileKey}`
- **Raw source**: `figma-sources/node-{nodeId}.md`
- **Screenshot**: captured during extraction (see raw source)
- **Asset URLs**: valid until `{expiry date}` (see raw source for full list)
```

### Step 6 — Self-audit before writing

Before writing the DESIGN.md, run a self-audit checklist:

- [ ] **Every hex color** in the DESIGN.md exists in either the Figma source OR the codebase (grep to verify)
- [ ] **Every opacity variant** actually used in the codebase is documented
- [ ] **Every spacing value** is expressed as a Tailwind class, not raw pixels (unless the project uses arbitrary values)
- [ ] **Every border-radius** uses the project's actual class (`rounded-full`, `rounded-lg`, etc.)
- [ ] **Every shadow** matches exactly what's in the codebase — no invented shadows
- [ ] **Every font weight** is actually loaded in the Google Fonts import
- [ ] **Every hover state** that exists in the codebase is documented
- [ ] **Responsive padding** uses the actual breakpoint pattern from the project
- [ ] **Button/input heights** list ALL variants used, not just one
- [ ] **The component snippets** use classes that actually appear in the project

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

1. **Zero invented values.** Every number in the DESIGN.md must come from either the Figma source code or a grep of the codebase. If a value can't be verified, flag it with `⚠️ unverified` rather than guessing.

2. **Tailwind-native output.** The DESIGN.md speaks Tailwind, not CSS. `px-6` not `padding: 24px`. `rounded-full` not `border-radius: 28px`. `h-10` not `height: 40px`. Exception: arbitrary values like `shadow-[...]` which are already Tailwind syntax.

3. **Figma values are hints, codebase values are truth.** When Figma says 25px and the codebase uses `px-6` (24px), document `px-6` (24px). The Figma is a design intent document; the codebase is the implementation reality. For NEW sections not yet implemented, use Figma values but flag them as `📐 from Figma — not yet in codebase`.

4. **Complete opacity coverage.** If `#121f46` appears at `/5`, `/10`, `/20`, `/30`, `/70` in the codebase, ALL five must be in the DESIGN.md. Missing opacity variants was the #1 accuracy gap in v1.

5. **All shadow types.** Document BOTH custom shadows (`shadow-[...]`) AND standard Tailwind shadows (`shadow-md`, `shadow-lg`) if both appear in the codebase. v1 missed `shadow-md` entirely.

6. **All interactive states.** Hover colors, focus rings, active states — if they exist in the codebase, they must be documented.

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
3. Re-run the codebase audit (Step 3) — values may have changed since last run
4. Identify new colors, components, or patterns not already documented
5. Add them to the appropriate sections
6. Flag any conflicts (e.g., same role mapped to different colors) for user resolution
7. Update the Figma Source Reference section (Section 10)
