# Spell Database Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a searchable spell browser to the Spells tab with ~550 PF1e CRB+APG spells, lazy-loaded from `spells.json`.

**Architecture:** External `spells.json` file with full spell data, lazy-loaded on first browser open. Full-screen overlay UI with filters (class, level, school, search). Spells added to character via level picker confirmation.

**Tech Stack:** Vanilla JS, CSS, HTML. No frameworks. `fetch()` for lazy loading. Single-file app + one JSON file.

## Global Constraints

- Single-file architecture: `index.html` + `spells.json` only
- No build tools, npm, or external dependencies
- Dark fantasy theme: bg `#1a1410`, panel `#2a2018`, border `#6b4c1e`, accent `#c9a84c`, text `#e8d5a3`
- System font stacks: `"Cinzel", Georgia, "Times New Roman", serif` (headers)
- i18n via `data-i18n` attributes + `i18n.en`/`i18n.ru` dictionaries
- All inputs dark: bg `#3a2a15`, text `#f0e0b0`
- `applyLanguage()` called after all DOM rebuilds
- No comments (`<!-- -->`) in HTML unless user requests

---

### Task 1: Create `spells.json` — PF1e CRB+APG Spell Database

**Files:**
- Create: `spells.json`

**Data scope:** ~550 unique spells from Pathfinder Core Rulebook + Advanced Player's Guide. Each spell needs: name, level, school, subschool, descriptors, classes (with level), castingTime, range, components, material, focus, duration, area, effect, targets, save, resistance, description.

**Spell list sources (CRB):**
- Wizard/Sorcerer: ~400 spells (levels 0-9)
- Cleric/Druid: ~400 spells (levels 0-9)
- Bard: ~200 spells (levels 0-6)
- Paladin: ~50 spells (levels 1-4)
- Ranger: ~50 spells (levels 1-4)

**Spell list sources (APG):**
- Alchemist: ~100 extracts (levels 1-6)
- Inquisitor: ~100 spells (levels 1-6)
- Oracle: ~200 spells (levels 0-9)
- Summoner: ~100 spells (levels 1-6)
- Witch: ~200 spells (levels 0-9)

**Description policy:** Include meaningful descriptions (2-5 sentences) for each spell. For very long spells, truncate to key mechanics.

- [ ] **Step 1: Research PF1e spell lists**

Compile spell lists from d20pfsrd.com or paizo.com SRD for each class. Create a master list of unique spell names.

- [ ] **Step 2: Create spells.json structure**

Start with the JSON array structure. Add ~50 high-priority spells first (Magic Missile, Fireball, Cure Light Wounds, etc.) to validate the format.

Spell entry format:
```json
{
  "name": "Magic Missile",
  "level": 1,
  "school": "Evocation",
  "subschool": null,
  "descriptors": ["force"],
  "classes": [{"name": "Wizard", "level": 1}, {"name": "Sorcerer", "level": 1}],
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
  "description": "A glowing sphere of energy streaks from your hand and strikes its target unerringly."
}
```

- [ ] **Step 3: Populate CRB spells (~400 spells)**

Add all CRB spells organized by class. Cover levels 0-9 for Wizard/Sorcerer, Cleric/Druid. Levels 0-6 for Bard. Levels 1-4 for Paladin/Ranger.

- [ ] **Step 4: Populate APG spells (~150 spells)**

Add APG spells for Alchemist, Inquisitor, Oracle, Summoner, Witch. Some overlap with CRB spells (different class access level).

- [ ] **Step 5: Validate JSON**

Run: `python3 -c "import json; d=json.load(open('spells.json')); print(f'{len(d)} spells loaded'); print('Sample:', d[0]['name'])"` to verify valid JSON and count.

Expected: ~550 spells, valid JSON, no syntax errors.

- [ ] **Step 6: Commit**

```bash
git add spells.json
git commit -m "feat: add PF1e CRB+APG spell database (~550 spells)"
```

---

### Task 2: Add Spell Browser Overlay HTML + CSS

**Files:**
- Modify: `index.html` (add overlay HTML after dice widget, add CSS in `<style>` block)

**Interfaces:**
- Consumes: None (new standalone component)
- Produces: DOM elements `#spellBrowserOverlay`, `#spellBrowserList`, `#spellBrowserSearch`, filter dropdowns

