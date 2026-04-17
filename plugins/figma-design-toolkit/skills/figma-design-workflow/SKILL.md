---
name: figma-design-workflow
description: Universal methodology for building screens in Figma with any design system. Load this skill before designing screens, layouts, or UI elements in Figma. Covers component-first decision tree, pre-flight audit, variable binding, and common pitfalls — file-agnostic.
---

# figma-design-workflow — Universal Design Methodology

Methodological rules for building screens in Figma, independent of file and design system.
Use together with `figma-console` (MCP mechanics) or `figma-cli` (JSX render).

---

## Decision tree — BEFORE building anything

Every UI element must go through this process:

```
1. Does a component exist in the file that FITS this element?
   → YES → importComponentByKeyAsync(key) + createInstance()

2. Does a component exist that ALMOST fits?
   (different context, similar look, different content)
   → YES → createInstance() + setProperties() for available variant props
            Check componentProperties — if text is not a property, accept
            fixed content or pick another variant

3. No component fits?
   → Build ONLY with design tokens:
      - colors: bind variables (setBoundVariableForPaint), NOT hardcoded hex/RGB
      - typography: values from the system (check existing components for patterns)
      - spacing: multiples of 4px or 8px
      - radius: values from the library (typically 4/6/8/12/16px)
```

> Never start with `createRectangle()` + `createText()` without going through the tree above.
> Building from raw shapes bypasses the entire design system — hardcoded colors, inconsistent typography,
> token changes don't propagate.

---

## Pre-flight — before every new screen

### Step 1: Discover components used in similar screens

The fastest way to find the right components is to read what existing pages use:

```javascript
// Adjust page name to the file
await figma.loadAllPagesAsync();
const page = figma.root.children.find(p => p.name.includes('YOUR_REFERENCE_PAGE'));
await figma.setCurrentPageAsync(page);

const frame = page.children[0]; // or find the right frame
const results = [];
for (const child of frame.children) {
  if (child.type !== 'INSTANCE') continue;
  const mc = await child.getMainComponentAsync();
  if (!mc) continue;
  results.push({
    name: child.name,
    key: mc.key,           // use this key to import
    variant: mc.name,      // full variant name
    size: { w: child.width, h: child.height }
  });
}
return results;
// Result: ready list of components with keys — copy to the next step
```

### Step 2: Discover available color variables

```javascript
const vars = await figma.variables.getLocalVariablesAsync();
const colors = vars
  .filter(v => v.resolvedType === 'COLOR')
  .map(v => ({ name: v.name, id: v.id }));
return { count: colors.length, sample: colors.slice(0, 20) };
// Use names to identify — then getVariableByIdAsync(id) to bind
```

### Step 3: Check file conventions

```javascript
await figma.loadAllPagesAsync();
const pages = figma.root.children.map(p => p.name);
// Find a sample frame and check its width
const refPage = figma.root.children.find(p => !p.name.startsWith('---'));
await figma.setCurrentPageAsync(refPage);
const frame = refPage.children[0];
return {
  pages,
  frameWidth: frame?.width,   // e.g. 1440, 1563, 1280
  frameHeight: frame?.height
};
```

### Step 4: Screenshot variant before using

Before instantiating a component, screenshot it to confirm the right variant:
```
figma_capture_screenshot({ nodeId: 'COMPONENT_NODE_ID' })
```

Check available variants after import:
```javascript
const comp = await figma.importComponentByKeyAsync('KEY');
const inst = comp.createInstance();
const props = Object.entries(inst.componentProperties)
  .map(([k, v]) => ({ key: k, type: v.type, value: v.value }));
inst.remove(); // remove the test instance
return props;
// If VARIANT props exist → you can change via setProperties({ 'Type': 'Primary' })
// If TEXT props exist → you can change via setProperties({ 'Label#123': 'New text' })
// No text props → content is fixed, cannot be changed
```

---

## Binding color variables

When the file has design tokens, **always** bind instead of hardcoded values:

```javascript
// By ID (more efficient — if you know the ID from the file catalog)
const v = await figma.variables.getVariableByIdAsync('VariableID:XXXX:YYYY');

// By name (more readable — when you don't know the ID)
const vars = await figma.variables.getLocalVariablesAsync();
const v = vars.find(x => x.name === 'colors/background/bg-brand');

// Apply to fills
const paint = figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', v
);
node.fills = [paint];

// Apply to strokes
node.strokes = [figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', borderVar
)];
```

---

## Building a new page — step-by-step workflow

