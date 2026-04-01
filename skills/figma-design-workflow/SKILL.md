---
name: figma-design-workflow
description: Universal methodology for building screens in Figma with any design system. Load this skill before designing screens, layouts, or UI elements in Figma. Covers component-first decision tree, pre-flight audit, variable binding, and common pitfalls — file-agnostic.
---

# figma-design-workflow — Universal Design Methodology

Zasady metodologiczne dla budowania ekranów w Figma, niezależne od pliku i design systemu.
Używaj razem z `figma-console` (mechanika MCP) lub `figma-cli` (JSX render).

---

## Decision tree — ZANIM zbudujesz cokolwiek

Każdy element UI musi przejść przez ten proces:

```
1. Czy w pliku istnieje komponent który PASUJE do tego elementu?
   → TAK → importComponentByKeyAsync(key) + createInstance()

2. Czy istnieje komponent który PRAWIE pasuje?
   (inny kontekst, zbliżony wygląd, inna treść)
   → TAK → createInstance() + setProperties() dla dostępnych variant props
            Sprawdź componentProperties — jeśli tekst nie jest property, zaakceptuj
            fixed content lub wybierz inny wariant

3. Żaden komponent nie pasuje?
   → Buduj WYŁĄCZNIE z design tokenów:
      - kolory: binduj zmienne (setBoundVariableForPaint), NIE hardcoded hex/RGB
      - typografia: wartości z systemu (sprawdź istniejące komponenty dla wzorców)
      - spacing: wielokrotności 4px lub 8px
      - radius: wartości z biblioteki (typowo 4/6/8/12/16px)
```

> Nigdy nie zacznij od `createRectangle()` + `createText()` bez przejścia przez powyższy tree.
> Budowanie z surowych kształtów omija cały design system — kolory hardcoded, typografia niespójna,
> zmiany w tokenach nie propagują się.

---

## Pre-flight — przed każdym nowym ekranem

### Krok 0: Font family (przed JAKĄKOLWIEK operacją na tekście)

Nigdy nie zakładaj "Inter". Odczytaj font z istniejącego węzła tekstowego w pliku:

```javascript
const sample = figma.currentPage.findOne(n => n.type === 'TEXT')
  ?? figma.root.children.flatMap(p => p.findAll(n => n.type === 'TEXT'))[0];
const fontFamily = sample?.fontName?.family ?? 'Unknown';
// Załaduj warianty których planujesz używać:
await figma.loadFontAsync({ family: fontFamily, style: 'Regular' });
await figma.loadFontAsync({ family: fontFamily, style: 'Medium' });
await figma.loadFontAsync({ family: fontFamily, style: 'SemiBold' });
return { fontFamily }; // sprawdź przed kontynuowaniem
```

### Krok 1: Odkryj komponenty używane w podobnych ekranach

Najszybszy sposób znaleźć właściwe komponenty to przeczytać co używają istniejące strony:

```javascript
// Dostosuj nazwę strony do pliku
await figma.loadAllPagesAsync();
const page = figma.root.children.find(p => p.name.includes('YOUR_REFERENCE_PAGE'));
await figma.setCurrentPageAsync(page);

const frame = page.children[0]; // lub znajdź właściwy frame
const results = [];
for (const child of frame.children) {
  if (child.type !== 'INSTANCE') continue;
  const mc = await child.getMainComponentAsync();
  if (!mc) continue;
  results.push({
    name: child.name,
    key: mc.key,           // użyj tego klucza do importu
    variant: mc.name,      // pełna nazwa wariantu
    size: { w: child.width, h: child.height }
  });
}
return results;
// Wynik: gotowa lista komponentów z kluczami — kopiuj do następnego kroku
```

### Krok 2: Odkryj dostępne zmienne kolorów

```javascript
const vars = await figma.variables.getLocalVariablesAsync();
const colors = vars
  .filter(v => v.resolvedType === 'COLOR')
  .map(v => ({ name: v.name, id: v.id }));
return { count: colors.length, sample: colors.slice(0, 20) };
// Użyj nazw do identyfikacji — potem getVariableByIdAsync(id) do bindowania
```

### Krok 3: Sprawdź konwencje pliku

