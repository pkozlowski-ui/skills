---
name: init-project
description: Bootstrap nowego projektu — kopiuje template startowy, generuje CLAUDE.md per-projekt, inicjalizuje memory, rejestruje weekly audit. Uruchamia się gdy użytkownik mówi "zainicjalizuj projekt", "ustaw Claude dla tego projektu", "nowy projekt".
---

# Skill: init-project

## Cel
Zaprogramować nowy projekt tak, żeby od razu działał z pełnym ekosystemem Piotra:
globalne CLAUDE.md dziedziczy automatycznie, project CLAUDE.md dostaje odpowiedni
preset, memory inicjalizowana, weekly audit zarejestrowany w /schedule.

## Auto-trigger
Uruchamia się gdy użytkownik mówi (w katalogu nowego projektu):
- "zainicjalizuj ten projekt"
- "ustaw Claude dla tego projektu"
- "nowy projekt — przygotuj"
- "zrób setup dla tego projektu"

## Prerekwizyty

Template startowy musi istnieć w `~/Documents/claude-project-template/`.
Jeśli nie istnieje — powiedz użytkownikowi że framework nie jest w pełni wdrożony
i zaproponuj utworzenie templatu najpierw.

## Protokół (5 kroków)

### Krok 1 — Sanity check
Sprawdź obecny katalog:
- `pwd` — czy jesteśmy w nowym katalogu projektu?
- `ls` — czy katalog jest pusty lub tylko z `README` / `.git`?
- Jeśli już istnieje `CLAUDE.md` w root → zapytaj użytkownika: "Widzę że projekt ma już CLAUDE.md. Nadpisać / uzupełnić / anulować?"

### Krok 2 — Pytania kontekstowe (AskUserQuestion)

**Pytanie 1: Typ projektu (single select)**
- Frontend prototype (React/Next.js + Tailwind)
- Fullstack mały (Next.js + API routes)
- Design-to-code (z Figmy do kodu, ciężki design system)
- Code-to-design (prototyp → Figma)
- Inne — wtedy doprecyzuj stack w pytaniu uzupełniającym

**Pytanie 2: Dodatkowe elementy (multi select)**
- Figma file — chcę podłączyć URL
- Design system — scaffold docs/design-system/ z 4-warstwową hierarchią
- Read-only text rule — nie dotykać copy marketingowego (hook ochronny)
- Weekly audit — automatyczny raport co poniedziałek
- Memory — inicjalizuj ~/.claude/projects/<cwd>/memory/

**Pytanie 3: Stack techniczny (tylko jeśli Pytanie 1 = "Inne")**
Open-ended, user wpisuje.

### Krok 3 — Kopiuj template i populuj

Na podstawie odpowiedzi:

1. **Kopiuj strukturę** z `~/Documents/claude-project-template/` do obecnego katalogu:
   ```
   CLAUDE.md (jako template do wypełnienia)
   .claude/settings.json
   .claude/hooks/ (puste)
   .agent/skills/ (puste)
   ```

2. **Populuj CLAUDE.md** — wypełnij placeholdery na podstawie odpowiedzi:
   - `{{PROJECT_TYPE}}` → "frontend prototype" / "fullstack" / etc.
   - `{{STACK}}` → "Next.js 16 + Tailwind v4" lub własny
   - `{{FIGMA_URL}}` → jeśli podany
   - `{{READ_ONLY_TEXT}}` → włącz sekcję i skopiuj hook jeśli zaznaczone

3. **Design system scaffold** (jeśli zaznaczone):
   - Utwórz `docs/design-system/01-foundations/` z placeholder `colors.md`
   - `02-primitives/`, `03-patterns/`, `04-page-blueprints/` jako puste katalogi z README

4. **Memory** (jeśli zaznaczone):
   - Utwórz `~/.claude/projects/<sanitized-cwd>/memory/MEMORY.md` z nagłówkiem indeksu
   - Skopiuj `project_context.md` template — ale **nie wypełniaj** konkretnymi wartościami, zostaw placeholdery do wypełnienia w pierwszej sesji

### Krok 4 — Weekly audit registration

Jeśli użytkownik wybrał weekly audit:
- Wywołaj `/schedule` skill
- Argument: `"co poniedziałek 10:00: uruchom weekly-audit dla projektu <nazwa-katalogu>"`

### Krok 5 — Raport końcowy

Pokaż użytkownikowi:
```
✓ CLAUDE.md utworzony (typ: frontend prototype)
✓ .claude/ settings.json skopiowany
✓ Memory zainicjalizowana w ~/.claude/projects/.../memory/
✓ Weekly audit zarejestrowany (poniedziałek 10:00)

Zalecane następne kroki:
1. Jeśli istnieje Figma file — powiedz mi URL żebym go zapamiętał
2. Zacommituj początkowy setup: 'chore: initial Claude Code setup'
3. Pracuj normalnie — skille auto-triggerują się
```

## Presety per typ projektu

### Frontend prototype
- CLAUDE.md: skill auto-triggers (browser-verify, design-system-lookup, motion-expert, responsive-verifier)
- Quality gates: UI changes → browser-verify
- Stack placeholder: Next.js/React + Tailwind

### Fullstack mały
- CLAUDE.md: + sekcja API (backend rules)
- Quality gates: browser-verify + testy API przed done
- Stack placeholder: Next.js + API routes + baza

### Design-to-code
- CLAUDE.md: + pełna sekcja DESIGN LANGUAGE (foundations/primitives/patterns/blueprints)
- Quality gates: design-system-lookup przed każdym komponentem, read-only text rule
- Scaffold pełnego `docs/design-system/` jeśli zaznaczone

### Code-to-design
- CLAUDE.md: skill auto-trigger code-to-figma-converter
- Figma file URL wymagane
- Brak read-only text rule (bo pracujesz w Figmie, nie w kodzie marketing site)

## Zasady

- **Nigdy nie nadpisuj bez pytania.** Jeśli plik istnieje w docelowym katalogu — zapytaj.
- **Placeholdery mają być widoczne.** `{{STACK}}` w CLAUDE.md jest lepszy niż "React (nie jestem pewny)". Piotr uzupełni przy pierwszej sesji.
- **Nie włączaj wszystkiego.** Tylko to co użytkownik wybrał. Minimum viable setup.
- **Commit to nie twoje zadanie.** Zaproponuj, ale nie commituj samodzielnie — Piotr decyduje.
