# Claude Project Template

Szkielet do inicjalizacji nowego projektu w ekosystemie Piotra.

## Jak użyć

**Automatycznie (zalecane):**
1. Przejdź do nowego katalogu projektu
2. Otwórz Claude Code (VS Code extension / Desktop)
3. Powiedz: **"zainicjalizuj ten projekt"**
4. Skill `init-project` pyta o typ i kopiuje tu odpowiednie pliki

**Manualnie:**
```
cp -r ~/Documents/claude-project-template/CLAUDE.md ./
cp -r ~/Documents/claude-project-template/.claude ./
cp -r ~/Documents/claude-project-template/.agent ./
# Wypełnij placeholdery {{...}} w CLAUDE.md
```

## Co jest w środku

- `CLAUDE.md` — template z placeholderami
- `.claude/settings.json` — permissions + Stop hook do retro
- `.claude/hooks/` — puste, na project-specific hooki
- `.agent/skills/` — puste, na project-specific skille
- `docs/design-system/` — szkielet 4-warstwowej hierarchii (tylko dla design-to-code)
- `memory-bootstrap/` — templates do podpięcia do `~/.claude/projects/<cwd>/memory/`

## Placeholdery do wypełnienia

W `CLAUDE.md`:
- `{{PROJECT_NAME}}`
- `{{PROJECT_TYPE}}` — frontend prototype / fullstack / design-to-code / code-to-design
- `{{STACK}}` — Next.js 16 + Tailwind v4 / itp.
- `{{FIGMA_URL}}` — opcjonalny URL
- `{{QUALITY_GATES}}` — bloker(y) project-specific, np. read-only text
- `{{FORBIDDEN_PATTERNS}}` — czego NIE robić
- `{{NOTES}}` — cokolwiek istotnego dla tego projektu
