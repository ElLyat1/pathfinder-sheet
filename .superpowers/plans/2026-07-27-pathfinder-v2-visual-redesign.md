# Plan: Visual Redesign — Game-UI Feel

## Problem Statement

The current v2 visual redesign has several issues:
1. **Buttons overflow** — 8 buttons in `.char-selector` don't fit on smaller screens; no wrapping/scrolling
2. **No mountain background** — current `body::before` is just dark radial gradients, no visible mountain silhouette
3. **Wood texture too subtle** — the `repeating-linear-gradient` at 6px cycle is barely visible
4. **Not game-like enough** — needs stronger visual hierarchy, glowing accents, panel depth, and atmospheric elements

## Design Direction

Think: **Pathfinder Kingmaker / Baldur's Gate character sheet** — dark parchment tones, visible atmospheric background, heavy ornamental borders, glowing gold accents, recessed input fields that feel like slots in a stone/wood UI.

## Changes

### 1. Mountain Silhouette Background (CSS-only)

Replace `body::before` with a layered CSS mountain landscape:

```
body::before — 3 layers:
  1. Sky gradient: dark purple-to-blue (#0d0b1a → #1a1025 → #0d0b0f)
  2. Mountain silhouettes: 3 polygon-like shapes using linear-gradient at angles
     - Far mountains: dark (#0a0810), smaller peaks
     - Mid mountains: slightly lighter (#12101a), medium peaks  
     - Near mountains: darkest (#080610), large peaks
  3. Ground: solid dark (#0d0b0f)
```

Use `clip-path: polygon()` on pseudo-elements OR layered `linear-gradient` with hard stops to create mountain shapes. The mountains should be subtle — dark silhouettes against a slightly lighter sky, visible but not distracting.

### 2. Improved Wood Panel Texture

Replace the current `.panel` background with a richer, more visible wood grain:

- Increase contrast between grain lines (current 2px cycle too tight)
- Add a subtle warm tint to the base color
- Add a "worn" look with slight color variation
- Add a visible inner border/groove effect

New texture approach:
```css
.panel {
  background:
    /* Wood grain — horizontal lines with natural variation */
    repeating-linear-gradient(
      180deg,
      rgba(58, 42, 28, 0.6) 0px,
      rgba(42, 30, 18, 0.8) 1px,
      rgba(52, 38, 24, 0.7) 3px,
      rgba(38, 26, 16, 0.9) 4px,
      rgba(48, 35, 22, 0.6) 6px,
      rgba(44, 32, 20, 0.8) 8px
    ),
    /* Vertical grain highlight */
    repeating-linear-gradient(
      90deg,
      rgba(80, 60, 35, 0.15) 0px,
      transparent 2px,
      transparent 8px
    ),
    /* Base color */
    linear-gradient(180deg, #2e2218 0%, #261c12 100%);
  border: 2px solid #5a421e;
  box-shadow:
    /* Outer bevel — light top, dark bottom */
    0 -1px 0 rgba(139, 105, 20, 0.3),
    0 1px 0 rgba(0, 0, 0, 0.5),
    /* Inner groove */
    inset 0 1px 0 rgba(255, 255, 255, 0.06),
    inset 0 -1px 0 rgba(0, 0, 0, 0.3),
    /* Drop shadow */
    0 4px 12px rgba(0, 0, 0, 0.5);
}
```

### 3. Game-UI Input Styling

Make inputs look like **recessed slots** in a game HUD:

- Deeper inset shadow
- Subtle inner glow on focus (gold pulse)
- Thicker bottom border (like a carved groove)
- Slightly larger padding for touch targets

### 4. Game-UI Button Styling

Make buttons look like **clickable game buttons** (think: inventory buttons in RPG):

- Thicker borders with bevel effect
- Gradient that suggests depth (light top, dark bottom)
- Stronger hover glow
- Active state that "presses in" convincingly
- Slightly rounded corners (4-5px)

### 5. Character Bar Overflow Fix

Add `flex-wrap: wrap` to `.char-selector` so buttons wrap to a second line on narrow screens. Also reduce button padding slightly and add `gap: 6px` to tighten spacing.

### 6. Title Banner Enhancement

Make the title more dramatic:
- Add a subtle glow/bloom effect behind the text
- Add ornamental lines (decorative rules) above and below
- Increase font size slightly

### 7. Tab Navigation Enhancement

Make tabs feel more like game menu tabs:
- Add background gradient to active tab
- Add subtle border/separator between tabs
- Add glow effect on active tab

## Files to Modify

- `/Users/ermand/Documents/Default Project/index.html` — CSS section only (lines 7-340)

## Acceptance Criteria

- [ ] Mountain silhouettes visible in background (dark, atmospheric, not distracting)
- [ ] Wood panel texture is clearly visible and looks like actual wood
- [ ] Inputs look like recessed game UI slots
- [ ] Buttons look like clickable game UI buttons with depth
- [ ] All 8 character bar buttons fit (wrap if needed)
- [ ] Title banner has glow effect and ornamental decorations
- [ ] Active tab has visual distinction (background/glow)
- [ ] Overall feel is "dark fantasy game UI" not "web form"
- [ ] No external dependencies (pure CSS)
- [ ] Print CSS still works (backgrounds hidden)
