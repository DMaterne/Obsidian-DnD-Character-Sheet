```dataviewjs
const CHARACTER_PATH = "Public/3 Backend/EIMER.md";

const ITEMS_FOLDER = "Public/3 Backend/Items/";
const ACTIONS_FOLDER = "Public/3 Backend/Actions/";
const SPELLS_FOLDER = "Public/3 Backend/Spells/";
const FEATURES_FOLDER = "Public/3 Backend/Features/";

const c = dv.page(CHARACTER_PATH);

function modFromScore(score) {
  return Math.floor((score - 10) / 2);
}

function modString(n) {
  return n >= 0 ? `+${n}` : `${n}`;
}

function profValue(isProf, base, profBonus) {
  return base + (isProf ? profBonus : 0);
}

function normalizeRef(ref) {
  return String(ref ?? "").trim();
}

function resolvePathRef(ref, folderPath) {
  const raw = normalizeRef(ref);
  if (!raw) return null;

  const candidates = [];

  if (raw.includes("/")) {
    candidates.push(raw);
    if (!raw.endsWith(".md")) candidates.push(`${raw}.md`);
  } else {
    candidates.push(`${folderPath}${raw}`);
    candidates.push(`${folderPath}${raw}.md`);
  }

  for (const candidate of candidates) {
    const file = app.vault.getAbstractFileByPath(candidate);
    if (file) return candidate;
  }

  return null;
}

function resolvePageRef(ref, folderPath) {
  const resolvedPath = resolvePathRef(ref, folderPath);
  if (!resolvedPath) return null;
  return dv.page(resolvedPath);
}

function uniqueActionObjects(entries) {
  const seen = new Set();
  const result = [];

  for (const entry of entries) {
    const sourceType = String(entry?.source_type ?? "");
    const sourcePath = String(entry?.source_item_path ?? entry?.file?.path ?? "");
    const name = String(entry?.name ?? entry?.file?.name ?? "");
    const actionType = String(entry?.action_type ?? "");
    const category = String(entry?.category ?? "");
    const key = `${sourceType}::${sourcePath}::${name}::${actionType}::${category}`;

    if (seen.has(key)) continue;
    seen.add(key);
    result.push(entry);
  }

  return result;
}

function makeInventoryInstanceId(itemPath, inventoryIndex, quantityIndex = 0) {
  return `${itemPath}::${inventoryIndex}::${quantityIndex}`;
}

function getAttunedInstanceIds() {
  return Array.isArray(c.attuned_items)
    ? c.attuned_items.map(x => String(x))
    : [];
}

function isInventoryInstanceAttuned(instanceId) {
  return getAttunedInstanceIds().includes(String(instanceId));
}

function getAttunedItemPaths() {
  const inventoryEntries = Array.isArray(c.inventory) ? c.inventory : [];
  const attunedInstanceIds = getAttunedInstanceIds();
  const paths = [];

  for (let inventoryIndex = 0; inventoryIndex < inventoryEntries.length; inventoryIndex++) {
    const entry = inventoryEntries[inventoryIndex];
    const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
    if (!itemPath) continue;

    const quantity = Math.max(1, Number(entry?.quantity ?? 1));

    for (let quantityIndex = 0; quantityIndex < quantity; quantityIndex++) {
      const instanceId = makeInventoryInstanceId(itemPath, inventoryIndex, quantityIndex);
      if (attunedInstanceIds.includes(instanceId)) {
        paths.push(itemPath);
      }
    }
  }

  return paths;
}

function isItemEquipped(entry) {
  return entry?.equipped === true;
}

function isItemAttuned(itemPath) {
  return getAttunedItemPaths().includes(itemPath);
}

function getAbilityScoreBase(name) {
  const key = String(name ?? "").trim().toLowerCase();

  if (key === "str") return Number(c.str ?? 10);
  if (key === "dex") return Number(c.dex ?? 10);
  if (key === "con") return Number(c.con ?? 10);
  if (key === "int") return Number(c.int ?? 10);
  if (key === "wis") return Number(c.wis ?? 10);
  if (key === "cha") return Number(c.cha ?? 10);

  return 10;
}

function getFormulaContext() {
  return {
    str: getAbilityScoreBase("str"),
    dex: getAbilityScoreBase("dex"),
    con: getAbilityScoreBase("con"),
    int: getAbilityScoreBase("int"),
    wis: getAbilityScoreBase("wis"),
    cha: getAbilityScoreBase("cha"),

    str_mod: modFromScore(getAbilityScoreBase("str")),
    dex_mod: modFromScore(getAbilityScoreBase("dex")),
    con_mod: modFromScore(getAbilityScoreBase("con")),
    int_mod: modFromScore(getAbilityScoreBase("int")),
    wis_mod: modFromScore(getAbilityScoreBase("wis")),
    cha_mod: modFromScore(getAbilityScoreBase("cha")),

    prof: Number(c.proficiency_bonus ?? 2)
  };
}

function evaluateBonusFormula(formula) {
  const expr = String(formula ?? "").trim();
  if (!expr) return null;

  const ctx = getFormulaContext();
  let safeExpr = expr;

  const replacements = [
    ["str_mod", ctx.str_mod],
    ["dex_mod", ctx.dex_mod],
    ["con_mod", ctx.con_mod],
    ["int_mod", ctx.int_mod],
    ["wis_mod", ctx.wis_mod],
    ["cha_mod", ctx.cha_mod],
    ["str", ctx.str],
    ["dex", ctx.dex],
    ["con", ctx.con],
    ["int", ctx.int],
    ["wis", ctx.wis],
    ["cha", ctx.cha],
    ["prof", ctx.prof]
  ];

  for (const [key, value] of replacements) {
    safeExpr = safeExpr.replace(new RegExp(`\\b${key}\\b`, "g"), String(value));
  }

  safeExpr = safeExpr.replace(/\bmin\s*\(/g, "Math.min(");
  safeExpr = safeExpr.replace(/\bmax\s*\(/g, "Math.max(");

  const stripped = safeExpr.replace(/Math\.(min|max)/g, "");
  if (!/^[0-9+\-*/().,\s]*$/.test(stripped)) {
    return null;
  }

  try {
    const result = Function(`"use strict"; return (${safeExpr});`)();
    return Number.isFinite(result) ? Number(result) : null;
  } catch {
    return null;
  }
}

function getAllCharacterFeatures() {
  const featureRefs = Array.isArray(c.features) ? c.features : [];

  return featureRefs
    .map(ref => resolvePageRef(ref, FEATURES_FOLDER))
    .filter(p => p)
    .sort((a, b) => {
      const nameA = String(a.name ?? a.file?.name ?? "");
      const nameB = String(b.name ?? b.file?.name ?? "");
      return nameA.localeCompare(nameB, "de");
    });
}

function getActiveBonusEffects() {
  const activeEffects = [];

  const inventoryEntries = Array.isArray(c.inventory) ? c.inventory : [];

  for (let inventoryIndex = 0; inventoryIndex < inventoryEntries.length; inventoryIndex++) {
    const entry = inventoryEntries[inventoryIndex];
    const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
    if (!itemPath) continue;

    const itemPage = dv.page(itemPath);
    if (!itemPage) continue;

    const bonuses = Array.isArray(itemPage.bonuses) ? itemPage.bonuses : [];
    if (bonuses.length === 0) continue;

    const equipped = isItemEquipped(entry);
    const quantity = Math.max(1, Number(entry?.quantity ?? 1));

    for (let quantityIndex = 0; quantityIndex < quantity; quantityIndex++) {
      const instanceId = makeInventoryInstanceId(itemPath, inventoryIndex, quantityIndex);
      const attuned = isInventoryInstanceAttuned(instanceId);

      for (const bonus of bonuses) {
        if (!bonus || typeof bonus !== "object") continue;

        const activeWhen = String(bonus.active_when ?? "equipped").trim().toLowerCase();
        const type = String(bonus.type ?? "").trim().toLowerCase();
        const value = Number(bonus.value ?? 0);

        const rawSetValue = bonus.set_value;
        const hasSetValue = rawSetValue !== undefined && rawSetValue !== null && rawSetValue !== "";
        const setValue = hasSetValue ? Number(rawSetValue) : null;

        const formula = String(bonus.formula ?? "").trim();

        if (!type) continue;
        if (
          !Number.isFinite(value) &&
          !(hasSetValue && Number.isFinite(setValue)) &&
          !formula
        ) continue;

        let isActive = false;

        if (activeWhen === "always") isActive = true;
        if (activeWhen === "equipped" && equipped) isActive = true;
        if (activeWhen === "attuned" && attuned) isActive = true;

        if (!isActive) continue;

        activeEffects.push({
          type,
          value: Number.isFinite(value) ? value : 0,
          set_value: hasSetValue && Number.isFinite(setValue) ? setValue : null,
          formula: formula || null,
          active_when: activeWhen,
          source_kind: "item",
          source_name: itemPage.name ?? itemPage.file?.name ?? "Unknown Item",
          source_path: itemPath,
          source_instance_id: instanceId
        });
      }
    }
  }

  const features = getAllCharacterFeatures();

  for (const featurePage of features) {
    const bonuses = Array.isArray(featurePage.bonuses) ? featurePage.bonuses : [];
    if (bonuses.length === 0) continue;

    const enabled = featurePage.enabled !== false;

    for (const bonus of bonuses) {
      if (!bonus || typeof bonus !== "object") continue;

      const activeWhen = String(bonus.active_when ?? "enabled").trim().toLowerCase();
      const type = String(bonus.type ?? "").trim().toLowerCase();
      const value = Number(bonus.value ?? 0);

      const rawSetValue = bonus.set_value;
      const hasSetValue = rawSetValue !== undefined && rawSetValue !== null && rawSetValue !== "";
      const setValue = hasSetValue ? Number(rawSetValue) : null;

      const formula = String(bonus.formula ?? "").trim();

      if (!type) continue;
      if (
        !Number.isFinite(value) &&
        !(hasSetValue && Number.isFinite(setValue)) &&
        !formula
      ) continue;

      let isActive = false;

      if (activeWhen === "always") isActive = true;
      if (activeWhen === "enabled" && enabled) isActive = true;

      if (!isActive) continue;

      activeEffects.push({
        type,
        value: Number.isFinite(value) ? value : 0,
        set_value: hasSetValue && Number.isFinite(setValue) ? setValue : null,
        formula: formula || null,
        active_when: activeWhen,
        source_kind: "feature",
        source_name: featurePage.name ?? featurePage.file?.name ?? "Unnamed Feature",
        source_path: featurePage.file?.path ?? null
      });
    }
  }

  return activeEffects;
}

function applyItemEffects(baseValue, type) {
  let result = Number(baseValue ?? 0);
  const targetType = String(type ?? "").trim().toLowerCase();

  const matchingEffects = getActiveBonusEffects().filter(b => b.type === targetType);

  const setValues = matchingEffects
    .filter(b => b.set_value != null && Number.isFinite(Number(b.set_value)))
    .map(b => Number(b.set_value));

  const formulaValues = matchingEffects
    .filter(b => b.formula)
    .map(b => evaluateBonusFormula(b.formula))
    .filter(v => v != null && Number.isFinite(v));

  if (setValues.length > 0 || formulaValues.length > 0) {
    const candidates = [...setValues, ...formulaValues];
    result = Math.max(...candidates);
  }

  const additiveBonus = matchingEffects.reduce((sum, b) => {
    return sum + Number(b.value ?? 0);
  }, 0);

  result += additiveBonus;
  return result;
}

function getAbilityScore(name) {
  const key = String(name ?? "").trim().toLowerCase();

  if (key === "str") return applyItemEffects(Number(c.str ?? 10), "str");
  if (key === "dex") return applyItemEffects(Number(c.dex ?? 10), "dex");
  if (key === "con") return applyItemEffects(Number(c.con ?? 10), "con");
  if (key === "int") return applyItemEffects(Number(c.int ?? 10), "int");
  if (key === "wis") return applyItemEffects(Number(c.wis ?? 10), "wis");
  if (key === "cha") return applyItemEffects(Number(c.cha ?? 10), "cha");

  return 10;
}

function getAbilityModByName(name) {
  return modFromScore(getAbilityScore(name));
}

function getProficiencyBonus() {
  return applyItemEffects(Number(c.proficiency_bonus ?? 2), "proficiency_bonus");
}

function getArmorClass() {
  return applyItemEffects(Number(c.ac ?? 10), "ac");
}

function getMaxHp() {
  return applyItemEffects(Number(c.hp_max ?? 0), "hp_max");
}

function getSpeed() {
  return applyItemEffects(Number(c.speed ?? 30), "speed");
}

function getInitiativeBonus() {
  const baseInit = c.initiative_bonus != null
    ? Number(c.initiative_bonus)
    : getAbilityModByName("dex");

  return applyItemEffects(baseInit, "initiative");
}

function getSavingThrowTotal(label, isProficient) {
  const abilityKey = String(label ?? "").trim().toLowerCase();
  const base = getAbilityModByName(abilityKey);
  const profBonus = getProficiencyBonus();

  let total = profValue(isProficient, base, profBonus);
  total = applyItemEffects(total, `${abilityKey}_save`);
  total = applyItemEffects(total, "saving_throws");
  return total;
}

const SKILL_KEY_MAP = {
  "Acrobatics": "acrobatics",
  "Animal Handling": "animal_handling",
  "Arcana": "arcana",
  "Athletics": "athletics",
  "Deception": "deception",
  "History": "history",
  "Insight": "insight",
  "Intimidation": "intimidation",
  "Investigation": "investigation",
  "Medicine": "medicine",
  "Nature": "nature",
  "Perception": "perception",
  "Performance": "performance",
  "Persuasion": "persuasion",
  "Religion": "religion",
  "Sleight of Hand": "sleight_of_hand",
  "Stealth": "stealth",
  "Survival": "survival"
};

function getSkillTotal(label, abilityName, isProficient) {
  const base = getAbilityModByName(abilityName);
  const profBonus = getProficiencyBonus();
  const skillKey = SKILL_KEY_MAP[label] ?? "";

  let total = profValue(isProficient, base, profBonus);
  if (skillKey) {
    total = applyItemEffects(total, skillKey);
  }

  return total;
}

async function toggleInventoryEquip(itemPathOrName) {
  const characterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);
  if (!characterFile) return;

  const resolvedItemPath =
    resolvePathRef(itemPathOrName, ITEMS_FOLDER) ?? String(itemPathOrName ?? "").trim();

  const itemPage = dv.page(resolvedItemPath);
  if (!itemPage) return;

  if (itemPage.equipment !== true) {
    new Notice("Dieses Item ist kein Equipment.");
    return;
  }

  await app.fileManager.processFrontMatter(characterFile, (fm) => {
    if (!Array.isArray(fm.inventory)) return;

    const entry = fm.inventory.find(invEntry => {
      const invPath =
        resolvePathRef(invEntry?.item, ITEMS_FOLDER) ?? String(invEntry?.item ?? "").trim();
      return invPath === resolvedItemPath;
    });

    if (!entry) return;

    entry.equipped = entry.equipped === true ? false : true;
  });
}

function getAllCharacterActions() {
  const collectedActions = [];

  const explicitActionRefs = Array.isArray(c.actions) ? c.actions : [];

  for (const actionRef of explicitActionRefs) {
    const actionPage = resolvePageRef(actionRef, ACTIONS_FOLDER);
    if (!actionPage) continue;

    const category = String(actionPage.category ?? "").trim().toLowerCase();
    if (category === "spell") continue;

    collectedActions.push({
      ...actionPage,
      source_type: "character",
      source_label: "Character Sheet",
      file: actionPage.file
    });
  }

  const inventoryEntries = Array.isArray(c.inventory) ? c.inventory : [];

  for (const entry of inventoryEntries) {
    const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
    if (!itemPath) continue;

    const itemPage = dv.page(itemPath);
    if (!itemPage) continue;

    const itemActions = Array.isArray(itemPage.actions) ? itemPage.actions : [];
    if (itemActions.length === 0) continue;

    for (const action of itemActions) {
      if (!action || typeof action !== "object") continue;

      const category = String(action.category ?? "").trim().toLowerCase();
      if (category === "spell") continue;

      collectedActions.push({
        ...action,
        source_type: "item",
        source_item_name: itemPage.name ?? itemPage.file?.name ?? "Unknown Item",
        source_item_path: itemPage.file?.path ?? itemPath,
        source_label: itemPage.name ?? itemPage.file?.name ?? "Unknown Item",
        file: {
          name: action.name ?? itemPage.name ?? itemPage.file?.name ?? "Unnamed Action",
          path: itemPage.file?.path ?? itemPath
        }
      });
    }
  }

  return uniqueActionObjects(collectedActions);
}

function getAllCharacterSpells() {
  const collectedSpells = [];

  const explicitSpellRefs = Array.isArray(c.spells) ? c.spells : [];

  for (const spellRef of explicitSpellRefs) {
    const spellPage = resolvePageRef(spellRef, SPELLS_FOLDER);
    if (!spellPage) continue;

    const category = String(spellPage.category ?? "").trim().toLowerCase();
    if (category && category !== "spell") continue;

    collectedSpells.push({
      ...spellPage,
      source_type: "character",
      source_label: "Character Sheet",
      file: spellPage.file
    });
  }

  const inventoryEntries = Array.isArray(c.inventory) ? c.inventory : [];

  for (const entry of inventoryEntries) {
    const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
    if (!itemPath) continue;

    const itemPage = dv.page(itemPath);
    if (!itemPage) continue;

    const itemActions = Array.isArray(itemPage.actions) ? itemPage.actions : [];
    if (itemActions.length === 0) continue;

    for (const action of itemActions) {
      if (!action || typeof action !== "object") continue;

      const category = String(action.category ?? "").trim().toLowerCase();
      if (category !== "spell") continue;

      collectedSpells.push({
        ...action,
        source_type: "item",
        source_item_name: itemPage.name ?? itemPage.file?.name ?? "Unknown Item",
        source_item_path: itemPage.file?.path ?? itemPath,
        source_label: itemPage.name ?? itemPage.file?.name ?? "Unknown Item",
        file: {
          name: action.name ?? itemPage.name ?? itemPage.file?.name ?? "Unnamed Spell",
          path: itemPage.file?.path ?? itemPath
        }
      });
    }
  }

  return uniqueActionObjects(collectedSpells);
}

async function renderMarkdownInto(container, markdownText, filePathForLinks = CHARACTER_PATH) {
  container.innerHTML = "";

  const md = String(markdownText ?? "");
  if (!md.trim()) {
    container.createEl("div", { text: "Keine Notizen eingetragen." });
    return;
  }

  const sourcePath = String(filePathForLinks ?? CHARACTER_PATH);

  const MDRenderer =
    (typeof MarkdownRenderer !== "undefined" && MarkdownRenderer) ||
    (typeof obsidian !== "undefined" && obsidian?.MarkdownRenderer) ||
    (typeof window !== "undefined" && window?.MarkdownRenderer) ||
    (typeof window !== "undefined" && window?.obsidian?.MarkdownRenderer);

  if (MDRenderer && typeof MDRenderer.renderMarkdown === "function") {
    await MDRenderer.renderMarkdown(md, container, sourcePath, null);
    return;
  }

  // Fallback, falls der Renderer in deinem DataviewJS-Kontext nicht verfügbar ist
  container.createEl("pre", { text: md });
}

if (!c) {
  dv.paragraph("Character-Datei nicht gefunden.");
} else {
  const wrapper = dv.el("div", "");
  wrapper.style.display = "flex";
  wrapper.style.flexDirection = "column";
  wrapper.style.gap = "18px";

  const headerRow = wrapper.createEl("div");
  headerRow.style.display = "grid";
  headerRow.style.gridTemplateColumns = "1fr 220px 260px";
  headerRow.style.gap = "18px";
  headerRow.style.alignItems = "stretch";

  const header = headerRow.createEl("div");
  header.style.padding = "16px";
  header.style.border = "1px solid var(--background-modifier-border)";
  header.style.borderRadius = "14px";
  header.style.display = "grid";
  header.style.gridTemplateColumns = "1fr 360px";
  header.style.gap = "16px";
  header.style.alignItems = "center";

  const headerText = header.createEl("div");

  const title = headerText.createEl("div", { text: c.name ?? "Unbenannter Charakter" });
  title.style.fontSize = "1.8em";
  title.style.fontWeight = "700";

  const subtitle = headerText.createEl("div");
  subtitle.style.opacity = "0.75";
  subtitle.style.marginTop = "4px";
  subtitle.style.display = "flex";
  subtitle.style.flexDirection = "column";
  subtitle.style.gap = "4px";

  function addHeaderInfoRow(parent, label, value) {
    const row = parent.createEl("div");
    const labelEl = row.createEl("span", { text: `${label}: ` });
    labelEl.style.fontWeight = "600";
    row.createEl("span", { text: String(value ?? "-") });
  }

  addHeaderInfoRow(subtitle, "Race", c.race ?? "Unknown Race");
  addHeaderInfoRow(subtitle, "Class", c.class ?? "Unknown Class");
  addHeaderInfoRow(
    subtitle,
    "Subclass",
    Array.isArray(c.subclass) ? c.subclass.filter(Boolean).join(", ") || "-" : (c.subclass ?? "-")
  );
  addHeaderInfoRow(subtitle, "Level", c.level ?? 1);

  const meta = headerText.createEl("div");
  meta.style.opacity = "0.7";
  meta.style.marginTop = "6px";
  meta.style.fontSize = "0.95em";
  meta.style.display = "flex";
  meta.style.flexDirection = "column";
  meta.style.gap = "4px";

  function addMetaInfoRow(parent, label, value) {
    const row = parent.createEl("div");
    const labelEl = row.createEl("span", { text: `${label}: ` });
    labelEl.style.fontWeight = "600";
    row.createEl("span", { text: String(value ?? "-") });
  }

  addMetaInfoRow(meta, "Background", c.background ?? "-");
  addMetaInfoRow(meta, "Alignment", c.alignment ?? "-");
  addMetaInfoRow(meta, "Player", c.player ?? "-");

  const portraitWrap = header.createEl("div");
  portraitWrap.style.display = "flex";
  portraitWrap.style.flexDirection = "column";
  portraitWrap.style.justifyContent = "center";
  portraitWrap.style.alignItems = "center";
  portraitWrap.style.gap = "8px";

  const portraitPath = c.portrait ?? "Public/1 Assets/Bilder/EIMER.png";
  const portraitFile = app.vault.getAbstractFileByPath(portraitPath);
  const portraitUrl = app.vault.adapter.getResourcePath(portraitPath);

  const portraitAnchor = portraitWrap.createEl("a");
  portraitAnchor.href = "#";
  portraitAnchor.style.display = "block";
  portraitAnchor.style.width = "240px";
  portraitAnchor.style.height = "240px";
  portraitAnchor.style.flexShrink = "0";

  portraitAnchor.addEventListener("click", async (evt) => {
    evt.preventDefault();
    if (portraitFile) {
      await app.workspace.getLeaf(true).openFile(portraitFile);
    }
  });

  const portrait = portraitAnchor.createEl("img");
  portrait.src = portraitUrl;
  portrait.alt = c.name ?? "Character Portrait";
  portrait.style.width = "240px";
  portrait.style.height = "240px";
  portrait.style.minWidth = "240px";
  portrait.style.maxWidth = "240px";
  portrait.style.minHeight = "240px";
  portrait.style.maxHeight = "240px";
  portrait.style.objectFit = "cover";
  portrait.style.objectPosition = "center center";
  portrait.style.display = "block";
  portrait.style.borderRadius = "20px";
  portrait.style.border = "1px solid var(--background-modifier-border)";
  portrait.style.background = "var(--background-secondary)";
  portrait.style.boxSizing = "border-box";

  const comValCard = headerRow.createEl("div");
  comValCard.style.padding = "16px";
  comValCard.style.border = "1px solid var(--background-modifier-border)";
  comValCard.style.borderRadius = "14px";
  comValCard.style.display = "flex";
  comValCard.style.flexDirection = "column";
  comValCard.style.gap = "10px";

  const comValTitle = comValCard.createEl("div", { text: "Combat Values" });
  comValTitle.style.fontWeight = "700";
  comValTitle.style.fontSize = "1.1em";
  comValTitle.style.marginBottom = "4px";

  const comValGrid = comValCard.createEl("div");
  comValGrid.style.display = "grid";
  comValGrid.style.gridTemplateColumns = "1fr";
  comValGrid.style.gap = "8px";

  const combatValues = [
    ["Armor Class", getArmorClass()],
    ["Initiative", modString(getInitiativeBonus())],
    ["Speed", `${getSpeed()} ft`],
  ];

  for (const [label, value] of combatValues) {
    const row = comValGrid.createEl("div");
    row.style.padding = "10px";
    row.style.border = "1px solid var(--background-modifier-border)";
    row.style.borderRadius = "10px";
    row.style.textAlign = "center";

    const labelEl = row.createEl("div", { text: label });
    labelEl.style.fontSize = "0.8em";
    labelEl.style.opacity = "0.7";
    labelEl.style.marginBottom = "4px";

    const valueEl = row.createEl("div", { text: String(value) });
    valueEl.style.fontSize = "1.2em";
    valueEl.style.fontWeight = "700";
  }

  const hpCard = headerRow.createEl("div");
  hpCard.style.padding = "16px";
  hpCard.style.border = "1px solid var(--background-modifier-border)";
  hpCard.style.borderRadius = "14px";
  hpCard.style.display = "flex";
  hpCard.style.flexDirection = "column";
  hpCard.style.gap = "12px";

  const hpTitle = hpCard.createEl("div", { text: "Hit Points" });
  hpTitle.style.fontWeight = "700";
  hpTitle.style.fontSize = "1.1em";

  const characterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);

  let currentHp = Number(c.hp_current ?? 0);
  const maxHp = getMaxHp();
  let tempHpValue = Number(c.hp_temp ?? 0);

  const hpMain = hpCard.createEl("div", {
    text: `${currentHp} / ${maxHp}`
  });
  hpMain.style.fontSize = "1.8em";
  hpMain.style.fontWeight = "700";
  hpMain.style.textAlign = "center";
  hpMain.style.padding = "8px";
  hpMain.style.border = "1px solid var(--background-modifier-border)";
  hpMain.style.borderRadius = "10px";

  const tempHp = hpCard.createEl("div", {
    text: `Temp HP: ${tempHpValue}`
  });
  tempHp.style.opacity = "0.8";
  tempHp.style.textAlign = "center";

  function refreshHpDisplay() {
    hpMain.setText(`${currentHp} / ${maxHp}`);
    tempHp.setText(`Temp HP: ${tempHpValue}`);
    tempInput.value = String(tempHpValue);
    deathSavesCard.style.display = currentHp <= 0 ? "block" : "none";
  }

  async function saveHpValues() {
    if (!characterFile) return;

    await app.fileManager.processFrontMatter(characterFile, (fm) => {
      fm.hp_current = currentHp;
      fm.hp_temp = tempHpValue;
    });
  }

  const actionRow = hpCard.createEl("div");
  actionRow.style.display = "grid";
  actionRow.style.gridTemplateColumns = "1fr auto auto";
  actionRow.style.gap = "8px";
  actionRow.style.alignItems = "center";

  const hpValueInput = actionRow.createEl("input");
  hpValueInput.type = "number";
  hpValueInput.placeholder = "Value";
  hpValueInput.style.width = "100%";
  hpValueInput.style.padding = "8px 10px";
  hpValueInput.style.border = "1px solid var(--background-modifier-border)";
  hpValueInput.style.borderRadius = "8px";
  hpValueInput.style.background = "var(--background-primary)";
  hpValueInput.style.color = "var(--text-normal)";

  const damageBtn = actionRow.createEl("button", { text: "Damage" });
  damageBtn.style.padding = "8px 12px";
  damageBtn.style.borderRadius = "8px";
  damageBtn.style.border = "1px solid var(--background-modifier-border)";
  damageBtn.style.cursor = "pointer";
  damageBtn.style.background = "var(--background-secondary)";
  damageBtn.style.color = "var(--text-normal)";
  damageBtn.style.fontWeight = "600";

  const healBtn = actionRow.createEl("button", { text: "Heal" });
  healBtn.style.padding = "8px 12px";
  healBtn.style.borderRadius = "8px";
  healBtn.style.border = "1px solid var(--background-modifier-border)";
  healBtn.style.cursor = "pointer";
  healBtn.style.background = "var(--background-secondary)";
  healBtn.style.color = "var(--text-normal)";
  healBtn.style.fontWeight = "600";

  const tempRow = hpCard.createEl("div");
  tempRow.style.display = "grid";
  tempRow.style.gridTemplateColumns = "70px 1fr auto";
  tempRow.style.gap = "8px";
  tempRow.style.alignItems = "center";

  const tempLabel = tempRow.createEl("div", { text: "Temp HP" });
  tempLabel.style.fontWeight = "600";

  const tempInput = tempRow.createEl("input");
  tempInput.type = "number";
  tempInput.placeholder = String(tempHpValue);
  tempInput.value = String(tempHpValue);
  tempInput.style.width = "100%";
  tempInput.style.padding = "8px 10px";
  tempInput.style.border = "1px solid var(--background-modifier-border)";
  tempInput.style.borderRadius = "8px";
  tempInput.style.background = "var(--background-primary)";
  tempInput.style.color = "var(--text-normal)";

  const tempSaveBtn = tempRow.createEl("button", { text: "Set" });
  tempSaveBtn.style.padding = "8px 12px";
  tempSaveBtn.style.borderRadius = "8px";
  tempSaveBtn.style.border = "1px solid var(--background-modifier-border)";
  tempSaveBtn.style.cursor = "pointer";
  tempSaveBtn.style.background = "var(--background-secondary)";
  tempSaveBtn.style.color = "var(--text-normal)";
  tempSaveBtn.style.fontWeight = "600";

  damageBtn.addEventListener("click", async () => {
    let amount = Number(hpValueInput.value ?? 0);
    if (!Number.isFinite(amount) || amount <= 0) return;

    amount = Math.floor(amount);

    if (tempHpValue > 0) {
      const absorbed = Math.min(tempHpValue, amount);
      tempHpValue -= absorbed;
      amount -= absorbed;
    }

    if (amount > 0) {
      currentHp = Math.max(0, currentHp - amount);
    }

    hpValueInput.value = "";
    refreshHpDisplay();
    await saveHpValues();
  });

  healBtn.addEventListener("click", async () => {
    let amount = Number(hpValueInput.value ?? 0);
    if (!Number.isFinite(amount) || amount <= 0) return;

    amount = Math.floor(amount);
    currentHp = Math.min(maxHp, currentHp + amount);

    hpValueInput.value = "";
    refreshHpDisplay();
    await saveHpValues();
  });

  tempSaveBtn.addEventListener("click", async () => {
    const value = Number(tempInput.value ?? 0);
    tempHpValue = Math.max(0, Math.floor(value || 0));
    refreshHpDisplay();
    await saveHpValues();
  });

  const deathSavesCard = hpCard.createEl("div");
  deathSavesCard.style.marginTop = "8px";
  deathSavesCard.style.padding = "10px";
  deathSavesCard.style.border = "1px solid var(--background-modifier-border)";
  deathSavesCard.style.borderRadius = "10px";
  deathSavesCard.style.display = currentHp <= 0 ? "block" : "none";

  const deathSavesTitle = deathSavesCard.createEl("div", { text: "Death Saves" });
  deathSavesTitle.style.fontWeight = "700";
  deathSavesTitle.style.marginBottom = "8px";

  function createDeathSaveRow(labelText) {
    const row = deathSavesCard.createEl("div");
    row.style.display = "grid";
    row.style.gridTemplateColumns = "70px repeat(3, 20px)";
    row.style.gap = "8px";
    row.style.alignItems = "center";
    row.style.marginBottom = "6px";

    const label = row.createEl("div", { text: labelText });
    label.style.fontWeight = "600";

    for (let i = 0; i < 3; i++) {
      const checkbox = row.createEl("input");
      checkbox.type = "checkbox";
    }
  }

  createDeathSaveRow("Fail");
  createDeathSaveRow("Success");

  const abilitiesCard = wrapper.createEl("div");
  abilitiesCard.style.padding = "8px";
  abilitiesCard.style.border = "1px solid var(--background-modifier-border)";
  abilitiesCard.style.borderRadius = "7px";

  const abilitiesTitle = abilitiesCard.createEl("div", { text: "Ability Scores" });
  abilitiesTitle.style.fontWeight = "700";
  abilitiesTitle.style.marginBottom = "6px";
  abilitiesTitle.style.fontSize = "1.1em";

  const abilityGrid = abilitiesCard.createEl("div");
  abilityGrid.style.display = "grid";
  abilityGrid.style.gridTemplateColumns = "repeat(6, 1fr)";
  abilityGrid.style.gap = "6px";

  const stats = [
    ["STR", getAbilityScore("str")],
    ["DEX", getAbilityScore("dex")],
    ["CON", getAbilityScore("con")],
    ["INT", getAbilityScore("int")],
    ["WIS", getAbilityScore("wis")],
    ["CHA", getAbilityScore("cha")],
  ];

  for (const [label, score] of stats) {
    const mod = modFromScore(score);

    const card = abilityGrid.createEl("div");
    card.style.padding = "14px";
    card.style.border = "1px solid var(--background-modifier-border)";
    card.style.borderRadius = "12px";
    card.style.textAlign = "center";

    const labelEl = card.createEl("div", { text: label });
    labelEl.style.fontSize = "0.85em";
    labelEl.style.opacity = "0.7";

    const modEl = card.createEl("div", { text: modString(mod) });
    modEl.style.fontSize = "1.5em";
    modEl.style.fontWeight = "700";
    modEl.style.marginTop = "6px";

    const scoreEl = card.createEl("div", { text: String(score) });
    scoreEl.style.fontSize = "0.85em";
    scoreEl.style.opacity = "0.75";
    scoreEl.style.marginTop = "4px";
  }

  const mainGrid = wrapper.createEl("div");
  mainGrid.style.display = "grid";
  mainGrid.style.gridTemplateColumns = "1fr 1fr 2fr";
  mainGrid.style.gap = "18px";

  const leftCol = mainGrid.createEl("div");
  leftCol.style.display = "flex";
  leftCol.style.flexDirection = "column";
  leftCol.style.gap = "9px";

  const middleCol = mainGrid.createEl("div");
  middleCol.style.display = "flex";
  middleCol.style.flexDirection = "column";
  middleCol.style.gap = "9px";

  const rightCol = mainGrid.createEl("div");
  rightCol.style.display = "flex";
  rightCol.style.flexDirection = "column";
  rightCol.style.gap = "9px";

  const savesCard = leftCol.createEl("div");
  savesCard.style.padding = "8px";
  savesCard.style.border = "1px solid var(--background-modifier-border)";
  savesCard.style.borderRadius = "7px";

  const savesTitle = savesCard.createEl("div", { text: "Saving Throws" });
  savesTitle.style.fontWeight = "700";
  savesTitle.style.marginBottom = "6px";
  savesTitle.style.fontSize = "1.1em";

  const saveData = [
    ["STR", c.str_save_prof ?? false],
    ["DEX", c.dex_save_prof ?? false],
    ["CON", c.con_save_prof ?? false],
    ["INT", c.int_save_prof ?? false],
    ["WIS", c.wis_save_prof ?? false],
    ["CHA", c.cha_save_prof ?? false],
  ];

  for (const [label, prof] of saveData) {
    const row = savesCard.createEl("div");
    row.style.display = "flex";
    row.style.justifyContent = "space-between";
    row.style.padding = "6px 0";
    row.style.borderBottom = "1px solid var(--background-modifier-border-hover)";

    row.createEl("div", { text: `${prof ? "●" : "○"} ${label}` });

    const totalSaveBonus = getSavingThrowTotal(label, prof);

    const right = row.createEl("div", {
      text: modString(totalSaveBonus)
    });
    right.style.fontWeight = "600";
  }

  const skillsCard = middleCol.createEl("div");
  skillsCard.style.padding = "6px";
  skillsCard.style.border = "1px solid var(--background-modifier-border)";
  skillsCard.style.borderRadius = "5px";

  const skillsTitle = skillsCard.createEl("div", { text: "Skills" });
  skillsTitle.style.fontWeight = "700";
  skillsTitle.style.marginBottom = "6px";
  skillsTitle.style.fontSize = "1.1em";

  const skills = [
    ["Acrobatics", "dex", c.acrobatics_prof ?? false],
    ["Animal Handling", "wis", c.animal_handling_prof ?? false],
    ["Arcana", "int", c.arcana_prof ?? false],
    ["Athletics", "str", c.athletics_prof ?? false],
    ["Deception", "cha", c.deception_prof ?? false],
    ["History", "int", c.history_prof ?? false],
    ["Insight", "wis", c.insight_prof ?? false],
    ["Intimidation", "cha", c.intimidation_prof ?? false],
    ["Investigation", "int", c.investigation_prof ?? false],
    ["Medicine", "wis", c.medicine_prof ?? false],
    ["Nature", "int", c.nature_prof ?? false],
    ["Perception", "wis", c.perception_prof ?? false],
    ["Performance", "cha", c.performance_prof ?? false],
    ["Persuasion", "cha", c.persuasion_prof ?? false],
    ["Religion", "int", c.religion_prof ?? false],
    ["Sleight of Hand", "dex", c.sleight_of_hand_prof ?? false],
    ["Stealth", "dex", c.stealth_prof ?? false],
    ["Survival", "wis", c.survival_prof ?? false],
  ];

  for (const [name, ability, prof] of skills) {
    const row = skillsCard.createEl("div");
    row.style.display = "flex";
    row.style.justifyContent = "space-between";
    row.style.padding = "6px 0";
    row.style.borderBottom = "1px solid var(--background-modifier-border-hover)";

    row.createEl("div", { text: `${prof ? "●" : "○"} ${name}` });

    const totalSkillValue = getSkillTotal(name, ability, prof);

    const value = row.createEl("div", {
      text: modString(totalSkillValue)
    });
    value.style.fontWeight = "600";
  }

  const tabCard = rightCol.createEl("div");
  tabCard.style.padding = "16px";
  tabCard.style.border = "1px solid var(--background-modifier-border)";
  tabCard.style.borderRadius = "14px";
  tabCard.style.display = "flex";
  tabCard.style.flexDirection = "column";
  tabCard.style.gap = "12px";

  const tabTitle = tabCard.createEl("div", { text: "Character Details" });
  tabTitle.style.fontWeight = "700";
  tabTitle.style.fontSize = "1.1em";

  const tabBar = tabCard.createEl("div");
  tabBar.style.display = "flex";
  tabBar.style.gap = "8px";
  tabBar.style.flexWrap = "wrap";

  const tabContent = tabCard.createEl("div");
  tabContent.style.padding = "10px";
  tabContent.style.border = "1px solid var(--background-modifier-border-hover)";
  tabContent.style.borderRadius = "10px";
  tabContent.style.minHeight = "220px";

  let activeTab = window.__dndActiveTab ?? "actions";
  const tabButtons = {};

  function clearEl(el) {
    el.innerHTML = "";
  }

  async function renderFeatures() {
    clearEl(tabContent);

    const title = tabContent.createEl("div", { text: "Features & Traits" });
    title.style.fontWeight = "700";
    title.style.fontSize = "1.05em";
    title.style.marginBottom = "10px";

    const features = getAllCharacterFeatures();

    if (features.length === 0) {
      tabContent.createEl("div", { text: "Keine Features eingetragen." });
      return;
    }

    const searchWrap = tabContent.createEl("div");
    searchWrap.style.marginBottom = "10px";

    const searchInput = searchWrap.createEl("input");
    searchInput.type = "text";
    searchInput.placeholder = "Search Features";
    searchInput.style.width = "100%";
    searchInput.style.padding = "8px 10px";
    searchInput.style.border = "1px solid var(--background-modifier-border)";
    searchInput.style.borderRadius = "8px";
    searchInput.style.background = "var(--background-primary)";
    searchInput.style.color = "var(--text-normal)";

    const tableWrap = tabContent.createEl("div");
    tableWrap.style.display = "flex";
    tableWrap.style.flexDirection = "column";
    tableWrap.style.gap = "6px";

    async function renderFeatureRows(filterText = "") {
      tableWrap.innerHTML = "";

      const header = tableWrap.createEl("div");
      header.style.display = "grid";
      header.style.gridTemplateColumns = "2fr 1fr";
      header.style.gap = "12px";
      header.style.padding = "8px 10px";
      header.style.fontWeight = "700";
      header.style.borderBottom = "1px solid var(--background-modifier-border)";

      header.createEl("div", { text: "Name" });
      header.createEl("div", { text: "Source" });

      const query = String(filterText ?? "").trim().toLowerCase();

      const filtered = features.filter(featurePage => {
        const name = String(featurePage.name ?? featurePage.file?.name ?? "").toLowerCase();
        const source = String(featurePage.source ?? "").toLowerCase();
        const notes = String(featurePage.notes ?? "").toLowerCase();

        if (!query) return true;
        return name.includes(query) || source.includes(query) || notes.includes(query);
      });

      if (filtered.length === 0) {
        const empty = tableWrap.createEl("div", { text: "Keine passenden Features gefunden." });
        empty.style.padding = "10px";
        empty.style.opacity = "0.7";
        return;
      }

      for (const featurePage of filtered) {
        const featureName = String(featurePage.name ?? featurePage.file?.name ?? "-");
        const featureSource = String(featurePage.source ?? "-");
        const featureNotes = String(featurePage.notes ?? "");
        const featureEnabled = featurePage.enabled !== false;

        const details = tableWrap.createEl("details");
        details.style.border = "1px solid var(--background-modifier-border)";
        details.style.borderRadius = "8px";
        details.style.overflow = "hidden";

        const summary = details.createEl("summary");
        summary.style.display = "grid";
        summary.style.gridTemplateColumns = "2fr 1fr";
        summary.style.gap = "12px";
        summary.style.padding = "10px";
        summary.style.alignItems = "center";
        summary.style.cursor = "pointer";
        summary.style.listStyle = "none";

        const leftWrap = summary.createEl("div");
        leftWrap.createEl("div", { text: featureName });

        const stateLine = leftWrap.createEl("div", {
          text: featureEnabled ? "Enabled" : "Disabled"
        });
        stateLine.style.fontSize = "0.8em";
        stateLine.style.opacity = "0.7";
        stateLine.style.marginTop = "2px";

        summary.createEl("div", { text: featureSource });

        const content = details.createEl("div");
        content.style.padding = "10px";
        content.style.borderTop = "1px solid var(--background-modifier-border)";
        content.style.background = "var(--background-primary-alt)";

        function addInfoRow(parent, label, value) {
          const row = parent.createEl("div");
          row.style.display = "flex";
          row.style.justifyContent = "space-between";
          row.style.gap = "10px";
          row.style.padding = "2px 0";

          const left = row.createEl("div", { text: label });
          left.style.opacity = "0.7";

          const right = row.createEl("div", { text: String(value ?? "-") });
          right.style.fontWeight = "600";
          right.style.textAlign = "right";
        }

        addInfoRow(content, "Source", featureSource);
        addInfoRow(content, "Enabled", featureEnabled ? "Yes" : "No");

        const notesTitle = content.createEl("div", { text: "Notes" });
        notesTitle.style.fontWeight = "600";
        notesTitle.style.marginTop = "8px";
        notesTitle.style.marginBottom = "6px";
        notesTitle.style.opacity = "0.85";

        const notesBox = content.createEl("div");
        notesBox.style.lineHeight = "1.6";
        await renderMarkdownInto(notesBox, featureNotes, featurePage.file?.path ?? CHARACTER_PATH);
      }
    }

    await renderFeatureRows();

    searchInput.addEventListener("input", async () => {
      await renderFeatureRows(searchInput.value);
    });
  }

  async function renderInventory() {
    clearEl(tabContent);

    const title = tabContent.createEl("div", { text: "Inventory" });
    title.style.fontWeight = "700";
    title.style.fontSize = "1.05em";
    title.style.marginBottom = "10px";

    const inventory = Array.isArray(c.inventory) ? c.inventory : [];

    if (inventory.length === 0) {
      tabContent.createEl("div", { text: "Kein Inventar eingetragen." });
      return;
    }

    const searchWrap = tabContent.createEl("div");
    searchWrap.style.marginBottom = "10px";

    const searchInput = searchWrap.createEl("input");
    searchInput.type = "text";
    searchInput.placeholder = "Search Inventory";
    searchInput.style.width = "100%";
    searchInput.style.padding = "8px 10px";
    searchInput.style.border = "1px solid var(--background-modifier-border)";
    searchInput.style.borderRadius = "8px";
    searchInput.style.background = "var(--background-primary)";
    searchInput.style.color = "var(--text-normal)";

    const tableWrap = tabContent.createEl("div");
    tableWrap.style.display = "flex";
    tableWrap.style.flexDirection = "column";
    tableWrap.style.gap = "6px";

    const resolvedInventory = inventory
      .map((entry, inventoryIndex) => {
        const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
        const itemPage = itemPath ? dv.page(itemPath) : null;

        if (!itemPage) return null;

        const quantity = Math.max(1, Number(entry?.quantity ?? 1));
        let attunedCount = 0;

        for (let quantityIndex = 0; quantityIndex < quantity; quantityIndex++) {
          const instanceId = makeInventoryInstanceId(itemPath, inventoryIndex, quantityIndex);
          if (isInventoryInstanceAttuned(instanceId)) attunedCount++;
        }

        return {
          entry,
          inventoryIndex,
          itemPage,
          itemPath,
          name: String(itemPage.name ?? itemPage.file?.name ?? "Unknown Item"),
          notes: String(itemPage.notes ?? ""),
          weight: Number(itemPage.weight ?? 0),
          quantity,
          equipped: entry?.equipped === true,
          equipment: itemPage?.equipment === true,
          attuned: attunedCount > 0,
          attunedCount
        };
      })
      .filter(x => x);

    function matchesFilter(item, filterText = "") {
      const name = String(item?.name ?? "").toLowerCase();
      const notes = String(item?.notes ?? "").toLowerCase();
      const query = String(filterText ?? "").trim().toLowerCase();

      if (!query) return true;
      return name.includes(query) || notes.includes(query);
    }

    function createSectionTitle(parent, text) {
      const sectionTitle = parent.createEl("div", { text });
      sectionTitle.style.fontWeight = "700";
      sectionTitle.style.fontSize = "1em";
      sectionTitle.style.marginTop = "8px";
      sectionTitle.style.marginBottom = "4px";
      sectionTitle.style.paddingBottom = "4px";
      sectionTitle.style.borderBottom = "1px solid var(--background-modifier-border)";
      return sectionTitle;
    }

    function createTableHeader(parent) {
      const header = parent.createEl("div");
      header.style.display = "grid";
      header.style.gridTemplateColumns = "2fr 80px 90px";
      header.style.gap = "12px";
      header.style.padding = "8px 10px";
      header.style.fontWeight = "700";
      header.style.borderBottom = "1px solid var(--background-modifier-border)";

      header.createEl("div", { text: "Name" });
      header.createEl("div", { text: "Qty" });
      header.createEl("div", { text: "Weight" });
    }

    async function createInventoryRow(parent, item) {
      const rowWeight = item.weight * item.quantity;

      const details = parent.createEl("details");
      details.style.border = "1px solid var(--background-modifier-border)";
      details.style.borderRadius = "8px";
      details.style.overflow = "hidden";

      const summary = details.createEl("summary");
      summary.style.display = "grid";
      summary.style.gridTemplateColumns = "2fr 80px 90px";
      summary.style.gap = "12px";
      summary.style.padding = "10px";
      summary.style.alignItems = "center";
      summary.style.cursor = "pointer";
      summary.style.listStyle = "none";

      const leftWrap = summary.createEl("div");
      leftWrap.createEl("div", { text: item.name });

      const statusBits = [];
      if (item.equipped) statusBits.push("Equipped");
      if (item.attunedCount > 0) statusBits.push(`Attuned ${item.attunedCount}/${item.quantity}`);

      if (statusBits.length > 0) {
        const sub = leftWrap.createEl("div", { text: statusBits.join(" • ") });
        sub.style.fontSize = "0.8em";
        sub.style.opacity = "0.7";
        sub.style.marginTop = "2px";
      }

      summary.createEl("div", { text: String(item.quantity) });
      summary.createEl("div", { text: String(rowWeight) });

      const content = details.createEl("div");
      content.style.padding = "10px";
      content.style.borderTop = "1px solid var(--background-modifier-border)";
      content.style.background = "var(--background-primary-alt)";

      function addInfoRow(parent, label, value) {
        const row = parent.createEl("div");
        row.style.display = "flex";
        row.style.justifyContent = "space-between";
        row.style.gap = "10px";
        row.style.padding = "2px 0";

        const left = row.createEl("div", { text: label });
        left.style.opacity = "0.7";

        const right = row.createEl("div", { text: String(value ?? "-") });
        right.style.fontWeight = "600";
        right.style.textAlign = "right";
      }

      addInfoRow(content, "Type", item.itemPage.type ?? "-");
      addInfoRow(content, "Quantity", item.quantity);
      addInfoRow(content, "Weight per Item", item.weight);
      addInfoRow(content, "Total Weight", rowWeight);
      addInfoRow(content, "Equipped", item.equipped ? "Yes" : "No");
      addInfoRow(content, "Attuned", `${item.attunedCount} / ${item.quantity}`);

      const notesTitle = content.createEl("div", { text: "Notes" });
      notesTitle.style.fontWeight = "600";
      notesTitle.style.marginTop = "8px";
      notesTitle.style.marginBottom = "6px";
      notesTitle.style.opacity = "0.85";

      const notesBox = content.createEl("div");
      notesBox.style.lineHeight = "1.6";
      await renderMarkdownInto(notesBox, item.notes, item.itemPage.file?.path ?? CHARACTER_PATH);

      if (item.itemPage.equipment === true) {
        const equipBtn = content.createEl("button", {
          text: item.equipped ? "Unequip" : "Equip"
        });

        equipBtn.style.marginTop = "10px";
        equipBtn.style.padding = "6px 10px";
        equipBtn.style.borderRadius = "8px";
        equipBtn.style.border = "1px solid var(--background-modifier-border)";
        equipBtn.style.cursor = "pointer";
        equipBtn.style.background = "var(--background-secondary)";
        equipBtn.style.color = "var(--text-normal)";
        equipBtn.style.fontWeight = "600";

        equipBtn.addEventListener("click", async (evt) => {
          evt.preventDefault();
          evt.stopPropagation();

          await toggleInventoryEquip(item.itemPath);

          window.__dndActiveTab = "inventory";
          activeTab = "inventory";
          await renderInventory();
        });
      }
    }

    async function renderInventoryRows(filterText = "") {
      tableWrap.innerHTML = "";

      const filtered = resolvedInventory.filter(item => matchesFilter(item, filterText));

      if (filtered.length === 0) {
        const empty = tableWrap.createEl("div", { text: "Keine passenden Einträge gefunden." });
        empty.style.padding = "10px";
        empty.style.opacity = "0.7";
        return;
      }

      const equippedItems = filtered
        .filter(item => item.equipment === true && item.equipped === true)
        .sort((a, b) => a.name.localeCompare(b.name, "de"));

      const normalItems = filtered
        .filter(item => !(item.equipment === true && item.equipped === true))
        .sort((a, b) => a.name.localeCompare(b.name, "de"));

      const maxCarryWeight = getAbilityScore("str") * 15;
      let totalWeight = 0;

      if (equippedItems.length > 0) {
        createSectionTitle(tableWrap, "Equipped Equipment");
        createTableHeader(tableWrap);

        for (const item of equippedItems) {
          totalWeight += item.weight * item.quantity;
          await createInventoryRow(tableWrap, item);
        }
      }

      if (normalItems.length > 0) {
        createSectionTitle(tableWrap, equippedItems.length > 0 ? "Inventory" : "Items");
        createTableHeader(tableWrap);

        for (const item of normalItems) {
          totalWeight += item.weight * item.quantity;
          await createInventoryRow(tableWrap, item);
        }
      }

      const totalRow = tableWrap.createEl("div");
      totalRow.style.display = "grid";
      totalRow.style.gridTemplateColumns = "2fr 80px 90px";
      totalRow.style.gap = "12px";
      totalRow.style.padding = "10px";
      totalRow.style.marginTop = "6px";
      totalRow.style.borderTop = "2px solid var(--background-modifier-border)";
      totalRow.style.fontWeight = "700";
      totalRow.style.alignItems = "center";

      totalRow.createEl("div", { text: "Total Weight" });
      totalRow.createEl("div", { text: "" });
      totalRow.createEl("div", { text: `${totalWeight} / ${maxCarryWeight}` });
    }

    await renderInventoryRows();

    searchInput.addEventListener("input", async () => {
      await renderInventoryRows(searchInput.value);
    });
  }

  async function renderActions() {
    clearEl(tabContent);

    const title = tabContent.createEl("div");
    title.style.display = "grid";
    title.style.gridTemplateColumns = "2fr 1fr 120px";
    title.style.gap = "12px";
    title.style.fontWeight = "700";
    title.style.marginBottom = "8px";
    title.style.padding = "0 10px";

    title.createEl("div", { text: "Actions" });
    title.createEl("div", { text: "Hit" });
    title.createEl("div", { text: "Damage" });

    const searchWrap = tabContent.createEl("div");
    searchWrap.style.marginBottom = "10px";

    const searchInput = searchWrap.createEl("input");
    searchInput.type = "text";
    searchInput.placeholder = "Search Actions";
    searchInput.style.width = "100%";
    searchInput.style.padding = "8px 10px";
    searchInput.style.border = "1px solid var(--background-modifier-border)";
    searchInput.style.borderRadius = "8px";
    searchInput.style.background = "var(--background-primary)";
    searchInput.style.color = "var(--text-normal)";

    const actionContainer = tabContent.createEl("div");

    function sectionLabel(type) {
      if (type === "action") return "Action";
      if (type === "bonus_action") return "Bonus Action";
      if (type === "reaction") return "Reaction";
      if (type === "other") return "Other";
      return type;
    }

    function addInfoRow(parent, label, value) {
      const row = parent.createEl("div");
      row.style.display = "flex";
      row.style.justifyContent = "space-between";
      row.style.gap = "10px";
      row.style.padding = "2px 0";

      const left = row.createEl("div", { text: label });
      left.style.opacity = "0.7";

      const right = row.createEl("div", { text: String(value ?? "-") });
      right.style.fontWeight = "600";
      right.style.textAlign = "right";
    }

    function getAbilityModForAction(statName) {
      return getAbilityModByName(statName);
    }

    const actions = getAllCharacterActions().filter(action => {
      const rawType = String(action.action_type ?? "").trim().toLowerCase();
      const type = rawType.replace(/\s+/g, "_");

      return (
        type === "" ||
        type === "action" ||
        type === "bonus_action" ||
        type === "reaction"
      );
    });

    const groupedActions = {
      action: [],
      bonus_action: [],
      reaction: [],
      other: []
    };

    for (const action of actions) {
      const rawType = String(action.action_type ?? "").trim().toLowerCase();
      const type = rawType.replace(/\s+/g, "_");

      if (groupedActions[type]) {
        groupedActions[type].push(action);
      } else {
        groupedActions.other.push(action);
      }
    }

    for (const key of Object.keys(groupedActions)) {
      groupedActions[key].sort((a, b) => {
        const nameA = String(a.name ?? a.file?.name ?? "");
        const nameB = String(b.name ?? b.file?.name ?? "");
        return nameA.localeCompare(nameB, "de");
      });
    }

    async function renderActionRows(filterText = "") {
      actionContainer.innerHTML = "";
      const query = String(filterText ?? "").trim().toLowerCase();

      for (const type of ["action", "bonus_action", "reaction", "other"]) {
        const entries = groupedActions[type].filter(action => {
          const name = String(action.name ?? action.file?.name ?? "").toLowerCase();
          const notes = String(action.notes ?? action.effect ?? "").toLowerCase();
          const mode = String(action.mode ?? "").toLowerCase();
          const damageType = String(action.damage_type ?? "").toLowerCase();
          const sourceItem = String(action.source_item_name ?? "").toLowerCase();

          if (!query) return true;
          return (
            name.includes(query) ||
            notes.includes(query) ||
            mode.includes(query) ||
            damageType.includes(query) ||
            sourceItem.includes(query)
          );
        });

        if (!entries || entries.length === 0) continue;

        const section = actionContainer.createEl("div");
        section.style.marginTop = "12px";

        const sectionTitle = section.createEl("div", { text: sectionLabel(type) });
        sectionTitle.style.fontWeight = "700";
        sectionTitle.style.fontSize = "1.05em";
        sectionTitle.style.marginBottom = "8px";
        sectionTitle.style.paddingBottom = "4px";
        sectionTitle.style.borderBottom = "1px solid var(--background-modifier-border)";

        for (const action of entries) {
          const details = section.createEl("details");
          details.style.border = "1px solid var(--background-modifier-border)";
          details.style.borderRadius = "10px";
          details.style.marginTop = "8px";
          details.style.padding = "0";
          details.style.overflow = "hidden";

          const summaryEl = details.createEl("summary");
          summaryEl.style.cursor = "pointer";
          summaryEl.style.padding = "10px";
          summaryEl.style.display = "grid";
          summaryEl.style.gridTemplateColumns = "2fr 1fr 120px";
          summaryEl.style.gap = "12px";
          summaryEl.style.alignItems = "center";
          summaryEl.style.fontWeight = "700";
          summaryEl.style.listStyle = "none";

          const leftWrap = summaryEl.createEl("div");

          const leftSummary = leftWrap.createEl("div", {
            text: action.name ?? action.file?.name ?? "Unnamed Action"
          });
          leftSummary.style.fontSize = "1.05em";

          const sourceText =
            action.source_type === "item" && action.source_item_name
              ? `From ${action.source_item_name}`
              : "";

          if (sourceText) {
            const sourceEl = leftWrap.createEl("div", { text: sourceText });
            sourceEl.style.fontSize = "0.8em";
            sourceEl.style.opacity = "0.7";
            sourceEl.style.fontWeight = "400";
            sourceEl.style.marginTop = "2px";
          }

          const attackStat = action.attack_stat ?? action.damage_bonus_stat ?? "str";
          const attackMod = getAbilityModForAction(attackStat);
          const profBonus = getProficiencyBonus();
          const isProficient = action.proficient ?? true;
          const toHitBonus = attackMod + (isProficient ? profBonus : 0);

          let damageTextA = "-";
          if (action.dice_count && action.dice_size) {
            damageTextA = `${action.dice_count}d${action.dice_size}`;

            const dmgBonus =
              Number(action.damage_bonus ?? 0) +
              getAbilityModForAction(action.damage_bonus_stat);

            if (dmgBonus > 0) damageTextA += `+${dmgBonus}`;
            if (dmgBonus < 0) damageTextA += `${dmgBonus}`;
          } else if (action.damage) {
            damageTextA = String(action.damage);
          }

          const hitText =
            String(action.mode ?? "").toLowerCase() === "attack"
              ? modString(toHitBonus)
              : "-";

          const hitEl = summaryEl.createEl("div", { text: hitText });
          hitEl.style.opacity = "0.7";
          hitEl.style.fontSize = "0.9em";
          hitEl.style.fontWeight = "400";

          const damageEl = summaryEl.createEl("div", { text: damageTextA });
          damageEl.style.opacity = "0.7";
          damageEl.style.fontSize = "0.9em";
          damageEl.style.fontWeight = "400";

          const card = details.createEl("div");
          card.style.padding = "10px";
          card.style.borderTop = "1px solid var(--background-modifier-border)";

          const infoBlock = card.createEl("div");

          addInfoRow(infoBlock, "Type", sectionLabel(type));
          addInfoRow(infoBlock, "Mode", action.mode ?? "-");
          addInfoRow(infoBlock, "Shape", action.shape ?? "-");

          if (action.source_item_name) {
            addInfoRow(infoBlock, "From", action.source_item_name);
          }

          if (action.range != null) addInfoRow(infoBlock, "Range", action.range);
          if (action.radius != null) addInfoRow(infoBlock, "Radius", action.radius);
          if (action.damage_type != null) addInfoRow(infoBlock, "Damage Type", action.damage_type);

          let damageText = "-";
          if (action.damage) {
            damageText = `${action.damage}${action.damage_type ? ` ${action.damage_type}` : ""}`;
          } else if (action.dice_count && action.dice_size) {
            const dmgBonus =
              Number(action.damage_bonus ?? 0) +
              getAbilityModForAction(action.damage_bonus_stat);

            damageText = `${action.dice_count}d${action.dice_size}`;
            if (dmgBonus > 0) damageText += ` + ${dmgBonus}`;
            if (dmgBonus < 0) damageText += ` ${dmgBonus}`;
            if (action.damage_type) damageText += ` ${action.damage_type}`;
          }
          addInfoRow(infoBlock, "Damage", damageText);

          if (String(action.mode ?? "").toLowerCase() === "attack") {
            addInfoRow(infoBlock, "To Hit", modString(toHitBonus));
          }

          if (action.save_ability) addInfoRow(infoBlock, "Save", String(action.save_ability).toUpperCase());
          if (action.resource_cost != null) addInfoRow(infoBlock, "Resource Cost", action.resource_cost);
          if (action.requires_los != null) addInfoRow(infoBlock, "Requires LoS", action.requires_los ? "Ja" : "Nein");
          if (action.friendly_fire != null) addInfoRow(infoBlock, "Friendly Fire", action.friendly_fire ? "Ja" : "Nein");

          const effectBlock = card.createEl("div");
          effectBlock.style.marginTop = "8px";

          const effectTitle = effectBlock.createEl("div", { text: "Notes / Effect" });
          effectTitle.style.fontWeight = "600";
          effectTitle.style.marginBottom = "4px";
          effectTitle.style.opacity = "0.85";

          const effectBox = effectBlock.createEl("div");
          effectBox.style.lineHeight = "1.6";
          await renderMarkdownInto(
            effectBox,
            action.effect ?? action.notes ?? "",
            action.file?.path ?? action.source_item_path ?? CHARACTER_PATH
          );
        }
      }

      if (actionContainer.innerHTML.trim() === "") {
        const empty = actionContainer.createEl("div", { text: "Keine passenden Actions gefunden." });
        empty.style.padding = "10px";
        empty.style.opacity = "0.7";
      }
    }

    await renderActionRows();

    searchInput.addEventListener("input", async () => {
      await renderActionRows(searchInput.value);
    });
  }

  async function renderSpells() {
    clearEl(tabContent);

    const title = tabContent.createEl("div", { text: "Spells" });
    title.style.fontWeight = "600";
    title.style.marginBottom = "10px";

    const searchWrap = tabContent.createEl("div");
    searchWrap.style.marginBottom = "10px";

    const searchInput = searchWrap.createEl("input");
    searchInput.type = "text";
    searchInput.placeholder = "Search Spells";
    searchInput.style.width = "100%";
    searchInput.style.padding = "8px 10px";
    searchInput.style.border = "1px solid var(--background-modifier-border)";
    searchInput.style.borderRadius = "8px";
    searchInput.style.background = "var(--background-primary)";
    searchInput.style.color = "var(--text-normal)";

    const spellcastingAbility = String(c.spellcasting_ability ?? "int").toLowerCase();
    const spellMod = getAbilityModByName(spellcastingAbility);
    const prof = getProficiencyBonus();
    const spellAttackBonus = c.spell_attack_bonus != null
      ? applyItemEffects(Number(c.spell_attack_bonus), "spell_attack_bonus")
      : applyItemEffects(spellMod + prof, "spell_attack_bonus");

    const spellSaveDC = c.spell_save_dc != null
      ? applyItemEffects(Number(c.spell_save_dc), "spell_save_dc")
      : applyItemEffects(8 + spellMod + prof, "spell_save_dc");

    function addSummaryBox(parent, label, value) {
      const box = parent.createEl("div");
      box.style.padding = "8px";
      box.style.border = "1px solid var(--background-modifier-border)";
      box.style.borderRadius = "8px";
      box.style.textAlign = "center";

      const labelEl = box.createEl("div", { text: label });
      labelEl.style.fontSize = "0.8em";
      labelEl.style.opacity = "0.7";

      const valueEl = box.createEl("div", { text: String(value) });
      valueEl.style.fontSize = "1.2em";
      valueEl.style.fontWeight = "700";
      valueEl.style.marginTop = "4px";
    }

    function addInfoRow(parent, label, value) {
      const row = parent.createEl("div");
      row.style.display = "flex";
      row.style.justifyContent = "space-between";
      row.style.gap = "10px";
      row.style.padding = "2px 0";

      const left = row.createEl("div", { text: label });
      left.style.opacity = "0.7";

      const right = row.createEl("div", { text: String(value ?? "-") });
      right.style.fontWeight = "600";
      right.style.textAlign = "right";
    }

    function levelLabel(level) {
      return level === 0 ? "Cantrips" : `Level ${level}`;
    }

    function addSpellSlotCheckboxes(parent, level, slotCount) {
      if (level === 0 || slotCount <= 0) return;

      const slotWrapper = parent.createEl("div");
      slotWrapper.style.display = "flex";
      slotWrapper.style.alignItems = "center";
      slotWrapper.style.gap = "8px";
      slotWrapper.style.flexWrap = "wrap";
      slotWrapper.style.marginTop = "6px";
      slotWrapper.style.marginBottom = "8px";

      const label = slotWrapper.createEl("div", { text: "Slots:" });
      label.style.fontSize = "0.9em";
      label.style.opacity = "0.75";

      for (let i = 0; i < slotCount; i++) {
        const boxLabel = slotWrapper.createEl("label");
        boxLabel.style.display = "flex";
        boxLabel.style.alignItems = "center";
        boxLabel.style.gap = "4px";
        boxLabel.style.cursor = "pointer";

        const checkbox = boxLabel.createEl("input");
        checkbox.type = "checkbox";
      }
    }

    const summaryGrid = tabContent.createEl("div");
    summaryGrid.style.display = "grid";
    summaryGrid.style.gridTemplateColumns = "repeat(3, 1fr)";
    summaryGrid.style.gap = "8px";
    summaryGrid.style.marginBottom = "12px";

    addSummaryBox(summaryGrid, "Modifier", modString(spellMod));
    addSummaryBox(summaryGrid, "Spell Attack", modString(spellAttackBonus));
    addSummaryBox(summaryGrid, "Spell Save DC", spellSaveDC);

    const spellContainer = tabContent.createEl("div");

    const spells = getAllCharacterSpells();

    if (!spells || spells.length === 0) {
      spellContainer.createEl("div", { text: "Keine Spells gefunden." });
      return;
    }

    const groupedSpells = {};

    for (const spell of spells) {
      const lvl = Number(spell.level ?? 0);
      if (!groupedSpells[lvl]) groupedSpells[lvl] = [];
      groupedSpells[lvl].push(spell);
    }

    for (const level of Object.keys(groupedSpells)) {
      groupedSpells[level].sort((a, b) => {
        const nameA = String(a.name ?? a.file?.name ?? "");
        const nameB = String(b.name ?? b.file?.name ?? "");
        return nameA.localeCompare(nameB, "de");
      });
    }

    async function renderSpellRows(filterText = "") {
      spellContainer.innerHTML = "";
      const query = String(filterText ?? "").trim().toLowerCase();

      const sortedLevels = Object.keys(groupedSpells)
        .map(Number)
        .sort((a, b) => a - b);

      for (const level of sortedLevels) {
        const levelSpells = groupedSpells[level].filter(spell => {
          const fields = [
            String(spell.name ?? spell.file?.name ?? ""),
            String(spell.school ?? ""),
            String(spell.notes ?? spell.effect ?? ""),
            String(spell.damage_type ?? ""),
            String(spell.source_item_name ?? ""),
            String(spell.casting_time ?? ""),
            String(spell.action_type ?? ""),
            String(spell.duration ?? ""),
            String(spell.range ?? ""),
            String(spell.save_ability ?? "")
          ]
            .join(" | ")
            .toLowerCase();

          const normalizedFields = fields
            .replace(/[_-]/g, " ")
            .replace(/\s+/g, " ");

          const normalizedQuery = String(query ?? "")
            .toLowerCase()
            .replace(/[_-]/g, " ")
            .replace(/\s+/g, " ")
            .trim();

          if (!normalizedQuery) return true;
          return normalizedFields.includes(normalizedQuery);
        });

        if (!levelSpells || levelSpells.length === 0) continue;

        const section = spellContainer.createEl("div");
        section.style.marginTop = "12px";

        const sectionTitle = section.createEl("div", { text: levelLabel(level) });
        sectionTitle.style.fontWeight = "700";
        sectionTitle.style.fontSize = "1.05em";
        sectionTitle.style.marginBottom = "8px";
        sectionTitle.style.paddingBottom = "4px";
        sectionTitle.style.borderBottom = "1px solid var(--background-modifier-border)";

        const slotCount = Number(c.spell_slots?.[level] ?? 0);
        addSpellSlotCheckboxes(section, level, slotCount);

        for (const spell of levelSpells) {
          const details = section.createEl("details");
          details.style.border = "1px solid var(--background-modifier-border)";
          details.style.borderRadius = "10px";
          details.style.marginTop = "8px";
          details.style.padding = "0";
          details.style.overflow = "hidden";

          const summaryEl = details.createEl("summary");
          summaryEl.style.cursor = "pointer";
          summaryEl.style.padding = "10px";
          summaryEl.style.display = "flex";
          summaryEl.style.justifyContent = "space-between";
          summaryEl.style.alignItems = "center";
          summaryEl.style.gap = "10px";
          summaryEl.style.fontWeight = "700";

          const leftWrap = summaryEl.createEl("div");

          const leftSummary = leftWrap.createEl("div", {
            text: spell.name ?? spell.file?.name ?? "Unnamed Spell"
          });
          leftSummary.style.fontSize = "1.05em";

          const sourceText =
            spell.source_type === "item" && spell.source_item_name
              ? `From ${spell.source_item_name}`
              : "";

          if (sourceText) {
            const sourceEl = leftWrap.createEl("div", { text: sourceText });
            sourceEl.style.fontSize = "0.8em";
            sourceEl.style.opacity = "0.7";
            sourceEl.style.fontWeight = "400";
            sourceEl.style.marginTop = "2px";
          }

          const rightSummary = summaryEl.createEl("div", {
            text: `${spell.school ?? "-"}`
          });
          rightSummary.style.opacity = "0.7";
          rightSummary.style.fontSize = "0.9em";
          rightSummary.style.fontWeight = "400";

          const card = details.createEl("div");
          card.style.padding = "10px";
          card.style.borderTop = "1px solid var(--background-modifier-border)";

          const infoBlock = card.createEl("div");

          if (spell.source_item_name) {
            addInfoRow(infoBlock, "From", spell.source_item_name);
          }

          addInfoRow(infoBlock, "Casting Time", spell.casting_time ?? spell.action_type ?? "-");
          addInfoRow(infoBlock, "Range", spell.range ?? "-");
          addInfoRow(infoBlock, "Duration", spell.duration ?? "-");
          addInfoRow(infoBlock, "Concentration", spell.concentration ? "Ja" : "Nein");

          if (spell.uses_attack_roll) {
            addInfoRow(infoBlock, "To Hit", modString(spellAttackBonus));
          } else {
            addInfoRow(infoBlock, "To Hit", "-");
          }

          if (spell.uses_save) {
            addInfoRow(
              infoBlock,
              "Save",
              `${String(spell.save_ability ?? "?").toUpperCase()} DC ${spellSaveDC}`
            );
          } else {
            addInfoRow(infoBlock, "Save", "-");
          }

          let damageText = "-";
          if (spell.damage) {
            damageText = `${spell.damage}${spell.damage_type ? ` ${spell.damage_type}` : ""}`;
          } else if (spell.dice_count && spell.dice_size) {
            damageText = `${spell.dice_count}d${spell.dice_size}${spell.damage_type ? ` ${spell.damage_type}` : ""}`;
          }
          addInfoRow(infoBlock, "Damage", damageText);

          const effectBlock = card.createEl("div");
          effectBlock.style.marginTop = "8px";

          const effectTitle = effectBlock.createEl("div", { text: "Effect" });
          effectTitle.style.fontWeight = "600";
          effectTitle.style.marginBottom = "4px";
          effectTitle.style.opacity = "0.85";

          const effectBox = effectBlock.createEl("div");
          effectBox.style.lineHeight = "1.6";
          await renderMarkdownInto(
            effectBox,
            spell.effect ?? spell.notes ?? "",
            spell.file?.path ?? spell.source_item_path ?? CHARACTER_PATH
          );
        }
      }

      if (spellContainer.innerHTML.trim() === "") {
        const empty = spellContainer.createEl("div", { text: "Keine passenden Spells gefunden." });
        empty.style.padding = "10px";
        empty.style.opacity = "0.7";
      }
    }

    await renderSpellRows();

    searchInput.addEventListener("input", async () => {
      await renderSpellRows(searchInput.value);
    });
  }

  async function renderAttunement() {
    clearEl(tabContent);

    const title = tabContent.createEl("div", { text: "Attunement" });
    title.style.fontWeight = "700";
    title.style.fontSize = "1.05em";
    title.style.marginBottom = "10px";

    const characterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);

    const attunementSlots = Number(c.attunement_slots ?? 3);
    let attunedItems = Array.isArray(c.attuned_items)
      ? c.attuned_items.map(x => String(x))
      : [];

    const inventoryEntries = Array.isArray(c.inventory) ? c.inventory : [];

    const attunableItems = [];

    for (let inventoryIndex = 0; inventoryIndex < inventoryEntries.length; inventoryIndex++) {
      const entry = inventoryEntries[inventoryIndex];
      const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
      const itemPage = itemPath ? dv.page(itemPath) : null;
      if (!itemPage) continue;

      const requiresAttunement = itemPage.attunement === true;
      if (!requiresAttunement) continue;

      const quantity = Math.max(1, Number(entry?.quantity ?? 1));

      for (let quantityIndex = 0; quantityIndex < quantity; quantityIndex++) {
        const instanceId = makeInventoryInstanceId(itemPath, inventoryIndex, quantityIndex);

        attunableItems.push({
          instanceId,
          inventoryIndex,
          quantityIndex,
          itemPath,
          itemPage,
          name: String(itemPage.name ?? itemPage.file?.name ?? "Unknown Item"),
          displayName:
            quantity > 1
              ? `${String(itemPage.name ?? itemPage.file?.name ?? "Unknown Item")} #${quantityIndex + 1}`
              : String(itemPage.name ?? itemPage.file?.name ?? "Unknown Item"),
          type: String(itemPage.type ?? "-"),
          notes: String(itemPage.notes ?? ""),
          quantity: 1,
          equipped: entry?.equipped === true,
          isAttuned: attunedItems.includes(instanceId)
        });
      }
    }

    const searchWrap = tabContent.createEl("div");
    searchWrap.style.marginBottom = "10px";

    const searchInput = searchWrap.createEl("input");
    searchInput.type = "text";
    searchInput.placeholder = "Search Attunement Items";
    searchInput.style.width = "100%";
    searchInput.style.padding = "8px 10px";
    searchInput.style.border = "1px solid var(--background-modifier-border)";
    searchInput.style.borderRadius = "8px";
    searchInput.style.background = "var(--background-primary)";
    searchInput.style.color = "var(--text-normal)";

    const slotsWrap = tabContent.createEl("div");
    slotsWrap.style.display = "flex";
    slotsWrap.style.flexDirection = "column";
    slotsWrap.style.gap = "8px";
    slotsWrap.style.marginBottom = "8px";

    const errorBox = tabContent.createEl("div");
    errorBox.style.display = "none";
    errorBox.style.marginBottom = "10px";
    errorBox.style.padding = "10px";
    errorBox.style.border = "1px solid var(--background-modifier-border)";
    errorBox.style.borderRadius = "8px";
    errorBox.style.background = "var(--background-secondary)";
    errorBox.style.color = "var(--text-normal)";

    const listWrap = tabContent.createEl("div");
    listWrap.style.display = "flex";
    listWrap.style.flexDirection = "column";
    listWrap.style.gap = "8px";

    function showError(message) {
      errorBox.setText(message);
      errorBox.style.display = "block";
    }

    function clearError() {
      errorBox.style.display = "none";
      errorBox.setText("");
    }

    async function saveAttunedItems() {
      if (!characterFile) return;

      await app.fileManager.processFrontMatter(characterFile, (fm) => {
        fm.attuned_items = attunedItems;
      });
    }

    function renderSlots() {
      slotsWrap.innerHTML = "";

      const slotsTitle = slotsWrap.createEl("div", {
        text: `Attuned Items: ${attunedItems.length} / ${attunementSlots}`
      });
      slotsTitle.style.fontWeight = "700";
      slotsTitle.style.fontSize = "1em";

      const slotGrid = slotsWrap.createEl("div");
      slotGrid.style.display = "grid";
      slotGrid.style.gridTemplateColumns = "repeat(auto-fit, minmax(180px, 1fr))";
      slotGrid.style.gap = "8px";

      for (let i = 0; i < attunementSlots; i++) {
        const card = slotGrid.createEl("div");
        card.style.padding = "10px";
        card.style.border = "1px solid var(--background-modifier-border)";
        card.style.borderRadius = "10px";
        card.style.background = "var(--background-primary-alt)";

        const slotLabel = card.createEl("div", { text: `Slot ${i + 1}` });
        slotLabel.style.fontSize = "0.8em";
        slotLabel.style.opacity = "0.7";
        slotLabel.style.marginBottom = "4px";

        const instanceId = attunedItems[i];
        if (instanceId) {
          const found = attunableItems.find(x => x.instanceId === instanceId);
          card.createEl("div", {
            text: found?.displayName ?? found?.name ?? instanceId
          });
        } else {
          const empty = card.createEl("div", { text: "Empty" });
          empty.style.opacity = "0.6";
        }
      }
    }

    function addInfoRow(parent, label, value) {
      const row = parent.createEl("div");
      row.style.display = "flex";
      row.style.justifyContent = "space-between";
      row.style.gap = "10px";
      row.style.padding = "2px 0";

      const left = row.createEl("div", { text: label });
      left.style.opacity = "0.7";

      const right = row.createEl("div", { text: String(value ?? "-") });
      right.style.fontWeight = "600";
      right.style.textAlign = "right";
    }

    async function createItemCard(parent, item) {
      const details = parent.createEl("details");
      details.style.border = "1px solid var(--background-modifier-border)";
      details.style.borderRadius = "10px";
      details.style.overflow = "hidden";

      const summary = details.createEl("summary");
      summary.style.display = "grid";
      summary.style.gridTemplateColumns = "2fr 100px 120px";
      summary.style.gap = "12px";
      summary.style.padding = "10px";
      summary.style.alignItems = "center";
      summary.style.cursor = "pointer";
      summary.style.listStyle = "none";

      const leftWrap = summary.createEl("div");

      const nameEl = leftWrap.createEl("div", { text: item.displayName ?? item.name });
      nameEl.style.fontWeight = "600";

      const typeEl = leftWrap.createEl("div", { text: item.type });
      typeEl.style.fontSize = "0.8em";
      typeEl.style.opacity = "0.7";
      typeEl.style.marginTop = "2px";

      const statusEl = summary.createEl("div", {
        text: item.isAttuned ? "Attuned" : "Not Attuned"
      });
      statusEl.style.fontSize = "0.9em";
      statusEl.style.opacity = "0.8";

      const buttonWrap = summary.createEl("div");

      const toggleBtn = buttonWrap.createEl("button", {
        text: item.isAttuned ? "Unattune" : "Attune"
      });
      toggleBtn.style.padding = "6px 10px";
      toggleBtn.style.borderRadius = "8px";
      toggleBtn.style.border = "1px solid var(--background-modifier-border)";
      toggleBtn.style.cursor = "pointer";
      toggleBtn.style.background = "var(--background-secondary)";
      toggleBtn.style.color = "var(--text-normal)";
      toggleBtn.style.fontWeight = "600";

      toggleBtn.addEventListener("click", async (evt) => {
        evt.preventDefault();
        evt.stopPropagation();
        clearError();

        const alreadyAttuned = attunedItems.includes(item.instanceId);

        if (alreadyAttuned) {
          attunedItems = attunedItems.filter(id => id !== item.instanceId);
        } else {
          if (attunedItems.length >= attunementSlots) {
            showError("No free attunement slots available.");
            return;
          }
          attunedItems.push(item.instanceId);
        }

        window.__dndActiveTab = "attunement";
        activeTab = "attunement";
        await saveAttunedItems();
        await renderAttunement();
      });

      const content = details.createEl("div");
      content.style.padding = "10px";
      content.style.borderTop = "1px solid var(--background-modifier-border)";
      content.style.background = "var(--background-primary-alt)";

      addInfoRow(content, "Type", item.type);
      addInfoRow(content, "Inventory Slot", item.inventoryIndex + 1);
      addInfoRow(content, "Copy", item.quantityIndex + 1);
      addInfoRow(content, "Equipped", item.equipped ? "Yes" : "No");
      addInfoRow(content, "Attunement", "Required");

      const notesTitle = content.createEl("div", { text: "Notes" });
      notesTitle.style.fontWeight = "600";
      notesTitle.style.marginTop = "8px";
      notesTitle.style.marginBottom = "4px";
      notesTitle.style.opacity = "0.85";

      const notesBox = content.createEl("div");
      notesBox.style.lineHeight = "1.6";
      await renderMarkdownInto(notesBox, item.notes, item.itemPage.file?.path ?? CHARACTER_PATH);
    }

    async function renderItemList(filterText = "") {
      listWrap.innerHTML = "";

      const query = String(filterText ?? "").trim().toLowerCase();

      const filtered = attunableItems
        .map(item => ({
          ...item,
          isAttuned: attunedItems.includes(item.instanceId)
        }))
        .filter(item => {
          const haystack = [
            item.name,
            item.displayName,
            item.type,
            item.notes
          ].join(" ").toLowerCase();

          return !query || haystack.includes(query);
        })
        .sort((a, b) => {
          if (a.isAttuned !== b.isAttuned) return a.isAttuned ? -1 : 1;
          return a.displayName.localeCompare(b.displayName, "de");
        });

      if (filtered.length === 0) {
        const empty = listWrap.createEl("div", {
          text: "No attunement-capable items found."
        });
        empty.style.padding = "10px";
        empty.style.opacity = "0.7";
        return;
      }

      const header = listWrap.createEl("div");
      header.style.display = "grid";
      header.style.gridTemplateColumns = "2fr 100px 120px";
      header.style.gap = "12px";
      header.style.padding = "8px 10px";
      header.style.fontWeight = "700";
      header.style.borderBottom = "1px solid var(--background-modifier-border)";

      header.createEl("div", { text: "Item" });
      header.createEl("div", { text: "Status" });
      header.createEl("div", { text: "Action" });

      for (const item of filtered) {
        await createItemCard(listWrap, item);
      }
    }

    renderSlots();
    await renderItemList();

    searchInput.addEventListener("input", async () => {
      await renderItemList(searchInput.value);
    });
  }

  async function renderNotes() {
    clearEl(tabContent);

    const title = tabContent.createEl("div", { text: "Notes" });
    title.style.fontWeight = "700";
    title.style.fontSize = "1.05em";
    title.style.marginBottom = "10px";

    const file = app.vault.getAbstractFileByPath(CHARACTER_PATH);

    if (!file || !file.path) {
      tabContent.createEl("div", {
        text: `Character-Datei nicht gefunden: ${CHARACTER_PATH}`,
      });
      return;
    }

    const content = await app.vault.cachedRead(file);

    function extractSection(markdown, headingName) {
      const lines = markdown.split(/\r?\n/);
      const target = headingName.trim().toLowerCase();

      let start = -1;
      let baseLevel = -1;

      for (let i = 0; i < lines.length; i++) {
        const match = lines[i].match(/^(#{1,6})\s+(.*?)\s*$/);
        if (!match) continue;

        const level = match[1].length;
        const text = match[2].trim().toLowerCase();

        if (text === target) {
          start = i + 1;
          baseLevel = level;
          break;
        }
      }

      if (start === -1) return null;

      let end = lines.length;
      for (let i = start; i < lines.length; i++) {
        const match = lines[i].match(/^(#{1,6})\s+(.*?)\s*$/);
        if (!match) continue;

        const level = match[1].length;
        if (level <= baseLevel) {
          end = i;
          break;
        }
      }

      return lines.slice(start, end).join("\n").trim();
    }

    const notesSection = extractSection(content, "Notes");

    if (!notesSection) {
      tabContent.createEl("div", { text: "Kein Abschnitt '## Notes' gefunden." });
      return;
    }

    const notesBox = tabContent.createEl("div");
    notesBox.style.padding = "10px";
    notesBox.style.border = "1px solid var(--background-modifier-border)";
    notesBox.style.borderRadius = "10px";
    notesBox.style.lineHeight = "1.6";

    await renderMarkdownInto(notesBox, notesSection, file.path);
  }

  function updateTabStyles() {
    for (const key in tabButtons) {
      const btn = tabButtons[key];
      const isActive = key === activeTab;

      btn.style.padding = "6px 10px";
      btn.style.borderRadius = "8px";
      btn.style.border = "1px solid var(--background-modifier-border)";
      btn.style.cursor = "pointer";
      btn.style.fontSize = "0.9em";
      btn.style.background = isActive
        ? "var(--interactive-accent)"
        : "var(--background-secondary)";
      btn.style.color = isActive
        ? "var(--text-on-accent)"
        : "var(--text-normal)";
    }
  }

  async function renderActiveTab() {
    window.__dndActiveTab = activeTab;
    updateTabStyles();

    try {
      if (activeTab === "actions") await renderActions();
      if (activeTab === "attunement") await renderAttunement();
      if (activeTab === "features") await renderFeatures();
      if (activeTab === "inventory") await renderInventory();
      if (activeTab === "spells") await renderSpells();
      if (activeTab === "notes") await renderNotes();
    } catch (err) {
      clearEl(tabContent);
      tabContent.createEl("div", {
        text: `Fehler beim Laden des Tabs: ${err.message ?? err}`
      });
      console.error(err);
    }

    updateTabStyles();
  }

  function makeTabButton(key, label) {
    const btn = tabBar.createEl("button", { text: label });
    btn.addEventListener("click", () => {
      activeTab = key;
      window.__dndActiveTab = key;
      renderActiveTab();
    });
    tabButtons[key] = btn;
  }

  makeTabButton("actions", "Actions");
  makeTabButton("attunement", "Attunement");
  makeTabButton("spells", "Spells");
  makeTabButton("inventory", "Inventory");
  makeTabButton("features", "Features");
  makeTabButton("notes", "Notes");

  renderActiveTab();

  const sensesCard = leftCol.createEl("div");
  sensesCard.style.padding = "16px";
  sensesCard.style.border = "1px solid var(--background-modifier-border)";
  sensesCard.style.borderRadius = "14px";

  const sensesTitle = sensesCard.createEl("div", { text: "Senses" });
  sensesTitle.style.fontWeight = "700";
  sensesTitle.style.marginBottom = "12px";
  sensesTitle.style.fontSize = "1.1em";

  const passivePerception = 10 + getSkillTotal("Perception", "wis", c.perception_prof ?? false);
  const passiveInvestigation = 10 + getSkillTotal("Investigation", "int", c.investigation_prof ?? false);
  const passiveInsight = 10 + getSkillTotal("Insight", "wis", c.insight_prof ?? false);

  const senseStats = [
    ["Passive Perception", passivePerception],
    ["Passive Investigation", passiveInvestigation],
    ["Passive Insight", passiveInsight],
  ];

  for (const [label, value] of senseStats) {
    const row = sensesCard.createEl("div");
    row.style.display = "flex";
    row.style.justifyContent = "space-between";
    row.style.padding = "6px 0";
    row.style.borderBottom = "1px solid var(--background-modifier-border-hover)";

    row.createEl("div", { text: label });

    const valueEl = row.createEl("div", { text: String(value) });
    valueEl.style.fontWeight = "600";
  }
}
```