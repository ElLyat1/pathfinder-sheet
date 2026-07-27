# Pathfinder 1e Character Sheet — Design Document

**Date:** 2026-07-27  
**Status:** Approved  
**Author:** opencode

---

## Overview

Single-file web application for creating and managing Pathfinder 1e characters. Dark fantasy aesthetic, offline-capable, localStorage persistence. Deployable to GitHub Pages.

## Constraints

- **Single HTML file** — no build tools, no server, no external dependencies
- **Offline-first** — works after first load via `file://` or static hosting
- **No Google Fonts** — system font stacks only (Cinzel → Georgia, Georgia → Crimson Text)
- **localStorage** — all state persisted per character, debounced auto-save

## Architecture

### File Structure

```
index.html
├── <style>    — CSS (theme, tabs, tables, print)
├── <body>     — HTML (6 tabs, forms, tables)
└── <script>   — JS (state, calculations, persistence, UI)
```

### State Management

Centralized state object with reactive recalculation:

```js
const state = {
  meta: { id, name, created },
  main: { name, class, level, race, alignment, deity, xp, xpNext, gender, age, height, weight, hair, eyes, skin },
  abilityScores: { str: { base, mod }, dex: { base, mod }, con: { base, mod }, int: { base, mod }, wis: { base, mod }, cha: { base, mod } },
  combat: {
    hp: { current, max, temp, nonlethal, status },
    ac: { total, touch, flatFooted, armor, shield, natural, dex, size, deflection, dodge, misc },
    cmb: { total, bab, strMod, size, misc },
    cmd: { total, bab, strMod, dexMod, size, misc },
    saves: { fort: { base, conMod, resistance, misc }, ref: { base, dexMod, resistance, misc }, will: { base, wisMod, resistance, misc } },
    initiative: { total, dexMod, misc },
    bab: { value, attacks: [] },
    speed: { ground, climb, swim, fly, armorReduction },
    dr: '', fastHealing: '', sr: '',
    energyResist: { fire: '', cold: '', electricity: '', acid: '', sonic: '' }
  },
  weapons: [ { name, damage, attackBonus, critRange, critMult, type, hand, range, notes } ], // up to 8
  armor: [ { equipped, name, acBonus, enhBonus, type, heavy, maxDex, checkPenalty, arcaneFail, notes } ], // up to 6
  conditions: { blinded: false, cowering: false, ... },
  skills: {
    acrobatics: { ranks: 0, classSkill: false, competence: 0, circumstance: 0, feat: 0, race: 0, misc: 0, armorPenalty: true },
    // ... full skill list
    knowledgeArcana: { ranks: 0, classSkill: false, trainedOnly: true, ... },
    // ... knowledge, perform, craft, profession
  },
  skillPoints: { available: 0, used: 0 },
  languages: '',
  featsAndFeatures: {
    feats: [ { name, description } ], // up to 20
    classTalents: [ { name, description } ], // up to 15
    classFeatures: [ { name, description } ], // up to 15
    racialFeatures: [ { name, description } ], // up to 10
    proficiencies: ''
  },
  spells: [ // up to 3 spell sets
    {
      casterClass: '',
      casterLevel: 0,
      ability: 'int',
      misc: 0,
      concentration: 0,
      slots: { 0: { total: 0, used: 0 }, 1: { total: 0, used: 0 }, ... 9: { total: 0, used: 0 } },
      spells: { 0: [ { name, expended, notes } ], 1: [ ... ], ... 9: [ ... ] } // up to 12 per level
    }
  ],
  inventory: {
    currency: { platinum: 0, gold: 0, silver: 0, copper: 0 },
    totalValueGP: 0,
    carryingCapacity: { light: 0, medium: 0, heavy: 0 },
    encumbrance: 'Light',
    carriedWeight: 0,
    sizeModifier: 'Medium',
    isQuadruped: false,
    notCarriedLocations: [],
    items: [ { qty, name, valuePerUnit, weightPerUnit, totalValue, totalWeight, location, notes } ] // up to 30
  },
  notes: { background: '', general: '', log: '' }
}
```

### Data Flow

```
Input change → update state[path] → recalculateDerived() → updateDisplay() → debounceSave()
```

1. Input has `data-bind="state.path"` attribute
2. On `input` event: write value to `state` at path
3. Call `recalculate()` — computes all derived fields
4. Update DOM elements with `data-display="derived.path"` attributes
5. After 300ms debounce: `localStorage.setItem(key, JSON.stringify(state))`

### Multi-Character Support

- **Index:** `pf1e_char_index` → `[{ id, name }, ...]`
- **Data:** `pf1e_char_<uuid>` → serialized state
- **Operations:** New, Duplicate, Delete, Rename, Switch
- **Export:** Single character or all characters as JSON
- **Import:** Adds new character (or overwrites by ID with confirmation)

## Visual Design

### Color Palette

| Element | Color |
|---------|-------|
| Background | `#1a1410` (very dark brown) |
| Panel bg | `#2a2018` |
| Panel border | `#6b4c1e` (aged gold/brown) |
| Accent / headers | `#c9a84c` (gold) |
| Text | `#e8d5a3` (warm cream) |
| Input fields | transparent, bottom border `#8b6914` |

