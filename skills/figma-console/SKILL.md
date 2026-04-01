---
name: figma-console
description: Prerequisite for figma-console MCP tools (figma_execute, figma_capture_screenshot, figma_search_components etc.). Load this skill before any figma_execute call. Use figma-console when you need full Plugin API access via Figma Desktop — complex component creation, variant management, variable binding, programmatic design operations that figma-cli's JSX syntax can't handle.
---

# figma-console — Desktop Bridge MCP

Steruje Figma Desktop przez WebSocket (Desktop Bridge plugin). Wymaga otwartej Figmy z działającym pluginem.

---

## Kiedy figma-console vs figma-cli vs use_figma

| Zadanie | Narzędzie |
|---------|-----------|
| JSX render, shadcn tokens, UI bloki | **figma-cli** |
| Kompleksowy Plugin API, warianty, variable binding | **figma-console** (`figma_execute`) |
| Odczyt design context, screenshot do kodu | `mcp__figma-desktop__` lub `mcp__figma__` |
| Fallback gdy desktop niedostępny | `use_figma` (claude.ai Figma) |

---

## Component-first — zasada nadrzędna dla design tasks

**ZANIM zbudujesz jakikolwiek element UI**, przejdź przez ten decision tree:

```
Czy istnieje komponent w pliku który PASUJE?
  → TAK: importComponentByKeyAsync(key) + createInstance()

Czy istnieje komponent który PRAWIE pasuje (inny kontekst, zbliżony wygląd)?
  → TAK: createInstance() + setProperties() dla variant props
         (jeśli tekst nie jest property → zaakceptuj fixed content lub wybierz inny wariant)

Żaden komponent nie pasuje?
  → Buduj z atomów, ALE wyłącznie z design tokenów:
     - kolory: setBoundVariableForPaint (NIE hardcoded RGB)
     - typografia: family/size/weight zgodne z systemem
     - spacing: wielokrotności 4px lub 8px
     - radius: wartości z biblioteką (6/8/12/16px)
```

> Reguła: każdy element UI to instancja komponentu — przyciski, inputy, karty, nawigacja, WSZYSTKO.
> Budowanie z `createRectangle()` + `createText()` omija cały design system.

### Krok 0 (przed każdym zadaniem designu): audit istniejących stron

Zanim zaczniesz budować, sprawdź co używają istniejące podobne strony w pliku — to gotowy "przepis":

```javascript
await figma.loadAllPagesAsync();
const page = figma.root.children.find(p => p.name.includes('Cart')); // dostosuj nazwę
await figma.setCurrentPageAsync(page);
const frame = page.children[0];
const results = [];
for (const inst of frame.children.filter(n => n.type === 'INSTANCE')) {
  const mc = await inst.getMainComponentAsync();
  results.push({ name: inst.name, key: mc?.key, variant: mc?.name });
}
return results;
// → lista komponentów z kluczami gotowa do użycia
```

### Bindowanie zmiennych kolorów (nie hardcoduj RGB)

Gdy plik ma design tokens (zmienne kolorów), ZAWSZE binduj:

```javascript
// Pobierz zmienną po ID (ID stabilne — nie zmienia się między sesjami)
const v = await figma.variables.getVariableByIdAsync('VariableID:...');
const paint = figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', v
);
node.fills = [paint];

// Lub pobierz wszystkie zmienne i znajdź po nazwie:
const vars = await figma.variables.getLocalVariablesAsync();
const colorMap = {};
vars.filter(v => v.resolvedType === 'COLOR').forEach(v => { colorMap[v.name] = v; });
// colorMap['colors/background/bg-brand'], colorMap['colors/text/text-heading'] itp.
```

### Screenshot wariantu przed użyciem

Przed `createInstance()` zrób screenshot komponentu żeby potwierdzić że to właściwy wariant:
```
figma_capture_screenshot({ nodeId: 'COMPONENT_NODE_ID' })
```

---

## Pre-flight checklist (każda sesja)

```
[ ] figma_get_status — sprawdź połączenie
[ ] figma_list_open_files — który plik jest aktywny?
[ ] figma_search_components — odśwież node IDs (stale między sesjami!)
[ ] figma_capture_screenshot — zrób screenshot strony przed zmianami
[ ] Znajdź wolne miejsce, nie nakładaj na istniejącą zawartość
```