```
1. Pre-flight: audit existing pages → list of components with keys
2. Check conventions: frameWidth, page names, hierarchy
3. Create new page (check if it already exists!)
4. Create main frame (matching width)
5. Build in sections — one figma_execute per section:
   a. Import component → createInstance() → append to frame
   b. Position (x=0, y=previous_section_y + height)
   c. figma_capture_screenshot → visual validation
6. Final validation of the entire screen
```

### New page — template

```javascript
await figma.loadAllPagesAsync();
const pageName = 'Checkout Step 1'; // adjust

// Don't duplicate
const existing = figma.root.children.find(p => p.name === pageName);
if (existing) {
  await figma.setCurrentPageAsync(existing);
  return { status: 'page_exists', id: existing.id };
}

const page = figma.createPage();
page.name = pageName;
await figma.setCurrentPageAsync(page);

const W = 1440; // adjust to file conventions
const frame = figma.createFrame();
frame.name = pageName;
frame.resize(W, 100); // height will grow as instances are appended
frame.x = 0;
frame.y = 0;
page.appendChild(frame);

return { pageId: page.id, frameId: frame.id };
```

### Laying out instances

```javascript
// Import and stack from top
const comp = await figma.importComponentByKeyAsync('COMPONENT_KEY');
const inst = comp.createInstance();
frame.appendChild(inst);
inst.x = 0;
inst.y = currentY; // track accumulated height
inst.resize(W, inst.height); // stretch to full width

currentY += inst.height;
return { instanceId: inst.id, newY: currentY };
```

---

## Common mistakes and how to avoid them

| Mistake | Fix |
|------|-----|
| Manual rect+text instead of instances | Always go through the decision tree first |
| Hardcoded `{ r: 0.12, g: 0.17, b: 0.30 }` | `setBoundVariableForPaint` with a token |
| Only navbar+footer as instances, rest manual | EVERY element is an instance |
| Skipping audit of existing pages | `getMainComponentAsync` on a similar page |
| Importing local component by key | Local components: `getNodeByIdAsync(nodeId)` |
| One giant `figma_execute` for the whole screen | Split into sections, screenshot after each |
| Wrong frame width (e.g. 1440 vs 1563) | Check `frame.width` on an existing page |
| Text in instance via direct edit | `setProperties({ 'Label#123': 'Text' })` |
| `getLocalVariables()` instead of async | `getLocalVariablesAsync()` |

---

## How to catalog a new design system

On first contact with a new Figma file, run this script and save the result to memory:

```javascript
await figma.loadAllPagesAsync();
const pages = figma.root.children.map(p => ({ name: p.name, id: p.id }));

// Find first meaningful page (skip pages with '---' prefix)
const refPage = figma.root.children.find(p => !p.name.startsWith('---') && p.children.length > 0);
await figma.setCurrentPageAsync(refPage);

// File conventions
const frame = refPage.children[0];
const frameWidth = frame?.width;

// Color variables
const vars = await figma.variables.getLocalVariablesAsync();
const colorVars = vars.filter(v => v.resolvedType === 'COLOR')
  .map(v => ({ name: v.name, id: v.id }));

// Components from instances on the reference page
const instances = refPage.findAll(n => n.type === 'INSTANCE');
const compMap = {};
for (const inst of instances.slice(0, 30)) {
  const mc = await inst.getMainComponentAsync();
  if (mc && mc.key) compMap[mc.key] = { name: mc.name, key: mc.key };
}

return {
  pages: pages.map(p => p.name),
  frameWidth,
  colorVarCount: colorVars.length,
  colorSample: colorVars.slice(0, 15),
  componentSample: Object.values(compMap)
};
// → Copy the result to memory/ as a reference for this file
```

---

## Relation to other skills

### Desktop Bridge path (preferred — faster, local)
| Skill | Role |
|-------|------|
| `figma-design-toolkit:figma-design-workflow` (this) | **Methodology** — decision tree, pre-flight, code patterns |
| `figma-design-toolkit:figma-console` | **Desktop mechanics** — figma_execute, error recovery, placement |
| `figma-design-toolkit:figma-cli` | **CLI** — JSX render, shadcn tokens, faster than MCP |

Load both when designing screens: `/figma-design-toolkit:figma-design-workflow` + `/figma-design-toolkit:figma-console`

### Cloud path (fallback — when Desktop Bridge unavailable)
| Skill | Role |
|-------|------|
| `figma:figma-use` | Plugin API prerequisite for cloud write ops |
| `figma:figma-generate-design` | Code/description → Figma screen (cloud) |
| `figma:figma-generate-library` | Design system from code (cloud) |
| `figma:figma-implement-design` | Figma design → production code |

> This skill (`figma-design-workflow`) is path-agnostic — the decision tree methodology
> and pre-flight audit apply to both Desktop Bridge and Cloud path.
