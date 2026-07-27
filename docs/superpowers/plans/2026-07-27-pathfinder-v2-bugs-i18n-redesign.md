# Pathfinder 1e Character Sheet — v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix two critical bugs (input focus loss, checkbox unresponsive), add RU/EN language toggle, and apply comprehensive visual redesign to the Pathfinder 1e character sheet.

**Architecture:** Single-file app (`index.html`, ~2000 lines). All changes in this one file. Selective re-render architecture replaces full innerHTML rebuilds on every keystroke. i18n dictionary for language toggle. CSS-only atmospheric effects for visual redesign.

**Tech Stack:** Vanilla HTML/CSS/JS, no frameworks, no external dependencies, localStorage persistence.

## Global Constraints

- Single-file app — no external dependencies, no build tools, no server required
- System font stacks only: `"Cinzel", Georgia, "Times New Roman", serif` (headers); `Georgia, "Crimson Text", serif` (body)
- localStorage persistence with 300ms debounce
- All PF1e formulas must remain correct after DOM changes
- Print CSS must still work (black-on-white, tabs hidden)
- Responsive layout must still work on mobile
- Git user: "ElLyat1", email: "vadim.rou7@gmail.com"

---

### Task 1: Fix Bug 1 — Selective re-render architecture

**Files:**
- Modify: `index.html` (JS section: `recalculate()`, `bindInputs()`, dynamic table render functions, input event handlers)

**What this produces:** Input focus is preserved during typing. Derived values update without destroying DOM elements.

**Interfaces:**
- Consumes: existing `state.*` object structure, existing `data-bind` and `data-display` attributes
- Produces: `recalculate()` (pure computation), `updateDerived()` (display-only), `renderAllDynamicTables()` (structural changes only)

- [ ] **Step 1: Refactor recalculate() to be pure computation**

In `index.html`, find the `recalculate()` function (currently around line 1167). Remove all render function calls from it. The function should ONLY update `state.*` values — zero DOM touches.

Current (broken):
```javascript
function recalculate() {
  // ... computation ...
  renderSkillsTable();
  renderKnowledgeTable();
  renderPerformTable();
  renderCraftProfession();
  renderSpellSets();
  renderInventoryTable();
  updateDisplay();
  saveToStorage();
}
```

