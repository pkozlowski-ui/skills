---
name: figma-cli
description: Custom Figma Desktop CLI (`figma-ds-cli`) at /Users/piotr/figma-cli. Use when creating components with JSX syntax, adding design tokens (shadcn/tailwind), building pre-made UI blocks, or using var: variable binding syntax. Faster than MCP because it connects directly to Figma Desktop via daemon.
---

# figma-cli — figma-ds-cli

CLI sterujące Figma Desktop bezpośrednio. Szybsze od MCP, działa przez lokalny daemon.

**Ścieżka:** `/Users/piotr/figma-cli`
**Komenda bazowa:** `node /Users/piotr/figma-cli/src/index.js <command>`

---

## Pre-flight checklist (przed każdą sesją)

- [ ] Figma Desktop jest otwarta
- [ ] Daemon działa: `node src/index.js daemon status` — jeśli nie, uruchom `node src/index.js connect`
- [ ] Sprawdź co jest na canvasie przed tworzeniem: `node src/index.js canvas info`
- [ ] Nigdy nie usuwaj istniejących node'ów

---

## Kiedy używać figma-cli vs figma-console

| Zadanie | Narzędzie |
|---------|-----------|
| Tworzenie komponentów via JSX | **figma-cli** (`render`) |
| Design tokens shadcn/tailwind | **figma-cli** (`tokens preset shadcn`) |
| Gotowe bloki UI (dashboard etc.) | **figma-cli** (`blocks create`) |
| Kompleksowe warianty komponentów | figma-console (`figma_execute`) |
| Bindowanie zmiennych do istniejących node'ów | figma-console lub figma-cli (`set fill "var:x"`) |
| Operacje na wielu stronach | figma-console |

---

## Kluczowe komendy

### Połączenie
```bash
node src/index.js connect          # Yolo mode (zalecane) — patch + daemon
node src/index.js connect --safe   # Safe mode — wymaga ręcznego uruchomienia pluginu
node src/index.js daemon status    # Sprawdź czy daemon działa
```

### Tworzenie (render)
```bash
# Prosty frame
node src/index.js render '<Frame name="Card" w={320} flex="col" bg="#18181b" rounded={12} p={24} gap={12}>
  <Text size={18} weight="bold" color="#fff" w="fill">Tytuł</Text>
  <Text size={14} color="#a1a1aa" w="fill">Opis</Text>
</Frame>'

# Z design tokens (shadcn)
node src/index.js render '<Frame bg="var:card" stroke="var:border" rounded={12} p={24}>
  <Text color="var:foreground" size={18} w="fill">Tytuł</Text>
</Frame>'
```

### Tokeny
```bash
node src/index.js tokens preset shadcn   # 244 primitives + 32 semantic (Light/Dark)
node src/index.js tokens tailwind        # 242 primitive colors
node src/index.js var list               # Pokaż istniejące zmienne
node src/index.js var visualize          # Swatch na canvasie
```

### Bloki UI
```bash
node src/index.js blocks list            # Dostępne bloki
node src/index.js blocks create dashboard-01  # Dashboard z sidebar, stats, chart
```

### Weryfikacja (OBOWIĄZKOWE po każdym create)
```bash
node src/index.js verify                 # Screenshot zaznaczonego node'a
node src/index.js verify "123:456"       # Screenshot konkretnego node'a
```

### Konwersja do komponentu
```bash
node src/index.js node to-component "NODE_ID"
```

---

## JSX syntax — najważniejsze zasady

### Tekst MUSI mieć `w="fill"` żeby zawijał się
```jsx
// ŹLE — tekst się tnie
<Text size={16} color="#fff">Długi tytuł który się utrnie</Text>

// DOBRZE — tekst się zawija
<Text size={16} color="#fff" w="fill">Długi tytuł który się zawija</Text>
```
**Reguła:** KAŻDY element `<Text>` w auto-layout frame musi mieć `w="fill"`. Bez wyjątków.

### Kolejność atrybutów layoutu ma znaczenie
```jsx
// Poprawny frame z auto-layout
<Frame w={320} flex="col" gap={16} p={24}>
  ...
</Frame>

// justify="between" NIE DZIAŁA — użyj grow={1} jako spacer
<Frame flex="row" items="center">
  <Frame>Logo</Frame>
  <Frame grow={1} />   {/* spacer */}
  <Frame>Przyciski</Frame>
</Frame>
```

### var: syntax dla zmiennych (shadcn)
```jsx
bg="var:card"              // fill tła
stroke="var:border"        // obrys
color="var:foreground"     // kolor tekstu (w <Text>)
bg="var:primary"           // primary color
bg="var:muted"             // muted background
```

### Ikony (Lucide, prawdziwe SVG)
```jsx
<Icon name="lucide:settings" size={20} color="#fff" />
<Icon name="lucide:chevron-right" size={16} color="var:muted-foreground" />
```

---

## Typowe błędy (silently fail — brak błędu!)

| Napisałeś | Powinno być |
|-----------|-------------|
| `layout="horizontal"` | `flex="row"` |
| `padding={24}` | `p={24}` |
| `fill="#fff"` | `bg="#fff"` |
| `cornerRadius={12}` | `rounded={12}` |
| `fontSize={18}` | `size={18}` |
| `fontWeight="bold"` | `weight="bold"` |
| `justify="between"` | `grow={1}` spacer |

---

## Safe Mode — krytyczna różnica

W Safe Mode: **`render-batch` NIE renderuje tekstu poprawnie.**
Dla komponentów z tekstem w Safe Mode, używaj `eval` z natywnym Figma Plugin API.

```bash
node src/index.js eval "(async () => {
  await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
  const frame = figma.createFrame();
  // ... reszta kodu
  return { id: frame.id };
})()"
```

---

## Workflow: tworzenie komponentu krok po kroku

1. **Sprawdź canvas** — `node src/index.js canvas info` (znajdź wolne miejsce)
2. **Sprawdź zmienne** — `node src/index.js var list` (czy są tokeny?)
3. **Render** — `node src/index.js render '<Frame ...>'`
4. **Verify** — `node src/index.js verify "NODE_ID"` (ZAWSZE)
5. **Konwertuj** — `node src/index.js node to-component "NODE_ID"` (jeśli potrzeba)
6. **Weryfikacja wizualna** — sprawdź screenshot, popraw jeśli coś nie gra

---

## Dokumentacja pełna

Pełna dokumentacja CLI: `/Users/piotr/figma-cli/CLAUDE.md`
Reference komendy: `/Users/piotr/figma-cli/REFERENCE.md` (jeśli istnieje)
