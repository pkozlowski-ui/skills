---
name: figma-cli
description: Custom Figma Desktop CLI (`figma-ds-cli`) at /Users/piotr/figma-cli. Use when creating components with JSX syntax, adding design tokens (shadcn/tailwind), building pre-made UI blocks, or using var: variable binding syntax. Faster than MCP because it connects directly to Figma Desktop via daemon.
---

# figma-cli — figma-ds-cli

CLI that controls Figma Desktop directly. Faster than MCP, runs via a local daemon.

**Path:** `/Users/piotr/figma-cli`
**Base command:** `node /Users/piotr/figma-cli/src/index.js <command>`

---

## Pre-flight checklist (before each session)

- [ ] Figma Desktop is open
- [ ] Daemon is running: `node src/index.js daemon status` — if not, run `node src/index.js connect`
- [ ] Check what's on the canvas before creating: `node src/index.js canvas info`
- [ ] Never delete existing nodes

---

## When to use figma-cli vs figma-console

| Task | Tool |
|---------|-----------|
| Creating components via JSX | **figma-cli** (`render`) |
| Design tokens shadcn/tailwind | **figma-cli** (`tokens preset shadcn`) |
| Pre-made UI blocks (dashboard etc.) | **figma-cli** (`blocks create`) |
| Complex component variants | figma-console (`figma_execute`) |
| Binding variables to existing nodes | figma-console or figma-cli (`set fill "var:x"`) |
| Operations across multiple pages | figma-console |

---

## Key commands

### Connection
```bash
node src/index.js connect          # Yolo mode (recommended) — patch + daemon
node src/index.js connect --safe   # Safe mode — requires manual plugin launch
node src/index.js daemon status    # Check if daemon is running
```

### Creating (render)
```bash
# Simple frame
node src/index.js render '<Frame name="Card" w={320} flex="col" bg="#18181b" rounded={12} p={24} gap={12}>
  <Text size={18} weight="bold" color="#fff" w="fill">Title</Text>
  <Text size={14} color="#a1a1aa" w="fill">Description</Text>
</Frame>'

# With design tokens (shadcn)
node src/index.js render '<Frame bg="var:card" stroke="var:border" rounded={12} p={24}>
  <Text color="var:foreground" size={18} w="fill">Title</Text>
</Frame>'
```

### Tokens
```bash
node src/index.js tokens preset shadcn   # 244 primitives + 32 semantic (Light/Dark)
node src/index.js tokens tailwind        # 242 primitive colors
node src/index.js var list               # Show existing variables
node src/index.js var visualize          # Swatch on canvas
```

### UI Blocks
```bash
node src/index.js blocks list            # Available blocks
node src/index.js blocks create dashboard-01  # Dashboard with sidebar, stats, chart
```

### Verification (REQUIRED after every create)
```bash
node src/index.js verify                 # Screenshot of selected node
node src/index.js verify "123:456"       # Screenshot of specific node
```

### Convert to component
```bash
node src/index.js node to-component "NODE_ID"
```

---

## JSX syntax — key rules

### Text MUST have `w="fill"` to wrap
```jsx
// BAD — text gets clipped
<Text size={16} color="#fff">Long title that gets cut off</Text>

// GOOD — text wraps properly
<Text size={16} color="#fff" w="fill">Long title that wraps properly</Text>
```
**Rule:** EVERY `<Text>` element inside an auto-layout frame must have `w="fill"`. No exceptions.

### Layout attribute order matters
```jsx
// Correct frame with auto-layout
<Frame w={320} flex="col" gap={16} p={24}>
  ...
</Frame>

// justify="between" DOES NOT WORK — use grow={1} as spacer
<Frame flex="row" items="center">
  <Frame>Logo</Frame>
  <Frame grow={1} />   {/* spacer */}
  <Frame>Buttons</Frame>
</Frame>
```

### var: syntax for variables (shadcn)
```jsx
bg="var:card"              // background fill
stroke="var:border"        // stroke
color="var:foreground"     // text color (in <Text>)
bg="var:primary"           // primary color
bg="var:muted"             // muted background
```

### Icons (Lucide, real SVG)
```jsx
<Icon name="lucide:settings" size={20} color="#fff" />
<Icon name="lucide:chevron-right" size={16} color="var:muted-foreground" />
```

---

## Common errors (silently fail — no error thrown!)

| You wrote | Should be |
|-----------|-------------|
| `layout="horizontal"` | `flex="row"` |
| `padding={24}` | `p={24}` |
| `fill="#fff"` | `bg="#fff"` |
| `cornerRadius={12}` | `rounded={12}` |
| `fontSize={18}` | `size={18}` |
| `fontWeight="bold"` | `weight="bold"` |
| `justify="between"` | `grow={1}` spacer |

---

## Safe Mode — critical difference

In Safe Mode: **`render-batch` does NOT render text correctly.**
For components with text in Safe Mode, use `eval` with native Figma Plugin API.

```bash
node src/index.js eval "(async () => {
  await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
  const frame = figma.createFrame();
  // ... rest of the code
  return { id: frame.id };
})()"
```

---

## Workflow: creating a component step by step

1. **Check canvas** — `node src/index.js canvas info` (find free space)
2. **Check variables** — `node src/index.js var list` (are tokens available?)
3. **Render** — `node src/index.js render '<Frame ...>'`
4. **Verify** — `node src/index.js verify "NODE_ID"` (ALWAYS)
5. **Convert** — `node src/index.js node to-component "NODE_ID"` (if needed)
6. **Visual validation** — check screenshot, fix if anything looks off

---

## Full documentation

Full CLI docs: `/Users/piotr/figma-cli/CLAUDE.md`
Command reference: `/Users/piotr/figma-cli/REFERENCE.md` (if exists)