New:
```javascript
function recalculate() {
  // Ability modifiers
  const ab = {};
  ['str', 'dex', 'con', 'int', 'wis', 'cha'].forEach(a => {
    const base = state.abilityScores[a].base || 10;
    state.abilityScores[a].mod = Math.floor((base - 10) / 2);
    ab[a] = state.abilityScores[a].mod;
  });

  // Max Dex from equipped armor
  const equippedArmor = (state.armor || []).filter(a => a.equipped);
  let maxDexCap = 999;
  equippedArmor.forEach(a => {
    if (a.maxDex !== '' && a.maxDex !== undefined && a.maxDex !== null) {
      const cap = Number(a.maxDex);
      if (!isNaN(cap) && cap < maxDexCap) maxDexCap = cap;
    }
  });
  const effectiveDex = maxDexCap === 999 ? ab.dex : Math.min(ab.dex, maxDexCap);

  // AC
  state.combat.ac.dex = effectiveDex;
  state.combat.ac.total = 10 + (state.combat.ac.armor || 0) + (state.combat.ac.shield || 0)
    + (state.combat.ac.natural || 0) + effectiveDex + (Number(state.combat.ac.size) || 0)
    + (state.combat.ac.deflection || 0) + (state.combat.ac.dodge || 0) + (state.combat.ac.misc || 0);
  state.combat.ac.touch = 10 + effectiveDex + (Number(state.combat.ac.size) || 0)
    + (state.combat.ac.deflection || 0) + (state.combat.ac.dodge || 0) + (state.combat.ac.misc || 0);
  state.combat.ac.flatFooted = state.combat.ac.total - ab.dex - (state.combat.ac.dodge || 0);

  // CMB / CMD (same as current)
  const bab = state.combat.bab.value || 0;
  const cmbSize = Number(state.combat.cmb.size) || 0;
  state.combat.cmb.strMod = ab.str;
  state.combat.cmb.total = bab + ab.str + cmbSize + (state.combat.cmb.misc || 0);
  state.combat.cmd.strMod = ab.str;
  state.combat.cmd.dexMod = ab.dex;
  state.combat.cmd.size = cmbSize;
  state.combat.cmd.total = 10 + bab + ab.str + ab.dex + cmbSize + (state.combat.cmd.misc || 0);

  // Saves (same as current)
  state.combat.saves.fort.conMod = ab.con;
  state.combat.saves.fort.total = (state.combat.saves.fort.base || 0) + ab.con
    + (state.combat.saves.fort.resistance || 0) + (state.combat.saves.fort.misc || 0);
  state.combat.saves.ref.dexMod = ab.dex;
  state.combat.saves.ref.total = (state.combat.saves.ref.base || 0) + ab.dex
    + (state.combat.saves.ref.resistance || 0) + (state.combat.saves.ref.misc || 0);
  state.combat.saves.will.wisMod = ab.wis;
  state.combat.saves.will.total = (state.combat.saves.will.base || 0) + ab.wis
    + (state.combat.saves.will.resistance || 0) + (state.combat.saves.will.misc || 0);

  // Initiative (same as current)
  state.combat.initiative.dexMod = ab.dex;
  state.combat.initiative.total = ab.dex + (state.combat.initiative.misc || 0);

  // Full Attack Progression (same as current)
  const meleeMod = ab.str;
  const rangedMod = ab.dex;
  const meleeAttacks = [bab];
  const rangedAttacks = [bab];
  if (bab >= 6) { meleeAttacks.push(bab - 5); rangedAttacks.push(bab - 5); }
  if (bab >= 11) { meleeAttacks.push(bab - 10); rangedAttacks.push(bab - 10); }
  if (bab >= 16) { meleeAttacks.push(bab - 15); rangedAttacks.push(bab - 15); }
  state.combat.bab.meleeAttacks = meleeAttacks.map(a => a + meleeMod + cmbSize);
  state.combat.bab.rangedAttacks = rangedAttacks.map(a => a + rangedMod + cmbSize);
  state.combat.bab.meleeString = state.combat.bab.meleeAttacks.map(a => (a >= 0 ? '+' : '') + a).join('/');
  state.combat.bab.rangedString = state.combat.bab.rangedAttacks.map(a => (a >= 0 ? '+' : '') + a).join('/');

  // HP Status (same as current)
  const hp = state.combat.hp;
  const current = hp.current || 0;
  const max = hp.max || 1;
  const nonlethal = hp.nonlethal || 0;
  const effective = current - nonlethal;
  const conMod = ab.con;
  if (effective >= max) hp.status = "Feelin' Fine!";
  else if (effective >= max * 0.5) hp.status = 'Hurt';
  else if (effective > 0) hp.status = 'Bloodied';
  else if (effective > -Math.abs(conMod)) hp.status = 'Dying';
  else hp.status = 'Dead';

  // Skill points
  if (!state.skillPoints) state.skillPoints = { available: 0, used: 0 };
  let usedPoints = 0;
  Object.values(state.skills || {}).forEach(sk => {
    usedPoints += Number(sk.ranks) || 0;
  });
  state.skillPoints.used = usedPoints;
  state.skillPoints.remaining = (state.skillPoints.available || 0) - usedPoints;

  // Spell concentration
  (state.spells || []).forEach((set) => {
    const abMod = state.abilityScores[set.ability]?.mod || 0;
    set.concentration = (Number(set.casterLevel) || 0) + abMod + (Number(set.misc) || 0);
  });

  // Inventory calculations
  if (!state.inventory) state.inventory = {};
  if (!state.inventory.currency) state.inventory.currency = { platinum: 0, gold: 0, silver: 0, copper: 0 };
  if (!state.inventory.carryingCapacity) state.inventory.carryingCapacity = { light: 0, medium: 0, heavy: 0 };

  const c = state.inventory.currency;
  state.inventory.totalValueGP = (Number(c.platinum) || 0) * 10
    + (Number(c.gold) || 0)
    + (Number(c.silver) || 0) / 10
    + (Number(c.copper) || 0) / 100;

  const strScore = Number(state.abilityScores.str.base) || 10;
  const [light, medium, heavy] = getCarryingCapacity(strScore);
  const sizeMod = Number(state.inventory.sizeModifier) || 1;
  const quadMod = state.inventory.isQuadruped ? 1.5 : 1;
  state.inventory.carryingCapacity.light = Math.round(light * sizeMod * quadMod);
  state.inventory.carryingCapacity.medium = Math.round(medium * sizeMod * quadMod);
  state.inventory.carryingCapacity.heavy = Math.round(heavy * sizeMod * quadMod);
  state.inventory.liftOverHead = state.inventory.carryingCapacity.heavy;
  state.inventory.liftOffGround = state.inventory.carryingCapacity.heavy * 2;
  state.inventory.pushDrag = state.inventory.carryingCapacity.heavy * 5;

  const notCarried = (state.inventory.notCarriedLocationsText || '').split(',').map(s => s.trim().toLowerCase()).filter(Boolean);
  state.inventory.carriedWeight = (state.inventory.items || []).reduce((sum, item) => {
    const loc = (item.location || '').trim().toLowerCase();
    if (notCarried.includes(loc)) return sum;
    return sum + (Number(item.qty) || 0) * (Number(item.weightPerUnit) || 0);
  }, 0);

  const cw = state.inventory.carriedWeight;
  const cap = state.inventory.carryingCapacity;
  if (cw <= cap.light) state.inventory.encumbrance = 'Light';
  else if (cw <= cap.medium) state.inventory.encumbrance = 'Medium';
  else if (cw <= cap.heavy) state.inventory.encumbrance = 'Heavy';
  else state.inventory.encumbrance = 'Overloaded';

  // Compute skill totals into state for data-display binding
  ensureSkillData();
  SKILL_DEFS.forEach(s => {
    const sk = state.skills[s.key];
    const calc = calcSkillTotal(sk, s.ability, s.armorPenalty);
    sk.total = calc.total;
    sk.classSkillBonus = calc.classSkillBonus;
    sk.abilityMod = calc.abilityMod;
    sk.armorPenaltyBonus = calc.armorPenalty;
  });
  KNOWLEDGE_SKILLS.forEach(name => {
    const key = 'knowledge' + name.replace(/[^a-zA-Z]/g, '');
    const sk = state.skills[key];
    const calc = calcSkillTotal(sk, 'int', false);
    sk.total = calc.total;
    sk.classSkillBonus = calc.classSkillBonus;
    sk.abilityMod = calc.abilityMod;
  });
  PERFORM_SKILLS.forEach(name => {
    const key = 'perform' + name.replace(/[^a-zA-Z]/g, '');
    const sk = state.skills[key];
    const calc = calcSkillTotal(sk, 'cha', false);
    sk.total = calc.total;
    sk.classSkillBonus = calc.classSkillBonus;
    sk.abilityMod = calc.abilityMod;
  });
  // Craft/Profession totals
  Object.entries(state.skills).filter(([k]) => k.startsWith('craft') || k.startsWith('profession')).forEach(([key, sk]) => {
    const ability = sk.ability || (key.startsWith('craft') ? 'int' : 'wis');
    const calc = calcSkillTotal(sk, ability, false);
    sk.total = calc.total;
  });
  // Inventory item totals
  (state.inventory.items || []).forEach(item => {
    item.totalValue = (Number(item.qty) || 0) * (Number(item.valuePerUnit) || 0);
    item.totalWeight = (Number(item.qty) || 0) * (Number(item.weightPerUnit) || 0);
  });
}
```

- [ ] **Step 2: Create updateDerived() function**

Add this new function right after `recalculate()`:

```javascript
function updateDerived() {
  document.querySelectorAll('[data-display]').forEach(el => {
    const value = getNestedValue(state, el.dataset.display);
    if (value !== undefined) {
      if (typeof value === 'number' && el.dataset.display.includes('.mod')) {
        el.textContent = value >= 0 ? '+' + value : String(value);
      } else if (typeof value === 'number') {
        el.textContent = String(value);
      } else {
        el.textContent = String(value);
      }
    }
  });
  saveToStorage();
}
```

- [ ] **Step 3: Create renderAllDynamicTables() function**

Add this new function:

```javascript
function renderAllDynamicTables() {
  renderSkillsTable();
  renderKnowledgeTable();
  renderPerformTable();
  renderCraftProfession();
  renderSpellSets();
  renderInventoryTable();
  renderWeaponsTable();
  renderArmorTable();
  renderFeatureList('featsList', 'feats');
  renderFeatureList('classTalentsList', 'classTalents');
  renderFeatureList('classFeaturesList', 'classFeatures');
  renderFeatureList('racialFeaturesList', 'racialFeatures');
}
```

- [ ] **Step 4: Update input event handler in bindInputs()**

Replace the current `bindInputs()` function:

```javascript
function bindInputs() {
  document.querySelectorAll('[data-bind]').forEach(input => {
    // Remove old listener by cloning
    const clone = input.cloneNode(true);
    input.parentNode.replaceChild(clone, input);

    const path = clone.dataset.bind;
    const existing = getNestedValue(state, path);
    if (existing !== undefined) {
      if (clone.type === 'checkbox') {
        clone.checked = !!existing;
      } else {
        clone.value = existing;
      }
    }

    // Use 'input' for text/number, 'change' for checkboxes too
    const handler = () => {
      let val;
      if (clone.type === 'checkbox') {
        val = clone.checked;
      } else if (clone.type === 'number') {
        val = Number(clone.value) || 0;
      } else {
        val = clone.value;
      }
      setNestedValue(state, path, val);
      recalculate();
      updateDerived();
    };

    clone.addEventListener('input', handler);
    if (clone.type === 'checkbox') {
      clone.addEventListener('change', handler);
    }
  });
}
```

