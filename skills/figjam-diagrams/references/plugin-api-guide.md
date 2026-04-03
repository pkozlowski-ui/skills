# FigJam Plugin API — Complete Reference

This file contains everything confirmed to work (and not work) in FigJam's Plugin API,
based on live testing. Trust this over general Figma documentation — FigJam is a subset.

---

## What exists vs. what doesn't

### ✅ Available in FigJam

```javascript
figma.createSection()          // Container / grouping
figma.createShapeWithText()    // Shapes with embedded text (the universal building block)
figma.createConnector()        // Arrow / connector lines
figma.createSticky()           // Sticky notes
figma.loadFontAsync({ family, style })  // Font loading (required before text)
node.resize(w, h)              // Resize any node
figma.currentPage              // Current page reference
figma.currentPage.findAll(fn)  // Find nodes by predicate
figma.getNodeById(id)          // Get node by ID string
figma.viewport.scrollAndZoomIntoView([...nodes])
figma.editorType               // Returns "figjam" in FigJam
```

### ❌ Does NOT exist in FigJam (will throw "TypeError: not a function")

```javascript
figma.createText()             // ❌ Use createShapeWithText() with empty fills instead
figma.createRectangle()        // ❌ Use createShapeWithText() with SQUARE shape
figma.createEllipse()          // ❌ Use createShapeWithText() with ELLIPSE shape
figma.createFrame()            // ❌ Use createSection() instead
figma.createComponent()        // ❌ Not available
node.resizeWithoutConstraints()// ❌ Use node.resize(w, h) instead
```

---

## ShapeWithTextNode

### Creating

```javascript
const shape = figma.createShapeWithText();
```

Default size after creation: **176 × 176 px**.

### Shape types (complete confirmed list from API)

```
"SQUARE"               rectangle (default)
"ELLIPSE"              circle / oval
"ROUNDED_RECTANGLE"    rectangle with rounded corners
"DIAMOND"              rotated square / rhombus
"HEXAGON"              hexagon
"PENTAGON"             pentagon
"OCTAGON"              octagon
"CHEVRON"              chevron / arrow-like
"TRIANGLE_UP"          upward triangle
"TRIANGLE_DOWN"        downward triangle
"PARALLELOGRAM_RIGHT"  leaning right
"PARALLELOGRAM_LEFT"   leaning left
"TRAPEZOID"            trapezoid
"STAR"                 star shape
"PLUS"                 plus / cross shape
"ARROW_LEFT"           arrow pointing left
"ARROW_RIGHT"          arrow pointing right
"SHIELD"               shield
"SPEECH_BUBBLE"        speech bubble
"PREDEFINED_PROCESS"   process box with dividers
"MANUAL_INPUT"         manual input (slanted top)
"DOCUMENT_SINGLE"      single document
"DOCUMENT_MULTIPLE"    stack of documents
"INTERNAL_STORAGE"     internal storage
"ENG_DATABASE"         database cylinder (engineering)
"ENG_QUEUE"            queue (engineering)
"ENG_FILE"             file (engineering)
"ENG_FOLDER"           folder (engineering)
"SUMMING_JUNCTION"     summing junction (circle with X)
"OR"                   OR gate (circle with +)
```

⚠ There is NO "USER", "PERSON", "ACTOR", "HUMAN", or "CYLINDER" shape type.
Use `importComponentByKeyAsync` (see below) to place FigJam library shapes like User.

### Resizing

```javascript
shape.resize(176, 88);         // ✅ correct
shape.resizeWithoutConstraints(176, 88);  // ❌ doesn't exist
```

Note: hexagons look best at **176 × 177**. Standard step boxes at **176 × 88**.

### Fills and strokes

```javascript
// Solid colour fill
shape.fills = [{ type: "SOLID", color: { r: 0.29, g: 0.62, b: 1.0 } }];

// Transparent (no background)
shape.fills = [];

// No border
shape.strokes = [];

// Stroke with colour and weight
shape.strokes = [{ type: "SOLID", color: { r: 0, g: 0, b: 0 } }];
shape.strokeWeight = 1;
```

### TextSublayer (shape.text)

All properties confirmed working:

```javascript
shape.text.characters          = "Label text";       // string, supports \n
shape.text.fontSize            = 11;                 // number (px)
shape.text.fontName            = { family: "Inter", style: "Regular" };
shape.text.fills               = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
shape.text.textAlignHorizontal = "CENTER";           // "LEFT" | "CENTER" | "RIGHT"
```

**Font must be loaded before setting characters or fontName:**
```javascript
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });
// then set text...
```

### Positioning

Always `parent.appendChild(shape)` **before** setting `x` and `y`.
Coordinates are **relative to the parent section's top-left corner**.