- [ ] **Step 1: Add CSS for spell browser overlay**

Add to the `<style>` block in `index.html`, after the existing `.comment-area` styles:

```css
.spell-browser-overlay {
  display: none;
  position: fixed;
  inset: 0;
  z-index: 3000;
  background: rgba(0,0,0,0.85);
  flex-direction: column;
}
.spell-browser-overlay.open { display: flex; }
.spell-browser-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: #2a2018;
  border-bottom: 2px solid #6b4c1e;
}
.spell-browser-header input[type="text"] {
  flex: 1;
  background: #3a2a15;
  border: 1px solid #6b4c1e;
  color: #f0e0b0;
  padding: 8px 12px;
  border-radius: 4px;
  font-family: inherit;
  font-size: 14px;
}
.spell-browser-header select {
  background: #3a2a15;
  border: 1px solid #6b4c1e;
  color: #f0e0b0;
  padding: 8px;
  border-radius: 4px;
  font-family: inherit;
}
.spell-browser-close {
  background: #8b1a1a;
  color: #e8d5a3;
  border: none;
  padding: 8px 16px;
  cursor: pointer;
  border-radius: 4px;
  font-family: 'Cinzel', Georgia, serif;
  font-size: 14px;
}
.spell-browser-close:hover { background: #a52020; }
.spell-browser-list {
  flex: 1;
  overflow-y: auto;
  padding: 8px 16px;
}
.spell-browser-item {
  background: #2a2018;
  border: 1px solid #3a2a15;
  border-radius: 4px;
  margin-bottom: 4px;
  cursor: pointer;
  transition: border-color 0.2s;
}
.spell-browser-item:hover { border-color: #6b4c1e; }
.spell-browser-item.expanded { border-color: #c9a84c; }
.spell-browser-item-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
}
.spell-browser-item-header .spell-name {
  flex: 1;
  color: #e8d5a3;
  font-family: 'Cinzel', Georgia, serif;
  font-size: 14px;
}
.spell-browser-item-header .spell-level {
  color: #c9a84c;
  font-size: 12px;
  min-width: 30px;
}
.spell-browser-item-header .spell-school {
  color: #8b6914;
  font-size: 12px;
}
.spell-browser-item-body {
  display: none;
  padding: 8px 12px 12px;
  border-top: 1px solid #3a2a15;
  color: #e8d5a3;
  font-size: 13px;
  line-height: 1.5;
}
.spell-browser-item.expanded .spell-browser-item-body { display: block; }
.spell-browser-item-body .spell-meta {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px 16px;
  margin-bottom: 8px;
  font-size: 12px;
}
.spell-browser-item-body .spell-meta span { color: #8b6914; }
.spell-browser-item-body .spell-meta span strong { color: #e8d5a3; }
.spell-browser-item-body .spell-desc {
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #3a2a15;
  color: #c4b08a;
}
.spell-browser-item-body .spell-add-btn {
  display: inline-block;
  margin-top: 10px;
  background: #6b4c1e;
  color: #e8d5a3;
  border: none;
  padding: 6px 14px;
  cursor: pointer;
  border-radius: 4px;
  font-family: 'Cinzel', Georgia, serif;
  font-size: 13px;
}
.spell-browser-item-body .spell-add-btn:hover { background: #8b6c2e; }
.spell-level-picker {
  display: none;
  margin-top: 8px;
  padding: 8px;
  background: rgba(0,0,0,0.3);
  border-radius: 4px;
}
.spell-level-picker.open { display: block; }
.spell-level-picker button {
  background: #3a2a15;
  color: #c9a84c;
  border: 1px solid #6b4c1e;
  padding: 4px 10px;
  margin: 2px;
  cursor: pointer;
  border-radius: 3px;
  font-family: 'Cinzel', Georgia, serif;
}
.spell-level-picker button:hover { background: #6b4c1e; color: #e8d5a3; }
.spell-browser-count {
  color: #8b6914;
  font-size: 12px;
  padding: 4px 16px;
}
```

- [ ] **Step 2: Add overlay HTML**

Add before the closing `</body>` tag (after the dice popup):