**Node IDs są stale między rozmowami** — nigdy nie używaj ID z poprzedniej sesji bez re-searchu.

---

## figma_execute — krytyczne zasady

Wiele zasad jest tych samych co w `use_figma`. Poniżej te, które są inne lub warte podkreślenia.

### 1. Kod JS — format

```javascript
// DOBRZE — czysty JS z top-level await i return
const frame = figma.createFrame();
frame.resize(200, 200);
return { id: frame.id, name: frame.name };

// ŹLE — nie owijaj w async IIFE (jest auto-wrapped)
(async () => { ... })()
```

### 2. Return jest jedynym kanałem wyjścia

- `console.log()` → niewidoczne, nie używaj
- `figma.notify()` → throws "not implemented", nie używaj
- Zawsze `return` z danymi: `{ createdNodeIds: [...], status: "ok" }`

### 3. Instances — tekst musi przez figma_set_instance_properties

```javascript
// ŹLE — FAILS SILENTLY (brak błędu, ale tekst się nie zmienia!)
const inst = figma.createInstance(component);
inst.findOne(n => n.type === 'TEXT').characters = 'Nowy tekst';

// DOBRZE — przez właściwości instancji
// Najpierw sprawdź dostępne właściwości:
return Object.keys(instance.componentProperties);
// Potem ustaw:
instance.setProperties({ 'Label#2:0': 'Nowy tekst' });
```

Lub użyj narzędzia `figma_set_instance_properties` bezpośrednio.

### 4. Sprawdzaj resultAnalysis.warning

figma_execute zwraca `resultAnalysis` — zawsze sprawdź `resultAnalysis.warning`:
- Empty arrays → operacja może być cicha porażka
- Null returns → node nie istnieje lub błędne ID

### 5. Placement — zawsze w Section/Frame, nigdy na pustym canvasie

```javascript
// DOBRZE — znajdź lub stwórz Section
let section = figma.currentPage.findOne(n => n.type === 'SECTION' && n.name === 'Components');
if (!section) {
  section = figma.createSection();
  section.name = 'Components';
  // Pozycjonuj poniżej istniejącej zawartości
  const maxY = Math.max(0, ...figma.currentPage.children.map(n => n.y + n.height)) + 100;
  section.y = maxY;
}
// Twórz wewnątrz section
const frame = figma.createFrame();
section.appendChild(frame);
```

### 6. Cleanup przy błędzie/retry

Jeśli skrypt się wysypał i zostawił częściowe artefakty — **usuń je przed retry**:

```javascript
// Znajdź i usuń puste/sieroty frames
const orphans = figma.currentPage.findAll(n => n.name === 'Temp' || n.children?.length === 0);
orphans.forEach(n => n.remove());
```

Nigdy nie buduj na zepsutej podstawie.

### 7. Strony — nie duplikuj

```javascript
// ZAWSZE sprawdź czy strona istnieje przed stworzeniem
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

### 8. Kolory — zakres 0–1 (nie 0–255)

```javascript
{ r: 1, g: 0, b: 0 }      // czerwony ✓
{ r: 255, g: 0, b: 0 }    // ŹLE ✗
```

### 9. FILL po appendChild

```javascript
parent.appendChild(child);
child.layoutSizingHorizontal = 'FILL'; // AFTER append, nie przed
```

### 10. Font PRZED operacją na tekście — sprawdź font family z pliku

```javascript
// NIE zakładaj 'Inter' — sprawdź faktyczny font z istniejącego węzła tekstowego
const sample = figma.currentPage.findOne(n => n.type === 'TEXT');
const fontFamily = sample?.fontName?.family ?? 'Inter';
await figma.loadFontAsync({ family: fontFamily, style: 'Regular' });
await figma.loadFontAsync({ family: fontFamily, style: 'SemiBold' });
await figma.loadFontAsync({ family: fontFamily, style: 'Medium' });
const text = figma.createText();
text.characters = 'Hello'; // dopiero po loadFontAsync
```

### 11. Lokalny vs opublikowany komponent

```javascript
// OPUBLIKOWANY (w library) → importComponentByKeyAsync
const comp = await figma.importComponentByKeyAsync('abc123key');

