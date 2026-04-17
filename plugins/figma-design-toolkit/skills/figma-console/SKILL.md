---
name: figma-console
description: Prerequisite for figma-console MCP tools (figma_execute, figma_capture_screenshot, figma_search_components etc.). Load this skill before any figma_execute call. Use figma-console when you need full Plugin API access via Figma Desktop — complex component creation, variant management, variable binding, programmatic design operations that figma-cli's JSX syntax can't handle.
---

# figma-console — Desktop Bridge MCP

Controls Figma Desktop via WebSocket (Desktop Bridge plugin). Requires Figma to be open with the plugin running.

---

## When figma-console vs figma-cli vs use_figma

| Task | Tool |
|---------|-----------|
| JSX render, shadcn tokens, UI blocks | **figma-cli** |
| Complex Plugin API, variants, variable binding | **figma-console** (`figma_execute`) |
| Read design context, screenshot for code | `mcp__figma-desktop__` or `mcp__figma__` |
| Fallback when desktop unavailable | `use_figma` (claude.ai Figma) |

---

## Component-first — primary rule for design tasks

**BEFORE building any UI element**, go through this decision tree:

```
Does a component exist in the file that FITS this element?
  → YES: importComponentByKeyAsync(key) + createInstance()

Does a component exist that ALMOST fits (different context, similar look)?
  → YES: createInstance() + setProperties() for variant props
         (if text is not a property → accept fixed content or pick another variant)

No component fits?
  → Build from atoms, BUT only with design tokens:
     - colors: setBoundVariableForPaint (NOT hardcoded RGB)
     - typography: family/size/weight consistent with the system
     - spacing: multiples of 4px or 8px
     - radius: values from the library (6/8/12/16px)
```

> Rule: every UI element is a component instance — buttons, inputs, cards, navigation, EVERYTHING.
> Building with `createRectangle()` + `createText()` bypasses the entire design system.

### Step 0 (before every design task): audit existing pages

Before building, check what similar existing pages in the file use — that's your ready-made "recipe":

```javascript
await figma.loadAllPagesAsync();
const page = figma.root.children.find(p => p.name.includes('Cart')); // adjust name
await figma.setCurrentPageAsync(page);
const frame = page.children[0];
const results = [];
for (const inst of frame.children.filter(n => n.type === 'INSTANCE')) {
  const mc = await inst.getMainComponentAsync();
  results.push({ name: inst.name, key: mc?.key, variant: mc?.name });
}
return results;
// → list of components with keys, ready to use
```

### Binding color variables (don't hardcode RGB)

When the file has design tokens (color variables), ALWAYS bind them:

```javascript
// Get variable by ID (ID is stable — doesn't change between sessions)
const v = await figma.variables.getVariableByIdAsync('VariableID:...');
const paint = figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', v
);
node.fills = [paint];

// Or get all variables and find by name:
const vars = await figma.variables.getLocalVariablesAsync();
const colorMap = {};
vars.filter(v => v.resolvedType === 'COLOR').forEach(v => { colorMap[v.name] = v; });
// colorMap['colors/background/bg-brand'], colorMap['colors/text/text-heading'] etc.
```

### Screenshot variant before using

Before `createInstance()`, screenshot the component to confirm it's the right variant:
```
figma_capture_screenshot({ nodeId: 'COMPONENT_NODE_ID' })
```

---

## Pre-flight checklist (each session)

```
[ ] figma_get_status — check connection
[ ] figma_list_open_files — which file is active?
[ ] figma_search_components — refresh node IDs (stale between sessions!)
[ ] figma_capture_screenshot — screenshot the page before making changes
[ ] Find free space, don't overlap existing content
```

**Node IDs are stale between conversations** — never use IDs from a previous session without re-searching.

---

## figma_execute — critical rules

Many rules are the same as in `use_figma`. Below are those that differ or are worth highlighting.

### 1. JS code — format

```javascript
// GOOD — plain JS with top-level await and return
const frame = figma.createFrame();
frame.resize(200, 200);
return { id: frame.id, name: frame.name };

// BAD — don't wrap in async IIFE (it's auto-wrapped)
(async () => { ... })()
```

### 2. Return is the only output channel

- `console.log()` → invisible, don't use
- `figma.notify()` → throws "not implemented", don't use
- Always `return` with data: `{ createdNodeIds: [...], status: "ok" }`

### 3. Instances — text must go through figma_set_instance_properties

```javascript
// BAD — FAILS SILENTLY (no error, but text doesn't change!)
const inst = figma.createInstance(component);
inst.findOne(n => n.type === 'TEXT').characters = 'New text';

// GOOD — via instance properties
// First check available properties:
return Object.keys(instance.componentProperties);
// Then set:
instance.setProperties({ 'Label#2:0': 'New text' });
```

Or use the `figma_set_instance_properties` tool directly.

### 4. Check resultAnalysis.warning

figma_execute returns `resultAnalysis` — always check `resultAnalysis.warning`:
- Empty arrays → operation may be a silent failure
- Null returns → node doesn't exist or wrong ID

### 5. Placement — always inside Section/Frame, never on blank canvas

```javascript
// GOOD — find or create a Section
let section = figma.currentPage.findOne(n => n.type === 'SECTION' && n.name === 'Components');
if (!section) {
  section = figma.createSection();
  section.name = 'Components';
  // Position below existing content
  const maxY = Math.max(0, ...figma.currentPage.children.map(n => n.y + n.height)) + 100;
  section.y = maxY;
}
// Create inside the section
const frame = figma.createFrame();
section.appendChild(frame);
```

### 6. Cleanup on error/retry