```html
<!-- Spell Browser Overlay -->
<div id="spellBrowserOverlay" class="spell-browser-overlay">
  <div class="spell-browser-header">
    <button class="spell-browser-close" onclick="closeSpellBrowser()" data-i18n="btn_close_browser">✕ Close</button>
    <span style="color:#c9a84c;font-family:'Cinzel',Georgia,serif;font-size:16px;" data-i18n="spell_browser_title">Spell Browser</span>
    <input type="text" id="spellBrowserSearch" placeholder="Search spells..." data-i18n-placeholder="ph_search_spells" oninput="filterSpellBrowser()">
    <select id="spellBrowserClass" onchange="filterSpellBrowser()">
      <option value="" data-i18n="filter_all_classes">All Classes</option>
    </select>
    <select id="spellBrowserLevel" onchange="filterSpellBrowser()">
      <option value="" data-i18n="filter_all_levels">All Levels</option>
      <option value="0" data-i18n="lbl_cantrip">Cantrip</option>
      <option value="1">1</option>
      <option value="2">2</option>
      <option value="3">3</option>
      <option value="4">4</option>
      <option value="5">5</option>
      <option value="6">6</option>
      <option value="7">7</option>
      <option value="8">8</option>
      <option value="9">9</option>
    </select>
    <select id="spellBrowserSchool" onchange="filterSpellBrowser()">
      <option value="" data-i18n="filter_all_schools">All Schools</option>
      <option value="Abjuration">Abjuration</option>
      <option value="Conjuration">Conjuration</option>
      <option value="Divination">Divination</option>
      <option value="Enchantment">Enchantment</option>
      <option value="Evocation">Evocation</option>
      <option value="Illusion">Illusion</option>
      <option value="Necromancy">Necromancy</option>
      <option value="Transmutation">Transmutation</option>
    </select>
  </div>
  <div class="spell-browser-count" id="spellBrowserCount"></div>
  <div class="spell-browser-list" id="spellBrowserList"></div>
</div>
```

- [ ] **Step 3: Verify overlay renders**

Open `index.html` in browser. The overlay should not be visible by default. Manually toggle it via browser console:
```javascript
document.getElementById('spellBrowserOverlay').classList.add('open');
```
Expected: Full-screen dark overlay with header, filters, and empty list area.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add spell browser overlay HTML and CSS"
```

---

### Task 3: Add Spell Browser JavaScript

**Files:**
- Modify: `index.html` (add JS functions in `<script>` block)

**Interfaces:**
- Consumes: `spells.json` (external file), `state.spells` (existing spell list state), `renderAllDynamicTables()`, `bindInputs()`, `recalculate()`, `updateDerived()`, `saveToStorage()`, `applyLanguage()`
- Produces: `loadSpellDatabase()`, `openSpellBrowser(setIndex)`, `closeSpellBrowser()`, `filterSpellBrowser()`, `renderSpellBrowserList(spells)`, `toggleSpellBrowserItem(index)`, `showSpellLevelPicker(spellIndex)`, `addSpellFromBrowser(setIndex, spellIndex, level)`

- [ ] **Step 1: Add spell database loader**

Add to the `<script>` block in `index.html`, before the DICE ROLLER section:

```javascript
// ============================================
// SPELL DATABASE
// ============================================
let spellDatabase = null;
let currentSpellSetIndex = -1;

async function loadSpellDatabase() {
  if (spellDatabase) return spellDatabase;
  try {
    const resp = await fetch('spells.json');
    if (!resp.ok) throw new Error('Failed to load spells.json');
    spellDatabase = await resp.json();
    return spellDatabase;
  } catch (err) {
    console.error('Failed to load spell database:', err);
    alert('Failed to load spell database. Make sure spells.json is in the same directory.');
    return null;
  }
}
```

- [ ] **Step 2: Add browser open/close functions**

```javascript
async function openSpellBrowser(setIndex) {
  currentSpellSetIndex = setIndex;
  const data = await loadSpellDatabase();
  if (!data) return;

  const overlay = document.getElementById('spellBrowserOverlay');
  overlay.classList.add('open');

  // Populate class filter from spell database
  const classSelect = document.getElementById('spellBrowserClass');
  const classes = [...new Set(data.flatMap(s => s.classes.map(c => c.name)))].sort();
  classSelect.innerHTML = '<option value="" data-i18n="filter_all_classes">All Classes</option>' +
    classes.map(c => `<option value="${c}">${c}</option>`).join('');

  // Pre-filter by casterClass if set
  const set = state.spells[setIndex];
  if (set && set.casterClass) {
    classSelect.value = set.casterClass;
  }

  filterSpellBrowser();
  applyLanguage();
}

