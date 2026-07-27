# Pathfinder 1e Character Sheet — v2: Bug Fixes, i18n, Visual Redesign

**Date:** 2026-07-27
**Status:** Approved
**File:** `index.html` (single-file app, ~2000 lines)

## Overview

This pass addresses two critical bugs, adds RU/EN language toggle, and applies a comprehensive visual redesign. The goal is to transform a functional-but-broken form into a polished, game-UI-style character sheet.

---

## Bug Fixes

### Bug 1 (CRITICAL): Input focus loss after every keystroke

**Root cause:** `recalculate()` calls 6 render functions (`renderSkillsTable`, `renderKnowledgeTable`, `renderPerformTable`, `renderCraftProfession`, `renderSpellSets`, `renderInventoryTable`) which all rebuild DOM via `innerHTML`. This destroys the `<input>` element the user is typing in, causing focus loss.

**Fix — Selective re-render architecture:**

1. **`recalculate()`** becomes pure computation — updates `state.*` values only. Zero DOM touches.
2. **`updateDerived()`** — new function that updates only `[data-display]` spans and derived cells. Never touches `<input>` elements.
3. **`renderAllDynamicTables()`** — new function that does full re-render of dynamic tables. Called ONLY on:
   - Character switch
   - Tab switch (first time visiting a tab)
   - Add/remove rows (weapons, armor, skills, spells, inventory, feats)
   - Page load
4. **Input event handler** calls `recalculate()` → `updateDerived()`. Never calls render functions.
5. **Dynamic table cells** that show derived values (skill totals, inventory totals, etc.) use `data-display` attributes and get updated by `updateDerived()` instead of being rebuilt.

**Derived cell update mechanism:**
- Table cells with computed values get `data-display="state.path"` attributes
- `updateDerived()` iterates `[data-display]` and sets `textContent` from state
- This avoids innerHTML rebuilds while keeping values current

### Bug 2: CS checkbox in Skills tab doesn't respond to clicks

**Root causes:**
1. `bindInputs()` uses `clone.value = existing` — but checkboxes need `clone.checked = existing`
2. The `input` event may not fire reliably on checkboxes in all browsers — needs `change` event

**Fix:**
- In `bindInputs()`, detect checkbox type: if `clone.type === 'checkbox'`, use `clone.checked = !!existing`
- Add both `input` and `change` event listeners for checkboxes
- For CS checkbox: `change` handler updates `state.skills[key].classSkill`, then calls `recalculate()` → `updateDerived()` to update the skill total display

---

## Language Toggle (RU / EN)

### Architecture

```javascript
const i18n = {
  en: { tab_main: "Main", tab_skills: "Skills", /* ... */ },
  ru: { tab_main: "Основное", tab_skills: "Навыки", /* ... */ }
};
let currentLang = localStorage.getItem('pf18n_lang') || 'en';
```

### Toggle mechanism
- Small "RU / EN" button in the top bar, left of the character selector
- On click: toggles `currentLang`, saves to `pf18n_lang` localStorage key, calls `applyLanguage()`
- `applyLanguage()` queries all elements with `data-i18n="key"` and sets `textContent`
- For placeholders: queries `data-i18n-placeholder` and sets the placeholder attribute

### Translation scope

**Translated (interface chrome):**
- Tab names
- Panel titles
- Field labels
- Button text
- Table headers
- Fixed skill names (Acrobatics → Акробатика, etc.)
- Knowledge/Perform type labels
- Status text ("Feelin' Fine!", "Hurt", etc.)
- Carrying capacity labels
- Condition names

**NOT translated (user data):**
- Character name, race, class, alignment
- Notes, background, campaign log
- Custom skill names (Craft/Profession)
- Custom weapon/armor/item names
- Any user-entered text in inputs/textareas

### Russian terminology

| English | Russian | Notes |
|---------|---------|-------|
| Armor Class (AC) | Класс Доспеха (КД) | Standard RU PF |
| Hit Points (HP) | Хиты (ХП) | Standard |
| Initiative | Инициатива | Standard |
| BAB | БМА | Базовый Модификатор Атаки |
| CMB | МБМ | Модификатор Боевых Маневров |
| CMD | ЗБМ | Защита от Боевых Маневров |
| Fortitude | Стойкость | Standard |
| Reflex | Рефлекс | Standard |
| Will | Воля | Standard |
| STR/DEX/CON/INT/WIS/CHA | Keep Latin | Standard in RU PF |
| Feats | Черты | Standard |
| Class Talents | Таланты класса | |
| Class Features | Классовые способности | |
| Racial Features | Расовые особенности | |
| Skills | Навыки | |
| Spells | Заклинания | |
| Inventory | Инвентарь | |
| Carrying Capacity | Несущая способность | |

---

## Visual Redesign

### Layout