If the script failed and left partial artifacts — **remove them before retrying**:

```javascript
// Find and remove empty/orphaned frames
const orphans = figma.currentPage.findAll(n => n.name === 'Temp' || n.children?.length === 0);
orphans.forEach(n => n.remove());
```

Never build on a broken foundation.

### 7. Pages — don't duplicate

```javascript
// ALWAYS check if page exists before creating
await figma.loadAllPagesAsync();
const existing = figma.root.children.find(p => p.name === 'Design System');
if (existing) {
  await figma.setCurrentPageAsync(existing);
} else {
  const newPage = figma.createPage();
  newPage.name = 'Design System';
  await figma.setCurrentPageAsync(newPage);
}
```

### 8. Colors — range 0–1 (not 0–255)

```javascript
{ r: 1, g: 0, b: 0 }      // red ✓
{ r: 255, g: 0, b: 0 }    // BAD ✗
```

### 9. FILL after appendChild

```javascript
parent.appendChild(child);
child.layoutSizingHorizontal = 'FILL'; // AFTER append, not before
```

### 10. Font BEFORE text operations

```javascript
await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
const text = figma.createText();
text.characters = 'Hello'; // only after loadFontAsync
```

---

## Visual Validation Loop (required)

After every operation that creates or modifies visual elements:

```
1. figma_capture_screenshot(nodeId: "NODE_ID")   ← prefer over figma_take_screenshot
2. Check: alignment, spacing, proportions, visual balance
3. Iterate if something looks off (max 3 times)
4. Final figma_capture_screenshot for confirmation
```

**figma_capture_screenshot vs figma_take_screenshot:**
- `figma_capture_screenshot` — via plugin runtime, sees changes **immediately**, preferred for validation
- `figma_take_screenshot` — via REST API (cloud), may not reflect latest state, good for broader page views

---

## Session Management

### Check connection
```
figma_get_status
```
If status not OK → `figma_reconnect`

### Multiple files
```
figma_list_open_files    → which files have Desktop Bridge plugin
figma_navigate           → switch active file
```

### Discover file structure
```javascript
// Start with verbosity='summary', depth=1 — not 'full' (consumes tokens!)
figma_get_file_data({ verbosity: 'summary', depth: 1 })
```

### Discover components (at session start)
```
figma_search_components({ query: 'Button', limit: 10 })
figma_search_components({ category: 'Card', limit: 10 })
```

### Check user selection
```
figma_get_selection()
figma_get_selection({ verbose: true })  // with fills/strokes/styles
```

---

## Incremental workflow — how to avoid bugs

1. **Inspect first** — screenshot + `figma_get_file_data(summary)` before creating anything
2. **One task per `figma_execute`** — don't build an entire screen in one call
3. **Return node IDs from every call** — you'll need them in subsequent steps
4. **Validate after each step** — `figma_capture_screenshot` after creating components
5. **Fix before continuing** — don't build on a broken state

### Suggested order for complex tasks

```
Step 1: Inspect — figma_get_file_data + figma_capture_screenshot (what's already there?)
Step 2: figma_search_components (what components are available?)
Step 3: Create Section/parent frame → return { sectionId }
Step 4: Create tokens/variables → validate figma_capture_screenshot
Step 5: Create components (one per call) → validate
Step 6: Compose layout from instances → validate
Step 7: Final validation of the whole screen
```

---

## Error Recovery

`figma_execute` is atomic — if the script throws, no changes are applied.

**On error:**
1. STOP — don't retry immediately
2. Read the error message carefully
3. If unclear — `figma_capture_screenshot` + `figma_get_file_data` to check state
4. Fix the script
5. Retry (safe — nothing was changed)

**On success but looks wrong:**
1. `figma_capture_screenshot(nodeId)` — not the whole page
2. Write a targeted fix script — don't rebuild everything from scratch

### Common errors and fixes

| Error | Cause | Fix |
|-------|-----------|-----|
| "not implemented" | `figma.notify()` | Remove it, use `return` |
| Text doesn't change, no error | Direct edit on instance | Use `setProperties()` or `figma_set_instance_properties` |
| `"Setting figma.currentPage is not supported"` | Sync setter | `await figma.setCurrentPageAsync(page)` |
| Empty array in result | Silent fail — node doesn't exist | Check ID, check page |
| FILL before appendChild | Crash on layoutSizing | Move FILL after appendChild |
| Font error | Missing loadFontAsync | Add `await figma.loadFontAsync(...)` before text ops |

---

## Pre-flight checklist (before figma_execute)

- [ ] `return` used as output (not console.log, not figma.notify)
- [ ] Code is NOT wrapped in async IIFE
- [ ] Colors in range 0–1
- [ ] `layoutSizingH/V = 'FILL'` set AFTER appendChild
- [ ] `loadFontAsync()` called BEFORE text operations
- [ ] Page switch via `await figma.setCurrentPageAsync(page)`
- [ ] Instances: text via `setProperties()`, not direct edit
- [ ] New nodes created inside Section/Frame (not on bare canvas)
- [ ] Node IDs returned from all created/modified elements
- [ ] Page duplication checked before creating

---

## Relation to figma-use skill

Many Plugin API rules are identical to `figma-use`. When you need detailed patterns:
- Components and variants → `/figma:figma-use` references/component-patterns.md
- Variables and binding → `/figma:figma-use` references/variable-patterns.md
- Text styles → `/figma:figma-use` references/text-style-patterns.md
- Full gotchas → `/figma:figma-use` references/gotchas.md

Difference: those skills use `use_figma`, this skill uses `figma_execute`. Plugin API rules are identical.
