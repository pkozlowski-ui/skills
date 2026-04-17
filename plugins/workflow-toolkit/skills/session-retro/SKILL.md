---
name: session-retro
description: Retrospektywa sesji — podsumowanie co zrobiono, aktualizacja memory projektu, wyłapanie cross-project lessons learned. Uruchamia się gdy użytkownik mówi "zakończ sesję", "kończymy", "tyle na dziś".
---

# Skill: session-retro

## Cel
Na koniec sesji: zapisać to co warto zapamiętać do memory, wyłapać cross-project lessons,
zasugerować commit jeśli są zmiany. Nie pozwolić żeby wiedza z sesji przepadła.

## Auto-trigger
Uruchamia się gdy użytkownik mówi:
- "zakończ sesję" / "kończymy" / "tyle na dziś"
- "skończmy" / "kończę na dziś"
- "zrób retro" / "podsumuj sesję"

## Protokół (5 kroków)

### Krok 1 — Krótkie podsumowanie (dla użytkownika)
1-3 zdania: co zrobiliśmy w tej sesji, co zostało ukończone, co ewentualnie niedokończone.
Format prosty, bez markdown headers:
> "W tej sesji: przeniesione 3 komponenty do v2, naprawiony bug z accent color, dodany skill X. Niedokończone: weryfikacja mobile dla VC page."

### Krok 2 — Ustal co warto zapamiętać

Zbadaj przebieg sesji pod kątem:

**Informacje o projekcie** (→ memory projektu):
- Nowe ścieżki plików które odkryłeś
- Nowe konwencje które poznałeś
- Decyzje projektowe podjęte przez użytkownika
- Stan prac (co w toku, co zaplanowane)

**Feedback użytkownika** (→ memory projektu):
- Korekty które dostałeś ("nie rób X")
- Pochwały dla niestandardowego podejścia (patrz auto-memory guidance)
- Preferencje pracy specyficzne dla tego projektu

**Cross-project lessons** (→ `~/.claude/LESSONS_LEARNED.md`):
- Wzorce które zadziałały — i prawdopodobnie zadziałają w innych projektach
- Błędy które popełniłeś — i można je zapobiec systemowo
- Meta-obserwacje o tym jak pracować z tym użytkownikiem

### Krok 3 — Zapisz do memory projektu

Ścieżka: `~/.claude/projects/<sanitized-cwd>/memory/`

- Jeśli katalog `memory/` nie istnieje — utwórz go
- Jeśli `MEMORY.md` nie istnieje — utwórz z nagłówkiem indeksu
- Dla każdego nowego faktu/feedbacku — utwórz lub zaktualizuj odpowiedni plik .md z frontmatter:
  ```yaml
  ---
  name: Tytuł
  description: Jedna linia
  type: project | feedback | user | reference
  ---
  ```
- Dodaj pointer do MEMORY.md: `- [Title](file.md) — one-line hook`

**NIE DUPLIKUJ.** Jeśli podobna informacja już istnieje — zaktualizuj istniejący plik zamiast tworzyć nowy.

### Krok 4 — Cross-project lessons

Jeśli znalazłeś pattern który prawdopodobnie dotyczy wielu projektów — dopisz do
`~/.claude/LESSONS_LEARNED.md`:

```md
## [YYYY-MM-DD] Krótki tytuł lekcji
- **Context:** projekt X, co się działo
- **Lesson:** co się nauczyłem
- **Rule:** jaka reguła wynika
- **Projects:** [projekt-1]
```

**Kryterium cross-project:** "Czy ten pattern pojawi się też w innym projekcie Piotra?"
Jeśli TAK (z dużym prawdopodobieństwem) → dopisz.
Jeśli NIE (specyficzne dla tego projektu) → memory projektu wystarczy.

### Krok 5 — Sprawdź git i zaproponuj commit

Uruchom `git status` (read-only, auto-allowed).

- **Są niezacommitowane zmiany** → zaproponuj commit: "Mamy X zmienionych plików. Zacommitować? Sugerowany message: '...'"
- **Brak zmian** → "Wszystko już zacommitowane."

Nie commituj automatycznie. Czekaj na potwierdzenie użytkownika.

## Zasady

- **Nie przedłużaj sesji.** Jeśli użytkownik mówi "kończymy" to znaczy że chce kończyć. Retro ma być szybkie (1-2 minuty dialogu), nie nową fazą pracy.
- **Nie wymyślaj faktów.** Jeśli nie pamiętasz dokładnie co się wydarzyło — powiedz że nie jesteś pewny, niech użytkownik dopowie.
- **Memory update jest opcjonalny.** Jeśli sesja nie wniosła nic nowego co warto zapamiętać — powiedz "nic nowego do memory" i przejdź do commitu.
- **LESSONS_LEARNED update jest rzadki.** Nie każda sesja daje cross-project lesson. Większość — nie.

## Przykłady

**Dobry retro (zwięzły):**
> "Sesja: naprawiony overflow hero na mobile, dodane tokeny dla purple accent.
> Memory: zaktualizowany project_context.md o nowy token.
> Brak cross-project lessons.
> Git: 3 pliki zmienione, proponuję commit: 'fix: mobile hero overflow + add vecten-purple token'."

**Zły retro (za długi):**
> ## Podsumowanie
> ### Co zrobiliśmy
> 1. ...
> 2. ...
> 3. ...
> ### Lessons Learned
> ...

Nie pisz tak. Piotr nie lubi podsumowań na końcu. Retro ma być konkretne i krótkie.
