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
6. Finalna walidacja całego ekranu
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
| Hardcoded `{ r: 0.12, g: 0.17, b: 0.30 }` | `setBoundVariableForPaint` z tokenem |
| Tylko navbar+footer jako instancje, reszta ręcznie | KAŻDY element to instancja |
| Pominięcie auditu istniejących stron | `getMainComponentAsync` na podobnej stronie |
| Import lokalnego komponentu przez key | Lokalne komponenty: `getNodeByIdAsync(nodeId)` |
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

| Skill | Rola |
|-------|------|
| `figma-design-workflow` (ten) | **Metodologia** — decision tree, pre-flight, wzorce kodu |
| `figma-console` | **Mechanika** — jak używać figma_execute, error recovery, placement |
| `figma-cli` | **Alternatywne narzędzie** — JSX render, shadcn tokens |

Załaduj oba gdy projektujesz ekrany: `/figma-design-workflow` + `/figma-console`.