function closeSpellBrowser() {
  document.getElementById('spellBrowserOverlay').classList.remove('open');
  currentSpellSetIndex = -1;
}
```

- [ ] **Step 3: Add filter and render functions**

```javascript
function filterSpellBrowser() {
  if (!spellDatabase) return;
  const search = (document.getElementById('spellBrowserSearch').value || '').toLowerCase();
  const cls = document.getElementById('spellBrowserClass').value;
  const lvl = document.getElementById('spellBrowserLevel').value;
  const school = document.getElementById('spellBrowserSchool').value;

  let filtered = spellDatabase;

  if (search) {
    filtered = filtered.filter(s => s.name.toLowerCase().includes(search));
  }
  if (cls) {
    filtered = filtered.filter(s => s.classes.some(c => c.name === cls));
  }
  if (lvl !== '') {
    const levelNum = parseInt(lvl);
    filtered = filtered.filter(s => s.level === levelNum);
  }
  if (school) {
    filtered = filtered.filter(s => s.school === school || (s.descriptors && s.descriptors.includes(school.toLowerCase())));
  }

  renderSpellBrowserList(filtered);
}

function renderSpellBrowserList(spells) {
  const container = document.getElementById('spellBrowserList');
  const countEl = document.getElementById('spellBrowserCount');
  countEl.textContent = `${spells.length} spells found`;

  container.innerHTML = spells.map((spell, i) => {
    const classList = spell.classes.map(c => `${c.name} ${c.level}`).join(', ');
    const descriptors = spell.descriptors && spell.descriptors.length ? ` [${spell.descriptors.join(', ')}]` : '';
    return `
      <div class="spell-browser-item" id="sbi-${i}" onclick="toggleSpellBrowserItem(${i})">
        <div class="spell-browser-item-header">
          <span class="spell-name">${spell.name}</span>
          <span class="spell-level">Lv${spell.level}</span>
          <span class="spell-school">${spell.school}${descriptors}</span>
        </div>
        <div class="spell-browser-item-body">
          <div class="spell-meta">
            <span><strong data-i18n="lbl_casting_time">Casting Time:</strong> ${spell.castingTime || '—'}</span>
            <span><strong data-i18n="lbl_range">Range:</strong> ${spell.range || '—'}</span>
            <span><strong data-i18n="lbl_components">Components:</strong> ${spell.components || '—'}</span>
            <span><strong data-i18n="lbl_duration">Duration:</strong> ${spell.duration || '—'}</span>
            ${spell.area ? `<span><strong data-i18n="lbl_area">Area:</strong> ${spell.area}</span>` : ''}
            ${spell.effect ? `<span><strong data-i18n="lbl_effect">Effect:</strong> ${spell.effect}</span>` : ''}
            ${spell.targets ? `<span><strong data-i18n="lbl_targets">Targets:</strong> ${spell.targets}</span>` : ''}
            <span><strong data-i18n="lbl_save">Save:</strong> ${spell.save || '—'}</span>
            <span><strong data-i18n="lbl_resistance">SR:</strong> ${spell.resistance || '—'}</span>
          </div>
          <div><strong data-i18n="lbl_classes">Classes:</strong> ${classList}</div>
          ${spell.material ? `<div><strong data-i18n="lbl_material">Material:</strong> ${spell.material}</div>` : ''}
          ${spell.focus ? `<div><strong data-i18n="lbl_focus">Focus:</strong> ${spell.focus}</div>` : ''}
          <div class="spell-desc">${spell.description || ''}</div>
          <button class="spell-add-btn" onclick="event.stopPropagation(); showSpellLevelPicker(${i})" data-i18n="btn_add_to_list">Add to spell list</button>
          <div class="spell-level-picker" id="slp-${i}">
            <span style="color:#8b6914;font-size:12px;" data-i18n="lbl_add_to_level">Add to Level:</span>
            ${[0,1,2,3,4,5,6,7,8,9].map(l => `<button onclick="event.stopPropagation(); addSpellFromBrowser(${i}, ${l})">${l}</button>`).join('')}
          </div>
        </div>
      </div>
    `;
  }).join('');

  applyLanguage();
}