- [ ] **Step 5: Update selectCharacter() to use renderAllDynamicTables()**

Replace the individual render calls in `selectCharacter()`:

```javascript
function selectCharacter(id) {
  currentCharId = id;
  const loaded = loadFromStorage(id);
  if (loaded) {
    state = loaded;
  }
  loadCharacterList();
  renderAbilityScores();
  renderAllDynamicTables();
  bindInputs();
  recalculate();
  updateDerived();
}
```

Do the same for `newCharacter()` and `duplicateCharacter()` — replace individual render calls with `renderAllDynamicTables()`.

- [ ] **Step 6: Update dynamic table render functions to use data-display for derived cells**

In `renderSkillsTable()`, change the skill total cell from:
```javascript
<td class="derived" data-display="skills.${s.key}.total">${calc.total}</td>
```
to:
```javascript
<td class="derived" data-display="skills.${s.key}.total"></td>
```

Same for Knowledge, Perform, and Craft/Profession tables. The `updateDerived()` function will populate these.

Also update the CS Bonus column and Ability Mod column to use `data-display`:
```javascript
<td class="derived" data-display="skills.${s.key}.classSkillBonus"></td>
<!-- ... -->
<td class="derived" data-display="skills.${s.key}.abilityMod"></td>
```

In `renderInventoryTable()`, change Total Value and Total Weight cells:
```javascript
<td class="derived" data-display="inventory.items.${i}.totalValue"></td>
<td class="derived" data-display="inventory.items.${i}.totalWeight"></td>
```

- [ ] **Step 7: Add "structural change" triggers for renderAllDynamicTables()**

In the `addWeapon()`, `removeWeapon()`, `addArmor()`, `removeArmor()`, `addSpellSet()`, `removeSpellSet()`, `addSpell()`, `removeSpell()`, `addInventoryItem()`, `removeInventoryItem()`, `addFeat()`, `addClassTalent()`, `addClassFeature()`, `addRacialFeature()`, `removeFeature()` functions, ensure they call `renderAllDynamicTables()` after modifying the array (they already call individual render functions — replace those with the unified call).

Also call `renderAllDynamicTables()` from `onArmorToggle()`.

- [ ] **Step 8: Test in browser**

