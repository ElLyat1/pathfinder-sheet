# Spell Database Design — Pathfinder 1e Character Sheet

## Overview

Add a searchable spell browser to the existing Spells tab. Users can browse ~550 PF1e spells (CRB + APG), view full descriptions, and add spells to their character's spell list. Manual entry remains available.

## Scope

- **Sources**: CRB + Advanced Player's Guide (~550 spells)
- **Spell data**: Full descriptions (name, level, school, casting time, range, duration, components, effect, save, resistance, description text)
- **Storage**: External `spells.json` file, lazy-loaded on first browser open
- **UI**: Full-screen overlay within the existing Spells tab (no new tab)
- **Language**: UI elements translated RU/EN; spell names stay in English (original)

## Architecture

### Files

```
index.html        — main app (modified: Spells tab + browser overlay + JS)
spells.json       — spell database (~550 entries, ~250-350KB)
```

### Spell Data Format (`spells.json`)

```json
[
  {
    "name": "Magic Missile",
    "level": 1,
    "school": "Evocation",
    "subschool": null,
    "descriptors": ["force"],
    "classes": [{"name": "Wizard", "level": 1}],
    "castingTime": "1 standard action",
    "range": "Medium (100 ft. + 10 ft./level)",
    "components": "V, S",
    "material": null,
    "focus": null,
    "duration": "Instantaneous",
    "area": null,
    "effect": "1 missile",
    "targets": null,
    "save": "None",
    "resistance": "Yes",
    "description": "A glowing sphere of energy..."
  }
]
```

### Spell Classes Reference

Each spell has a `classes` array with class name and the level at which the class gains access:

```json
"classes": [
  {"name": "Wizard", "level": 1},
  {"name": "Sorcerer", "level": 1}
]
```

The `class` field matches against the Spell Set's `casterClass` for pre-filtering.

## UI Design

### Integration Point

On the Spells tab, each Spell Set gets a "Browse" button next to the Spell List header:

```
Spell Set 1 — Wizard
  Spell List              [Browse]
  Level 0: ...
  Level 1: ...
```

- "Browse" is only enabled when `casterClass` is set in the Spell Set
- "Browse" is disabled/greyed when `casterClass` is empty

### Browser Overlay

Full-screen dark overlay with:

```
┌─────────────────────────────────────────┐
│  ✕ Close        Spell Browser           │
├─────────────────────────────────────────┤
│  [🔍 Search...]  Class ▼  Level ▼  School ▼ │
├─────────────────────────────────────────┤
│  ⚡ Magic Missile     Lv1  Evocation   │
│  🔥 Fireball         Lv3  Evocation   │
│  💚 Cure Light Wounds Lv1  Conjuration│
│  ... (scrollable list, max-height: calc)│
└─────────────────────────────────────────┘
```

When a spell is clicked, its description expands inline (accordion):

```
│  ⚡ Magic Missile     Lv1  Evocation   │
│  ┌─────────────────────────────────┐    │
│  │  Evocation [force]              │    │
│  │  Casting: 1 standard action     │    │
│  │  Range: Medium (100+10/level)   │    │
│  │  Components: V, S               │    │
│  │  Duration: Instantaneous        │    │
│  │  ─────────────────────          │    │
│  │  Full description text here...  │    │
│  │  [Add to spell list]            │    │
│  └─────────────────────────────────┘    │
```

### Filters

| Filter | Type | Options |
|--------|------|---------|
| Search | Text input | Fuzzy match on spell name |
| Class | Dropdown | All, Wizard, Cleric, Druid, Bard, Sorcerer, Paladin, Ranger, etc. |
| Level | Dropdown | All, 0 (Cantrip), 1, 2, 3, 4, 5, 6, 7, 8, 9 |
| School | Dropdown | All, Abjuration, Conjuration, Divination, Enchantment, Evocation, Illusion, Necromancy, Transmutation |

- Filters update the list instantly (Array.filter on ~550 items)
- When opened from a Spell Set with `casterClass` set, the Class filter auto-selects that class

### Add Flow