function toggleSpellBrowserItem(index) {
  const el = document.getElementById('sbi-' + index);
  if (el) el.classList.toggle('expanded');
}

function showSpellLevelPicker(spellIndex) {
  const el = document.getElementById('slp-' + spellIndex);
  if (el) el.classList.toggle('open');
}
```

- [ ] **Step 4: Add spell addition function**

```javascript
function addSpellFromBrowser(spellIndex, level) {
  if (currentSpellSetIndex < 0 || !spellDatabase) return;
  const spell = spellDatabase[spellIndex];
  if (!spell) return;

  const set = state.spells[currentSpellSetIndex];
  if (!set) return;

  if (!set.spells[level]) set.spells[level] = [];

  const description = [
    spell.school + (spell.descriptors && spell.descriptors.length ? ` [${spell.descriptors.join(', ')}]` : ''),
    spell.castingTime ? `Casting: ${spell.castingTime}` : '',
    spell.range ? `Range: ${spell.range}` : '',
    spell.components ? `Components: ${spell.components}` : '',
    spell.duration ? `Duration: ${spell.duration}` : '',
    spell.area ? `Area: ${spell.area}` : '',
    spell.effect ? `Effect: ${spell.effect}` : '',
    spell.targets ? `Targets: ${spell.targets}` : '',
    spell.save ? `Save: ${spell.save}` : '',
    spell.resistance ? `SR: ${spell.resistance}` : '',
    '',
    spell.description || ''
  ].filter(Boolean).join('\n');

  set.spells[level].push({
    name: spell.name,
    expended: false,
    notes: description
  });

  closeSpellBrowser();
  renderAllDynamicTables();
  bindInputs();
  recalculate();
  updateDerived();
  saveToStorage();
}
```

- [ ] **Step 5: Add keyboard shortcut (Escape to close)**

Add to the INIT section at the bottom of the script:

```javascript
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') closeSpellBrowser();
});
```

- [ ] **Step 6: Verify all functions work**

Open browser console, manually test:
```javascript
// Load database
await loadSpellDatabase(); // Should return ~550 spells
// Open browser for spell set 0
openSpellBrowser(0); // Should open overlay with all spells
// Filter
document.getElementById('spellBrowserClass').value = 'Wizard';
filterSpellBrowser(); // Should show only Wizard spells
// Close
closeSpellBrowser(); // Should close overlay
```

Expected: All functions work without errors.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add spell browser JavaScript (load, filter, render, add)"
```

---

### Task 4: Add "Browse" Button to Spell Sets + i18n Keys

**Files:**
- Modify: `index.html` — `renderSpellSets()` function, `i18n.en`, `i18n.ru`

**Interfaces:**
- Consumes: `openSpellBrowser(setIndex)` from Task 3
- Produces: Updated `renderSpellSets()` with Browse button, complete i18n keys

- [ ] **Step 1: Add Browse button to renderSpellSets**

In the `renderSpellSets()` function, find the Spell List panel title line:
```javascript
<div class="panel-title" style="margin-top:12px;" data-i18n="lbl_spell_list">Spell List</div>
```

Replace with:
```javascript
<div class="panel-title" style="margin-top:12px;display:flex;justify-content:space-between;align-items:center;">
  <span data-i18n="lbl_spell_list">Spell List</span>
  <button onclick="openSpellBrowser(${si})" style="background:#6b4c1e;color:#e8d5a3;border:none;padding:4px 12px;cursor:pointer;font-size:12px;border-radius:4px;font-family:'Cinzel',Georgia,serif;" data-i18n="btn_browse_spells">Browse</button>
</div>
```

Also add logic to disable the button when casterClass is empty:
- After the button HTML, add: `${!set.casterClass ? '<style>[data-set="' + si + '"] .spell-browse-btn{opacity:0.5;pointer-events:none;}</style>' : ''}`

