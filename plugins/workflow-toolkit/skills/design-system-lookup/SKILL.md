---
name: design-system-lookup
description: Szukanie istniejącego wzorca w design systemie PRZED stworzeniem nowego komponentu lub sekcji. Zapobiega wymyślaniu klas i wzorców które już istnieją.
---

# Skill: design-system-lookup

## Cel
Znalezienie właściwego wzorca w design systemie PRZED stworzeniem nowego komponentu lub sekcji.
Zapobiega wymyślaniu klas Tailwind i wzorców które już istnieją.

## Auto-trigger
Uruchamia się gdy:
- Tworzysz nowy komponent lub sekcję
- Pada pytanie "jaki wzorzec użyć do..."
- Nie jesteś pewny czy dany pattern już istnieje
- Zaczynasz redesign lub nowy blok UI

## Protokół

### Krok 1 — Sprawdź czy projekt ma design system
Po kolei sprawdź:

| Lokalizacja | Zawiera |
|---|---|
| `docs/design-system/` | Pełna dokumentacja warstwowa (foundations/primitives/patterns/blueprints) |
| `docs/components/` lub `docs/ui/` | Alternatywna nazwa katalogu |
| `src/app/components/` lub `src/components/` | Implementacje — czytaj nazwy komponentów |
| `tailwind.config.js` / `theme.css` | Tokeny (kolory, spacing, typography) |
| `DesignLanguagePage.tsx` / `StyleGuidePage.tsx` | Runtime preview wzorców |

**Jeśli żadne z powyższych nie istnieje** — powiadom użytkownika że projekt nie ma sformalizowanego design systemu i zapytaj jak chce pracować.

### Krok 2 — Określ kategorię tego czego szukasz

| Pytasz o... | Szukaj w warstwie... |
|---|---|
| kolory, font, spacing, easing, motion | Foundations (tokeny, `docs/design-system/01-foundations/`) |
| button, card, badge, input, tile | Primitives (`02-primitives/`) |
| hero, CTA, quote, metrics, timeline | Patterns (`03-patterns/`) |
| struktura całej strony | Blueprints (`04-page-blueprints/`) |

### Krok 3 — Przeszukaj pliki
Użyj Grep/Glob aby znaleźć istniejący wzorzec:
- Szukasz karty? → `Grep "Card" src/app/components/` i czytaj pierwsze 3 wyniki
- Szukasz hero? → `Glob **/Hero*.tsx` i czytaj najnowszy
- Szukasz wzorca → czytaj odpowiedni plik .md w `docs/design-system/`

### Krok 4 — Decyzja

| Znalazłeś... | Akcja |
|---|---|
| Identyczny wzorzec | Użyj go, nie wymyślaj od nowa |
| Częściowe dopasowanie | Rozszerz istniejący (dodaj wariant, props, opcję) |
| Nic podobnego | Zapytaj użytkownika **zanim** stworzysz nowy wzorzec — nie zakładaj |

### Krok 5 — Weryfikacja tokenów
Przed napisaniem pierwszej linii CSS sprawdź:
- Kolory → użyj CSS variables / klas Tailwind z projektowych tokenów, nie hex inline
- Spacing → użyj skali projektu, nie `mt-[37px]`
- Font sizes → użyj skali, nie `text-[17px]`

Jeśli potrzebujesz wartości której nie ma w tokenach — zapytaj użytkownika przed dodaniem.

## Zasady ogólne (uniwersalne)

### Centralne zarządzanie
Każdy element wizualny ma być zarządzalny z poziomu design systemu.
Zmiana tokenu/komponentu w design systemie propaguje się na cały projekt — bez edycji stron.

### Strony = kompozycja, nie twórczość
Pliki stron (`pages/`, `app/`) używają **wyłącznie** zarządzanych komponentów.
Zakazane: inline anonymous divs, one-off helpery, hardcoded hex kolory w stronach.

### Jeśli komponent nie istnieje
1. Stwórz go w odpowiednim katalogu komponentów (nie w pliku strony)
2. Udokumentuj w design systemie (jeśli projekt ma dokumentację)
3. Dopiero potem użyj w stronie

## Przykłady

```
Użytkownik: "Dodaj sekcję ze statystykami"
→ Sprawdź docs/design-system/03-patterns/ pod kątem "metrics" / "stats"
→ Jeśli jest → użyj wzorca
→ Jeśli nie ma → zapytaj: "Nie widzę wzorca dla metrics. Zrobić nowy w ramach design systemu czy ad hoc?"

Użytkownik: "Nowy przycisk CTA"
→ Sprawdź src/app/components/ pod kątem Button
→ Prawdopodobnie istnieje — użyj go z odpowiednim wariantem
→ Jeśli wariant nie istnieje → rozszerz istniejący komponent, nie twórz równoległego
```
