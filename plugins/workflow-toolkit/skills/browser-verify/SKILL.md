---
name: browser-verify
description: Weryfikacja wizualna UI w przeglądarce (desktop + mobile) przed zgłoszeniem zadania jako ukończonego. Uruchamia się automatycznie po każdej zmianie UI.
---

# Skill: browser-verify

## Cel
Potwierdzenie że zmiany UI wyglądają poprawnie w przeglądarce — na desktop i mobile —
przed zgłoszeniem zadania jako ukończonego.

## Auto-trigger
Uruchamia się **automatycznie po każdej zmianie UI**, bez pytania użytkownika.
Wyjątek: zmiany wyłącznie w logice/danych, bez wpływu na warstwę wizualną.

## Protokół (5 kroków)

### Krok 1 — Dev server
Sprawdź czy serwer developerski już działa. Jeśli nie, uruchom go standardową komendą projektu:
- Next.js / React: `npm run dev` (port 3000 domyślnie)
- Vite: `npm run dev` (port 5173)
- Inne: sprawdź `package.json` → `scripts.dev`

Poczekaj na komunikat "Ready" / "Local:" przed kontynuowaniem.

### Krok 2 — Screenshot desktop
Otwórz zmodyfikowaną stronę/komponent w przeglądarce przy viewport **1440px**.
Użyj jednego z:
- `mcp__claude_ai_Figma__get_screenshot` z URL strony
- Playwright screenshot: `npx playwright screenshot --viewport-size=1440,900 <url> /tmp/desktop.png`
- Headless Chrome: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --screenshot=/tmp/desktop.png --window-size=1440,900 <url>`

### Krok 3 — Screenshot mobile
Zmień viewport na **375px** (iPhone SE / standardowy mobile).
Ta sama metoda co wyżej, tylko `--window-size=375,812`.

### Krok 4 — Checklist wizualny
Sprawdź oba screenshoty pod kątem:

| Punkt | Opis |
|-------|------|
| Nakładające się elementy | Żaden element nie zachodzi na drugi |
| Tekst niezmieniony | Copy identyczne z oryginałem (jeśli projekt ma READ-ONLY TEXT rule) |
| Brak broken layout | Grid/flex renderuje się prawidłowo na obu breakpointach |
| Spacing/padding | Rytm odstępów zgodny z design systemem projektu |
| Animacje | Fade-in, scroll triggers nie powodują flash ani skoku pozycji |
| Kolory | Zgodne z tokenami projektu (brak przypadkowych hex) |
| Brak console errors | Brak red errors w konsoli przeglądarki |

### Krok 5 — Decyzja
- **Wszystko OK** → zgłoś zadanie jako ukończone, pokaż screenshoty użytkownikowi
- **Problem znaleziony** → napraw, wróć do Kroku 2
- **Problem niejasny** → zapytaj użytkownika zanim wprowadzisz zmiany

## Integracja z innymi skillami
- Po `subpage-redesign-protocol` — uruchom `responsive-verifier` PRZED `browser-verify`
  (responsywność najpierw, pełna weryfikacja wizualna na końcu)
- Po dużej zmianie layoutu — zawsze `responsive-verifier` najpierw

## Uwagi
- Nigdy nie pomijaj tego skilla "bo zmiana była mała" — najmniejsza zmiana CSS może wpłynąć na layout
- Screenshots są dowodem dla użytkownika że weryfikacja się odbyła — zawsze je pokazuj
- Jeśli serwer dev nie odpala się — zgłoś problem, nie zgaduj że "pewnie działa"