// LOKALNY (nieopublikowany) → getNodeByIdAsync + createInstance
// nodeId pochodzi z figma_search_components → results[].nodeId
const comp = await figma.getNodeByIdAsync('337:15141');
const inst = comp.createInstance();
```

> `importComponentByKeyAsync` na lokalnym komponencie rzuca `"cannot import unpublished"`. Zawsze sprawdź czy komponent jest opublikowany — jeśli nie, użyj `getNodeByIdAsync`.

### 12. setProperties() — weryfikuj efekt wizualny, nie tylko wartość property

```javascript
// setProperties zmienia wartość w componentProperties, ale może NIE zmienić
// węzła tekstowego jeśli text property jest "ghost" (niezwiązana z TextNode)

inst.setProperties({ 'Label#xxx': 'Nowy tekst' });

// Weryfikuj: sprawdź faktyczne węzły tekstowe
const actualText = inst.findAll(n => n.type === 'TEXT').map(t => t.characters);
// Jeśli actualText nie zawiera 'Nowy tekst' → property jest ghost → użyj fallback

// FALLBACK: detach + direct edit (zawsze działa)
const detached = inst.detachInstance();
const t = detached.findOne(n => n.type === 'TEXT' && n.visible);
await figma.loadFontAsync(t.fontName);
t.characters = 'Nowy tekst';
// Uwaga: detach traci link do komponentu — akceptowalne dla custom labels na konkretnym ekranie
```

### 13. layoutSizingVertical — FIXED domyślnie, HUG musi być jawne

```javascript
// Nowe frames mają layoutSizingVertical = 'FIXED' domyślnie
// Auto-layout NIE przełącza na HUG automatycznie po dodaniu tekstu
// HUG musi być ustawione JAWNIE i ZAWSZE PO appendChild()