### Typography

- Headers: `"Cinzel", Georgia, "Times New Roman", serif`
- Body: `Georgia, "Crimson Text", serif`

### Layout

- Tab navigation at top
- Compact, dense layout (physical character sheet feel)
- `@media print`: hide tabs, stack sections, black-on-white

## Tabs

### Tab 1: MAIN

**Character Info (top bar):**
Name, Class (+ level), Race, Alignment, Deity, XP / XP to next, Gender, Age, Height, Weight, Hair, Eyes, Skin

**Ability Scores:**
STR, DEX, CON, INT, WIS, CHA — base input, auto modifier:
```
modifier = floor((score - 10) / 2)
```

**Hit Points:**
Current / Max / Temp / Nonlethal inputs.
HP Status (informational, not official PF1e rule):
- ≥ 100%: "Feelin' Fine!"
- 50–99%: "Hurt"
- 1–49%: "Bloodied"
- ≤ 0 and > -CON: "Dying"
- ≤ -CON: "Dead"

**Armor Class:**
```
AC = 10 + Armor + Shield + Natural + min(Dex mod, MaxDex) + Size + Deflection + Dodge + Misc
Touch AC = 10 + Dex mod (capped by armor MaxDex) + Size + Deflection + Dodge + Misc
Flat-footed AC = AC - Dex mod - Dodge
```

Size modifiers: Fine +8, Diminutive +4, Tiny +2, Small +1, Medium 0, Large -1, Huge -2, Gargantuan -4, Colossal -8

**CMB / CMD:**
```
CMB = BAB + STR mod + Size mod (CMB-size: Fine -8, Dim -4, Tiny -2, Small -1, Med 0, Large +1, Huge +2, Garg +4, Col +8)
CMD = 10 + BAB + STR mod + DEX mod + Size mod (CMB-style) + Misc
```

**Saves:**
```
Fort = Base Fort + CON mod + Resistance + Misc
Ref  = Base Ref + DEX mod + Resistance + Misc
Will = Base Will + WIS mod + Resistance + Misc
```

**Initiative:**
```
Initiative = DEX mod + Misc
```

**Full Attack Progression:**
```
attacks = [BAB]
if BAB >= 6:  attacks.push(BAB - 5)
if BAB >= 11: attacks.push(BAB - 10)
if BAB >= 16: attacks.push(BAB - 15)
```
Each entry + STR/DEX mod + Size + Misc. Display separately for Melee and Ranged.

**Speed:** Ground / Climb / Swim / Fly — manual inputs.
Heavy armor reduces speed: -10ft for base 30ft, -5ft for base 20ft.

**DR, Fast Healing, SR, Energy Resistances** — manual inputs.

**Weapons table (up to 8):**
Name | Damage | Attack bonus | Crit range | Crit mult | Type (M/R) | Hand | Range | Notes

**Armor table (up to 6):**
Equipped | Name | AC bonus | Enh bonus | Type (A/S/N/D) | Heavy? | Max Dex | Check Penalty | Arcane Fail % | Notes

Equipped toggle recalculates: AC, total check penalty (sum of equipped), MaxDex cap (LOWEST among equipped), speed reduction.

**Common Conditions:** Checkboxes for Blinded, Cowering, Dazzled, Deafened, Entangled, Exhausted, Fatigued, Frightened, Grappled, Panicked, Pinned, Prone, Shaken, Sickened, Stunned, Haste, Prayer, Enlarge, Reduce, Heroism

### Tab 2: SKILLS

Columns: Class Skill | Skill Name | Ability | Total (auto) | Ranks | Class Skill Bonus (auto) | Competence | Circumstance | Feat | Race | Misc | Armor Penalty (auto) | Ability Mod (auto)

```
Class Skill Bonus = 3 if (classSkill AND ranks > 0) else 0
Armor Penalty = -(sum equipped check penalties) if armorPenalty else 0
Total = Ranks + ClassSkillBonus + AbilityMod + Competence + Circumstance + Feat + Race + Misc + ArmorPenalty
```

**Skill list (armor-check-affected marked with \*):**
Acrobatics (Dex\*), Appraise (Int), Bluff (Cha), Climb (Str\*), Diplomacy (Cha), Disable Device (Int\*), Disguise (Cha), Escape Artist (Dex\*), Fly (Dex\*), Handle Animal (Cha), Heal (Wis), Intimidate (Cha), Linguistics (Int), Perception (Wis), Ride (Dex\*), Sense Motive (Wis), Sleight of Hand (Dex\*), Spellcraft (Int), Stealth (Dex\*), Survival (Wis), Swim (Str\*), Use Magic Device (Cha)

Knowledge (Int, trained-only): Arcana, Dungeoneering, Engineering, Geography, History, Local, Nature, Nobility & Royalty, Religion, The Planes, Other

Perform (Cha): Act, Comedy, Dance, Keyboard, Oratory, Percussion, Sing, String Instrument, Wind Instrument, Other