- **Main/Abilities/Notes tabs:** centered column, max-width 550px
- **Skills/Inventory/Spells tabs:** wider layout, max-width 950px (tables need room)
- CSS class `.tab-panel.wide` for table-heavy tabs
- On screens < 700px: all tabs go full-width

### Background

CSS-only atmospheric background (no external images):
- Base: dark gradient (`#0d0b0f` → `#1a1410`)
- Layered radial gradients for depth (dark navy, deep brown)
- Subtle noise texture via `repeating-conic-gradient` at very low opacity
- Dark vignette at edges via radial gradient overlay

### Title Banner

- "PATHFINDER" text at top of page, above the tab bar
- CSS text styling: `font-variant: small-caps`, heavy letter-spacing (0.3em)
- Gold gradient text effect via `background-clip: text` with `linear-gradient`
- Original typography treatment — not copying Paizo's logo design

### Wood Panel Texture

CSS-generated wood grain on `.panel` elements:
- Repeating linear gradients with slight color variations in brown tones
- Multiple gradient layers at slight angles for organic feel
- Beveled frame effect: `box-shadow` with outer highlight (`0 1px 0 rgba(255,255,255,0.1)`) + inner shadow (`inset 0 2px 4px rgba(0,0,0,0.5)`)
- Border: 1px solid with subtle gradient effect

### Input Styling

- Recessed/inset look: `box-shadow: inset 0 2px 4px rgba(0,0,0,0.5)`
- Background: slightly lighter than panel (`rgba(255,255,255,0.05)`)
- Focus glow: `box-shadow: inset 0 2px 4px rgba(0,0,0,0.5), 0 0 8px rgba(201,168,76,0.5)`
- Border-bottom: 1px solid `#8b6914` (existing, keep)
- Text color: `#e8d5a3` (existing, keep)

### Button Styling

- Gradient background: subtle dark-to-darker brown (`#6b4c1e` → `#4a3515`)
- Border: 1px solid `#8b6914`
- Border-radius: 3px
- Active state: `box-shadow: inset 0 2px 4px rgba(0,0,0,0.5)`, slightly darker background
- Hover: lighter background + subtle glow

### Custom Checkboxes

- Hide default browser checkbox (`appearance: none` or `display: none` + reposition)
- Custom `<span>` indicator: 16x16px square, 1px solid `#8b6914` border
- Checked state: gold background (`#c9a84c`) with dark checkmark (✓ via `::after` pseudo-element or Unicode)
- Transition: smooth 0.15s on background/border color

### Tab Navigation

- Active tab: gold underline (`border-bottom: 2px solid #c9a84c`), brighter text (`#c9a84c`)
- Inactive: dimmer text (`#8b6914`), no underline
- Hover: subtle text brightening + faint glow

### Section Headers (Panel Titles)

- Ornamental divider pattern: `═══ ✦ ═══` flanking the text
- Implemented via `::before` and `::after` pseudo-elements with CSS content
- Small decorative diamond/rune character in the center
- Color: `#c9a84c` (gold, consistent with existing)

### General Polish

- Consistent spacing: 12px panel padding, 8px field-group margin, 10px grid gap
- Font sizes: 14px body, 16px panel titles, 11px field labels, 12px table headers
- Scrollbar styling: dark track, gold thumb (existing, keep)
- Contrast: text remains readable against wood texture and background

---

## Build Order

1. Fix Bug 1 (input focus loss) — blocks everything else being testable
2. Fix Bug 2 (CS checkbox)
3. Add language toggle + RU translations
4. Redesign: page background + title banner
5. Redesign: centered wood-panel layout (wider for table tabs)
6. Redesign: input/button/checkbox styling
7. Final pass: verify no regressions on calculation logic

---

## Constraints

- Single-file app — no external dependencies, no build tools
- System font stacks only: `"Cinzel", Georgia, "Times New Roman", serif` (headers); `Georgia, "Crimson Text", serif` (body)
- localStorage persistence with 300ms debounce
- All PF1e formulas must remain correct after DOM changes
- Print CSS must still work (black-on-white, tabs hidden)
- Responsive layout must still work on mobile

---

## Verification Checklist

After all changes:
- [ ] Type a full sentence into Name field without mouse — cursor preserved, no refocus
- [ ] Toggle CS checkbox on 3 skills — each visually updates AND total recalculates (+3 when ranks > 0)
- [ ] Switch language RU ↔ EN — all labels update, no data loss
- [ ] Refresh page — language preference persists
- [ ] All AC/saves/CMB/CMD calculations still correct
- [ ] Skills totals still correct with class skill + armor penalty
- [ ] Carrying capacity still correct for STR scores
- [ ] Print preview still clean (black-on-white)
- [ ] Responsive layout works on narrow screens
- [ ] Export/Import still works with all data