1. User clicks "Add to spell list" in an expanded spell card
2. A small inline picker appears: "Add to Level: [0] [1] [2] ... [9]"
3. User clicks a level number
4. The spell is added to `state.spells[setIndex].spells[level]` with:
   - `name`: spell name
   - `notes`: spell description (truncated or full)
   - `expended`: false
5. Overlay closes
6. `renderAllDynamicTables()`, `bindInputs()`, `recalculate()`, `updateDerived()`, `saveToStorage()` are called

## Data Loading

### Lazy Load Strategy

```javascript
let spellDatabase = null;

async function loadSpellDatabase() {
  if (spellDatabase) return spellDatabase;
  const resp = await fetch('spells.json');
  spellDatabase = await resp.json();
  return spellDatabase;
}
```

- Loaded on first "Browse" click
- Cached in memory for session
- No IndexedDB needed (550 items fits in memory)

### Pre-filtering

When the Spell Set has a `casterClass`:
1. On browser open, auto-set Class filter to that class
2. Only show spells where `classes[].name` matches `casterClass`

When `casterClass` is empty:
- Show all spells, no pre-filter

## Spell Database Content

### Sources

- **Pathfinder Core Rulebook (CRB)**: ~400 spells
- **Advanced Player's Guide (APG)**: ~150 spells (Alchemist, Inquisitor, Oracle, Summoner, Witch)

### Classes Included

| Class | Source | Spells |
|-------|--------|--------|
| Wizard | CRB | ~400 |
| Sorcerer | CRB | ~400 |
| Cleric | CRB | ~400 |
| Druid | CRB | ~400 |
| Bard | CRB | ~200 |
| Paladin | CRB | ~50 |
| Ranger | CRB | ~50 |
| Alchemist | APG | ~100 |
| Inquisitor | APG | ~100 |
| Oracle | APG | ~200 |
| Summoner | APG | ~100 |
| Witch | APG | ~200 |

### Spell Count Target

~550 unique spells (some shared across classes).

## i18n Keys

### EN

```javascript
btn_browse_spells: "Browse",
spell_browser_title: "Spell Browser",
filter_class: "Class",
filter_level: "Level",
filter_school: "School",
filter_all: "All",
lbl_cantrip: "Cantrip",
btn_add_to_list: "Add to spell list",
lbl_add_to_level: "Add to Level:",
lbl_casting_time: "Casting Time",
lbl_components: "Components",
lbl_material: "Material",
lbl_focus: "Focus",
lbl_area: "Area",
lbl_targets: "Targets",
lbl_save: "Save",
lbl_resistance: "Spell Resistance",
```

### RU

```javascript
btn_browse_spells: "Поиск",
spell_browser_title: "Реестр заклинаний",
filter_class: "Класс",
filter_level: "Уровень",
filter_school: "Школа",
filter_all: "Все",
lbl_cantrip: "Заговор",
btn_add_to_list: "Добавить в список",
lbl_add_to_level: "Добавить на уровень:",
lbl_casting_time: "Время сотворения",
lbl_components: "Компоненты",
lbl_material: "Материальный",
lbl_focus: "Фокус",
lbl_area: "Область",
lbl_targets: "Цели",
lbl_save: "Спасбросок",
lbl_resistance: "Сопротивление магии",
```

## Implementation Steps

1. Create `spells.json` with ~550 PF1e CRB+APG spells
2. Add spell browser overlay HTML to `index.html`
3. Add spell browser CSS (dark theme matching existing)
4. Add spell browser JS (loadSpellDatabase, openSpellBrowser, filterSpells, renderSpellList, addSpellFromBrowser)
5. Add "Browse" button to Spell List header in renderSpellSets
6. Add i18n keys for browser UI
7. Test: add spells from browser, verify they appear in spell list
8. Test: RU/EN language toggle works in browser
9. Test: pre-filtering by casterClass works
10. Commit and push

## Constraints

- Single-file architecture maintained (spells.json is the only external file)
- No build tools, no npm, no frameworks
- Dark fantasy theme consistent with existing UI
- Print CSS: spell browser overlay hidden in print mode
- Offline-capable (spells.json served from same origin)