```javascript
await figma.loadAllPagesAsync();
const pages = figma.root.children.map(p => p.name);
// Znajdź przykładowy frame i sprawdź jego szerokość
const refPage = figma.root.children.find(p => !p.name.startsWith('---'));
await figma.setCurrentPageAsync(refPage);
const frame = refPage.children[0];
return {
  pages,
  frameWidth: frame?.width,   // np. 1440, 1563, 1280
  frameHeight: frame?.height
};
```

### Krok 4: Screenshot wariantu przed użyciem

Zanim zinstancjonujesz komponent, zrób screenshot żeby potwierdzić właściwy wariant:
```
figma_capture_screenshot({ nodeId: 'COMPONENT_NODE_ID' })
```

Sprawdź dostępne warianty po imporcie:
```javascript
const comp = await figma.importComponentByKeyAsync('KEY');
const inst = comp.createInstance();
const props = Object.entries(inst.componentProperties)
  .map(([k, v]) => ({ key: k, type: v.type, value: v.value }));
inst.remove(); // usuń testową instancję
return props;
// Jeśli są VARIANT props → możesz zmienić przez setProperties({ 'Type': 'Primary' })
// Jeśli są TEXT props → możesz zmienić przez setProperties({ 'Label#123': 'Nowy tekst' })
// Brak text props → treść fixed, nie można zmienić
```

---

## Nazwy warstw i struktura layerów — obowiązkowe standardy

### Nazewnictwo — zawsze opisowe

Każdy node stworzony przez `figma_execute` musi mieć nazwę odzwierciedlającą jego rolę w hierarchii.

```
✓  frame.name = 'Slot 9:00am'
✓  frame.name = 'Attendees Row'
✓  frame.name = 'Action Row'
✓  frame.name = 'Divider'
✗  (domyślne 'Frame', 'Group', 'Rectangle' — niedopuszczalne)
```

Reguła: `node.name = 'descriptive-name'` to **obowiązkowa linia** przy każdym `figma.createFrame()`, `createText()`, `createRectangle()`.

Wzorzec nazwy: `[Rola] [Kontekst]` — np. `"Time Pill"`, `"Content Area"`, `"Title Row"`, `"Join Button"`.

### Auto-layout — świadoma decyzja dla każdego frame'a

Każdy nowy frame wymaga jawnej decyzji o `layoutMode`:

```
VERTICAL   → dzieci układają się w kolumnie (sekcje, listy, karty, formularze)
HORIZONTAL → dzieci układają się w wierszu (headery, action bary, wiersze tabel, pills)
NONE       → tylko gdy absolutne pozycjonowanie jest konieczne (tabele z grid, overlay)
             + ZAWSZE z komentarzem w kodzie: // layoutMode NONE — tabela z fixed positions
```

`layoutMode = 'NONE'` bez uzasadnienia = red flag wymagający przeglądu.

**Sizing po appendChild — obowiązkowa kolejność:**
```javascript
parent.appendChild(child);
// DOPIERO PO appendChild ustaw sizing:
child.layoutSizingHorizontal = 'FILL'; // lub 'FIXED' / 'HUG'
child.layoutSizingVertical = 'HUG';    // domyślnie jest FIXED — jawnie ustaw HUG
```

**Kolejność dla resize z auto-layoutem:**
```javascript
frame.layoutMode = 'HORIZONTAL';         // 1. ustaw tryb
frame.resize(targetWidth, frame.height); // 2. resize
frame.layoutSizingHorizontal = 'FIXED';  // 3. zablokuj szerokość (inaczej wraca do HUG)
```

### Hierarchia — brak sierot

Każdy nowo stworzony node musi być dzieckiem właściwego parenta. Przed zakończeniem skryptu sprawdź: żaden nowy node nie leży bezpośrednio na `figma.currentPage` — wszystko idzie do target frame/section.

```javascript
// Anti-pattern — sierota na canvasie:
const orphan = figma.createFrame(); // ← jeszcze nie ma parenta!
// Poprawnie — od razu append:
const f = figma.createFrame();
f.name = 'Content Row';
parentFrame.appendChild(f); // ← natychmiast po stworzeniu
```

### Separator / divider

Separator = frame `h=1` z `fillStyleId` (paint style) lub stroke na rodzicu.
NIE: `createRectangle()` bez nazwy i stylu, NIE: hardcoded kolor `{ r: 0.9, g: 0.9, b: 0.9 }`.

```javascript
const divider = figma.createFrame();
divider.name = 'Divider';
divider.resize(parentWidth, 1);
await divider.setFillStyleIdAsync(outlineStyleId); // binduj paint style, nie hex
```