parent.appendChild(rowFrame);
rowFrame.layoutSizingHorizontal = 'FILL';  // zajmij całą szerokość parenta
rowFrame.layoutSizingVertical = 'HUG';     // skurczy się do wysokości zawartości
// ✗ NIE ustawiaj HUG przed appendChild — nie zadziała
```

### 14. Kolejność: layoutMode → resize() → layoutSizingHorizontal

```javascript
// Ustawienie layoutMode zmienia sizing na HUG — resize() przywraca FIXED
// Wymagana kolejność:
frame.layoutMode = 'HORIZONTAL';          // (1) najpierw layout mode
frame.resize(768, frame.height);          // (2) potem resize do żądanego wymiaru
frame.layoutSizingHorizontal = 'FIXED';   // (3) na końcu zablokuj sizing
// ✗ Odwrotna kolejność → frame wraca do HUG szerokości tekstu
```

---

## Visual Validation Loop (obowiązkowy)

Po każdej operacji tworzącej/modyfikującej elementy wizualne:

```
1. figma_capture_screenshot(nodeId: "NODE_ID")   ← preferuj nad figma_take_screenshot
2. Sprawdź: alignment, spacing, proporcje, visual balance
3. Iteruj jeśli coś nie gra (max 3 razy)
4. Ostateczne figma_capture_screenshot dla potwierdzenia
```

**figma_capture_screenshot vs figma_take_screenshot:**
- `figma_capture_screenshot` — przez plugin runtime, widzi zmiany **natychmiast**, preferowany do walidacji
- `figma_take_screenshot` — przez REST API (chmura), może nie mieć najnowszego stanu, dobry do szerszych widoków strony

---

## Session Management

### Sprawdź połączenie
```
figma_get_status
```
Jeśli status nie OK → `figma_reconnect`

### Wiele plików
```
figma_list_open_files    → które pliki mają Desktop Bridge plugin
figma_navigate           → przełącz aktywny plik
```

### Odkryj strukturę pliku
```javascript
// Zacznij od verbosity='summary', depth=1 — nie 'full' (pochłania tokeny!)
figma_get_file_data({ verbosity: 'summary', depth: 1 })
```

### Odkryj komponenty (na początku sesji)
```
figma_search_components({ query: 'Button', limit: 10 })
figma_search_components({ category: 'Card', limit: 10 })
```

### Sprawdź co użytkownik zaznaczył
```
figma_get_selection()
figma_get_selection({ verbose: true })  // z fills/strokes/styles
```

---

## Incremental workflow — jak unikać bugów

1. **Inspect first** — screenshot + `figma_get_file_data(summary)` zanim cokolwiek stworzysz
2. **Jedno zadanie per `figma_execute`** — nie buduj całego ekranu w jednym wywołaniu
3. **Return node IDs z każdego call** — potrzebujesz ich w kolejnych krokach
4. **Waliduj po każdym kroku** — `figma_capture_screenshot` po tworzeniu komponentów
5. **Napraw przed kontynuowaniem** — nie buduj na błędnym stanie

### Sugerowana kolejność dla złożonych zadań

```
Krok 1: Inspect — figma_get_file_data + figma_capture_screenshot (co już jest?)
Krok 2: figma_search_components (jakie komponenty są dostępne?)
Krok 3: Stwórz Section/parent frame → return { sectionId }
Krok 4: Stwórz tokeny/zmienne → waliduj figma_capture_screenshot
Krok 5: Stwórz komponenty (jeden per call) → waliduj
Krok 6: Złóż layout z instancji → waliduj
Krok 7: Finalna walidacja całości
```

---

## Error Recovery

`figma_execute` jest atomiczne — jeśli skrypt rzuci błąd, żadne zmiany nie są aplikowane.

**Gdy błąd:**
1. STOP — nie retry natychmiast
2. Przeczytaj error message dokładnie
3. Jeśli niejasny — `figma_capture_screenshot` + `figma_get_file_data` żeby sprawdzić stan
4. Napraw skrypt
5. Retry (bezpieczne — nic nie zostało zmienione)

**Gdy sukces ale wygląda źle:**
1. `figma_capture_screenshot(nodeId)` — nie całej strony
2. Napisz targeted fix script — nie odtwarzaj wszystkiego od zera

### Typowe błędy i fix

| Error | Przyczyna | Fix |
|-------|-----------|-----|
| "not implemented" | `figma.notify()` | Usuń, użyj `return` |
| Tekst się nie zmienia, brak błędu | Direct edit instancji | Użyj `setProperties()` lub `figma_set_instance_properties` |
| `"Setting figma.currentPage is not supported"` | Sync setter | `await figma.setCurrentPageAsync(page)` |
| Empty array w result | Silent fail — node nie istnieje | Sprawdź ID, sprawdź stronę |
| FILL przed appendChild | Crash na layoutSizing | Przenieś FILL po appendChild |
| Font error | Brak loadFontAsync | Dodaj `await figma.loadFontAsync(...)` przed text ops |

---

## Pre-flight checklist (przed figma_execute)

- [ ] `return` używane jako output (nie console.log, nie figma.notify)
- [ ] Kod NIE jest owiniętyw async IIFE
- [ ] Kolory w zakresie 0–1
- [ ] `layoutSizingH/V = 'FILL'` ustawiane PO appendChild
- [ ] `loadFontAsync()` wywołane PRZED text operations
- [ ] Page switch przez `await figma.setCurrentPageAsync(page)`
- [ ] Instancje: tekst przez `setProperties()`, nie direct edit
- [ ] Nowe node'y tworzone wewnątrz Section/Frame (nie na gołym canvasie)
- [ ] Zwracane node IDs ze wszystkich stworzonych/zmodyfikowanych elementów
- [ ] Sprawdzona duplikacja stron przed tworzeniem

---

## Relacja z figma-use skill

Wiele zasad Plugin API jest identycznych z `figma-use`. Gdy potrzebujesz szczegółowych wzorców:
- Komponenty i warianty → `/figma:figma-use` references/component-patterns.md
- Zmienne i bindowanie → `/figma:figma-use` references/variable-patterns.md
- Text styles → `/figma:figma-use` references/text-style-patterns.md
- Pełne gotchas → `/figma:figma-use` references/gotchas.md

Różnica: tamte skille używają `use_figma`, ten skill używa `figma_execute`. Zasady Plugin API są identyczne.