Craft ×5 (Int, custom name), Profession ×5 (Wis, trained-only, custom name)

Skill Points: Manual "Available" field, auto-summed "Used" from all ranks.

Languages: Free-text comma-separated.

### Tab 3: ABILITIES

- **Feats** — up to 20 rows: Name | Description (collapsible textarea)
- **Class Talents** — up to 15 rows: Name | Description
- **Class Features** — up to 15 rows: Name | Description
- **Racial Features & Traits** — up to 10 rows: Name | Description
- **Proficiencies** — free text area

### Tab 4: SPELLS

Up to 3 spell sets. Each set:
- Caster class label, Caster Level input, Ability selector (Int/Wis/Cha), Misc
- Concentration = Caster Level + Ability Mod + Misc
- Spell slots 0–9: Total input + Used counter + remaining auto
- Spell list 0–9: up to 12 slots each — Name | Expended | Notes

### Tab 5: INVENTORY

**Currency:** Platinum / Gold / Silver / Copper
```
Total GP = Platinum×10 + Gold + Silver/10 + Copper/100
```

**Carrying Capacity (PF1e Core Rulebook Table):**
```
STR 1–20 light/medium/heavy (lbs):
1: 3/6/10, 2: 6/13/20, 3: 10/20/30, 4: 13/26/40, 5: 16/33/50
6: 20/40/60, 7: 23/46/70, 8: 26/53/80, 9: 30/60/90, 10: 33/66/100
11: 38/76/115, 12: 43/86/130, 13: 50/100/150, 14: 58/116/175
15: 66/133/200, 16: 76/153/230, 17: 86/173/260, 18: 100/200/300
19: 116/233/350, 20: 133/266/400
```

STR 21+: each +10 STR quadruples capacity. **Note:** This formula is approximate — verify against SRD (d20pfsrd.com/gamemastering/carrying-capacity) for edge cases like STR 30, 40 before shipping.

Size modifiers: Large ×2, Huge ×4, Small ×3/4. Quadruped: ×1.5.

Derived: Lift Over Head = Heavy, Lift Off Ground = Heavy×2, Push/Drag = Heavy×5.

Encumbrance: Informational badge only (Medium: -3 skill penalty, max Dex +3, run x3; Heavy: -6, max Dex +1, speed reduced). NOT auto-applied to skills tab.

**Inventory table (up to 30):**
Qty | Item Name | Value/unit (gp) | Weight/unit (lbs) | Total Value (auto) | Total Weight (auto) | Location | Notes

Items in non-carried locations excluded from encumbrance calculation.

### Tab 6: NOTES

Three text areas: Background/Backstory, General Notes, Party/Campaign Log.

## Key Formulas Summary

| Derived | Formula |
|---------|---------|
| Ability Mod | `floor((score - 10) / 2)` |
| AC | `10 + Armor + Shield + Natural + min(Dex, MaxDex) + Size + Deflection + Dodge + Misc` |
| Touch AC | `10 + Dex(mod) + Size + Deflection + Dodge + Misc` |
| Flat-footed | `AC - Dex - Dodge` |
| CMB | `BAB + Str(mod) + Size(CMB)` |
| CMD | `10 + BAB + Str(mod) + Dex(mod) + Size(CMB)` |
| Fort | `BaseFort + Con(mod) + Resistance + Misc` |
| Ref | `BaseRef + Dex(mod) + Resistance + Misc` |
| Will | `BaseWill + Wis(mod) + Resistance + Misc` |
| Initiative | `Dex(mod) + Misc` |
| Full Attack | `[BAB, BAB-5(if≥6), BAB-10(if≥11), BAB-15(if≥16)]` + mods |
| Skill Total | `Ranks + ClassSkillBonus + AbilityMod + Competence + Circumstance + Feat + Race + Misc + ArmorPenalty` |
| Concentration | `CasterLevel + AbilityMod + Misc` |
| Total GP | `Plat×10 + Gold + Silver/10 + Copper/100` |

## Print CSS

- Hide tab navigation
- Show all 6 sections stacked vertically
- Black text on white background
- Page breaks between major sections
- Hide interactive elements (buttons, toggles)

## Build Order

1. HTML skeleton + tab navigation + dark theme CSS
2. Multi-character selector + localStorage scaffolding
3. Character info + ability scores + modifiers
4. HP / AC / CMB-CMD / Saves / Initiative / full-attack progression
5. Weapons and armor tables with equipped toggle
6. Skills tab with full auto-calculation
7. Spells tab
8. Inventory tab with carrying capacity table and encumbrance
9. Abilities/Feats tab
10. Notes tab
11. Export/Import JSON
12. Print CSS
13. Final polish and responsive layout check

## Deployment

1. GitHub repo `pf1e-character-sheet`
2. `index.html` in root
3. GitHub Pages enabled (main branch)
4. README.md with usage instructions

## Out of Scope (v1)

- Auto-applying condition effects to calculations
- Auto-deriving skill points from class/level/Int
- Spell slot auto-generation from class tables
- Equipment cost/weight auto-lookups
- Digital dice rolling
- Character portraits