---

## Spacing — odkryj system, nie zgaduj

### Pre-flight: 3 kroki przed użyciem jakiegokolwiek spacing

Zawsze wykonuj w tej kolejności. Zatrzymaj się na kroku który daje wynik.

**Krok A: Sprawdź czy istnieją spacing variables (FLOAT)**

```javascript
const vars = await figma.variables.getLocalVariablesAsync();
const spacingVars = vars.filter(v => v.resolvedType === 'FLOAT');
return { count: spacingVars.length, sample: spacingVars.slice(0, 20).map(v => ({ name: v.name, id: v.id })) };
// Jeśli count > 0 → ZAWSZE używaj variables (patrz sekcja "Bindowanie spacing variables")
// Jeśli count = 0 → przejdź do Kroku B
```

**Krok B: Odczytaj spacing z istniejących ekranów**

```javascript
const samples = [];
function sampleSpacing(n, depth = 0) {
  if (depth > 2) return;
  if (n.layoutMode && n.layoutMode !== 'NONE' && (n.itemSpacing > 0 || n.paddingTop > 0 || n.paddingLeft > 0)) {
    samples.push({ name: n.name, gap: n.itemSpacing, pt: n.paddingTop, pb: n.paddingBottom, pl: n.paddingLeft, pr: n.paddingRight });
  }
  n.children?.forEach(c => sampleSpacing(c, depth + 1));
}
await figma.loadAllPagesAsync();
const refPage = figma.root.children.find(p => !p.name.startsWith('---') && p.children.length > 0);
await figma.setCurrentPageAsync(refPage);
refPage.children.slice(0, 3).forEach(f => sampleSpacing(f));
return samples.slice(0, 20);
// Wynik: lista używanych wartości → użyj ich jako punktu odniesienia
```

**Krok C: Skopiuj spacing z podobnego istniejącego komponentu**

```javascript
// Znajdź komponent o podobnej roli (Card, Row, Panel, List)
const ref = figma.currentPage.findOne(n =>
  n.type === 'FRAME' && n.layoutMode !== 'NONE' &&
  (n.name.includes('Card') || n.name.includes('Row') || n.name.includes('Panel'))
);
return { gap: ref?.itemSpacing, pt: ref?.paddingTop, pb: ref?.paddingBottom, pl: ref?.paddingLeft, pr: ref?.paddingRight };
// Kopiuj wartości 1:1 — spójność ważniejsza niż "prawidłowe" wartości abstrakcyjne
```

### Bindowanie spacing variables (gdy Krok A zwrócił wyniki)

```javascript
const vars = await figma.variables.getLocalVariablesAsync();
const spacingMap = {};
vars.filter(v => v.resolvedType === 'FLOAT').forEach(v => { spacingMap[v.name] = v; });

// Przykład nazw — dostosuj do konwencji pliku (spacing/md, gap-md, space-16 itp.)
frame.setBoundVariable('itemSpacing', spacingMap['spacing/md']);
frame.setBoundVariable('paddingTop',    spacingMap['spacing/sm']);
frame.setBoundVariable('paddingBottom', spacingMap['spacing/sm']);
frame.setBoundVariable('paddingLeft',   spacingMap['spacing/md']);
frame.setBoundVariable('paddingRight',  spacingMap['spacing/md']);

// Fallback gdy dana variable nie istnieje:
if (!spacingMap['spacing/md']) frame.itemSpacing = 16;
```

Dostępne właściwości do bindowania: `itemSpacing`, `paddingTop`, `paddingBottom`, `paddingLeft`, `paddingRight`, `width`, `height`, `cornerRadius`, `topLeftRadius` itd.

> **Reguła spójności:** jeśli plik używa spacing variables — **wszystkie** spacing muszą przez nie przechodzić. Nigdy nie mieszaj variables z hardcoded px dla tej samej klasy wartości (np. nie `paddingTop = spacingVar` i `paddingBottom = 12`). Mieszanie psuje refactoring tokenów.

### Fallback scale (gdy Kroki A–C nie dały wyników)

Stosuj wyłącznie gdy nie masz żadnego punktu odniesienia z pliku. Wszystkie wartości to wielokrotności 4px.