```javascript
parent.appendChild(shape);
shape.x = 265;   // distance from left edge of section
shape.y = 200;   // distance from top edge of section
```

### Height adjustment for hexagons

Hexagons are taller (177px vs 88px for rectangles). To visually centre them with adjacent
rectangles in a horizontal flow:

```javascript
// Rectangle step at y=200, height=88 → centre at y=244
// Hexagon at y=200-44=156, height=177 → centre at y=244.5  ✓
shape.y = isHex ? stepY - 44 : stepY;
```

---

## ConnectorNode

### Creating

```javascript
const conn = figma.createConnector();
parent.appendChild(conn);       // append FIRST
conn.connectorStart = { endpointNodeId: nodeA.id, magnet: "AUTO" };
conn.connectorEnd   = { endpointNodeId: nodeB.id, magnet: "AUTO" };
conn.connectorEndStrokeCap   = "ARROW_LINES";  // arrow head at end
conn.connectorStartStrokeCap = "NONE";          // no arrow at start
```

### Magnet values

`"AUTO"` lets FigJam pick the best attachment point. Alternatives: `"TOP"`, `"BOTTOM"`, `"LEFT"`, `"RIGHT"`.

### Stroke cap values

`"NONE"` | `"ARROW_LINES"` | `"ARROW_EQUILATERAL"` | `"CIRCLE_FILLED"` | `"DIAMOND_FILLED"` | `"TRIANGLE_FILLED"`

### Styling connectors

```javascript
conn.strokes = [{ type: "SOLID", color: { r: 0.7, g: 0.7, b: 0.7 } }];
conn.strokeWeight = 2;
```

---

## SectionNode

### Creating

```javascript
const section = figma.createSection();
section.name = "Section title";   // shown as label in FigJam
section.x    = 1000;              // page-level coordinates
section.y    = 2000;
```

### Behaviour

- Sections **auto-expand** to fit their children — no need to set width/height
- Children's `x` and `y` are **relative to the section origin**
- Sections can be nested
- The section's displayed title is `section.name`

### Finding existing sections

```javascript
const sections = figma.currentPage.findAll(n =>
  n.type === "SECTION" && n.name.startsWith("Option ")
);
```

---

## Finding free space on the board

```javascript
// Calculate the bottom edge of all existing content
const allNodes = figma.currentPage.children;
let maxY = 0;
for (const n of allNodes) {
  const bottom = n.y + (n.height || 0);
  if (bottom > maxY) maxY = bottom;
}
const newContentY = maxY + 400; // 400px gap below existing content
```

---

---

## FigJam Library Components (importComponentByKeyAsync)

For shapes that don't exist as `createShapeWithText()` types, use `figma.importComponentByKeyAsync(key)`.
This is how you access FigJam's built-in Advanced Shapes library.

### Confirmed component keys

| Shape | Key | Size | Notes |
|---|---|---|---|
| **User** (Shapes > Advanced > User) | `b3abe69f2adf828533836dc0e44328a9f74c706c` | 88×88 | Label via `"Text Label [PS_TEXT]"` text node |

### Usage pattern

```javascript
// mkActor must be async because importComponentByKeyAsync is async
async function mkActor(parent, label, x, y) {
  const comp = await figma.importComponentByKeyAsync("b3abe69f2adf828533836dc0e44328a9f74c706c");
  const inst = comp.createInstance();
  inst.resize(88, 88);
  parent.appendChild(inst);
  inst.x = x; inst.y = y;
  // IMPORTANT: target by name, not by index — other text nodes exist inside the component
  const labelNode = inst.findOne(n => n.type === "TEXT" && n.name === "Text Label [PS_TEXT]");
  if (labelNode) labelNode.characters = label;
  return inst;
}
```

### Calling async functions in use_figma

Since `use_figma` scripts run in an async context, you can `await` at any level:
```javascript
await mkActor(section, "Admin", 51, 240);
await mkActor(section, "Reseller", 200, 240);
```

---

## Performance tips

- All creation happens in a single synchronous frame — no need for batching tricks
- Avoid calling `figma.currentPage.findAll()` with expensive predicates inside loops
- `figma.viewport.scrollAndZoomIntoView([...nodes])` at the end of the script centres the view

---

## Diagnostic: identifying API errors

If you get `TypeError: not a function` with no line number:
1. Check for any call to `createText`, `createRectangle`, `createEllipse`, `createFrame` — remove them
2. Check for `resizeWithoutConstraints` — replace with `resize`
3. Verify fonts are loaded before any `text.characters` assignment
4. Verify `parent.appendChild(node)` happens before `node.x = ...`

The most common mistake is calling a function that doesn't exist in FigJam.
Always use the verified helper library from SKILL.md rather than writing raw Plugin API calls.