Open `index.html`. Test:
- Type a full sentence into the Name field without clicking — cursor must stay, no refocus needed
- Change ability scores — modifiers update, AC/saves update, no focus loss
- Type in skill ranks — total updates, no focus loss
- Toggle CS checkbox — it responds, total recalculates with +3 bonus
- Add/remove weapons, armor, spells, inventory items — tables re-render correctly
- Switch characters — all data loads correctly

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "fix: selective re-render architecture — input focus preserved, checkbox fixed"
```

---

### Task 2: Add i18n dictionary and applyLanguage()

**Files:**
- Modify: `index.html` (JS section: add i18n object, add `applyLanguage()`, add `data-i18n` attributes to HTML)

**What this produces:** Complete EN/RU translation dictionary, language toggle button, all UI labels translatable.

**Interfaces:**
- Consumes: existing HTML structure with panel titles, field labels, tab buttons
- Produces: `i18n` object, `currentLang` variable, `applyLanguage()` function, `data-i18n` attributes on all translatable elements

- [ ] **Step 1: Add i18n dictionary to JS**

Add near the top of the `<script>` section (after the state variables):

```javascript
const i18n = {
  en: {
    // Tab names
    tab_main: "Main",
    tab_skills: "Skills",
    tab_abilities: "Abilities",
    tab_spells: "Spells",
    tab_inventory: "Inventory",
    tab_notes: "Notes",
    // Character selector
    btn_new: "New",
    btn_dup: "Dup",
    btn_rename: "Rename",
    btn_del: "Delete",
    btn_export: "Export",
    btn_export_all: "Export All",
    btn_import: "Import",
    // Main tab — Character Info
    panel_character_info: "Character Info",
    lbl_name: "Name",
    lbl_player: "Player",
    lbl_race: "Race",
    lbl_class_level: "Class & Level",
    lbl_alignment: "Alignment",
    lbl_deity: "Deity",
    lbl_size: "Size",
    lbl_age: "Age",
    lbl_gender: "Gender",
    lbl_height: "Height",
    lbl_weight: "Weight",
    lbl_eyes: "Eyes",
    lbl_hair: "Hair",
    lbl_skin: "Skin",
    // Main tab — Ability Scores
    panel_ability_scores: "Ability Scores",
    // Main tab — Hit Points
    panel_hit_points: "Hit Points",
    lbl_current_hp: "Current HP",
    lbl_max_hp: "Max HP",
    lbl_temp_hp: "Temp HP",
    lbl_nonlethal: "Nonlethal",
    lbl_status: "Status",
    // Main tab — Armor Class
    lbl_total_ac: "Total AC",
    lbl_touch_ac: "Touch AC",
    lbl_flat_footed: "Flat-Footed",
    lbl_armor_bonus: "Armor Bonus",
    lbl_shield_bonus: "Shield Bonus",
    lbl_natural_armor: "Natural Armor",
    lbl_size_mod: "Size Mod",
    lbl_deflection: "Deflection",
    lbl_dodge: "Dodge",
    lbl_misc_ac: "Misc AC",
    lbl_dex_mod: "Dex Mod",
    // Main tab — Initiative
    panel_initiative: "Initiative",
    lbl_total: "Total",
    lbl_misc_init: "Misc",
    // Main tab — CMB/CMD
    panel_combat_maneuvers: "Combat Maneuvers",
    lbl_cmb_total: "МБМ Total",
    lbl_bab: "БМА",
    lbl_str_mod: "Str Mod",
    lbl_size_cmb: "Size",
    lbl_misc_cmb: "Misc",
    lbl_cmd_total: "ЗБМ Total",
    lbl_cmd_misc: "Misc",
    // Main tab — Saves
    panel_saves: "Saving Throws",
    lbl_fort: "Fortitude",
    lbl_ref: "Reflex",
    lbl_will: "Will",
    lbl_base_save: "Base",
    lbl_ability_mod: "Ability",
    lbl_resistance: "Resistance",
    lbl_misc_save: "Misc",
    // Main tab — Attack Progression
    panel_attack_progression: "Attack Progression",
    lbl_melee: "Melee",
    lbl_ranged: "Ranged",
    // Main tab — Speed
    panel_speed: "Speed",
    lbl_ground: "Ground (ft)",
    lbl_climb: "Climb (ft)",
    lbl_swim: "Swim (ft)",
    lbl_fly: "Fly (ft)",
    // Main tab — Defensive Abilities
    panel_defensive: "Defensive Abilities",
    lbl_dr: "Damage Reduction",
    lbl_fast_healing: "Fast Healing",
    lbl_sr: "Spell Resistance",
    lbl_fire_resist: "Fire Resist",
    lbl_cold_resist: "Cold Resist",
    lbl_elec_resist: "Elec Resist",
    lbl_acid_resist: "Acid Resist",
    lbl_sonic_resist: "Sonic Resist",
    // Main tab — Weapons
    panel_weapons: "Weapons",
    btn_add_weapon: "+ Add Weapon",
    th_name: "Name",
    th_damage: "Damage",
    th_atk_bonus: "Atk Bonus",
    th_crit: "Crit",
    th_type: "Type",
    th_hand: "Hand",
    th_range: "Range",
    th_notes: "Notes",
    // Main tab — Armor
    panel_armor: "Armor",
    btn_add_armor: "+ Add Armor",
    th_eq: "Eq",
    th_ac_bonus: "AC Bonus",
    th_enh: "Enh",
    th_heavy: "Heavy?",
    th_max_dex: "Max Dex",
    th_check_pen: "Check Pen",
    th_arcane_fail: "Arcane Fail",
    // Main tab — Conditions
    panel_conditions: "Common Conditions",
    cond_blinded: "Blinded",
    cond_cowering: "Cowering",
    cond_dazzled: "Dazzled",
    cond_deafened: "Deafened",
    cond_entangled: "Entangled",
    cond_exhausted: "Exhausted",
    cond_fatigued: "Fatigued",
    cond_frightened: "Frightened",
    cond_grappled: "Grappled",
    cond_panicked: "Panicked",
    cond_pinned: "Pinned",
    cond_prone: "Prone",
    cond_shaken: "Shaken",
    cond_sickened: "Sickened",
    cond_stunned: "Stunned",
    cond_haste: "Haste",
    cond_prayer: "Prayer",
    cond_enlarge: "Enlarge",
    cond_reduce: "Reduce",
    cond_heroism: "Heroism",
    // Skills tab
    panel_skill_points: "Skill Points",
    lbl_available: "Available",
    lbl_used: "Used",
    lbl_remaining: "Remaining",
    panel_skills: "Skills",
    th_cs: "CS",
    th_skill: "Skill",
    th_ability: "Ability",
    th_skill_total: "Total",
    th_ranks: "Ranks",
    th_cs_bonus: "CS Bonus",
    th_comp: "Comp",
    th_circ: "Circ",
    th_feat: "Feat",
    th_race: "Race",
    th_misc_skill: "Misc",
    th_armor: "Armor",
    th_ab_mod: "Ab Mod",
    panel_knowledge: "Knowledge Skills",
    panel_perform: "Perform Skills",
    panel_craft_prof: "Craft / Profession",
    lbl_craft: "Craft (Int)",
    lbl_profession: "Profession (Wis, Trained Only)",
    btn_add_craft: "+ Add Craft",
    btn_add_profession: "+ Add Profession",
    panel_languages: "Languages",
    // Spells tab
    btn_add_spell_set: "+ Add Spell Set",
    lbl_caster_class: "Caster Class",
    lbl_caster_level: "Caster Level",
    lbl_ability: "Ability",
    lbl_misc_spell: "Misc",
    lbl_concentration: "Concentration Check",
    lbl_spell_slots: "Spell Slots",
    lbl_spell_list: "Spell List",
    lbl_total_val: "Total (GP)",
    // Inventory tab
    panel_currency: "Currency",
    lbl_platinum: "Platinum",
    lbl_gold: "Gold",
    lbl_silver: "Silver",
    lbl_copper: "Copper",
    panel_carrying: "Carrying Capacity",
    lbl_size_mod_inv: "Size Modifier",
    lbl_quadruped: "Quadruped",
    lbl_current_load: "Current Load (lbs)",
    lbl_light_load: "Light Load",
    lbl_medium_load: "Medium Load",
    lbl_heavy_load: "Heavy Load",
    lbl_lift_over: "Lift Over Head",
    lbl_lift_off: "Lift Off Ground",
    lbl_push_drag: "Push / Drag",
    lbl_encumbrance: "Encumbrance",
    lbl_info_only: "Informational only — not auto-applied to skills",
    panel_inventory: "Inventory",
    btn_add_item: "+ Add Item",
    th_qty: "Qty",
    th_item: "Item",
    th_value_unit: "Value/unit (gp)",
    th_weight_unit: "Weight/unit (lbs)",
    th_total_value: "Total Value",
    th_total_weight: "Total Weight",
    th_location: "Location",
    panel_not_carried: "Not-Carried Locations",
    lbl_not_carried_desc: "Items in these locations are excluded from carried weight:",
    // Abilities tab
    panel_feats: "Feats",
    btn_add_feat: "+ Add",
    panel_class_talents: "Class Talents / Rogue Talents / Discoveries",
    btn_add_talent: "+ Add",
    panel_class_features: "Class Features",
    btn_add_feature: "+ Add",
    panel_racial: "Racial Features & Traits",
    btn_add_racial: "+ Add",
    panel_proficiencies: "Proficiencies",
    // Notes tab
    panel_background: "Background / Backstory",
    panel_general_notes: "General Notes",
    panel_campaign_log: "Party / Campaign Log",
    // HP Status
    status_fine: "Feelin' Fine!",
    status_hurt: "Hurt",
    status_bloodied: "Bloodied",
    status_dying: "Dying",
    status_dead: "Dead",
    // Encumbrance
    enc_light: "Light",
    enc_medium: "Medium",
    enc_heavy: "Heavy",
    enc_overloaded: "Overloaded",
    // Select options
    size_small: "Small",
    size_medium: "Medium",
    size_large: "Large",
    size_huge: "Huge",
    type_melee: "Melee",
    type_ranged: "Ranged",
    type_armor: "Armor",
    type_shield: "Shield",
    type_natural: "Natural",
    type_deflection: "Deflection",
    ability_int: "Intelligence",
    ability_wis: "Wisdom",
    ability_cha: "Charisma",
    // Language toggle
    lang_toggle: "EN",
  },
  ru: {
    // Tab names
    tab_main: "Основное",
    tab_skills: "Навыки",
    tab_abilities: "Черты",
    tab_spells: "Заклинания",
    tab_inventory: "Инвентарь",
    tab_notes: "Заметки",
    // Character selector
    btn_new: "Новый",
    btn_dup: "Копия",
    btn_rename: "Переименовать",
    btn_del: "Удалить",
    btn_export: "Экспорт",
    btn_export_all: "Экспорт всех",
    btn_import: "Импорт",
    // Main tab — Character Info
    panel_character_info: "Информация о персонаже",
    lbl_name: "Имя",
    lbl_player: "Игрок",
    lbl_race: "Раса",
    lbl_class_level: "Класс и уровень",
    lbl_alignment: "Мировоззрение",
    lbl_deity: "Божество",
    lbl_size: "Размер",
    lbl_age: "Возраст",
    lbl_gender: "Пол",
    lbl_height: "Рост",
    lbl_weight: "Вес",
    lbl_eyes: "Глаза",
    lbl_hair: "Волосы",
    lbl_skin: "Кожа",
    // Main tab — Ability Scores
    panel_ability_scores: "Параметры",
    // Main tab — Hit Points
    panel_hit_points: "Хиты (ХП)",
    lbl_current_hp: "Текущие ХП",
    lbl_max_hp: "Макс. ХП",
    lbl_temp_hp: "Врем. ХП",
    lbl_nonlethal: "Несмертельный",
    lbl_status: "Статус",
    // Main tab — Armor Class
    lbl_total_ac: "Класс Доспеха (КД)",
    lbl_touch_ac: "Касание",
    lbl_flat_footed: "С захватом",
    lbl_armor_bonus: "Бонус доспеха",
    lbl_shield_bonus: "Бонус щита",
    lbl_natural_armor: "Природная броня",
    lbl_size_mod: "Мод. размера",
    lbl_deflection: "Отражение",
    lbl_dodge: "Уклонение",
    lbl_misc_ac: "Разное КД",
    lbl_dex_mod: "Мод. Лов",
    // Main tab — Initiative
    panel_initiative: "Инициатива",
    lbl_total: "Итого",
    lbl_misc_init: "Разное",
    // Main tab — CMB/CMD
    panel_combat_maneuvers: "Боевые маневры",
    lbl_cmb_total: "МБМ Итого",
    lbl_bab: "БМА",
    lbl_str_mod: "Мод. Сил",
    lbl_size_cmb: "Размер",
    lbl_misc_cmb: "Разное",
    lbl_cmd_total: "ЗБМ Итого",
    lbl_cmd_misc: "Разное",
    // Main tab — Saves
    panel_saves: "Спасброски",
    lbl_fort: "Стойкость",
    lbl_ref: "Рефлекс",
    lbl_will: "Воля",
    lbl_base_save: "Базовый",
    lbl_ability_mod: "Параметр",
    lbl_resistance: "Сопротивление",
    lbl_misc_save: "Разное",
    // Main tab — Attack Progression
    panel_attack_progression: "Прогрессия атак",
    lbl_melee: "Ближний бой",
    lbl_ranged: "Дальний бой",
    // Main tab — Speed
    panel_speed: "Скорость",
    lbl_ground: "Земля (фт)",
    lbl_climb: "Лазание (фт)",
    lbl_swim: "Плавание (фт)",
    lbl_fly: "Полёт (фт)",
    // Main tab — Defensive Abilities
    panel_defensive: "Защитные способности",
    lbl_dr: "Снижение урона",
    lbl_fast_healing: "Быстрое лечение",
    lbl_sr: "Сопротивление магии",
    lbl_fire_resist: "Огонь",
    lbl_cold_resist: "Холод",
    lbl_elec_resist: "Электричество",
    lbl_acid_resist: "Кислота",
    lbl_sonic_resist: "Звук",
    // Main tab — Weapons
    panel_weapons: "Оружие",
    btn_add_weapon: "+ Добавить оружие",
    th_name: "Название",
    th_damage: "Урон",
    th_atk_bonus: "Бонус атаки",
    th_crit: "Крит",
    th_type: "Тип",
    th_hand: "Рука",
    th_range: "Дальность",
    th_notes: "Заметки",
    // Main tab — Armor
    panel_armor: "Доспехи",
    btn_add_armor: "+ Добавить доспех",
    th_eq: "Надет",
    th_ac_bonus: "Бонус КД",
    th_enh: "Улучш",
    th_heavy: "Тяжёл?",
    th_max_dex: "Макс. Лов",
    th_check_pen: "Штраф проверки",
    th_arcane_fail: "Шанс неудачи",
    // Main tab — Conditions
    panel_conditions: "Распространённые состояния",
    cond_blinded: "Ослеплён",
    cond_cowering: "В ужасе",
    cond_dazzled: "Ослаблен",
    cond_deafened: "Оглушён",
    cond_entangled: "Опутан",
    cond_exhausted: "Изнеможение",
    cond_fatigued: "Усталость",
    cond_frightened: "Испуган",
    cond_grappled: "В захвате",
    cond_panicked: "В панике",
    cond_pinned: "Прижат",
    cond_prone: "Лёжа",
    cond_shaken: "Напуган",
    cond_sickened: "Тошнота",
    cond_stunned: "Оглушён",
    cond_haste: "Ускорение",
    cond_prayer: "Молитва",
    cond_enlarge: "Увеличение",
    cond_reduce: "Уменьшение",
    cond_heroism: "Героизм",
    // Skills tab
    panel_skill_points: "Очки навыков",
    lbl_available: "Доступно",
    lbl_used: "Потрачено",
    lbl_remaining: "Осталось",
    panel_skills: "Навыки",
    th_cs: "КН",
    th_skill: "Навык",
    th_ability: "Параметр",
    th_skill_total: "Итого",
    th_ranks: "Ранги",
    th_cs_bonus: "Бонус КН",
    th_comp: "Компет",
    th_circ: "Ситуац",
    th_feat: "Черта",
    th_race: "Раса",
    th_misc_skill: "Разное",
    th_armor: "Доспех",
    th_ab_mod: "Мод.парам",
    panel_knowledge: "Знания",
    panel_perform: "Исполнение",
    panel_craft_prof: "Ремесло / Профессия",
    lbl_craft: "Ремесло (Инт)",
    lbl_profession: "Профессия (Мдр, Только обученный)",
    btn_add_craft: "+ Добавить ремесло",
    btn_add_profession: "+ Добавить профессию",
    panel_languages: "Языки",
    // Spells tab
    btn_add_spell_set: "+ Добавить набор заклинаний",
    lbl_caster_class: "Класс заклинателя",
    lbl_caster_level: "Уровень заклинателя",
    lbl_ability: "Параметр",
    lbl_misc_spell: "Разное",
    lbl_concentration: "Проверка концентрации",
    lbl_spell_slots: "Ячейки заклинаний",
    lbl_spell_list: "Список заклинаний",
    lbl_total_val: "Итого (зм)",
    // Inventory tab
    panel_currency: "Валюта",
    lbl_platinum: "Платина",
    lbl_gold: "Золото",
    lbl_silver: "Серебро",
    lbl_copper: "Медь",
    panel_carrying: "Несущая способность",
    lbl_size_mod_inv: "Мод. размера",
    lbl_quadruped: "Четвероногий",
    lbl_current_load: "Текущая нагрузка (фн)",
    lbl_light_load: "Лёгкая",
    lbl_medium_load: "Средняя",
    lbl_heavy_load: "Тяжёлая",
    lbl_lift_over: "Поднять над головой",
    lbl_lift_off: "Поднять с земли",
    lbl_push_drag: "Толкнуть / Волочить",
    lbl_encumbrance: "Нагрузка",
    lbl_info_only: "Только для справки — не влияет на навыки",
    panel_inventory: "Инвентарь",
    btn_add_item: "+ Добавить предмет",
    th_qty: "Кол-во",
    th_item: "Предмет",
    th_value_unit: "Цена/шт (зм)",
    th_weight_unit: "Вес/шт (фн)",
    th_total_value: "Общая цена",
    th_total_weight: "Общий вес",
    th_location: "Расположение",
    panel_not_carried: "Места без переноски",
    lbl_not_carried_desc: "Предметы в этих местах не включены в нагрузку:",
    // Abilities tab
    panel_feats: "Черты",
    btn_add_feat: "+ Добавить",
    panel_class_talents: "Таланты класса / Таланты разведчика / Открытия",
    btn_add_talent: "+ Добавить",
    panel_class_features: "Классовые способности",
    btn_add_feature: "+ Добавить",
    panel_racial: "Расовые особенности и черты",
    btn_add_racial: "+ Добавить",
    panel_proficiencies: "Владение",
    // Notes tab
    panel_background: "Предыстория",
    panel_general_notes: "Общие заметки",
    panel_campaign_log: "Журнал кампании / группы",
    // HP Status
    status_fine: "В порядке!",
    status_hurt: "Ранен",
    status_bloodied: "Истекает кровью",
    status_dying: "Умирает",
    status_dead: "Мёртв",
    // Encumbrance
    enc_light: "Лёгкая",
    enc_medium: "Средняя",
    enc_heavy: "Тяжёлая",
    enc_overloaded: "Перегрузка",
    // Select options
    size_small: "Маленький",
    size_medium: "Средний",
    size_large: "Большой",
    size_huge: "Огромный",
    type_melee: "Ближний",
    type_ranged: "Дальний",
    type_armor: "Доспех",
    type_shield: "Щит",
    type_natural: "Природный",
    type_deflection: "Отражение",
    ability_int: "Интеллект",
    ability_wis: "Мудрость",
    ability_cha: "Харизма",
    // Language toggle
    lang_toggle: "RU",
  }
};