Actually, simpler approach — just set `disabled` attribute:
```javascript
<button onclick="openSpellBrowser(${si})" style="background:#6b4c1e;color:#e8d5a3;border:none;padding:4px 12px;cursor:pointer;font-size:12px;border-radius:4px;font-family:'Cinzel',Georgia,serif;${!set.casterClass ? 'opacity:0.5;pointer-events:none;' : ''}" data-i18n="btn_browse_spells">Browse</button>
```

- [ ] **Step 2: Add EN i18n keys**

Find the EN i18n dictionary (around line 1280) and add before the closing `}`:

```javascript
btn_browse_spells: "Browse",
spell_browser_title: "Spell Browser",
filter_all_classes: "All Classes",
filter_all_levels: "All Levels",
filter_all_schools: "All Schools",
btn_close_browser: "✕ Close",
btn_add_to_list: "Add to spell list",
lbl_add_to_level: "Add to Level:",
lbl_casting_time: "Casting Time",
lbl_range: "Range",
lbl_components: "Components",
lbl_duration: "Duration",
lbl_area: "Area",
lbl_effect: "Effect",
lbl_targets: "Targets",
lbl_save: "Save",
lbl_resistance: "SR",
lbl_classes: "Classes",
lbl_material: "Material",
lbl_focus: "Focus",
ph_search_spells: "Search spells...",
```

- [ ] **Step 3: Add RU i18n keys**

Find the RU i18n dictionary (around line 1560) and add:

```javascript
btn_browse_spells: "Поиск",
spell_browser_title: "Реестр заклинаний",
filter_all_classes: "Все классы",
filter_all_levels: "Все уровни",
filter_all_schools: "Все школы",
btn_close_browser: "✕ Закрыть",
btn_add_to_list: "Добавить в список",
lbl_add_to_level: "Добавить на уровень:",
lbl_casting_time: "Время сотворения",
lbl_range: "Дальность",
lbl_components: "Компоненты",
lbl_duration: "Длительность",
lbl_area: "Область",
lbl_effect: "Эффект",
lbl_targets: "Цели",
lbl_save: "Спасбросок",
lbl_resistance: "Сопрот. магии",
lbl_classes: "Классы",
lbl_material: "Материальный",
lbl_focus: "Фокус",
ph_search_spells: "Поиск заклинаний...",
```

- [ ] **Step 4: Verify Browse button appears**

Open the Spells tab. Each Spell Set should have a "Browse" button next to "Spell List" title.
- If `casterClass` is empty → button is greyed out / disabled
- If `casterClass` is set → button is active
- Click active button → overlay opens

- [ ] **Step 5: Test full flow**

1. Add a Spell Set, set Caster Class to "Wizard"
2. Click "Browse" → overlay opens, pre-filtered to Wizard spells
3. Search "Magic Missile" → found
4. Click it → expands with full description
5. Click "Add to spell list" → level picker appears
6. Click "1" → spell added to Level 1, overlay closes
7. Verify spell appears in the spell list with name and description

- [ ] **Step 6: Test RU language**

1. Switch to RU
2. Open spell browser → title should be "Реестр заклинаний"
3. Filters should show Russian labels
4. Click a spell → metadata labels in Russian
5. Add spell → works correctly

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add Browse button to spell sets + i18n keys for spell browser"
```

---

### Task 5: Polish and Integration Testing

**Files:**
- Modify: `index.html` (if needed for fixes)

**Interfaces:**
- Consumes: All previous tasks
- Produces: Final working spell browser

- [ ] **Step 1: Test print mode**

Open print dialog (Ctrl+P). The spell browser overlay should not appear in print.

Add to the `@media print` section:
```css
.spell-browser-overlay { display: none !important; }
```

- [ ] **Step 2: Test edge cases**

- Empty spell database (corrupt/missing file) → graceful error message
- No casterClass set → Browse button disabled
- Multiple spell sets → each opens its own browser instance
- Very long spell descriptions → scrollable card
- Mobile responsive → overlay adapts to small screens

- [ ] **Step 3: Test all filter combinations**

- Class + Level
- Class + School
- All filters combined
- Search + filters
- Clear all filters → shows all spells

- [ ] **Step 4: Final commit and push**

```bash
git add index.html
git commit -m "feat: spell browser — polish, print CSS, edge cases"
git push
```
