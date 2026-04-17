# Design System

Czterowarstwowa hierarchia:

| Warstwa | Zawiera | Katalog |
|---|---|---|
| **01-foundations** | Kolory, typografia, spacing, motion, breakpoints | `01-foundations/` |
| **02-primitives** | Button, Card, Badge, Input, Tile — building blocks | `02-primitives/` |
| **03-patterns** | Hero, CTA, Metrics, Timeline — kompozycje primitives | `03-patterns/` |
| **04-page-blueprints** | Układ całych stron — kolejność sekcji, whitelist komponentów | `04-page-blueprints/` |

## Zasada centralnego zarządzania

Każdy element wizualny musi być zarządzalny z tego poziomu. Zmiana tutaj propaguje
się na cały projekt — strony/komponenty nie definiują własnych wartości.

## Jak dodać nowy element

1. **Foundation** (kolor/token) → dopisz do `01-foundations/` i do CSS variables
2. **Primitive** (nowy komponent atomowy) → dokumentacja w `02-primitives/`, kod w components
3. **Pattern** (złożenie primitives) → dokumentacja w `03-patterns/`, kod w components
4. **Blueprint** (cała strona) → `04-page-blueprints/` opisuje strukturę