let currentLang = localStorage.getItem('pf18n_lang') || 'en';
```

- [ ] **Step 2: Add applyLanguage() function**

```javascript
function applyLanguage() {
  const strings = i18n[currentLang] || i18n.en;
  document.querySelectorAll('[data-i18n]').forEach(el => {
    const key = el.dataset.i18n;
    if (strings[key] !== undefined) {
      el.textContent = strings[key];
    }
  });
  document.querySelectorAll('[data-i18n-placeholder]').forEach(el => {
    const key = el.dataset.i18nPlaceholder;
    if (strings[key] !== undefined) {
      el.placeholder = strings[key];
    }
  });
  // Update lang toggle button text
  const langBtn = document.getElementById('langToggle');
  if (langBtn) langBtn.textContent = i18n[currentLang].lang_toggle;
}
```

- [ ] **Step 3: Add toggleLanguage() function**

```javascript
function toggleLanguage() {
  currentLang = currentLang === 'en' ? 'ru' : 'en';
  localStorage.setItem('pf18n_lang', currentLang);
  applyLanguage();
}
```

- [ ] **Step 4: Add data-i18n attributes to HTML elements**

Go through all HTML elements and add `data-i18n="key"` attributes. This is a large but mechanical change. Key locations:

- Tab buttons: `<button class="tab-btn" data-tab="main" data-i18n="tab_main">Main</button>`
- Panel titles: `<div class="panel-title" data-i18n="panel_character_info">Character Info</div>`
- Field labels: `<label class="field-label" data-i18n="lbl_name">Name</label>`
- Table headers: `<th data-i18n="th_cs">CS</th>`
- Buttons: `<button data-i18n="btn_add_weapon">+ Add Weapon</button>`
- Condition labels: `<label><input type="checkbox" ...> <span data-i18n="cond_blinded">Blinded</span></label>`

For dynamic table headers (skills, weapons, etc.), the `data-i18n` attributes go on the `<th>` elements in the `<thead>`.

- [ ] **Step 5: Add language toggle button to HTML**

In the `.char-selector` bar, add a toggle button before the character select:

```html
<div class="char-selector">
  <button id="langToggle" onclick="toggleLanguage()" class="lang-btn">EN</button>
  <select id="charSelect">...</select>
  <!-- ... existing buttons ... -->