| Kontekst                        | gap    | paddingH | paddingV |
|---------------------------------|--------|----------|----------|
| Ikon / tekst obok siebie        | 4–6    | —        | —        |
| Chip / badge / pill             | 4–6    | 8–10     | 3–5      |
| Row w liście / wiersz tabeli    | 8      | 0        | 8–10     |
| Wewnątrz karty / panelu         | 12     | 16       | 12–16    |
| Między sekcjami w karcie        | 12     | —        | —        |
| Content area (główna zawartość) | 24     | 24       | 24       |
| Między kartami na stronie       | 16–24  | —        | —        |
| Kolumny layoutu (top-level)     | 32     | —        | —        |

---

## Bindowanie zmiennych kolorów

Gdy plik ma design tokens, **zawsze** binduj zamiast hardcoded wartości:

```javascript
// Po ID (wydajniejsze — jeśli znasz ID z katalogu pliku)
const v = await figma.variables.getVariableByIdAsync('VariableID:XXXX:YYYY');

// Po nazwie (bardziej czytelne — gdy nie znasz ID)
const vars = await figma.variables.getLocalVariablesAsync();
const v = vars.find(x => x.name === 'colors/background/bg-brand');

// Aplikuj do fills
const paint = figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', v
);
node.fills = [paint];

// Aplikuj do strokes
node.strokes = [figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', borderVar
)];
```

---

## Budowanie nowej strony — workflow krok po kroku

```
1. Pre-flight: audit istniejących stron → lista komponentów z kluczami
2. Sprawdź konwencje: frameWidth, nazwy stron, hierarchia
3. Stwórz nową stronę (sprawdź czy już istnieje!)
4. Stwórz główny frame (zgodna szerokość)
5. Buduj sekcjami — jeden figma_execute per sekcja:
   a. Importuj komponent → createInstance() → append do frame
   b. Pozycjonuj (x=0, y=poprzednia_sekcja_y + wysokość)
   c. figma_capture_screenshot → walidacja wizualna
6. Audit kolorów (przed finalną walidacją)
7. Finalna walidacja całego ekranu
```

### Audit kolorów — uruchom po zbudowaniu każdej sekcji

Cel: zero hardcoded fills/strokes na węzłach niebędących instancjami.

```javascript
function auditColors(sectionNode) {
  const issues = [];
  function check(n) {
    if (n.type === 'INSTANCE') return; // instancje zarządzają własnymi stylami
    // Hardcoded fill (SOLID bez powiązanego stylu)
    if (n.fills?.some(f => f.type === 'SOLID') && !n.fillStyleId)
      issues.push({ id: n.id, name: n.name, issue: 'hardcoded fill' });
    // Hardcoded stroke
    if (n.strokes?.some(s => s.type === 'SOLID') && !n.strokeStyleId)
      issues.push({ id: n.id, name: n.name, issue: 'hardcoded stroke' });
    if (n.children) for (const child of n.children) check(child);
  }
  check(sectionNode);
  return { ok: issues.length === 0, issues };
  // Cel: { ok: true, issues: [] }
  // Każdy wpis w issues → podepnij paint style zamiast hardcoded wartości
}
return auditColors(figma.currentPage.findOne(n => n.id === 'TARGET_SECTION_ID'));
```

### Nowa strona — szablon

```javascript
await figma.loadAllPagesAsync();
const pageName = 'Checkout Step 1'; // dostosuj

// Nie duplikuj
const existing = figma.root.children.find(p => p.name === pageName);
if (existing) {
  await figma.setCurrentPageAsync(existing);
  return { status: 'page_exists', id: existing.id };
}

const page = figma.createPage();
page.name = pageName;
await figma.setCurrentPageAsync(page);

const W = 1440; // dostosuj do konwencji pliku
const frame = figma.createFrame();
frame.name = pageName;
frame.resize(W, 100); // wysokość wyrośnie przy appendowaniu
frame.x = 0;
frame.y = 0;
page.appendChild(frame);

return { pageId: page.id, frameId: frame.id };
```

### Układanie instancji

```javascript
// Importuj i układaj od góry
const comp = await figma.importComponentByKeyAsync('COMPONENT_KEY');
const inst = comp.createInstance();
frame.appendChild(inst);
inst.x = 0;
inst.y = currentY; // śledź akumulowaną wysokość
inst.resize(W, inst.height); // rozciągnij do pełnej szerokości

currentY += inst.height;
return { instanceId: inst.id, newY: currentY };
```

---

## Typowe błędy i jak ich unikać