</div>
```

- [ ] **Step 6: Call applyLanguage() on page load**

In the `loadCharacterList()` function (or at the end of the script), add:

```javascript
applyLanguage();
```

Also call `applyLanguage()` in `selectCharacter()` after rendering.

- [ ] **Step 7: Update renderSkillsTable() to use data-i18n for skill names**

In `renderSkillsTable()`, change skill name cells from:
```javascript
<td>${s.name}${s.trainedOnly ? ' *' : ''}</td>
```
to:
```javascript
<td data-i18n="skill_${s.key}">${s.name}${s.trainedOnly ? ' *' : ''}</td>
```

Add corresponding entries to the i18n dictionary for all 22 skill names.

- [ ] **Step 8: Test in browser**

- Click language toggle — all labels switch between EN/RU
- Refresh page — language preference persists
- Character data (name, notes, custom entries) is NOT translated
- All calculations still work
- No missing translations (no `data-i18n` key without a dictionary entry)

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: i18n language toggle with EN/RU translations"
```

---

### Task 3: Visual redesign — background + title banner

**Files:**
- Modify: `index.html` (CSS section + HTML for title)

**What this produces:** Atmospheric CSS-only background, stylized "PATHFINDER" title banner.

**Interfaces:**
- Consumes: existing CSS structure
- Produces: new background styles, title banner HTML + CSS

- [ ] **Step 1: Add atmospheric background CSS**

Replace the `body` background styling:

```css
body {
  font-family: Georgia, "Crimson Text", serif;
  background: #0d0b0f;
  color: #e8d5a3;
  font-size: 14px;
  line-height: 1.4;
  min-height: 100vh;
  position: relative;
}

body::before {
  content: '';
  position: fixed;
  inset: 0;
  z-index: -2;
  background:
    radial-gradient(ellipse at 20% 50%, rgba(30, 20, 15, 0.8) 0%, transparent 50%),
    radial-gradient(ellipse at 80% 20%, rgba(20, 15, 25, 0.6) 0%, transparent 40%),
    radial-gradient(ellipse at 50% 80%, rgba(25, 18, 10, 0.7) 0%, transparent 50%),
    radial-gradient(ellipse at 50% 50%, rgba(15, 12, 18, 0.9) 0%, transparent 70%),
    linear-gradient(180deg, #0d0b0f 0%, #1a1410 50%, #0d0b0f 100%);
}

body::after {
  content: '';
  position: fixed;
  inset: 0;
  z-index: -1;
  background: radial-gradient(ellipse at center, transparent 40%, rgba(0,0,0,0.6) 100%);
  pointer-events: none;
}
```

- [ ] **Step 2: Add title banner HTML**

After `<body>`, before the tab-nav, add:

```html
<div class="title-banner">
  <h1 class="title-text">PATHFINDER</h1>
  <div class="title-subtitle">1st Edition Character Sheet</div>
</div>
```

- [ ] **Step 3: Add title banner CSS**

```css
.title-banner {
  text-align: center;
  padding: 30px 20px 15px;
  position: relative;
}

.title-text {
  font-family: "Cinzel", Georgia, "Times New Roman", serif;
  font-size: 42px;
  font-variant: small-caps;
  letter-spacing: 0.3em;
  background: linear-gradient(180deg, #c9a84c 0%, #a08030 40%, #c9a84c 60%, #8b6914 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-shadow: none;
  margin: 0;
  line-height: 1.2;
}

.title-subtitle {
  font-family: Georgia, serif;
  font-size: 13px;
  color: #8b6914;
  letter-spacing: 0.15em;
  margin-top: 4px;
}
```

- [ ] **Step 4: Test in browser**

- Background shows layered dark gradients with vignette effect
- Title "PATHFINDER" displays with gold gradient text effect
- Subtitle shows below
- No visual overlap with content

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: atmospheric CSS background and PATHFINDER title banner"
```

---

### Task 4: Visual redesign — centered wood-panel layout

**Files:**
- Modify: `index.html` (CSS + HTML structure)

**What this produces:** Centered column layout with wood-grain panel texture, wider layout for table-heavy tabs.

**Interfaces:**
- Consumes: existing panel/section structure
- Produces: new layout CSS, `.wide` class for table tabs

- [ ] **Step 1: Add centered layout CSS**

```css
.content-wrapper {
  max-width: 550px;
  margin: 0 auto;
  padding: 0 15px 40px;
}

.tab-panel.wide {
  max-width: 950px;
}

@media (max-width: 700px) {
  .content-wrapper {
    max-width: 100%;
  }
  .tab-panel.wide {
    max-width: 100%;
  }
}
```

- [ ] **Step 2: Add .wide class to table-heavy tabs**

In the HTML, add `wide` class to Skills, Inventory, and Spells tab panels:

```html
<div id="tab-skills" class="tab-panel wide">
<div id="tab-inventory" class="tab-panel wide">
<div id="tab-spells" class="tab-panel wide">
```

- [ ] **Step 3: Wrap tab panels in content-wrapper**

Wrap all `.tab-panel` divs inside a `<div class="content-wrapper">`.

- [ ] **Step 4: Add wood-grain panel texture CSS**

Replace the current `.panel` styling:

```css
.panel {
  background:
    repeating-linear-gradient(
      87deg,
      rgba(42, 32, 24, 0.95) 0px,
      rgba(38, 28, 20, 0.95) 2px,
      rgba(45, 35, 25, 0.95) 4px,
      rgba(40, 30, 22, 0.95) 6px
    ),
    repeating-linear-gradient(
      90deg,
      rgba(50, 38, 28, 0.3) 0px,
      transparent 1px,
      transparent 3px
    );
  border: 1px solid #5a421e;
  border-radius: 4px;
  padding: 14px;
  margin-bottom: 14px;
  box-shadow:
    0 1px 0 rgba(255, 255, 255, 0.05),
    inset 0 1px 3px rgba(0, 0, 0, 0.4),
    0 2px 8px rgba(0, 0, 0, 0.3);
  position: relative;
}