| Błąd | Fix |
|------|-----|
| Ręczne rect+text zamiast instancji | Zawsze przejdź przez decision tree najpierw |
| Hardcoded `{ r: 0.12, g: 0.17, b: 0.30 }` | `setFillStyleIdAsync(styleId)` lub `setBoundVariableForPaint` z tokenem |
| Tylko navbar+footer jako instancje, reszta ręcznie | KAŻDY element to instancja |
| Pominięcie auditu istniejących stron | `getMainComponentAsync` na podobnej stronie |
| Import lokalnego (nieopublikowanego) komponentu przez key | Lokalne: `getNodeByIdAsync(nodeId).createInstance()` — `importComponentByKeyAsync` tylko dla opublikowanych |
| Text property ustawiona, brak efektu wizualnego | Diagnozuj: porównaj `componentProperties` vs `findAll(TEXT).characters`; fallback: `detachInstance()` + direct edit |
| Frame 100px zamiast huggowania zawartości | `layoutSizingVertical` domyślnie FIXED — po appendChild: `child.layoutSizingVertical = 'HUG'` |
| Frame wraca do HUG po `resize()` | Zła kolejność: (1) `layoutMode`, (2) `resize()`, (3) `layoutSizingHorizontal = 'FIXED'` |
| Warstwy z domyślnymi nazwami Frame/Group | `node.name = 'descriptive-name'` przy każdym `createFrame()` |
| Frame bez `layoutMode` (dzieci w losowych pozycjach) | Każdy frame = świadoma decyzja: VERTICAL / HORIZONTAL / NONE z uzasadnieniem |
| Sieroty na canvasie | `parentFrame.appendChild(node)` natychmiast po stworzeniu |
| Zakładanie "Inter" jako font family | Krok 0 pre-flight: `findOne(TEXT).fontName.family` |
| Jeden wielki `figma_execute` na cały ekran | Dziel na sekcje, screenshot po każdej |
| Zła szerokość frame'a (np. 1440 vs 1563) | Sprawdź `frame.width` na istniejącej stronie |
| Text w instancji przez direct edit | `setProperties({ 'Label#123': 'Tekst' })` |
| `getLocalVariables()` zamiast async | `getLocalVariablesAsync()` |

---

## Jak katalogować nowy design system

Przy pierwszym kontakcie z nowym plikiem Figma, uruchom ten skrypt i zapisz wynik do memory:

```javascript
await figma.loadAllPagesAsync();
const pages = figma.root.children.map(p => ({ name: p.name, id: p.id }));

// Znajdź pierwszą sensowną stronę (pomiń strony z '---' prefix)
const refPage = figma.root.children.find(p => !p.name.startsWith('---') && p.children.length > 0);
await figma.setCurrentPageAsync(refPage);

// Konwencje pliku
const frame = refPage.children[0];
const frameWidth = frame?.width;

// Zmienne kolorów
const vars = await figma.variables.getLocalVariablesAsync();
const colorVars = vars.filter(v => v.resolvedType === 'COLOR')
  .map(v => ({ name: v.name, id: v.id }));

// Komponenty z instancji na stronie referencyjnej
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
// → Skopiuj wynik do memory/ jako reference dla tego pliku
```

---

## Relacja z innymi skillami

### Desktop Bridge path (preferowany — szybszy, lokalny)
| Skill | Rola |
|-------|------|
| `figma-design-toolkit:figma-design-workflow` (ten) | **Metodologia** — decision tree, pre-flight, wzorce kodu |
| `figma-design-toolkit:figma-console` | **Mechanika Desktop** — figma_execute, error recovery, placement |
| `figma-design-toolkit:figma-cli` | **CLI** — JSX render, shadcn tokens, szybszy niż MCP |

Załaduj oba gdy projektujesz ekrany: `/figma-design-toolkit:figma-design-workflow` + `/figma-design-toolkit:figma-console`

### Cloud path (fallback — gdy Desktop Bridge niedostępny)
| Skill | Rola |
|-------|------|
| `figma:figma-use` | Plugin API prerequisite dla cloud write ops |
| `figma:figma-generate-design` | Kod/opis → ekran Figma (cloud) |
| `figma:figma-generate-library` | Design system z kodu (cloud) |
| `figma:figma-implement-design` | Figma design → kod produkcyjny |

> Ten skill (`figma-design-workflow`) jest niezależny od ścieżki — metodologia decision tree
> i pre-flight audit stosują się zarówno do Desktop Bridge jak i Cloud path.