.panel::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: 4px;
  border: 1px solid rgba(201, 168, 76, 0.1);
  pointer-events: none;
}
```

- [ ] **Step 5: Add panel title ornament CSS**

```css
.panel-title {
  font-family: "Cinzel", Georgia, "Times New Roman", serif;
  color: #c9a84c;
  font-size: 15px;
  margin-bottom: 10px;
  border-bottom: 1px solid rgba(107, 76, 30, 0.6);
  padding-bottom: 6px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.panel-title::before,
.panel-title::after {
  content: '═';
  color: #6b4c1e;
  font-size: 12px;
}

.panel-title::before { content: '═══ ✦ '; }
.panel-title::after { content: ' ✦ ═══'; }
```

- [ ] **Step 6: Test in browser**

- Panels have subtle wood-grain texture (horizontal stripes)
- Panels have beveled shadow effect (outer highlight + inner shadow)
- Panel titles show ornamental dividers: ═══ ✦ Title ✦ ═══
- Main/Abilities/Notes tabs are centered at 550px max
- Skills/Inventory/Spells tabs expand to 950px
- On narrow screens, all tabs go full-width

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: centered wood-panel layout with grain texture and ornaments"
```

---

### Task 5: Visual redesign — input/button/checkbox styling

**Files:**
- Modify: `index.html` (CSS section)

**What this produces:** Recessed inputs with gold focus glow, tactile buttons, custom checkboxes.

**Interfaces:**
- Consumes: existing input/button/checkbox elements
- Produces: new CSS for all interactive elements

- [ ] **Step 1: Restyle inputs**

Replace current input CSS:

```css
input, select, textarea {
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid #5a421e;
  border-bottom: 1px solid #8b6914;
  color: #e8d5a3;
  font-family: Georgia, serif;
  font-size: 14px;
  padding: 4px 6px;
  border-radius: 2px;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.4);
  transition: border-color 0.2s, box-shadow 0.2s;
}

input:focus, select:focus, textarea:focus {
  outline: none;
  border-color: #c9a84c;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.4), 0 0 8px rgba(201, 168, 76, 0.4);
}

textarea {
  border: 1px solid #5a421e;
  resize: vertical;
  width: 100%;
}
```

- [ ] **Step 2: Restyle buttons**

Add button styles:

```css
button, .btn {
  background: linear-gradient(180deg, #6b4c1e 0%, #4a3515 100%);
  color: #e8d5a3;
  border: 1px solid #8b6914;
  padding: 5px 12px;
  cursor: pointer;
  font-family: Georgia, serif;
  font-size: 13px;
  border-radius: 3px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
  transition: background 0.15s, box-shadow 0.15s;
}

button:hover {
  background: linear-gradient(180deg, #7a5a28 0%, #5a421e 100%);
  box-shadow: 0 1px 4px rgba(201, 168, 76, 0.3);
}

button:active {
  background: linear-gradient(180deg, #4a3515 0%, #3a2a10 100%);
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.5);
}
```

- [ ] **Step 3: Custom checkboxes**

```css
input[type="checkbox"] {
  appearance: none;
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  border: 1px solid #8b6914;
  border-radius: 2px;
  background: rgba(255, 255, 255, 0.04);
  cursor: pointer;
  position: relative;
  vertical-align: middle;
  box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.3);
  transition: background 0.15s, border-color 0.15s;
}

input[type="checkbox"]:checked {
  background: #c9a84c;
  border-color: #c9a84c;
}

input[type="checkbox"]:checked::after {
  content: '✓';
  position: absolute;
  top: -1px;
  left: 2px;
  color: #1a1410;
  font-size: 13px;
  font-weight: bold;
}

input[type="checkbox"]:focus {
  box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.3), 0 0 6px rgba(201, 168, 76, 0.4);
}
```

- [ ] **Step 4: Style tab navigation**

```css
.tab-btn {
  background: none;
  border: none;
  color: #6b5020;
  font-family: "Cinzel", Georgia, "Times New Roman", serif;
  font-size: 14px;
  padding: 10px 16px;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: color 0.2s, border-color 0.2s, text-shadow 0.2s;
}

.tab-btn:hover {
  color: #c9a84c;
  text-shadow: 0 0 8px rgba(201, 168, 76, 0.3);
}

.tab-btn.active {
  color: #c9a84c;
  border-bottom-color: #c9a84c;
  text-shadow: 0 0 10px rgba(201, 168, 76, 0.4);
}
```

- [ ] **Step 5: Style character selector**

```css
.char-selector {
  background: rgba(42, 32, 24, 0.9);
  border-bottom: 1px solid #5a421e;
  padding: 8px 15px;
  display: flex;
  align-items: center;
  gap: 10px;
  backdrop-filter: blur(4px);
}

.char-selector select {
  background: rgba(26, 20, 16, 0.9);
  color: #e8d5a3;
  border: 1px solid #5a421e;
  padding: 4px 8px;
  font-family: Georgia, serif;
  border-radius: 3px;
}
```

- [ ] **Step 6: Style language toggle button**

```css
.lang-btn {
  background: linear-gradient(180deg, #4a3515 0%, #3a2a10 100%);
  color: #c9a84c;
  border: 1px solid #6b4c1e;
  padding: 4px 10px;
  font-family: "Cinzel", Georgia, serif;
  font-size: 12px;
  font-weight: bold;
  letter-spacing: 1px;
  border-radius: 3px;
  cursor: pointer;
  min-width: 36px;
}

.lang-btn:hover {
  background: linear-gradient(180deg, #5a421e 0%, #4a3515 100%);
}
```

- [ ] **Step 7: Test in browser**

- Inputs have recessed/inset shadow appearance
- Focus shows gold glow ring
- Buttons have gradient, border, and pressed state on click
- Checkboxes are custom-styled with gold checkmark when checked
- Tab navigation has gold underline on active, glow on hover
- Language toggle button styled distinctly

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: game UI input/button/checkbox styling with wood texture"
```

---

### Task 6: Final polish — regression testing + print/responsive updates

**Files:**
- Modify: `index.html` (CSS: print styles, responsive breakpoints)

**What this produces:** Verified working app with no regressions, updated print CSS for new layout.

**Interfaces:**
- Consumes: all previous tasks
- Produces: final polished version

- [ ] **Step 1: Update print CSS for new layout**

Update the `@media print` block to account for new classes:

```css
@media print {
  body { background: white; color: black; font-size: 12px; }
  body::before, body::after { display: none !important; }
  .title-banner { display: none !important; }
  .tab-nav, .char-selector { display: none !important; }
  .content-wrapper { max-width: 100%; padding: 0; }
  .tab-panel { display: block !important; page-break-inside: avoid; margin-bottom: 20px; }
  .tab-panel.wide { max-width: 100%; }
  .panel {
    background: white;
    border: 1px solid #ccc;
    box-shadow: none;
  }
  .panel::before { display: none; }
  .panel-title { color: #333; border-bottom: 1px solid #ccc; }
  .panel-title::before, .panel-title::after { display: none; }
  input, select, textarea {
    border: 1px solid #999;
    color: black;
    background: transparent;
    box-shadow: none;
  }
  input[type="checkbox"] {
    appearance: auto;
    -webkit-appearance: auto;
    background: transparent;
    border: 1px solid #999;
  }
  input[type="checkbox"]:checked::after { display: none; }
  .derived { color: #333; }
  .field-label { color: #666; }
  th { background: #eee; color: #333; }
  td { background: white; }
  button { display: none !important; }
  h1, h2, h3, h4 { color: #333; }
  .lang-btn { display: none !important; }
}
```

- [ ] **Step 2: Update responsive breakpoints**

Ensure responsive CSS still works with new layout:

```css
@media (max-width: 900px) {
  .grid-6 { grid-template-columns: repeat(3, 1fr); }
  .grid-5 { grid-template-columns: repeat(3, 1fr); }
  .grid-4 { grid-template-columns: repeat(2, 1fr); }
  .grid-3 { grid-template-columns: 1fr 1fr; }
}

@media (max-width: 700px) {
  .title-text { font-size: 28px; letter-spacing: 0.2em; }
  .title-banner { padding: 20px 15px 10px; }
  .content-wrapper { max-width: 100%; }
  .tab-panel.wide { max-width: 100%; }
}

@media (max-width: 600px) {
  .grid-6 { grid-template-columns: repeat(2, 1fr); }
  .grid-5 { grid-template-columns: 1fr 1fr; }
  .grid-4 { grid-template-columns: 1fr; }
  .grid-3 { grid-template-columns: 1fr; }
  .grid-2 { grid-template-columns: 1fr; }
  .tab-btn { padding: 8px 10px; font-size: 12px; }
  table { font-size: 11px; }
}
```

- [ ] **Step 3: Regression test — verify all calculations**

Open `index.html` and test:

1. **Ability scores:** Change STR to 16 — modifier shows +3
2. **AC:** Set armor bonus to 4, DEX mod should apply — total AC correct
3. **Touch AC:** Should exclude armor/shield/natural
4. **Flat-Footed:** Should exclude dodge bonus
5. **CMB:** BAB + STR mod + size — correct
6. **CMD:** 10 + BAB + STR + DEX + size — correct
7. **Saves:** Base + ability mod + resistance + misc — correct for each
8. **Initiative:** DEX mod + misc — correct
9. **Full Attack:** BAB 6 shows two attacks (+1/+1), BAB 11 shows three
10. **Skills:** Acrobatics with DEX 14 (+2 mod), 5 ranks, CS checked = 10 total
11. **Skill Points:** Available 20, used sums all ranks, remaining = available - used
12. **Spells:** Caster level 5, INT 18 (+4) = concentration 9
13. **Inventory:** STR 10 light load 33 lbs, carrying capacity correct
14. **Encumbrance:** Changes from Light → Medium → Heavy as weight increases
15. **Weapons/Armor:** Add/remove works, equipped armor affects AC
16. **Export/Import:** Export character, import it back, all data preserved
17. **Language toggle:** RU/EN switches all labels, data preserved
18. **Input focus:** Type full sentence in Name — no focus loss
19. **CS checkbox:** Toggle on 3 skills — each recalculates correctly
20. **Print:** Cmd+P shows clean black-on-white layout

- [ ] **Step 4: Final commit**

```bash
git add index.html
git commit -m "feat: v2 complete — bug fixes, i18n, visual redesign"
```
