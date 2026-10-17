```dataviewjs
const CHARACTER_FOLDER = "Public/3 Backend/Characters/";

const CLASSES_FOLDER = "Public/3 Backend/Classes/";

const SUBCLASSES_FOLDER = "Public/3 Backend/Subclasses/";

const ITEMS_FOLDER = "Public/3 Backend/Items/";

const FEATS_FOLDER = "Public/3 Backend/Feats/";

const SPECIES_FOLDER = "Public/3 Backend/Species/";

const BACKGROUNDS_FOLDER = "Public/3 Backend/Backgrounds/";

const SPELLS_FOLDER = "Public/3 Backend/Spells/";

  

const ABILITIES = [

  ["str", "STR"], ["dex", "DEX"], ["con", "CON"],

  ["int", "INT"], ["wis", "WIS"], ["cha", "CHA"]

];

  

function clampInt(v, min, max, fallback) {

  const n = Math.floor(Number(v));

  return Number.isFinite(n) ? Math.max(min, Math.min(max, n)) : fallback;

}

function modFromScore(v) { return Math.floor((Number(v) - 10) / 2); }

function modString(v) { return Number(v) >= 0 ? `+${v}` : `${v}`; }

function toPlainArray(value) {

  if (Array.isArray(value)) return [...value];

  if (value == null) return [];

  try {

    if (typeof value?.array === "function") return [...value.array()];

    if (typeof value?.values === "function") return [...value.values()];

    if (typeof value?.[Symbol.iterator] === "function" && typeof value !== "string") return [...value];

  } catch {}

  return [];

}

function directMarkdownFiles(folder) {

  const prefix = folder.endsWith("/") ? folder : folder + "/";

  return app.vault.getMarkdownFiles()

    .filter(f => f.path.startsWith(prefix) && !f.path.slice(prefix.length).includes("/"))

    .sort((a,b) => a.basename.localeCompare(b.basename, "de"));

}

function pageFor(file) { return file ? dv.page(file.path) : null; }

function styleCard(el) {

  el.style.padding = "16px";

  el.style.border = "1px solid var(--background-modifier-border)";

  el.style.borderRadius = "14px";

  el.style.background = "var(--background-primary)";

}

function styleInput(el) {

  el.style.cssText += `

    width: 100%;

    min-width: 0;

    box-sizing: border-box;

  `;

}

function makeButton(parent, text) {

  const b = parent.createEl("button", {text});

  b.style.padding = "8px 12px";

  b.style.borderRadius = "8px";

  b.style.border = "1px solid var(--background-modifier-border)";

  b.style.cursor = "pointer";

  return b;

}

function resolveItem(ref) {

  const raw = String(ref ?? "").trim();

  if (!raw) return null;

  const candidates = raw.includes("/")

    ? [raw, raw.endsWith(".md") ? raw : raw + ".md"]

    : [`${ITEMS_FOLDER}${raw}`, `${ITEMS_FOLDER}${raw}.md`];

  for (const path of candidates) {

    const file = app.vault.getAbstractFileByPath(path);

    if (file) return {file, page: dv.page(path), path};

  }

  return null;

}

  

const characterFiles = directMarkdownFiles(CHARACTER_FOLDER);

const classFiles = directMarkdownFiles(CLASSES_FOLDER);

const subclassFiles = directMarkdownFiles(SUBCLASSES_FOLDER);

const itemFiles = directMarkdownFiles(ITEMS_FOLDER);

const featFiles = directMarkdownFiles(FEATS_FOLDER);

const speciesFiles = directMarkdownFiles(SPECIES_FOLDER);

const backgroundFiles = directMarkdownFiles(BACKGROUNDS_FOLDER);

const spellFiles = directMarkdownFiles(SPELLS_FOLDER);

  

const root = dv.el("div", "");

root.style.maxWidth = "1100px";

root.style.margin = "0 auto";

root.style.display = "flex";

root.style.flexDirection = "column";

root.style.gap = "14px";

  

const header = root.createEl("div");

styleCard(header);

header.createEl("div", {text:"Character Builder"}).style.cssText =

  "font-size:1.8em;font-weight:700";

header.createEl("div", {

  text:"Character creation and persistent character configuration"

}).style.cssText = "opacity:.7;margin-top:4px";

  

if (!characterFiles.length) {

  root.createEl("div", {text:`No characters found directly in ${CHARACTER_FOLDER}`});

  return;

}

  

const top = root.createEl("div");

styleCard(top);

top.style.display = "grid";

top.style.gridTemplateColumns = "1fr auto";

top.style.gap = "12px";

top.style.alignItems = "end";

  

const selectorWrap = top.createEl("div");

selectorWrap.createEl("div", {text:"Character"}).style.cssText =

  "font-weight:600;font-size:.85em;margin-bottom:6px";

const characterSelect = selectorWrap.createEl("select");

characterSelect.style.cssText += `

  width: 100%;

  min-width: 350px;

  max-width: 100%;

`;

styleInput(characterSelect);

  

for (const file of characterFiles) {

  const pg = pageFor(file);

  const opt = characterSelect.createEl("option", {

    text:`${pg?.name ?? file.basename} — ${file.path}`

  });

  opt.value = file.path;

}

  

if (

  window.__dndBuilderCharacterPath &&

  characterFiles.some(f => f.path === window.__dndBuilderCharacterPath)

) characterSelect.value = window.__dndBuilderCharacterPath;

  

const saveBtn = makeButton(top, "Save Changes");

saveBtn.style.background = "var(--interactive-accent)";

saveBtn.style.color = "var(--text-on-accent)";

saveBtn.style.fontWeight = "700";

  

const tabCard = root.createEl("div");

styleCard(tabCard);

tabCard.style.padding = "10px";

  

const tabBar = tabCard.createEl("div");

tabBar.style.display = "flex";

tabBar.style.flexWrap = "wrap";

tabBar.style.gap = "8px";

  

const content = root.createEl("div");

styleCard(content);

content.style.minHeight = "420px";

  

const status = root.createEl("div");

status.style.fontSize = ".85em";

status.style.opacity = ".7";

  

let currentFile = null;

let currentPage = null;

let draft = null;

let dirty = false;

let activeTab = window.__dndBuilderTab ?? "class";

const tabButtons = {};

  

function markDirty() {

  dirty = true;

  saveBtn.setText("Save Changes *");

  status.setText("Unsaved changes");

}

function clearDirty() {

  dirty = false;

  saveBtn.setText("Save Changes");

  status.setText("Saved state loaded");

}

function selectedClassFile(name) {

  return classFiles.find(f => {

    const p = pageFor(f);

    return String(p?.name ?? f.basename) === String(name ?? "");

  }) ?? null;

}

function selectedClassPage() {

  return pageFor(selectedClassFile(draft?.class));

}

function classLevelData() {

  const cp = selectedClassPage();

  const level = String(draft?.level ?? 1);

  return cp?.levels?.[level] ?? cp?.levels?.[Number(level)] ?? null;

}

function getSubclassConfig() {

  const cfg = selectedClassPage()?.subclass;

  return cfg && typeof cfg === "object" ? cfg : null;

}

function availableSubclassOptions() {

  const className = String(draft?.class ?? "").trim().toLowerCase();

  if (!className) return [];

  

  const found = [];

  const seen = new Set();

  

  // Preferred method: discover every subclass file whose frontmatter says

  // `class: <selected class>`.

  for (const file of subclassFiles) {

    const page = pageFor(file);

    if (!page) continue;

  

    const ownerRaw = page.class ?? page.parent_class ?? page.base_class ?? "";

    const owners = Array.isArray(ownerRaw) ? ownerRaw : [ownerRaw];

    const belongs = owners.some(x => String(x ?? "").trim().toLowerCase() === className);

    if (!belongs) continue;

  

    const name = String(page.name ?? file.basename).trim();

    if (!name) continue;

    const key = file.path.toLowerCase();

    if (seen.has(key)) continue;

    seen.add(key);

    found.push({name, file:file.path});

  }

  

  // Backward compatibility: also accept explicitly declared class options.

  const cfg = getSubclassConfig();

  const declared = Array.isArray(cfg?.options) ? cfg.options : [];

  for (const option of declared) {

    if (!option) continue;

    const name = String(option.name ?? option.file ?? "").trim();

    if (!name) continue;

    const path = String(option.file ?? "").trim();

    const key = (path || name).toLowerCase();

    if (seen.has(key)) continue;

    seen.add(key);

    found.push({name, file:path || null});

  }

  

  return found.sort((a,b)=>a.name.localeCompare(b.name,"en"));

}

function getSubclassName() {

  return Array.isArray(draft?.subclass)

    ? String(draft.subclass.find(Boolean) ?? "")

    : String(draft?.subclass ?? "");

}

  

/* Matching calculation rules for the Character Sheet.
   Source types: inventory (including attuned), species, species choices,
   class/subclass/species/manual features, ASI and feat choices. */
function builderProfBase() {
  const level = Math.max(1, Math.min(20, Number(draft?.level ?? 1)));
  return 2 + Math.floor((level - 1) / 4);
}
function baseContext() {
  const ctx = {};
  for (const [k] of ABILITIES) {
    ctx[k] = Number(draft?.[k] ?? 10);
    ctx[`${k}_mod`] = modFromScore(ctx[k]);
  }
  ctx.prof = builderProfBase();
  ctx.character_level = Number(draft?.level ?? 1);
  return ctx;
}
function evaluateSimpleFormula(formula, ctx = baseContext()) {
  let expr = String(formula ?? "").trim();
  if (!expr) return null;
  const keys = ["character_level","str_mod","dex_mod","con_mod","int_mod","wis_mod","cha_mod",
                "str","dex","con","int","wis","cha","prof"];
  for (const key of keys) expr = expr.replace(new RegExp(`\\b${key}\\b`, "g"), String(ctx[key] ?? 0));
  expr = expr.replace(/\bmin\s*\(/g,"Math.min(").replace(/\bmax\s*\(/g,"Math.max(");
  const stripped = expr.replace(/Math\.(min|max)/g, "");
  if (!/^[0-9+\-*/().,\s]*$/.test(stripped)) return null;
  try {
    const x = Function(`"use strict";return (${expr})`)();
    return Number.isFinite(x) ? Number(x) : null;
  } catch { return null; }
}
function builderResolvePage(ref, folder) {
  const raw = String(ref ?? "").trim();
  if (!raw) return null;
  const candidates = raw.includes("/")
    ? [raw, raw.endsWith(".md") ? raw : raw + ".md"]
    : [`${folder}${raw}`, `${folder}${raw}.md`];
  for (const path of candidates) {
    const page = dv.page(path);
    if (page) return page;
  }
  return null;
}
function builderAsiEntries() {
  const choices = draft?.asi_choices && typeof draft.asi_choices === "object" ? draft.asi_choices : {};
  return Object.entries(choices).filter(([level, choice]) =>
    choice && (!Number.isFinite(Number(level)) || Number(level) <= Number(draft?.level ?? 1)));
}
function builderFeaturePages() {
  const refs = [...(Array.isArray(draft?.features) ? draft.features : [])];
  const level = Number(draft?.level ?? 1);
  const cp = selectedClassPage();
  for (const entry of Array.isArray(cp?.features) ? cp.features : [])
    if (entry && Number(entry.level ?? 1) <= level && entry.feature) refs.push(entry.feature);
  const config = cp?.subclass;
  const subclassName = getSubclassName().trim();
  const options = Array.isArray(config?.options) ? config.options : [];
  const option = options.find(o => String(o?.name ?? "").trim().toLowerCase() === subclassName.toLowerCase());
  const sp = subclassName && (option?.file || options.length === 0)
    ? builderResolvePage(option?.file ?? subclassName, SUBCLASSES_FOLDER) : null;
  const unlockLevel = Number(config?.unlock_level ?? 1);
  if (level >= unlockLevel)
    for (const entry of Array.isArray(sp?.features) ? sp.features : [])
      if (entry && Number(entry.level ?? unlockLevel) <= level && entry.feature) refs.push(entry.feature);
  const species = builderResolvePage(draft?.species, SPECIES_FOLDER);
  for (const entry of Array.isArray(species?.features) ? species.features : []) {
    const ref = typeof entry === "string" ? entry : entry?.feature ?? entry?.file ?? entry?.path;
    if (ref) refs.push(ref);
  }
  const pages = [], seen = new Set();
  const pushPage = (page) => {
    const path = String(page?.file?.path ?? "");
    if (page && path && !seen.has(path)) { seen.add(path); pages.push(page); }
  };
  refs.forEach(ref => pushPage(builderResolvePage(ref, "Public/3 Backend/Features/")));
  for (const [, choice] of builderAsiEntries())
    if (String(choice?.type ?? "").toLowerCase() === "feat") pushPage(selectedFeatPage(choice.feat));
  return pages;
}

function builderIsWearingArmor() {
  for (const entry of (Array.isArray(draft?.inventory) ? draft.inventory : [])) {
    if (entry?.equipped !== true) continue;
    const item = resolveItem(entry.item)?.page;
    if (!item) continue;
    const tokens = [item.type, item.category, item.armor_type, item.armor_category,
                    item.item_type, item.equipment_category, item.subtype]
      .map(x => String(x ?? "").toLowerCase().replace(/[_-]/g, " "));
    const isShield = tokens.some(x => /\bshield\b/.test(x)) ||
      /\bshield\b/i.test(String(item.name ?? item.file?.name ?? ""));
    if (isShield) continue;
    if (tokens.some(x => /\b(light armor|medium armor|heavy armor|armor)\b/.test(x))) return true;
    if (item.armor_class != null || item.armor_base != null) return true;
  }
  return false;
}
function builderBonusCondition(b, defaultWhen, sourceActive = true, equipped = false, attuned = false) {
  const when = String(b?.active_when ?? defaultWhen).trim().toLowerCase();
  if (when === "always") return sourceActive;
  if (when === "enabled" || when === "selected") return sourceActive;
  if (when === "equipped") return equipped;
  if (when === "attuned") return attuned;
  if (when === "unarmored") return sourceActive && !builderIsWearingArmor();
  if (when === "armored") return sourceActive && builderIsWearingArmor();
  return false;
}

function builderActiveEffects() {
  const effects = [];
  const append = (bonuses) => {
    for (const b of Array.isArray(bonuses) ? bonuses : []) {
      const type = String(b?.type ?? "").trim().toLowerCase();
      if (type) effects.push({type, value:Number(b.value ?? 0), set_value:b.set_value,
        formula:b.formula, calculation:String(b.calculation ?? "").trim().toLowerCase()});
    }
  };
  const inventory = Array.isArray(draft?.inventory) ? draft.inventory : [];
  const attuned = Array.isArray(draft?.attuned_items) ? draft.attuned_items.map(String) : [];
  inventory.forEach((entry, i) => {
    const resolved = resolveItem(entry?.item);
    const item = resolved?.page;
    if (!item) return;
    const count = Math.max(1, Number(entry.quantity ?? 1));
    for (let j = 0; j < count; j++) {
      const instanceId = `${resolved.path}::${i}::${j}`;
      for (const b of Array.isArray(item.bonuses) ? item.bonuses : []) {
        const when = String(b?.active_when ?? "equipped").trim().toLowerCase();
        const active = builderBonusCondition(b, "equipped", true,
          entry.equipped === true, attuned.includes(instanceId));
        if (active) append([b]);
      }
    }
  });
  const species = builderResolvePage(draft?.species, SPECIES_FOLDER);
  append(species?.bonuses);
  append(species?.scaling_bonuses);
  const defs = species?.choices ?? {};
  const state = draft?.species_choices ?? {};
  if (defs.ability_increase && state.ability_increase)
    append([{type:state.ability_increase,value:Number(defs.ability_increase.amount ?? 1)}]);
  if (defs.ability_increases) {
    const selected = Array.isArray(state.ability_increases) ? state.ability_increases : [];
    const count = Math.max(1, Number(defs.ability_increases.count ?? selected.length ?? 1));
    selected.slice(0,count).forEach(type => append([{type,value:Number(defs.ability_increases.amount ?? 1)}]));
  }
  for (const feature of builderFeaturePages()) {
    if (feature.enabled === false) continue;
    for (const b of Array.isArray(feature.bonuses) ? feature.bonuses : []) {
      const when = String(b?.active_when ?? "enabled").trim().toLowerCase();
      if (builderBonusCondition(b, "enabled", feature.enabled !== false)) append([b]);
    }
    append(feature.scaling_bonuses);
  }
  for (const [, choice] of builderAsiEntries()) {
    if (String(choice?.type ?? "").trim().toLowerCase() !== "feat") continue;
    const feat = selectedFeatPage(choice.feat);
    const choiceDefs = feat?.choices ?? {};
    const choiceState = choice.feat_choices ?? {};
    for (const field of ["ability_increase","resilient_ability"])
      if (choiceDefs[field] && choiceState[field])
        append([{type:choiceState[field],value:Number(choiceDefs[field].amount ?? 1)}]);
  }
  return effects;
}
function builderApplyEffects(baseValue, type, effects = builderActiveEffects()) {
  const key = String(type ?? "").trim().toLowerCase();
  const matches = effects.filter(b => b.type === key);
  const setters = [];
  let additive = 0;
  for (const b of matches) {
    const n = Number(b.value ?? 0);
    if (Number.isFinite(n)) additive += n;
    if (b.set_value !== undefined && b.set_value !== null && b.set_value !== ""
        && Number.isFinite(Number(b.set_value))) setters.push(Number(b.set_value));
    if (b.formula && !(key === "ac" && b.calculation === "base")) {
      const value = evaluateSimpleFormula(b.formula);
      if (value != null) setters.push(value);
    }
  }
  if (key === "ac") {
    const ctx = baseContext();
    for (const [ability] of ABILITIES) {
      ctx[ability] = calculatedAbility(ability).final;
      ctx[`${ability}_mod`] = modFromScore(ctx[ability]);
    }
    const bases = matches.filter(x => x.calculation === "base" && x.formula)
      .map(x => evaluateSimpleFormula(x.formula,ctx)).filter(Number.isFinite);
    if (bases.length) return Math.max(...bases) + additive;
  }
  // Keep a higher existing score when set_value sets a minimum.
  return (setters.length ? Math.max(Number(baseValue), ...setters) : Number(baseValue)) + additive;
}
function getAsiBonus(key) {
  let total = 0;
  for (const [, choice] of builderAsiEntries()) {
    if (String(choice.type ?? "asi").trim().toLowerCase() !== "asi") continue;
    if (String(choice.ability_1 ?? "").trim().toLowerCase() === key) total++;
    if (String(choice.ability_2 ?? "").trim().toLowerCase() === key) total++;
  }
  return total;
}
function calculatedAbility(key) {
  const base = Number(draft?.[key] ?? 10);
  const final = builderApplyEffects(base + getAsiBonus(key), key);
  return {base, bonus:final-base, final};
}

function getAllAsiLevels() {

  const cp=selectedClassPage();

  const className=String(cp?.name ?? draft?.class ?? "").trim().toLowerCase();

  

  // Preferred explicit backend form:

  // asi_levels: [4, 8, 12, 16, 19]

  if(Array.isArray(cp?.asi_levels)){

    return cp.asi_levels.map(Number).filter(Number.isFinite).sort((a,b)=>a-b);

  }

  

  // Also recognize ASI entries in the class feature progression.

  const features=Array.isArray(cp?.features)?cp.features:[];

  const fromFeatures=features.filter(f=>{

    const ref=String(f?.feature ?? f?.name ?? "").trim();

    const normalized=ref.toLowerCase();

    if(normalized==="ability score improvement" || normalized==="asi") return true;

  

    // If the class entry points to a Feature file, allow that file to identify itself.

    const fp=ref ? (

      dv.page(`Public/3 Backend/Features/${ref}`) ??

      dv.page(`Public/3 Backend/Features/${ref}.md`) ??

      (ref.includes("/") ? dv.page(ref) : null)

    ) : null;

    const kind=String(fp?.type ?? fp?.choice_type ?? "").trim().toLowerCase();

    return kind==="asi" || kind==="ability_score_improvement" || kind==="asi_or_feat";

  }).map(f=>Number(f?.level)).filter(Number.isFinite);

  

  if(fromFeatures.length) return [...new Set(fromFeatures)].sort((a,b)=>a-b);

  

  // 2014 fallback schedules, so the builder remains usable even before

  // every class backend has explicit asi_levels.

  const schedules={

    artificer:[4,8,12,16,19],

    barbarian:[4,8,12,16,19],

    bard:[4,8,12,16,19],

    cleric:[4,8,12,16,19],

    druid:[4,8,12,16,19],

    fighter:[4,6,8,12,14,16,19],

    monk:[4,8,12,16,19],

    paladin:[4,8,12,16,19],

    ranger:[4,8,12,16,19],

    rogue:[4,8,10,12,16,19],

    sorcerer:[4,8,12,16,19],

    warlock:[4,8,12,16,19],

    wizard:[4,8,12,16,19]

  };

  return schedules[className] ?? [];

}

function getAsiLevels() {

  return getAllAsiLevels().filter(level=>level<=Number(draft?.level ?? 1));

}

function featDisplayName(file) {

  const p=pageFor(file);

  return String(p?.name ?? file.basename);

}

function selectedFeatPage(ref) {

  const raw=String(ref ?? "").trim();

  if(!raw) return null;

  if(raw.includes("/")) return dv.page(raw) ?? dv.page(raw.endsWith(".md")?raw:raw+".md");

  return dv.page(`${FEATS_FOLDER}${raw}`) ?? dv.page(`${FEATS_FOLDER}${raw}.md`);

}

function featChoiceState(choice) {

  if(!choice.feat_choices || typeof choice.feat_choices!=="object") choice.feat_choices={};

  return choice.feat_choices;

}

  
  

function sectionTitle(text, subtext="") {

  const h = content.createEl("div");

  h.createEl("div", {text}).style.cssText = "font-size:1.35em;font-weight:700";

  if (subtext) h.createEl("div", {text:subtext}).style.cssText =

    "opacity:.65;margin-top:3px;margin-bottom:16px";

  else h.style.marginBottom = "16px";

}

function field(parent, label) {

  const w = parent.createEl("div");

  w.createEl("div", {text:label}).style.cssText =

    "font-size:.85em;font-weight:600;opacity:.8;margin-bottom:6px";

  return w;

}

  

function renderClass() {

  content.innerHTML = "";

  sectionTitle("Class", "Class, level, subclass and level-based choices.");

  

  const grid = content.createEl("div");

  grid.style.display = "grid";

  grid.style.gridTemplateColumns = "2fr 1fr";

  grid.style.gap = "12px";

  

  const cw = field(grid, "Class");

  const sel = cw.createEl("select"); styleInput(sel);

  sel.createEl("option",{text:"— Select Class —"}).value = "";

  for (const f of classFiles) {

    const p = pageFor(f), name = String(p?.name ?? f.basename);

    const o = sel.createEl("option",{text:name}); o.value=name;

  }

  if (draft.class && !classFiles.some(f => String(pageFor(f)?.name ?? f.basename) === draft.class)) {

    const o=sel.createEl("option",{text:`${draft.class} (missing file)`}); o.value=draft.class;

  }

  sel.value=draft.class ?? "";

  sel.onchange=()=>{draft.class=sel.value; draft.subclass=""; draft.class_choices={}; markDirty(); renderClass();};

  

  const lw = field(grid,"Level");

  const lr = lw.createEl("div");

  lr.style.display="grid"; lr.style.gridTemplateColumns="40px 1fr 40px"; lr.style.gap="6px";

  const minus=makeButton(lr,"−");

  const li=lr.createEl("input"); li.type="number"; li.min="1"; li.max="20"; li.value=String(draft.level); styleInput(li); li.style.textAlign="center";

  const plus=makeButton(lr,"+");

  const changeLevel=n=>{draft.level=clampInt(n,1,20,1); markDirty(); renderClass();};

  minus.onclick=()=>changeLevel(draft.level-1);

  plus.onclick=()=>changeLevel(draft.level+1);

  li.onchange=()=>changeLevel(li.value);

  

  const cfg=getSubclassConfig();

  const unlock=Number(cfg?.unlock_level ?? 1);

  if (cfg && Number(draft.level)>=unlock) {

    const sw=field(content, cfg.label ?? "Subclass");

    sw.style.marginTop="14px";

    const ss=sw.createEl("select"); styleInput(ss);

    const subclassOptions=availableSubclassOptions();

    ss.createEl("option",{text:subclassOptions.length ? "— Select —" : "— No matching subclass files —"}).value="";

    for (const option of subclassOptions) {

      if (!option) continue;

      const name=String(option.name ?? option.file ?? "");

      const o=ss.createEl("option",{text:name}); o.value=name;

    }

    const selectedSubclass=getSubclassName();

    if(selectedSubclass && !subclassOptions.some(x=>String(x.name??"")===selectedSubclass)){

      const o=ss.createEl("option",{text:`${selectedSubclass} (not matched)`});o.value=selectedSubclass;

    }

    ss.value=selectedSubclass;

    ss.onchange=()=>{draft.subclass=ss.value;markDirty();};

  }

  

  const prog=content.createEl("div");

  prog.style.marginTop="22px";

  prog.createEl("div",{text:"Class Progression"}).style.cssText="font-weight:700;font-size:1.05em;margin-bottom:8px";

  

  const cp=selectedClassPage();

  const features=Array.isArray(cp?.features) ? cp.features : [];

  const unlocked=features.filter(x=>Number(x?.level ?? 1)<=Number(draft.level));

  

  if (cp) {

    draft.class_choices=draft.class_choices&&typeof draft.class_choices==="object"

      ? draft.class_choices : {};

  

    const profBox=content.createEl("div");

    profBox.style.cssText="margin-top:18px;padding:14px;border:1px solid var(--background-modifier-border);border-radius:12px";

    profBox.createEl("div",{text:"Class Proficiencies"}).style.cssText="font-weight:700;font-size:1.05em;margin-bottom:10px";

  

    const profDefs =

      (cp.proficiency_choices && typeof cp.proficiency_choices==="object" ? cp.proficiency_choices : null) ??

      (cp.proficiencies?.choices && typeof cp.proficiencies.choices==="object" ? cp.proficiencies.choices : null) ??

      {};

  

    const fixed=cp.proficiencies && typeof cp.proficiencies==="object" ? cp.proficiencies : {};

    const fixedParts=[];

    for(const key of ["armor","weapons","tools","saving_throws","skills"]){

      const vals=Array.isArray(fixed[key]) ? fixed[key] : [];

      if(vals.length) fixedParts.push(`${key.replaceAll("_"," ")}: ${vals.join(", ").replaceAll("_"," ")}`);

    }

    if(fixedParts.length){

      profBox.createEl("div",{text:`Granted: ${fixedParts.join(" • ")}`})

        .style.cssText="font-size:.85em;opacity:.7;margin-bottom:12px";

    }

  

    function classChoiceSelect(label,key,index,options,placeholder){

      if(!Array.isArray(draft.class_choices[key])) draft.class_choices[key]=[];

      const current=String(draft.class_choices[key][index]??"");

      const w=field(profBox,label), select=w.createEl("select"); styleInput(select);

      select.createEl("option",{text:placeholder}).value="";

      const opts=[...new Set((options??[]).map(String))];

      if(current && !opts.includes(current)) opts.unshift(current);

      for(const value of opts){

        const o=select.createEl("option",{text:value.replaceAll("_"," ")}); o.value=value;

      }

      select.value=current;

      select.onchange=()=>{

        draft.class_choices[key][index]=select.value;

        markDirty();

        renderClass();

      };

    }

  

    function renderClassChoiceGroup(key,def,defaults,label,placeholder){

      if(!def || typeof def!=="object") return false;

      const count=Math.max(1,Number(def.count??def.choose??1));

      const options=Array.isArray(def.options) ? def.options : defaults;

      if(!Array.isArray(options) || !options.length) return false;

      for(let i=0;i<count;i++) classChoiceSelect(`${label} ${i+1}`,key,i,options,placeholder);

      return true;

    }

  

    let rendered=false;

    rendered = renderClassChoiceGroup("skills",profDefs.skills,SKILL_OPTIONS,"Skill","— Select Skill —") || rendered;

    rendered = renderClassChoiceGroup("tools",profDefs.tools,TOOL_OPTIONS,"Tool","— Select Tool —") || rendered;

    rendered = renderClassChoiceGroup("languages",profDefs.languages,LANGUAGE_OPTIONS,"Language","— Select Language —") || rendered;

  

    // Compatibility with class files that store choice objects directly in proficiencies.

    if(!rendered){

      rendered = renderClassChoiceGroup("skills",fixed.skills,SKILL_OPTIONS,"Skill","— Select Skill —") || rendered;

      rendered = renderClassChoiceGroup("tools",fixed.tools,TOOL_OPTIONS,"Tool","— Select Tool —") || rendered;

      rendered = renderClassChoiceGroup("languages",fixed.languages,LANGUAGE_OPTIONS,"Language","— Select Language —") || rendered;

    }

  

    if(!rendered && !fixedParts.length){

      profBox.createEl("div",{text:"No class proficiency choices found in this class backend."}).style.opacity=".65";

    }

  }

  

  if (!cp) {

    prog.createEl("div",{text:"No class backend loaded."}).style.opacity=".65";

  } else {

    const levelData=classLevelData();

    const summary=prog.createEl("div");

    summary.style.display="flex"; summary.style.gap="8px"; summary.style.flexWrap="wrap"; summary.style.marginBottom="10px";

    const chips=[

      `Level ${draft.level}`,

      `Proficiency ${levelData?.proficiency_bonus != null ? modString(levelData.proficiency_bonus) : "-"}`,

      `Attunement ${levelData?.attunement_slots ?? "-"}`

    ];

    for (const text of chips) {

      const chip=summary.createEl("span",{text});

      chip.style.cssText="padding:5px 9px;border:1px solid var(--background-modifier-border);border-radius:999px;font-size:.85em";

    }

    if (!unlocked.length) prog.createEl("div",{text:"No unlocked class features found."});

    for (const f of unlocked) {

      const row=prog.createEl("div");

      row.style.cssText="display:grid;grid-template-columns:80px 1fr;gap:10px;padding:8px 0;border-bottom:1px solid var(--background-modifier-border-hover)";

      row.createEl("div",{text:`Level ${f.level ?? 1}`}).style.opacity=".65";

      row.createEl("div",{text:String(f.feature ?? "-")});

    }

  }

  

  const choices=content.createEl("div");

  choices.style.marginTop="22px";

  choices.createEl("div",{text:"Level Choices"}).style.cssText="font-weight:700;font-size:1.05em;margin-bottom:10px";

  const asiLevels=getAsiLevels();

  draft.asi_choices=draft.asi_choices&&typeof draft.asi_choices==="object"?draft.asi_choices:{};

  

  if(!asiLevels.length) {

    const allAsi=getAllAsiLevels();

    const msg=allAsi.length

      ? `No ASI unlocked yet. Next Ability Score Improvement: Level ${allAsi.find(x=>x>Number(draft.level)) ?? allAsi[allAsi.length-1]}.`

      : "No ASI progression found for this class. Add asi_levels to the class backend.";

    choices.createEl("div",{text:msg}).style.opacity=".65";

  }

  

  for(const asiLevel of asiLevels){

    const key=String(asiLevel);

    const choice=draft.asi_choices[key]??{type:"asi",ability_1:"",ability_2:"",feat:""};

    draft.asi_choices[key]=choice;

  

    const card=choices.createEl("div");

    card.style.cssText="padding:14px;border:1px solid var(--background-modifier-border);border-radius:12px;margin-bottom:10px";

    const top=card.createEl("div");

    top.style.cssText="display:flex;justify-content:space-between;gap:10px;align-items:center;margin-bottom:10px";

    top.createEl("div",{text:`Level ${asiLevel} — Ability Score Improvement`}).style.fontWeight="700";

  

    const type=top.createEl("select");styleInput(type);type.style.width="150px";

    for(const [v,l] of [["asi","ASI"],["feat","Feat"]]){const o=type.createEl("option",{text:l});o.value=v;}

    type.value=String(choice.type??"asi");

    const body=card.createEl("div");

  

    function draw(){

      body.innerHTML="";

      if(String(choice.type??"asi")==="asi"){

        body.createEl("div",{text:"Choose two +1 increases. The same ability twice gives +2."})

          .style.cssText="font-size:.85em;opacity:.65;margin-bottom:10px";

        const grid=body.createEl("div");grid.style.cssText="display:grid;grid-template-columns:1fr 1fr;gap:10px";

        function score(label,prop){

          const w=field(grid,label),sel=w.createEl("select");styleInput(sel);

          sel.createEl("option",{text:"— Select Ability —"}).value="";

          for(const [ak,al] of ABILITIES){const o=sel.createEl("option",{text:al});o.value=ak;}

          sel.value=String(choice[prop]??"");

          sel.onchange=()=>{choice[prop]=sel.value;choice.feat="";markDirty();};

        }

        score("Increase 1 (+1)","ability_1");score("Increase 2 (+1)","ability_2");

      }else{

        const w=field(body,"Feat"),sel=w.createEl("select");styleInput(sel);

        sel.createEl("option",{text:"— Select Feat —"}).value="";

        for(const file of featFiles){const o=sel.createEl("option",{text:featDisplayName(file)});o.value=file.path;}

        if(choice.feat&&!featFiles.some(f=>f.path===choice.feat)){const o=sel.createEl("option",{text:`${choice.feat} (missing file)`});o.value=choice.feat;}

        sel.value=String(choice.feat??"");

        sel.onchange=()=>{

          choice.feat=sel.value;

          choice.ability_1="";

          choice.ability_2="";

          choice.feat_choices={};

          markDirty();

          draw();

        };

        if(!featFiles.length) body.createEl("div",{text:`No feat files found in ${FEATS_FOLDER}`})

          .style.cssText="font-size:.85em;opacity:.65;margin-top:8px";

  

        const featPage=selectedFeatPage(choice.feat);

        const definitions=featPage?.choices;

        if(featPage && definitions && typeof definitions==="object"){

          const state=featChoiceState(choice);

          const choiceWrap=body.createEl("div");

          choiceWrap.style.cssText="margin-top:12px;padding-top:12px;border-top:1px solid var(--background-modifier-border-hover)";

          choiceWrap.createEl("div",{text:"Feat Choices"}).style.cssText="font-weight:700;margin-bottom:10px";

  

          function singleSelect(title,key,options){

            const w=field(choiceWrap,title), cs=w.createEl("select");styleInput(cs);

            cs.createEl("option",{text:"— Select —"}).value="";

            for(const option of options??[]){

              const o=cs.createEl("option",{text:String(option).toUpperCase()});o.value=String(option);

            }

            cs.value=String(state[key]??"");

            cs.onchange=()=>{state[key]=cs.value;markDirty();};

          }

  

          if(definitions.ability_increase){

            const d=definitions.ability_increase;

            singleSelect(`Ability Increase (+${Number(d.amount??1)})`,"ability_increase",Array.isArray(d.options)?d.options:[]);

          }

  

          if(definitions.resilient_ability){

            const d=definitions.resilient_ability;

            singleSelect("Resilient Ability (+1 & Save Proficiency)","resilient_ability",Array.isArray(d.options)?d.options:[]);

          }

  

          if(definitions.weapon_proficiencies){

            const d=definitions.weapon_proficiencies;

            const count=Math.max(1,Number(d.count??1));

            const note=choiceWrap.createEl("div",{text:`Choose ${count} weapon proficiencies.`});

            note.style.cssText="font-size:.85em;opacity:.65;margin-bottom:8px";

            if(!Array.isArray(state.weapon_proficiencies)) state.weapon_proficiencies=[];

            for(let wi=0;wi<count;wi++){

              const w=field(choiceWrap,`Weapon ${wi+1}`);

              const inp=w.createEl("input");inp.type="text";inp.placeholder="e.g. longsword";styleInput(inp);

              inp.value=String(state.weapon_proficiencies[wi]??"");

              inp.onchange=()=>{state.weapon_proficiencies[wi]=inp.value.trim().toLowerCase();markDirty();};

            }

          }

  

          if(definitions.skill_or_tool_proficiencies){

            const d=definitions.skill_or_tool_proficiencies;

            const count=Math.max(1,Number(d.count??1));

            if(!Array.isArray(state.skill_or_tool_proficiencies)) state.skill_or_tool_proficiencies=[];

            const note=choiceWrap.createEl("div",{text:`Choose ${count} skill or tool proficiencies.`});

            note.style.cssText="font-size:.85em;opacity:.65;margin-bottom:8px";

            for(let pi=0;pi<count;pi++){

              const row=choiceWrap.createEl("div");

              row.style.cssText="display:grid;grid-template-columns:130px 1fr;gap:8px;margin-bottom:8px";

              const typeSel=row.createEl("select");styleInput(typeSel);

              for(const t of ["skill","tool"]){const o=typeSel.createEl("option",{text:t==="skill"?"Skill":"Tool"});o.value=t;}

              const existing=state.skill_or_tool_proficiencies[pi]??{type:"skill",value:""};

              typeSel.value=existing.type??"skill";

              const inp=row.createEl("input");inp.type="text";inp.placeholder="e.g. perception / thieves_tools";styleInput(inp);

              inp.value=existing.value??"";

              const save=()=>{state.skill_or_tool_proficiencies[pi]={type:typeSel.value,value:inp.value.trim().toLowerCase()};markDirty();};

              typeSel.onchange=save;inp.onchange=save;

            }

          }

        }

      }

    }

    type.onchange=()=>{choice.type=type.value;if(choice.type==="asi")choice.feat="";else{choice.ability_1="";choice.ability_2="";}markDirty();draw();};

    draw();

  }

}

  
  
  

const SKILL_OPTIONS = [

  "acrobatics","animal_handling","arcana","athletics","deception","history",

  "insight","intimidation","investigation","medicine","nature","perception",

  "performance","persuasion","religion","sleight_of_hand","stealth","survival"

];

  

const TOOL_OPTIONS = [

  "alchemists_supplies","brewers_supplies","calligraphers_supplies",

  "carpenters_tools","cartographers_tools","cobblers_tools","cooks_utensils",

  "glassblowers_tools","jewelers_tools","leatherworkers_tools","masons_tools",

  "painters_supplies","potters_tools","smiths_tools","tinkers_tools",

  "weavers_tools","woodcarvers_tools","disguise_kit","forgery_kit",

  "herbalism_kit","navigators_tools","poisoners_kit","thieves_tools",

  "gaming_set","musical_instrument","vehicles_land","vehicles_water"

];

  

const LANGUAGE_OPTIONS = [

  "Common","Dwarvish","Elvish","Giant","Gnomish","Goblin","Halfling","Orc",

  "Abyssal","Celestial","Draconic","Deep Speech","Infernal","Primordial",

  "Sylvan","Undercommon"

];

  

function renderBackground() {

  content.innerHTML="";

  sectionTitle("Background", "Choose a PHB 2014 background and configure its proficiency choices.");

  

  const w=field(content,"Background");

  const sel=w.createEl("select"); styleInput(sel);

  sel.createEl("option",{text:"— Select Background —"}).value="";

  

  for(const file of backgroundFiles){

    const pg=pageFor(file);

    const opt=sel.createEl("option",{text:String(pg?.name ?? file.basename)});

    opt.value=file.path;

  }

  

  // Accept both legacy names ("Soldier") and the new file path.

  let selected=String(draft.background ?? "");

  if(selected && !selected.includes("/")){

    const match=backgroundFiles.find(f=>{

      const pg=pageFor(f);

      return String(pg?.name ?? f.basename).toLowerCase()===selected.toLowerCase();

    });

    if(match) selected=match.path;

  }

  sel.value=selected;

  

  sel.onchange=()=>{

    draft.background=sel.value;

    draft.background_choices={};

    markDirty();

    renderBackground();

  };

  

  if(!backgroundFiles.length){

    content.createEl("div",{text:`No background files found in ${BACKGROUNDS_FOLDER}`}).style.opacity=".65";

    return;

  }

  if(!sel.value) return;

  

  const bg=dv.page(sel.value);

  if(!bg) return;

  draft.background=sel.value;

  draft.background_choices=draft.background_choices&&typeof draft.background_choices==="object"

    ? draft.background_choices : {};

  

  const info=content.createEl("div");

  info.style.cssText="margin-top:14px;padding:14px;border:1px solid var(--background-modifier-border);border-radius:12px";

  info.createEl("div",{text:String(bg.name ?? "Background")}).style.cssText="font-weight:700;font-size:1.05em";

  if(bg.source) info.createEl("div",{text:String(bg.source)}).style.cssText="font-size:.85em;opacity:.65;margin-top:3px";

  

  const prof=bg.proficiencies&&typeof bg.proficiencies==="object"?bg.proficiencies:{};

  const parts=[];

  for(const key of ["skills","tools","languages"]){

    const vals=Array.isArray(prof[key])?prof[key]:[];

    if(vals.length) parts.push(`${key}: ${vals.join(", ").replaceAll("_"," ")}`);

  }

  if(parts.length) info.createEl("div",{text:`Granted: ${parts.join(" • ")}`}).style.cssText="margin-top:10px;font-size:.9em";

  

  const features=Array.isArray(bg.features)?bg.features:[];

  if(features.length) info.createEl("div",{text:`Feature: ${features.join(", ")}`}).style.cssText="margin-top:6px;font-size:.9em";

  

  const defs=bg.proficiency_choices&&typeof bg.proficiency_choices==="object"?bg.proficiency_choices:{};

  

  function bgChoiceSelect(label,key,index,options,placeholder){

    if(!Array.isArray(draft.background_choices[key])) draft.background_choices[key]=[];

    const current=String(draft.background_choices[key][index]??"");

    const fw=field(content,label), cs=fw.createEl("select"); styleInput(cs);

    cs.createEl("option",{text:placeholder}).value="";

    const opts=[...new Set((options??[]).map(String))];

    if(current&&!opts.includes(current)) opts.unshift(current);

    for(const value of opts){

      const o=cs.createEl("option",{text:value.replaceAll("_"," ")});o.value=value;

    }

    cs.value=current;

    cs.onchange=()=>{draft.background_choices[key][index]=cs.value;markDirty();};

  }

  

  for(const [key,def] of Object.entries(defs)){

    if(!def||typeof def!=="object") continue;

    const count=Math.max(1,Number(def.count??def.choose??1));

    const defaults=key==="skills"?SKILL_OPTIONS:key==="tools"?TOOL_OPTIONS:key==="languages"?LANGUAGE_OPTIONS:[];

    const options=Array.isArray(def.options)?def.options:defaults;

    for(let i=0;i<count;i++){

      const singular=key==="languages"?"Language":key==="tools"?"Tool":key==="skills"?"Skill":key;

      bgChoiceSelect(`${singular} ${i+1}`,key,i,options,`— Select ${singular} —`);

    }

  }

}

  

function renderSpecies() {

  content.innerHTML="";

  sectionTitle("Species", "Choose a species/race and configure the choices defined by its backend file.");

  

  const speciesField=field(content,"Species");

  const speciesSelect=speciesField.createEl("select"); styleInput(speciesSelect);

  speciesSelect.createEl("option",{text:"— Select Species —"}).value="";

  

  for(const file of speciesFiles){

    const pg=pageFor(file);

    const opt=speciesSelect.createEl("option",{text:String(pg?.name ?? file.basename)});

    opt.value=file.path;

  }

  

  speciesSelect.value=String(draft.species ?? "");

  speciesSelect.onchange=()=>{

    draft.species=speciesSelect.value;

    draft.species_choices={};

    markDirty();

    renderSpecies();

  };

  

  if(!speciesFiles.length){

    content.createEl("div",{text:`No species files found in ${SPECIES_FOLDER}`})

      .style.cssText="margin-top:10px;font-size:.85em;opacity:.65";

    return;

  }

  

  const speciesPage=draft.species ? dv.page(draft.species) : null;

  if(!speciesPage) return;

  

  const info=content.createEl("div");

  info.style.cssText="margin-top:14px;padding:12px;border:1px solid var(--background-modifier-border);border-radius:10px";

  const facts=[];

  if(speciesPage.source) facts.push(`Source: ${speciesPage.source}`);

  if(speciesPage.size) facts.push(`Size: ${speciesPage.size}`);

  if(speciesPage.speed!=null) facts.push(`Speed: ${speciesPage.speed} ft`);

  if(Array.isArray(speciesPage.languages)) facts.push(`Languages: ${speciesPage.languages.join(", ")}`);

  info.createEl("div",{text:facts.join(" • ")}).style.cssText="font-size:.9em;opacity:.75";

  if(speciesPage.notes) info.createEl("div",{text:String(speciesPage.notes)}).style.cssText="margin-top:8px";

  

  const defs=speciesPage.choices;

  if(!defs || typeof defs!=="object") return;

  if(!draft.species_choices || typeof draft.species_choices!=="object") draft.species_choices={};

  const state=draft.species_choices;

  

  const box=content.createEl("div");

  box.style.cssText="margin-top:14px;padding:14px;border:1px solid var(--background-modifier-border);border-radius:10px";

  box.createEl("div",{text:"Species Choices"}).style.cssText="font-weight:700;margin-bottom:12px";

  

  function optionsFor(def){

    return Array.isArray(def?.options) ? def.options.map(String) : [];

  }

  function singleSelect(label,key,options,placeholder="— Select —"){

    const w=field(box,label), sel=w.createEl("select"); styleInput(sel);

    sel.createEl("option",{text:placeholder}).value="";

    for(const value of options){

      const o=sel.createEl("option",{text:String(value).replaceAll("_"," ")}); o.value=String(value);

    }

    sel.value=String(state[key]??"");

    sel.onchange=()=>{state[key]=sel.value;markDirty();};

  }

  

  function proficiencySelect(label,key,index,options,placeholder="— Select —"){

    const isMulti = index !== null;

    if(isMulti && !Array.isArray(state[key])) state[key]=[];

  

    const current = isMulti ? String(state[key][index] ?? "") : String(state[key] ?? "");

    const w=field(box,label);

    const sel=w.createEl("select"); styleInput(sel);

    sel.createEl("option",{text:placeholder}).value="";

  

    const normalizedOptions=[...options];

    // Preserve older/custom values already stored on the character.

    if(current && !normalizedOptions.some(v=>String(v).toLowerCase()===current.toLowerCase())){

      normalizedOptions.unshift(current);

    }

  

    for(const value of normalizedOptions){

      const o=sel.createEl("option",{text:String(value).replaceAll("_"," ")});

      o.value=String(value);

    }

    sel.value=current;

  

    sel.onchange=()=>{

      if(isMulti) state[key][index]=sel.value;

      else state[key]=sel.value;

      markDirty();

    };

  }

  

  function textChoices(label,key,count,placeholder){

    if(!Array.isArray(state[key])) state[key]=[];

    for(let i=0;i<count;i++){

      const w=field(box,`${label} ${i+1}`);

      const inp=w.createEl("input"); inp.type="text"; inp.placeholder=placeholder; styleInput(inp);

      inp.value=String(state[key][i]??"");

      inp.onchange=()=>{state[key][i]=inp.value.trim().toLowerCase();markDirty();};

    }

  }

  

  for(const [key,def] of Object.entries(defs)){

    if(!def || typeof def!=="object") continue;

    const count=Math.max(1,Number(def.count??1));

    const amount=Number(def.amount??1);

    const opts=optionsFor(def);

  

    if(key==="ability_increase"){

      singleSelect(`Ability Increase (+${amount})`,key,opts);

    } else if(key==="ability_increases"){

      if(!Array.isArray(state[key])) state[key]=[];

      for(let i=0;i<count;i++){

        const w=field(box,`Ability Increase ${i+1} (+${amount})`);

        const sel=w.createEl("select"); styleInput(sel);

        sel.createEl("option",{text:"— Select Ability —"}).value="";

        for(const value of opts){

          const o=sel.createEl("option",{text:String(value).toUpperCase()}); o.value=value;

        }

        sel.value=String(state[key][i]??"");

        sel.onchange=()=>{

          state[key][i]=sel.value;

          if(def.distinct===true){

            state[key]=state[key].filter((v,idx)=>!v || state[key].indexOf(v)===idx);

          }

          markDirty(); renderSpecies();

        };

      }

    } else if(key==="feat"){

      const w=field(box,"Feat"), sel=w.createEl("select"); styleInput(sel);

      sel.createEl("option",{text:"— Select Feat —"}).value="";

      for(const file of featFiles){

        const o=sel.createEl("option",{text:featDisplayName(file)}); o.value=file.path;

      }

      sel.value=String(state[key]??"");

      sel.onchange=()=>{state[key]=sel.value;markDirty();};

    } else if(opts.length){

      singleSelect(key.replaceAll("_"," "),key,opts);

    } else if(key.includes("skill") && count>1){

      for(let i=0;i<count;i++) proficiencySelect(`Skill Proficiency ${i+1}`,key,i,SKILL_OPTIONS,"— Select Skill —");

    } else if(key.includes("skill")){

      proficiencySelect("Skill Proficiency",key,null,SKILL_OPTIONS,"— Select Skill —");

    } else if(key.includes("tool") && count>1){

      for(let i=0;i<count;i++) proficiencySelect(`Tool Proficiency ${i+1}`,key,i,TOOL_OPTIONS,"— Select Tool —");

    } else if(key.includes("tool")){

      proficiencySelect("Tool Proficiency",key,null,TOOL_OPTIONS,"— Select Tool —");

    } else if(key.includes("language") && count>1){

      for(let i=0;i<count;i++) proficiencySelect(`Language ${i+1}`,key,i,LANGUAGE_OPTIONS,"— Select Language —");

    } else if(key.includes("language")){

      proficiencySelect("Language",key,null,LANGUAGE_OPTIONS,"— Select Language —");

    } else {

      const w=field(box,key.replaceAll("_"," "));

      const inp=w.createEl("input"); inp.type="text"; styleInput(inp);

      inp.value=String(state[key]??"");

      inp.onchange=()=>{state[key]=inp.value.trim();markDirty();};

    }

  }

}

  

function renderAbilities() {

  content.innerHTML="";

  sectionTitle("Abilities","Base values and calculated values used by the character sheet.");

  

  const head=content.createEl("div");

  head.style.cssText="display:grid;grid-template-columns:1.2fr 1fr 1fr 1fr 1fr;gap:8px;padding:8px;font-weight:700;border-bottom:1px solid var(--background-modifier-border)";

  for (const t of ["Ability","Base / Raw","Bonuses","Final","Modifier"]) head.createEl("div",{text:t});

  

  for (const [key,label] of ABILITIES) {

    const calc=calculatedAbility(key);

    const row=content.createEl("div");

    row.style.cssText="display:grid;grid-template-columns:1.2fr 1fr 1fr 1fr 1fr;gap:8px;align-items:center;padding:10px 8px;border-bottom:1px solid var(--background-modifier-border-hover)";

    row.createEl("div",{text:label}).style.fontWeight="700";

  

    const input=row.createEl("input"); input.type="number"; input.min="1"; input.max="30"; input.value=String(calc.base); styleInput(input);

    input.onchange=()=>{

      draft[key]=clampInt(input.value,1,30,10);

      markDirty(); renderAbilities();

    };

    row.createEl("div",{text:calc.bonus===0 ? "—" : modString(calc.bonus)});

    const final=row.createEl("div",{text:String(calc.final)}); final.style.fontWeight="700";

    const mod=row.createEl("div",{text:modString(modFromScore(calc.final))}); mod.style.fontWeight="700";

  }

  

  const note=content.createEl("div");

  note.style.cssText="margin-top:16px;padding:12px;border:1px solid var(--background-modifier-border);border-radius:10px;opacity:.75";

  note.setText("Base / Raw is stored in the character backend. The current Bonuses column already reflects active item ability bonuses. ASI, Species and Background sources can be added here without overwriting the raw score.");

}

  

function renderEquipment() {

  content.innerHTML="";

  sectionTitle("Equipment","Search for an item, set the quantity, then add it to the character inventory. Equipped state is controlled only from the character sheet.");

  

  const addCard=content.createEl("div");

  addCard.style.cssText="display:flex;flex-direction:column;gap:8px;margin-bottom:16px;padding:12px;border:1px solid var(--background-modifier-border);border-radius:10px";

  

  const search=addCard.createEl("input");

  search.type="search";

  search.placeholder="Search item…";

  styleInput(search);

  

  const controls=addCard.createEl("div");

  controls.style.cssText="display:grid;grid-template-columns:minmax(220px,1fr) 100px auto;gap:8px;align-items:center";

  

  const itemSelect=controls.createEl("select");

  styleInput(itemSelect);

  

  const addQty=controls.createEl("input");

  addQty.type="number";

  addQty.min="1";

  addQty.max="999";

  addQty.value="1";

  addQty.title="Quantity";

  styleInput(addQty);

  

  const addBtn=makeButton(controls,"Add");

  

  function refillItemSelect() {

    const previous=itemSelect.value;

    itemSelect.innerHTML="";

    itemSelect.createEl("option",{text:"— Select Item —"}).value="";

  

    const q=String(search.value ?? "").trim().toLowerCase();

    const matches=itemFiles

      .map(f=>({file:f,page:pageFor(f)}))

      .filter(x=>{

        if(!q) return true;

        const haystack=[

          x.page?.name ?? x.file.basename,

          x.page?.type ?? "",

          x.page?.category ?? "",

          x.page?.notes ?? ""

        ].join(" ").toLowerCase();

        return haystack.includes(q);

      });

  

    for (const {file,page} of matches) {

      const name=String(page?.name ?? file.basename);

      const type=String(page?.type ?? "").trim();

      const label=type ? `${name} — ${type}` : name;

      const o=itemSelect.createEl("option",{text:label});

      o.value=file.path;

    }

  

    if([...itemSelect.options].some(o=>o.value===previous)) itemSelect.value=previous;

  }

  

  refillItemSelect();

  search.addEventListener("input",refillItemSelect);

  

  addBtn.onclick=()=>{

    if (!itemSelect.value) {

      new Notice("Select an item first.");

      return;

    }

  

    const amount=clampInt(addQty.value,1,999,1);

    draft.inventory=Array.isArray(draft.inventory)?draft.inventory:[];

  

    const existing=draft.inventory.find(e=>{

      const r=resolveItem(e?.item);

      return r?.path===itemSelect.value;

    });

  

    if(existing) existing.quantity=Math.max(1,Number(existing.quantity ?? 1))+amount;

    else draft.inventory.push({item:itemSelect.value,quantity:amount});

  

    markDirty();

    renderEquipment();

  };

  

  const inv=Array.isArray(draft.inventory)?draft.inventory:[];

  if (!inv.length) {

    content.createEl("div",{text:"Inventory is empty."}).style.opacity=".65";

    return;

  }

  

  for (let i=0;i<inv.length;i++) {

    const entry=inv[i], resolved=resolveItem(entry?.item), item=resolved?.page;

    const row=content.createEl("div");

    row.style.cssText="display:grid;grid-template-columns:minmax(220px,2fr) 90px 90px;gap:10px;align-items:center;padding:10px 0;border-bottom:1px solid var(--background-modifier-border-hover)";

  

    const nameWrap=row.createEl("div");

    nameWrap.createEl("div",{text:String(item?.name ?? resolved?.file?.basename ?? entry?.item ?? "Unknown Item")}).style.fontWeight="600";

    nameWrap.createEl("div",{text:String(item?.type ?? "")}).style.cssText="font-size:.8em;opacity:.6";

  

    const qty=row.createEl("input");

    qty.type="number";

    qty.min="1";

    qty.max="999";

    qty.value=String(Math.max(1,Number(entry.quantity ?? 1)));

    styleInput(qty);

    qty.onchange=()=>{

      entry.quantity=Math.max(1,clampInt(qty.value,1,999,1));

      markDirty();

    };

  

    const remove=makeButton(row,"Remove");

    remove.onclick=()=>{

      draft.inventory.splice(i,1);

      markDirty();

      renderEquipment();

    };

  }

}

  

function spellRefPath(ref) {

  const raw=String(ref ?? "").trim();

  if(!raw) return "";

  if(raw.includes("/")) return raw.endsWith(".md") ? raw : raw+".md";

  return `${SPELLS_FOLDER}${raw.endsWith(".md") ? raw : raw+".md"}`;

}

  

function spellSupportsClass(spell, className) {

  const wanted=String(className ?? "").trim().toLowerCase();

  if(!wanted) return false;

  const classes=Array.isArray(spell?.classes) ? spell.classes : (spell?.classes ? [spell.classes] : []);

  return classes.some(x=>String(x ?? "").trim().toLowerCase()===wanted);

}

  

function getPreparedSpellLimit() {

  const className=String(draft?.class ?? "").trim().toLowerCase();

  if(className!=="artificer") return null;

  const intScore=clampInt(draft?.int,1,30,10);

  const intMod=modFromScore(intScore);

  return Math.max(1, intMod + Math.floor(Math.max(1,Number(draft?.level ?? 1))/2));

}

  

function renderSpells() {

  content.innerHTML="";

  sectionTitle("Spells","Search all spell files and manage which spells the character knows and has prepared.");

  

  draft.known_spells=toPlainArray(draft.known_spells);

  draft.prepared_spells=toPlainArray(draft.prepared_spells);

  

  // IMPORTANT:

  // Spell `classes:` tags are intentionally NOT required here.

  // The builder reads every Markdown spell in SPELLS_FOLDER. This keeps the

  // spell backend simple and allows manual/custom spell lists without having

  // to maintain duplicate class metadata in every spell file.

  const available=spellFiles

    .map(file=>({file,page:pageFor(file)}))

    .filter(x=>x.page && String(x.page.category ?? "spell").toLowerCase()==="spell")

    .sort((a,b)=>{

      const la=Number(a.page.level ?? 0), lb=Number(b.page.level ?? 0);

      if(la!==lb) return la-lb;

      return String(a.page.name ?? a.file.basename).localeCompare(String(b.page.name ?? b.file.basename),"de");

    });

  

  // Prepared-spell maximum intentionally disabled.

  // const preparedLimit=getPreparedSpellLimit();

  const preparedLimit=null;

  

  const info=content.createEl("div");

  info.style.cssText="display:flex;gap:12px;flex-wrap:wrap;margin-bottom:10px;padding:10px;border:1px solid var(--background-modifier-border);border-radius:10px";

  info.createEl("span",{text:`Available: ${available.length}`});

  info.createEl("span",{text:`Known: ${draft.known_spells.length}`});

  info.createEl("span",{text:preparedLimit==null ? `Prepared: ${draft.prepared_spells.length}` : `Prepared: ${draft.prepared_spells.length}/${preparedLimit}`});

  

  const filters=content.createEl("div");

  filters.style.cssText="display:grid;grid-template-columns:minmax(220px,1fr) 130px;gap:8px;margin-bottom:12px";

  

  const search=filters.createEl("input");

  search.type="search";

  search.placeholder="Search spells…";

  styleInput(search);

  

  const levelFilter=filters.createEl("select");

  styleInput(levelFilter);

  levelFilter.createEl("option",{text:"All levels"}).value="all";

  levelFilter.createEl("option",{text:"Cantrips"}).value="0";

  for(let lvl=1;lvl<=9;lvl++) levelFilter.createEl("option",{text:`Level ${lvl}`}).value=String(lvl);

  

  const list=content.createEl("div");

  

  function renderSpellList() {

    list.innerHTML="";

    const q=String(search.value ?? "").trim().toLowerCase();

    const wantedLevel=levelFilter.value;

  

    const filtered=available.filter(({file,page})=>{

      const level=Number(page.level ?? 0);

      if(wantedLevel!=="all" && level!==Number(wantedLevel)) return false;

      if(!q) return true;

      const haystack=[

        page.name ?? file.basename,

        page.school ?? "",

        page.effect ?? "",

        page.notes ?? "",

        page.damage_type ?? ""

      ].join(" ").toLowerCase();

      return haystack.includes(q);

    });

  

    if(!filtered.length){

      list.createEl("div",{text:"No matching spells found."}).style.opacity=".65";

      return;

    }

  

    let currentLevel=null;

    for(const {file,page} of filtered){

      const level=Number(page.level ?? 0);

      if(level!==currentLevel){

        currentLevel=level;

        const h=list.createEl("div",{text:level===0?"Cantrips":`Level ${level}`});

        h.style.cssText="font-weight:700;font-size:1.05em;margin-top:14px;padding-bottom:5px;border-bottom:1px solid var(--background-modifier-border)";

      }

  

      const ref=file.path;

      const known=draft.known_spells.map(spellRefPath).includes(ref);

      const prepared=draft.prepared_spells.map(spellRefPath).includes(ref);

  

      const row=list.createEl("div");

      row.style.cssText="display:grid;grid-template-columns:minmax(220px,1fr) 100px 110px;gap:12px;align-items:center;padding:8px 4px;border-bottom:1px solid var(--background-modifier-border-hover)";

  

      const name=row.createEl("div");

      name.createEl("div",{text:String(page.name ?? file.basename)}).style.fontWeight="600";

      const meta=[level===0?"Cantrip":`Level ${level}`,String(page.school ?? "")].filter(Boolean).join(" · ");

      name.createEl("div",{text:meta}).style.cssText="font-size:.8em;opacity:.6";

  

      const knownLabel=row.createEl("label");

      knownLabel.style.cssText="display:flex;gap:6px;align-items:center";

      const knownBox=knownLabel.createEl("input");

      knownBox.type="checkbox";

      knownBox.checked=known;

      knownLabel.createEl("span",{text:"Known"});

  

      const prepLabel=row.createEl("label");

      prepLabel.style.cssText="display:flex;gap:6px;align-items:center";

      const prepBox=prepLabel.createEl("input");

      prepBox.type="checkbox";

      prepBox.checked=prepared;

      prepLabel.createEl("span",{text:"Prepared"});

      prepBox.disabled=!known || level===0;

  

      knownBox.onchange=()=>{

        const knownSet=new Set(draft.known_spells.map(spellRefPath));

        const prepSet=new Set(draft.prepared_spells.map(spellRefPath));

  

        if(knownBox.checked) knownSet.add(ref);

        else {

          knownSet.delete(ref);

          prepSet.delete(ref);

        }

  

        draft.known_spells=[...knownSet];

        draft.prepared_spells=[...prepSet];

        markDirty();

        renderSpellList();

      };

  

      prepBox.onchange=()=>{

        const prepSet=new Set(draft.prepared_spells.map(spellRefPath));

  

        if(prepBox.checked){

          // Prepared-spell limit intentionally disabled:

          // if(preparedLimit!=null && prepSet.size>=preparedLimit){

          //   new Notice(`Prepared spell limit reached (${preparedLimit}).`);

          //   prepBox.checked=false;

          //   return;

          // }

          prepSet.add(ref);

        } else {

          prepSet.delete(ref);

        }

  

        draft.prepared_spells=[...prepSet];

        markDirty();

        renderSpellList();

      };

    }

  }

  

  search.addEventListener("input",renderSpellList);

  levelFilter.addEventListener("change",renderSpellList);

  renderSpellList();

}

  

function renderPlaceholder(type) {

  content.innerHTML="";

  sectionTitle(type, `${type} configuration will use its own backend definitions.`);

  const box=content.createEl("div");

  box.style.cssText="padding:18px;border:1px dashed var(--background-modifier-border);border-radius:12px";

  box.createEl("div",{text:`${type} system prepared`}).style.fontWeight="700";

  box.createEl("div",{

    text:`Next, ${type.toLowerCase()} files can define features, bonuses and choices just like the class backend.`

  }).style.cssText="opacity:.65;margin-top:6px";

}

  

function updateTabs() {

  for (const [key,b] of Object.entries(tabButtons)) {

    const on=key===activeTab;

    b.style.background=on?"var(--interactive-accent)":"var(--background-secondary)";

    b.style.color=on?"var(--text-on-accent)":"var(--text-normal)";

  }

}

function renderTab() {

  window.__dndBuilderTab=activeTab;

  updateTabs();

  if (activeTab==="class") renderClass();

  else if (activeTab==="background") renderBackground();

  else if (activeTab==="species") renderSpecies();

  else if (activeTab==="abilities") renderAbilities();

  else if (activeTab==="equipment") renderEquipment();

  else if (activeTab==="spells") renderSpells();

}

for (const [key,label] of [

  ["class","Class"],["background","Background"],["species","Species"],

  ["abilities","Abilities"],["equipment","Equipment"],["spells","Spells"]

]) {

  const b=makeButton(tabBar,label);

  b.onclick=()=>{activeTab=key;renderTab();};

  tabButtons[key]=b;

}

  

function loadCharacter(path) {

  const file=app.vault.getAbstractFileByPath(path);

  const pg=file?dv.page(path):null;

  if (!file || !pg) { new Notice("Character could not be loaded."); return; }

  currentFile=file; currentPage=pg; window.__dndBuilderCharacterPath=path;

  draft={

    class:String(pg.class ?? ""),

    subclass:Array.isArray(pg.subclass)?[...pg.subclass]:String(pg.subclass ?? ""),

    level:clampInt(pg.level,1,20,1),

    str:clampInt(pg.str,1,30,10), dex:clampInt(pg.dex,1,30,10),

    con:clampInt(pg.con,1,30,10), int:clampInt(pg.int,1,30,10),

    wis:clampInt(pg.wis,1,30,10), cha:clampInt(pg.cha,1,30,10),

    inventory:Array.isArray(pg.inventory) ? pg.inventory.map(e=>({...e})) : [],

    known_spells:toPlainArray(pg.known_spells),

    prepared_spells:toPlainArray(pg.prepared_spells),

    asi_choices:pg.asi_choices&&typeof pg.asi_choices==="object"?JSON.parse(JSON.stringify(pg.asi_choices)):{},

    class_choices:pg.class_choices&&typeof pg.class_choices==="object"?JSON.parse(JSON.stringify(pg.class_choices)):{},

    background:String(pg.background ?? ""),

    background_choices:pg.background_choices&&typeof pg.background_choices==="object"?JSON.parse(JSON.stringify(pg.background_choices)):{},

    species:String(pg.species ?? ""),

    species_choices:pg.species_choices&&typeof pg.species_choices==="object"?JSON.parse(JSON.stringify(pg.species_choices)):{}

  };

  clearDirty(); renderTab();

}

  

async function saveCharacter() {

  if (!currentFile || !draft) {

    new Notice("No character selected.");

    return;

  }

  

  // Resolve the selected file again from the vault instead of relying on a stale object.

  const selectedPath=String(characterSelect.value ?? currentFile.path ?? "").trim();

  const targetFile=app.vault.getAbstractFileByPath(selectedPath);

  

  if(!targetFile || targetFile.extension!=="md"){

    status.setText("Save failed: character file not found");

    new Notice(`Character file not found: ${selectedPath}`);

    return;

  }

  

  saveBtn.disabled=true;

  status.setText(`Saving ${selectedPath}…`);

  

  try {

    const expectedLevel=clampInt(draft.level,1,20,1);

    const expectedClass=String(draft.class ?? "");

  

    await app.fileManager.processFrontMatter(targetFile, fm => {

      fm.class=expectedClass;

  

      const subclass=String(

        Array.isArray(draft.subclass) ? (draft.subclass[0] ?? "") : (draft.subclass ?? "")

      ).trim();

      fm.subclass=subclass || null;

  

      fm.level=expectedLevel;

  

      for(const [key] of ABILITIES){

        fm[key]=clampInt(draft[key],1,30,10);

      }

  

      fm.inventory=JSON.parse(JSON.stringify(draft.inventory ?? []));

      fm.known_spells=JSON.parse(JSON.stringify(draft.known_spells ?? []));

      fm.prepared_spells=JSON.parse(JSON.stringify(draft.prepared_spells ?? []));

      fm.asi_choices=JSON.parse(JSON.stringify(draft.asi_choices ?? {}));

      fm.class_choices=JSON.parse(JSON.stringify(draft.class_choices ?? {}));

      fm.background=String(draft.background ?? "").trim() || null;

      fm.background_choices=JSON.parse(JSON.stringify(draft.background_choices ?? {}));

      fm.species=String(draft.species ?? "").trim() || null;

      fm.species_choices=JSON.parse(JSON.stringify(draft.species_choices ?? {}));

    });

  

    // Wait until Obsidian metadata cache sees the newly written frontmatter.

    let verified=null;

    for(let attempt=0;attempt<20;attempt++){

      await new Promise(resolve=>setTimeout(resolve,100));

      const fm=app.metadataCache.getFileCache(targetFile)?.frontmatter;

      if(fm && Number(fm.level)===expectedLevel && String(fm.class ?? "")===expectedClass){

        verified=fm;

        break;

      }

    }

  

    if(!verified){

      // processFrontMatter already completed; verify directly from the file text as fallback.

      const raw=await app.vault.read(targetFile);

      if(!raw.includes(`level: ${expectedLevel}`)){

        throw new Error("The file was written, but the saved frontmatter could not be verified.");

      }

    }

  

    currentFile=targetFile;

    currentPage=dv.page(targetFile.path) ?? currentPage;

    dirty=false;

    saveBtn.setText("Save Changes");

    status.setText(`Saved: ${targetFile.path}`);

    new Notice(`Saved ${targetFile.basename}`);

  

    try { app.workspace.trigger("dataview:refresh-views"); } catch(e) {}

  } catch(err) {

    console.error("Character Builder save error:",err);

    status.setText(`Save failed: ${err?.message ?? err}`);

    new Notice(`Save failed: ${err?.message ?? err}`);

  } finally {

    saveBtn.disabled=false;

  }

}

  

characterSelect.onchange=()=>{

  if (dirty && !confirm("Discard unsaved changes and switch character?")) {

    characterSelect.value=currentFile?.path ?? characterSelect.value; return;

  }

  loadCharacter(characterSelect.value);

};

saveBtn.onclick=saveCharacter;

loadCharacter(characterSelect.value);
const CHARACTERS_FOLDER = "Public/3 Backend/Characters/";
const BUILDER_PATH = "Public/3 Backend/Character Builder.md";
const DEFAULT_CHARACTER_PATH = "Public/3 Backend/Characters/EIMER Test.md";

function directCharacterFiles() {
  const prefix = CHARACTERS_FOLDER.endsWith("/") ? CHARACTERS_FOLDER : CHARACTERS_FOLDER + "/";
  return app.vault.getMarkdownFiles()
    .filter(file =>
      file.path.startsWith(prefix) &&
      !file.path.slice(prefix.length).includes("/")
    )
    .sort((a,b) => a.basename.localeCompare(b.basename, "de"));
}

const characterFiles = directCharacterFiles();

// Store the selected character in the SHEET note itself.
// This is persistent and Dataview automatically re-evaluates when frontmatter changes.
const sheetPage = dv.current();
const savedCharacterPath = String(sheetPage?.selected_character ?? "").trim();

let CHARACTER_PATH = characterFiles.some(file => file.path === savedCharacterPath)
  ? savedCharacterPath
  : (characterFiles.some(file => file.path === DEFAULT_CHARACTER_PATH)
      ? DEFAULT_CHARACTER_PATH
      : (characterFiles[0]?.path ?? DEFAULT_CHARACTER_PATH));

const ITEMS_FOLDER = "Public/3 Backend/Items/";
const ACTIONS_FOLDER = "Public/3 Backend/Actions/";
const SPELLS_FOLDER = "Public/3 Backend/Spells/";
const FEATURES_FOLDER = "Public/3 Backend/Features/";
const FEATS_FOLDER = "Public/3 Backend/Feats/";
const SPECIES_FOLDER = "Public/3 Backend/Species/";
const CLASSES_FOLDER = "Public/3 Backend/Classes/";
const SUBCLASSES_FOLDER = "Public/3 Backend/Subclasses/";
const BACKGROUNDS_FOLDER = "Public/3 Backend/Backgrounds/";

const c = dv.page(CHARACTER_PATH);

function modFromScore(score) {
  return Math.floor((score - 10) / 2);
}

function modString(n) {
  return n >= 0 ? `+${n}` : `${n}`;
}
function toPlainArray(value) {
  if (Array.isArray(value)) return [...value];
  if (value == null) return [];
  try {
    if (typeof value?.array === "function") return [...value.array()];
    if (typeof value?.values === "function") return [...value.values()];
    if (typeof value?.[Symbol.iterator] === "function" && typeof value !== "string") return [...value];
  } catch {}
  return [];
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

    prof: Number(getClassProgressionValue("proficiency_bonus", c.proficiency_bonus ?? 2)),
    character_level: Number(c.level ?? 1)
  };
}

function evaluateBonusFormula(formula, contextOverride = null) {
  const expr = String(formula ?? "").trim();
  if (!expr) return null;

  const ctx = contextOverride ?? getFormulaContext();
  let safeExpr = expr;

  const replacements = [
    ["character_level", ctx.character_level],
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

function getClassPage() {
  const className = String(c.class ?? "").trim();
  if (!className) return null;

  return resolvePageRef(className, CLASSES_FOLDER);
}

function getClassLevelData() {
  const classPage = getClassPage();
  if (!classPage || !classPage.levels) return null;

  const characterLevel = String(Number(c.level ?? 1));
  return classPage.levels?.[characterLevel] ?? classPage.levels?.[Number(characterLevel)] ?? null;
}

function getClassProgressionValue(key, fallback = null) {
  const levelData = getClassLevelData();
  if (!levelData || levelData[key] == null) return fallback;
  return levelData[key];
}

function getMaxSpellSlots() {
  // 1) Prefer explicit character slots when present. This keeps legacy
  // characters and manually configured casters fully compatible.
  const characterSlots = c.spell_slots;
  if (characterSlots && typeof characterSlots === "object") {
    const normalized = {};
    let hasPositiveSlot = false;
    for (let level = 1; level <= 9; level++) {
      const value = Number(characterSlots?.[String(level)] ?? characterSlots?.[level] ?? 0);
      normalized[String(level)] = Number.isFinite(value) ? Math.max(0, value) : 0;
      if (normalized[String(level)] > 0) hasPositiveSlot = true;
    }
    if (hasPositiveSlot) return normalized;
  }

  // 2) Read either backend layout used by our class files:
  //    levels[level] or progression[level].
  const classPage = getClassPage();
  const levelKey = String(Math.max(1, Number(c.level ?? 1)));
  const levelData =
    classPage?.levels?.[levelKey] ??
    classPage?.progression?.[levelKey] ??
    null;

  const classSlots = levelData?.spell_slots;
  if (classSlots && typeof classSlots === "object") return classSlots;

  // 3) Artificer 2014/Eberron half-caster progression. The current rebuilt
  // Artificer backend contains progression data but no spell_slots entries,
  // so without this fallback the UI calculates zero slots and renders no boxes.
  if (String(classPage?.name ?? c.class ?? "").trim().toLowerCase() === "artificer") {
    const ARTIFICER_SLOTS = {
      1:[2,0,0,0,0],  2:[2,0,0,0,0],
      3:[3,0,0,0,0],  4:[3,0,0,0,0],
      5:[4,2,0,0,0],  6:[4,2,0,0,0],
      7:[4,3,0,0,0],  8:[4,3,0,0,0],
      9:[4,3,2,0,0], 10:[4,3,2,0,0],
      11:[4,3,3,0,0],12:[4,3,3,0,0],
      13:[4,3,3,1,0],14:[4,3,3,1,0],
      15:[4,3,3,2,0],16:[4,3,3,2,0],
      17:[4,3,3,3,1],18:[4,3,3,3,1],
      19:[4,3,3,3,2],20:[4,3,3,3,2]
    };
    const row = ARTIFICER_SLOTS[Math.max(1, Math.min(20, Number(c.level ?? 1)))] ?? [];
    return Object.fromEntries(row.map((value, index) => [String(index + 1), value]));
  }

  return {};
}

function getAttunementSlots() {
  return Number(getClassProgressionValue("attunement_slots", c.attunement_slots ?? 3));
}

function getClassFeatureRefs() {
  const classPage = getClassPage();
  if (!classPage) return [];

  const characterLevel = Number(c.level ?? 1);
  const classFeatures = Array.isArray(classPage.features) ? classPage.features : [];

  return classFeatures
    .filter(entry => {
      if (!entry || typeof entry !== "object") return false;
      return Number(entry.level ?? 1) <= characterLevel;
    })
    .map(entry => entry.feature)
    .filter(ref => ref);
}

function getSubclassConfig() {
  const classPage = getClassPage();
  const config = classPage?.subclass;
  return config && typeof config === "object" ? config : null;
}

function getSubclassOptions() {
  const config = getSubclassConfig();
  return Array.isArray(config?.options) ? config.options : [];
}

function getSelectedSubclassName() {
  if (Array.isArray(c.subclass)) {
    return String(c.subclass.find(Boolean) ?? "").trim();
  }
  return String(c.subclass ?? "").trim();
}

function getSubclassPage() {
  const subclassName = getSelectedSubclassName();
  if (!subclassName) return null;

  const options = getSubclassOptions();
  const matchingOption = options.find(option => {
    if (!option || typeof option !== "object") return false;
    return String(option.name ?? "").trim().toLowerCase() === subclassName.toLowerCase();
  });

  // Prefer the path declared by the class. This prevents accidentally loading
  // a subclass that does not belong to the selected class.
  if (matchingOption?.file) {
    return resolvePageRef(matchingOption.file, SUBCLASSES_FOLDER);
  }

  // Backward-compatible fallback for classes that do not yet declare options.
  if (options.length === 0) {
    return resolvePageRef(subclassName, SUBCLASSES_FOLDER);
  }

  return null;
}

function getSubclassFeatureRefs() {
  const config = getSubclassConfig();
  const subclassPage = getSubclassPage();
  if (!subclassPage) return [];

  const characterLevel = Number(c.level ?? 1);
  const unlockLevel = Number(config?.unlock_level ?? 1);
  if (characterLevel < unlockLevel) return [];

  const subclassFeatures = Array.isArray(subclassPage.features) ? subclassPage.features : [];

  return subclassFeatures
    .filter(entry => {
      if (!entry || typeof entry !== "object") return false;
      return Number(entry.level ?? unlockLevel) <= characterLevel;
    })
    .map(entry => entry.feature)
    .filter(ref => ref);
}


function getCharacterSpeciesPage() {
  const ref = String(c.species ?? "").trim();
  if (!ref) return null;
  return resolvePageRef(ref, SPECIES_FOLDER);
}


function getSpeciesFeatureRefs() {
  const speciesPage = getCharacterSpeciesPage();
  if (!speciesPage) return [];

  const entries = Array.isArray(speciesPage.features) ? speciesPage.features : [];

  return entries
    .map(entry => {
      if (typeof entry === "string") return entry;
      if (entry && typeof entry === "object") return entry.feature ?? entry.file ?? entry.path ?? null;
      return null;
    })
    .filter(Boolean);
}

function getSpeciesChoiceBonusEffects() {
  const speciesPage = getCharacterSpeciesPage();
  if (!speciesPage) return [];

  const defs = speciesPage.choices && typeof speciesPage.choices === "object"
    ? speciesPage.choices
    : {};
  const state = c.species_choices && typeof c.species_choices === "object"
    ? c.species_choices
    : {};
  const result = [];

  const addAbility = (ability, amount, sourceName) => {
    const key = String(ability ?? "").trim().toLowerCase();
    const n = Number(amount ?? 0);
    if (!["str","dex","con","int","wis","cha"].includes(key) || !Number.isFinite(n)) return;
    result.push({
      type: key, value: n, set_value: null, formula: null,
      active_when: "selected", source_kind: "species_choice",
      source_name: sourceName, source_path: speciesPage.file?.path ?? null
    });
  };

  if (defs.ability_increase) {
    addAbility(state.ability_increase, defs.ability_increase.amount ?? 1,
      speciesPage.name ?? "Species");
  }

  if (defs.ability_increases) {
    const selected = Array.isArray(state.ability_increases) ? state.ability_increases : [];
    const amount = Number(defs.ability_increases.amount ?? 1);
    const count = Math.max(1, Number(defs.ability_increases.count ?? selected.length ?? 1));
    selected.slice(0, count).forEach(a => addAbility(a, amount, speciesPage.name ?? "Species"));
  }

  return result;
}

function getSpeciesSpeed() {
  const speciesPage = getCharacterSpeciesPage();
  const speed = Number(speciesPage?.speed);
  return Number.isFinite(speed) ? speed : null;
}

function getActiveAsiChoiceEntries() {
  // IMPORTANT: read the raw stored choices here. Calling getActiveAsiChoices()
  // from this function would recurse indefinitely.
  const choices = c.asi_choices && typeof c.asi_choices === "object"
    ? c.asi_choices
    : {};
  const level = Number(c.level ?? 1);

  return Object.entries(choices).filter(([levelKey, choice]) => {
    if (!choice) return false;
    const unlockLevel = Number(levelKey);

    // Numeric keys are ASI unlock levels ("4", "8", ...).
    // Non-numeric legacy keys remain active for backward compatibility.
    return !Number.isFinite(unlockLevel) || unlockLevel <= level;
  });
}

function getActiveAsiChoices() {
  return getActiveAsiChoiceEntries().map(([, choice]) => choice);
}

function getAllCharacterFeatures() {
  const manualFeatureRefs = Array.isArray(c.features) ? c.features : [];
  const classFeatureRefs = getClassFeatureRefs();
  const subclassFeatureRefs = getSubclassFeatureRefs();
  const speciesFeatureRefs = getSpeciesFeatureRefs();

  const seenPaths = new Set();
  const features = [];

  // Every source only grants references. The actual mechanics live in
  // Public/3 Backend/Features/. Missing files are ignored safely.
  const allFeatureRefs = [
    ...manualFeatureRefs,
    ...classFeatureRefs,
    ...subclassFeatureRefs,
    ...speciesFeatureRefs
  ];

  for (const ref of allFeatureRefs) {
    const resolvedPath = resolvePathRef(ref, FEATURES_FOLDER);
    if (!resolvedPath || seenPaths.has(resolvedPath)) continue;

    const page = dv.page(resolvedPath);
    if (!page) continue;

    seenPaths.add(resolvedPath);
    features.push(page);
  }

  // Feats selected through ASI choices are also Features for display,
  // bonuses and resources. They stay in the Feats folder; no duplication
  // in c.features is required.
  const asiChoices = getActiveAsiChoices();

  for (const choice of asiChoices) {
    if (!choice || String(choice.type ?? "").trim().toLowerCase() !== "feat") continue;

    const featRef = String(choice.feat ?? "").trim();
    if (!featRef) continue;

    const resolvedPath = resolvePathRef(featRef, FEATS_FOLDER);
    if (!resolvedPath || seenPaths.has(resolvedPath)) continue;

    const page = dv.page(resolvedPath);
    if (!page) continue;

    seenPaths.add(resolvedPath);
    features.push(page);
  }

  return features.sort((a, b) => {
    const nameA = String(a.name ?? a.file?.name ?? "");
    const nameB = String(b.name ?? b.file?.name ?? "");
    return nameA.localeCompare(nameB, "de");
  });
}


function getSelectedFeatPages() {
  const choices = getActiveAsiChoices();
  const result = [];

  for (const choice of choices) {
    if (!choice || String(choice.type ?? "").toLowerCase() !== "feat") continue;
    const ref = String(choice.feat ?? "").trim();
    if (!ref) continue;
    const page = resolvePageRef(ref, FEATS_FOLDER);
    if (page) result.push(page);
  }

  return result;
}

function getFeatProficiencies(category) {
  const key = String(category ?? "").trim().toLowerCase();
  const values = [];

  for (const feat of getSelectedFeatPages()) {
    const profs = feat.proficiencies;
    if (!profs || typeof profs !== "object") continue;
    const entries = Array.isArray(profs[key]) ? profs[key] : [];
    for (const entry of entries) {
      const value = String(entry ?? "").trim().toLowerCase();
      if (value && !values.includes(value)) values.push(value);
    }
  }

  const choices = getActiveAsiChoices();
  for (const choice of choices) {
    if (!choice || String(choice.type ?? "").toLowerCase() !== "feat") continue;
    const state = choice.feat_choices && typeof choice.feat_choices === "object"
      ? choice.feat_choices : {};
    const feat = resolvePageRef(choice.feat, FEATS_FOLDER);

    if (key === "saves" && feat?.choices?.resilient_ability?.grants_matching_save_proficiency === true) {
      const ability = String(state.resilient_ability ?? "").trim().toLowerCase();
      if (ability && !values.includes(ability)) values.push(ability);
    }

    if (key === "weapons" && Array.isArray(state.weapon_proficiencies)) {
      for (const entry of state.weapon_proficiencies) {
        const value=String(entry ?? "").trim().toLowerCase();
        if(value && !values.includes(value)) values.push(value);
      }
    }

    if ((key === "skills" || key === "tools") && Array.isArray(state.skill_or_tool_proficiencies)) {
      const expectedType = key === "skills" ? "skill" : "tool";
      for (const entry of state.skill_or_tool_proficiencies) {
        if (String(entry?.type ?? "").toLowerCase() !== expectedType) continue;
        const value=String(entry?.value ?? "").trim().toLowerCase();
        if(value && !values.includes(value)) values.push(value);
      }
    }
  }
  return values;
}

function hasFeatProficiency(category, value) {
  return getFeatProficiencies(category)
    .includes(String(value ?? "").trim().toLowerCase());
}


// A shield is not armor for the Unarmored Defense condition.
function characterIsWearingArmor() {
  for (const entry of (Array.isArray(c.inventory) ? c.inventory : [])) {
    if (entry?.equipped !== true) continue;
    const page = resolvePageRef(entry.item, ITEMS_FOLDER);
    if (!page) continue;
    const tokens = [page.type, page.category, page.armor_type, page.armor_category,
                    page.item_type, page.equipment_category, page.subtype]
      .map(x => String(x ?? "").toLowerCase().replace(/[_-]/g, " "));
    const isShield = tokens.some(x => /\bshield\b/.test(x)) ||
      /\bshield\b/i.test(String(page.name ?? page.file?.name ?? ""));
    if (isShield) continue;
    if (tokens.some(x => /\b(light armor|medium armor|heavy armor|armor)\b/.test(x))) return true;
    // Explicit armor-class entries are also recognizable when type is unspecified.
    if (page.armor_class != null || page.armor_base != null) return true;
  }
  return false;
}
function bonusConditionSatisfied(bonus, defaultWhen, sourceActive = true, equipped = false, attuned = false) {
  const when = String(bonus?.active_when ?? defaultWhen).trim().toLowerCase();
  if (when === "always") return sourceActive;
  if (when === "enabled" || when === "selected") return sourceActive;
  if (when === "equipped") return equipped;
  if (when === "attuned") return attuned;
  if (when === "unarmored") return sourceActive && !characterIsWearingArmor();
  if (when === "armored") return sourceActive && characterIsWearingArmor();
  return false;
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

        isActive = bonusConditionSatisfied(bonus, "equipped", true, equipped, attuned);

        if (!isActive) continue;

        activeEffects.push({
          type,
          value: Number.isFinite(value) ? value : 0,
          set_value: hasSetValue && Number.isFinite(setValue) ? setValue : null,
          formula: formula || null,
          calculation: String(bonus.calculation ?? "").trim().toLowerCase(),
          active_when: activeWhen,
          source_kind: "item",
          source_name: itemPage.name ?? itemPage.file?.name ?? "Unknown Item",
          source_path: itemPath,
          source_instance_id: instanceId
        });
      }
    }
  }

  const speciesPage = getCharacterSpeciesPage();

  if (speciesPage) {
    const speciesBonuses = [
      ...(Array.isArray(speciesPage.bonuses) ? speciesPage.bonuses : []),
      ...(Array.isArray(speciesPage.scaling_bonuses) ? speciesPage.scaling_bonuses : [])
    ];

    for (const bonus of speciesBonuses) {
      if (!bonus || typeof bonus !== "object") continue;

      const type = String(bonus.type ?? "").trim().toLowerCase();
      const value = Number(bonus.value ?? 0);
      const formula = String(bonus.formula ?? "").trim();
      const rawSetValue = bonus.set_value;
      const hasSetValue = rawSetValue !== undefined && rawSetValue !== null && rawSetValue !== "";
      const setValue = hasSetValue ? Number(rawSetValue) : null;

      if (!type) continue;
      if (!Number.isFinite(value) && !formula && !(hasSetValue && Number.isFinite(setValue))) continue;

      activeEffects.push({
        type,
        value: Number.isFinite(value) ? value : 0,
        set_value: hasSetValue && Number.isFinite(setValue) ? setValue : null,
        formula: formula || null,
        calculation: String(bonus.calculation ?? "").trim().toLowerCase(),
        active_when: "selected",
        source_kind: "species",
        source_name: speciesPage.name ?? speciesPage.file?.name ?? "Species",
        source_path: speciesPage.file?.path ?? null
      });
    }

    activeEffects.push(...getSpeciesChoiceBonusEffects());
  }

  const features = getAllCharacterFeatures();

  for (const featurePage of features) {
    const enabled = featurePage.enabled !== false;
    const bonuses = Array.isArray(featurePage.bonuses) ? featurePage.bonuses : [];
    const scalingBonuses = Array.isArray(featurePage.scaling_bonuses) ? featurePage.scaling_bonuses : [];

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
      if (!Number.isFinite(value) && !(hasSetValue && Number.isFinite(setValue)) && !formula) continue;

      const isActive = bonusConditionSatisfied(bonus, "enabled", enabled);
      if (!isActive) continue;

      activeEffects.push({
        type,
        value: Number.isFinite(value) ? value : 0,
        set_value: hasSetValue && Number.isFinite(setValue) ? setValue : null,
        formula: formula || null,
        calculation: String(bonus.calculation ?? "").trim().toLowerCase(),
        active_when: activeWhen,
        source_kind: String(featurePage.type ?? "").toLowerCase() === "feat" ? "feat" : "feature",
        source_name: featurePage.name ?? featurePage.file?.name ?? "Unnamed Feature",
        source_path: featurePage.file?.path ?? null
      });
    }

    for (const bonus of scalingBonuses) {
      if (!enabled || !bonus || typeof bonus !== "object") continue;
      const type = String(bonus.type ?? "").trim().toLowerCase();
      const formula = String(bonus.formula ?? "").trim();
      const value = Number(bonus.value ?? 0);
      if (!type || (!formula && !Number.isFinite(value))) continue;

      activeEffects.push({
        type,
        value: Number.isFinite(value) ? value : 0,
        set_value: null,
        formula: formula || null,
        active_when: "enabled",
        source_kind: "feature_scaling",
        source_name: featurePage.name ?? featurePage.file?.name ?? "Unnamed Feature",
        source_path: featurePage.file?.path ?? null
      });
    }
  }

  const asiChoices = getActiveAsiChoices();
  for (const choice of asiChoices) {
    if (!choice || String(choice.type ?? "").toLowerCase() !== "feat") continue;
    const feat = resolvePageRef(choice.feat, FEATS_FOLDER);
    const defs = feat?.choices;
    const state = choice.feat_choices && typeof choice.feat_choices === "object" ? choice.feat_choices : {};

    if (defs?.ability_increase) {
      const ability=String(state.ability_increase ?? "").trim().toLowerCase();
      const amount=Number(defs.ability_increase.amount ?? 1);
      if (ability && Number.isFinite(amount)) activeEffects.push({
        type:ability,value:amount,set_value:null,formula:null,active_when:"selected",
        source_kind:"feat_choice",source_name:feat?.name ?? "Feat Choice",source_path:feat?.file?.path ?? null
      });
    }

    if (defs?.resilient_ability) {
      const ability=String(state.resilient_ability ?? "").trim().toLowerCase();
      const amount=Number(defs.resilient_ability.amount ?? 1);
      if (ability && Number.isFinite(amount)) activeEffects.push({
        type:ability,value:amount,set_value:null,formula:null,active_when:"selected",
        source_kind:"feat_choice",source_name:feat?.name ?? "Feat Choice",source_path:feat?.file?.path ?? null
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

  const baseFormulas = targetType === "ac"
    ? matchingEffects.filter(b => b.calculation === "base" && b.formula)
    : [];
  // AC base alternatives use *final* ability modifiers, including ASI, species and items.
  // For example: 10 + dex_mod + con_mod, optionally combined with shield bonuses.
  const formulaContext = targetType === "ac" ? {
    ...getFormulaContext(),
    ...Object.fromEntries(["str","dex","con","int","wis","cha"].flatMap(k =>
      [[k,getAbilityScore(k)],[`${k}_mod`,getAbilityModByName(k)]]))
  } : null;
  const evaluate = (formula) => evaluateBonusFormula(formula, formulaContext);
  const baseCandidates = baseFormulas.map(b => evaluate(b.formula))
    .filter(v => v != null && Number.isFinite(v));
  const normalFormulas = matchingEffects.filter(b => b.formula && b.calculation !== "base")
    .map(b => evaluate(b.formula)).filter(v => v != null && Number.isFinite(v));
  if (targetType === "ac" && baseCandidates.length) {
    // A base formula replaces the stored raw AC when its condition is met.
    result = Math.max(...baseCandidates);
  } else if (setValues.length || normalFormulas.length) {
    result = Math.max(result, ...setValues, ...normalFormulas);
  }

  const additiveBonus = matchingEffects.reduce((sum, b) => {
    return sum + Number(b.value ?? 0);
  }, 0);

  result += additiveBonus;
  return result;
}

function getAsiAbilityBonus(key) {
  const choices = getActiveAsiChoices();
  let total = 0;

  // Only ordinary ASI selections belong here.
  // Feat ability bonuses are already collected by getActiveBonusEffects().
  for (const choice of choices) {
    if (!choice || String(choice.type ?? "asi").trim().toLowerCase() !== "asi") continue;
    if (String(choice.ability_1 ?? "").trim().toLowerCase() === key) total += 1;
    if (String(choice.ability_2 ?? "").trim().toLowerCase() === key) total += 1;
  }
  return total;
}

function getAbilityScore(name) {
  const key = String(name ?? "").trim().toLowerCase();
  if (!["str","dex","con","int","wis","cha"].includes(key)) return 10;

  const raw = Number(c[key] ?? 10);

  // Species bonuses, species choices and feat ability bonuses are already
  // supplied by getActiveBonusEffects() inside applyItemEffects().
  // Only ordinary ASI selections have to be added to the raw score here.
  const withAsi = raw + getAsiAbilityBonus(key);
  return applyItemEffects(withAsi, key);
}

function getAbilityModByName(name) {
  return modFromScore(getAbilityScore(name));
}

function getProficiencyBonus() {
  // D&D 5e proficiency bonus is determined by total character level:
  // 1-4 +2, 5-8 +3, 9-12 +4, 13-16 +5, 17-20 +6.
  // Do not depend on a class backend field here; older/newer backends use
  // different progression layouts and that previously caused stale +2 values.
  const level = Math.max(1, Math.min(20, Number(c.level ?? 1)));
  const baseProf = 2 + Math.floor((level - 1) / 4);
  return applyItemEffects(baseProf, "proficiency_bonus");
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


function proficiencyDisplayName(value) {
  const raw = String(value ?? "").trim();
  if (!raw) return "";
  const special = {
    str:"STR", dex:"DEX", con:"CON", int:"INT", wis:"WIS", cha:"CHA",
    light:"Light Armor", medium:"Medium Armor", heavy:"Heavy Armor",
    shields:"Shields", simple:"Simple Weapons", martial:"Martial Weapons"
  };
  const key = raw.toLowerCase().replace(/\s+/g, "_");
  if (special[key]) return special[key];
  return raw
    .replace(/_/g, " ")
    .replace(/\b\w/g, ch => ch.toUpperCase());
}

function getAllProficienciesByCategory() {
  const result = {
    armor:new Set(), weapons:new Set(), tools:new Set(),
    skills:new Set(), saving_throws:new Set(), languages:new Set()
  };

  const add = (category, raw) => {
    if (!result[category] || raw == null) return;
    const values = Array.isArray(raw) ? raw : [raw];
    for (const value of values) {
      const v = String(value ?? "").trim();
      if (v) result[category].add(v);
    }
  };

  const addPageProficiencies = page => {
    const p = page?.proficiencies;
    if (!p || typeof p !== "object") return;
    add("armor", p.armor);
    add("weapons", p.weapons);
    add("tools", p.tools);
    add("skills", p.skills);
    add("saving_throws", p.saving_throws ?? p.saves);
    add("languages", p.languages);
  };

  // Fixed grants from backend sources.
  addPageProficiencies(getClassPage());
  addPageProficiencies(getCharacterSpeciesPage());
  addPageProficiencies(getBackgroundPage());
  for (const feature of getAllCharacterFeatures()) addPageProficiencies(feature);
  for (const feat of getSelectedFeatPages()) addPageProficiencies(feat);

  // Class choices.
  const cc = c.class_choices && typeof c.class_choices === "object" ? c.class_choices : {};
  add("skills", cc.skills); add("tools", cc.tools);
  add("weapons", cc.weapons); add("armor", cc.armor); add("languages", cc.languages);

  // Background choices.
  const bc = c.background_choices && typeof c.background_choices === "object" ? c.background_choices : {};
  add("skills", bc.skills); add("tools", bc.tools);
  add("weapons", bc.weapons); add("armor", bc.armor); add("languages", bc.languages);

  // Species choices can use descriptive keys such as skill_proficiency,
  // tool_proficiency and extra_language.
  const sc = c.species_choices && typeof c.species_choices === "object" ? c.species_choices : {};
  for (const [key, value] of Object.entries(sc)) {
    const k = String(key).toLowerCase();
    if (k.includes("skill")) add("skills", value);
    else if (k.includes("tool")) add("tools", value);
    else if (k.includes("weapon")) add("weapons", value);
    else if (k.includes("armor")) add("armor", value);
    else if (k.includes("language")) add("languages", value);
  }

  // Existing effective helpers preserve legacy character fields and feat choices.
  for (const v of getEffectiveSkillProficiencies()) add("skills", v);
  for (const v of getEffectiveSaveProficiencies()) add("saving_throws", v);
  for (const category of ["armor","weapons","tools","skills","languages"]) {
    for (const v of getFeatProficiencies(category)) add(category, v);
  }
  for (const v of getFeatProficiencies("saves")) add("saving_throws", v);

  return result;
}

function normalizedProfKey(value) {
  return String(value ?? "").trim().toLowerCase().replace(/\s+/g, "_");
}

function getBackgroundPage() {
  const raw = String(c.background ?? "").trim();
  if (!raw) return null;

  // New builder stores a full backend path; legacy characters may only store
  // a background name such as "Soldier".
  return resolvePageRef(raw, BACKGROUNDS_FOLDER);
}

function getEffectiveSkillProficiencies() {
  const result = new Set();

  // Legacy character booleans remain supported.
  const legacySkills = [
    "acrobatics","animal_handling","arcana","athletics","deception","history",
    "insight","intimidation","investigation","medicine","nature","perception",
    "performance","persuasion","religion","sleight_of_hand","stealth","survival"
  ];
  for (const key of legacySkills) {
    if (c[`${key}_prof`] === true) result.add(key);
  }

  // Choices made in the class builder.
  const classSkills = Array.isArray(c.class_choices?.skills) ? c.class_choices.skills : [];
  for (const skill of classSkills) if (skill) result.add(normalizedProfKey(skill));

  // Fixed proficiencies granted by the selected background backend.
  const backgroundPage = getBackgroundPage();
  const backgroundSkills = Array.isArray(backgroundPage?.proficiencies?.skills)
    ? backgroundPage.proficiencies.skills : [];
  for (const skill of backgroundSkills) if (skill) result.add(normalizedProfKey(skill));

  // Choices made in the background builder.
  const backgroundSkillsChosen = Array.isArray(c.background_choices?.skills)
    ? c.background_choices.skills : [];
  for (const skill of backgroundSkillsChosen) if (skill) result.add(normalizedProfKey(skill));

  // Choices made in the species builder. Species backends may use either one
  // skill field or several differently named skill choice fields.
  const speciesChoices = c.species_choices && typeof c.species_choices === "object"
    ? c.species_choices : {};
  for (const [key, raw] of Object.entries(speciesChoices)) {
    if (!String(key).toLowerCase().includes("skill")) continue;
    const values = Array.isArray(raw) ? raw : [raw];
    for (const skill of values) if (skill) result.add(normalizedProfKey(skill));
  }

  return result;
}

function isSkillProficient(skillKey) {
  return getEffectiveSkillProficiencies().has(normalizedProfKey(skillKey));
}

function getEffectiveSaveProficiencies() {
  const result = new Set();
  for (const ability of ["str","dex","con","int","wis","cha"]) {
    if (c[`${ability}_save_prof`] === true) result.add(ability);
  }

  // Class backend is authoritative for class saving-throw proficiencies.
  const classPage = getClassPage();
  const saves = Array.isArray(classPage?.proficiencies?.saving_throws)
    ? classPage.proficiencies.saving_throws : [];
  for (const save of saves) if (save) result.add(normalizedProfKey(save));

  return result;
}

function isSaveProficient(abilityKey) {
  return getEffectiveSaveProficiencies().has(normalizedProfKey(abilityKey));
}

function getSavingThrowTotal(label, isProficient) {
  const abilityKey = String(label ?? "").trim().toLowerCase();
  const base = getAbilityModByName(abilityKey);
  const profBonus = getProficiencyBonus();
  const proficient = Boolean(isProficient) || hasFeatProficiency("saves", abilityKey);

  let total = profValue(proficient, base, profBonus);
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

function getFeatureResourceDefinition(featurePage) {
  let resource = featurePage?.resource;
  if ((!resource || typeof resource !== "object") && featurePage?.resources) {
    resource = Array.isArray(featurePage.resources)
      ? (featurePage.resources[0] ?? null)
      : featurePage.resources;
  }
  if (!resource || typeof resource !== "object") return null;

  const id = String(resource.id ?? "").trim();
  if (!id) return null;

  let maxUses = null;

  if (resource.max_formula != null) {
    maxUses = evaluateBonusFormula(resource.max_formula);
  } else if (resource.max != null) {
    maxUses = Number(resource.max);
  } else if (resource.max_uses != null) {
    maxUses = Number(resource.max_uses);
  }

  if (!Number.isFinite(Number(maxUses))) return null;

  const minimum = Number(resource.minimum ?? 0);
  maxUses = Math.max(Number.isFinite(minimum) ? minimum : 0, Math.floor(Number(maxUses)));

  return {
    id,
    label: String(resource.label ?? "Uses"),
    max: maxUses,
    recharge: String(resource.recharge ?? "").trim().toLowerCase(),
    display: String(resource.display ?? "checkboxes").trim().toLowerCase()
  };
}



function getItemResourceDefinition(itemPage) {
  const resource = itemPage?.resource;
  if (!resource || typeof resource !== "object") return null;

  const id = String(resource.id ?? "").trim();
  if (!id) return null;

  let maxUses = null;
  if (resource.max_formula != null) maxUses = evaluateBonusFormula(resource.max_formula);
  else if (resource.max != null) maxUses = Number(resource.max);
  else if (resource.max_uses != null) maxUses = Number(resource.max_uses);
  if (!Number.isFinite(Number(maxUses))) return null;

  const minimum = Number(resource.minimum ?? 0);
  maxUses = Math.max(Number.isFinite(minimum) ? minimum : 0, Math.floor(Number(maxUses)));

  return {
    id,
    label: String(resource.label ?? "Uses"),
    max: maxUses,
    recharge: String(resource.recharge ?? "").trim().toLowerCase(),
    rechargeAmount: String(resource.recharge_amount ?? "").trim().toLowerCase(),
    display: String(resource.display ?? "checkboxes").trim().toLowerCase()
  };
}

function getItemResourceUsed(instanceId, resourceId) {
  const all = c.item_resources;
  if (!all || typeof all !== "object") return 0;
  const instance = all[instanceId];
  if (!instance || typeof instance !== "object") return 0;
  const state = instance[resourceId];
  if (!state || typeof state !== "object") return 0;
  return Math.max(0, Number(state.used ?? 0));
}

async function setItemResourceUsed(instanceId, resourceId, used, maxUses = null) {
  const characterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);
  if (!characterFile) {
    new Notice("Character-Datei nicht gefunden.");
    return;
  }
  const nextUsed = Math.max(0, maxUses == null ? Number(used ?? 0) : Math.min(Number(maxUses), Number(used ?? 0)));
  await app.fileManager.processFrontMatter(characterFile, (fm) => {
    if (!fm.item_resources || typeof fm.item_resources !== "object" || Array.isArray(fm.item_resources)) fm.item_resources = {};
    if (!fm.item_resources[instanceId] || typeof fm.item_resources[instanceId] !== "object") fm.item_resources[instanceId] = {};
    if (!fm.item_resources[instanceId][resourceId] || typeof fm.item_resources[instanceId][resourceId] !== "object") fm.item_resources[instanceId][resourceId] = {};
    fm.item_resources[instanceId][resourceId].used = nextUsed;
  });
}

function rollSimpleDiceFormula(formula) {
  const match = String(formula ?? "").trim().toLowerCase().match(/^(\d+)d(\d+)([+-]\d+)?$/);
  if (!match) return null;
  const count = Number(match[1]);
  const sides = Number(match[2]);
  const bonus = Number(match[3] ?? 0);
  if (count < 1 || sides < 1 || count > 100) return null;
  let total = bonus;
  for (let i = 0; i < count; i++) total += 1 + Math.floor(Math.random() * sides);
  return total;
}

function getFeatureResourceUsed(resourceId) {
  const resources = c.resources;
  if (!resources || typeof resources !== "object") return 0;
  const state = resources[resourceId];
  if (!state || typeof state !== "object") return 0;
  return Math.max(0, Number(state.used ?? 0));
}

async function setFeatureResourceUsed(resourceId, used, maxUses = null) {
  const characterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);
  if (!characterFile) {
    new Notice("Character-Datei nicht gefunden.");
    return;
  }

  const nextUsed = Math.max(
    0,
    maxUses == null ? Number(used ?? 0) : Math.min(Number(maxUses), Number(used ?? 0))
  );

  await app.fileManager.processFrontMatter(characterFile, (fm) => {
    if (!fm.resources || typeof fm.resources !== "object" || Array.isArray(fm.resources)) {
      fm.resources = {};
    }
    if (!fm.resources[resourceId] || typeof fm.resources[resourceId] !== "object") {
      fm.resources[resourceId] = {};
    }
    fm.resources[resourceId].used = nextUsed;
  });
}

function getResourceIdsForRest(restType) {
  const ids = new Set();

  const speciesPage = getCharacterSpeciesPage();
  if (speciesPage) {
    const def = getFeatureResourceDefinition(speciesPage);
    if (def) {
      const recharge = String(def.recharge ?? "").toLowerCase();
      if (
        (restType === "short" && recharge === "short_rest") ||
        (restType === "long" && (recharge === "short_rest" || recharge === "long_rest"))
      ) ids.add(def.id);
    }
  }

  for (const featurePage of getAllCharacterFeatures()) {
    const def = getFeatureResourceDefinition(featurePage);
    if (!def) continue;

    const shouldReset =
      def.recharge === restType ||
      (restType === "long_rest" && def.recharge === "short_rest");

    if (shouldReset) ids.add(def.id);
  }

  return ids;
}

function resetRechargeResourcesInPlace(value, restType) {
  if (!value) return;

  if (Array.isArray(value)) {
    for (const entry of value) resetRechargeResourcesInPlace(entry, restType);
    return;
  }

  if (typeof value !== "object") return;

  const recharge = String(value.recharge ?? "").trim().toLowerCase();
  const shouldReset =
    recharge === restType ||
    (restType === "long_rest" && recharge === "short_rest");

  if (shouldReset) {
    const max = Number(value.max ?? value.max_uses ?? NaN);
    if (Number.isFinite(max)) {
      if ("current" in value) value.current = max;
      if ("uses_remaining" in value) value.uses_remaining = max;
      if ("used" in value) value.used = 0;
      if ("uses_used" in value) value.uses_used = 0;
    }
  }

  for (const child of Object.values(value)) {
    if (child && typeof child === "object") {
      resetRechargeResourcesInPlace(child, restType);
    }
  }
}

async function performRest(restType) {
  const characterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);
  if (!characterFile) {
    new Notice("Character-Datei nicht gefunden.");
    return;
  }

  const isLongRest = restType === "long_rest";
  const maxHp = getMaxHp();
  const characterLevel = Math.max(1, Number(c.level ?? 1));
  const featureResourceIdsToReset = getResourceIdsForRest(restType);

  await app.fileManager.processFrontMatter(characterFile, (fm) => {
    // Legacy/generic resources that carry their recharge rule in character state.
    resetRechargeResourcesInPlace(fm.resources, restType);
    resetRechargeResourcesInPlace(fm.feature_resources, restType);

    // Preferred model: feature defines recharge, character stores only used state.
    if (!fm.resources || typeof fm.resources !== "object" || Array.isArray(fm.resources)) {
      fm.resources = {};
    }
    for (const resourceId of featureResourceIdsToReset) {
      if (!fm.resources[resourceId] || typeof fm.resources[resourceId] !== "object") {
        fm.resources[resourceId] = {};
      }
      fm.resources[resourceId].used = 0;
    }

    if (!isLongRest) return;

    // Long Rest: restore HP and clear temporary HP.
    fm.hp_current = maxHp;
    fm.hp_temp = 0;

    // All spell slots become available again.
    const maxSlots = getMaxSpellSlots();
    if (!fm.spell_slots_used || typeof fm.spell_slots_used !== "object") {
      fm.spell_slots_used = {};
    }
    for (const level of Object.keys(maxSlots ?? {})) {
      fm.spell_slots_used[String(level)] = 0;
    }

    // Reset death saves if these fields are used later.
    if ("death_save_successes" in fm) fm.death_save_successes = 0;
    if ("death_save_failures" in fm) fm.death_save_failures = 0;

    // 2014-style Hit Dice recovery: at least one, up to half the total dice.
    if ("hit_dice_used" in fm) {
      const used = Math.max(0, Number(fm.hit_dice_used ?? 0));
      const recovered = Math.max(1, Math.floor(characterLevel / 2));
      fm.hit_dice_used = Math.max(0, used - recovered);
    }
  });

  new Notice(isLongRest ? "Long Rest abgeschlossen." : "Short Rest abgeschlossen.");
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

  // Inline actions from Species.
  const speciesPage = getCharacterSpeciesPage();
  if (speciesPage) {
    const speciesActions = Array.isArray(speciesPage.actions) ? speciesPage.actions : [];
    for (const action of speciesActions) {
      if (!action || typeof action !== "object") continue;
      const activation = String(action.activation ?? action.action_type ?? "other")
        .trim().toLowerCase().replace(/\s+/g, "_");
      const actionType = ["action","bonus_action","reaction"].includes(activation) ? activation : "other";

      collectedActions.push({
        ...action,
        action_type: actionType,
        source_type: "species",
        source_label: speciesPage.name ?? speciesPage.file?.name ?? "Species",
        file: {
          name: action.name ?? speciesPage.name ?? "Species Action",
          path: speciesPage.file?.path ?? CHARACTER_PATH
        }
      });
    }
  }

  // Inline actions from Features and Feats.
  for (const featurePage of getAllCharacterFeatures()) {
    const inlineActions = Array.isArray(featurePage.actions) ? featurePage.actions : [];
    for (const action of inlineActions) {
      if (!action || typeof action !== "object") continue;

      const activation = String(action.activation ?? action.action_type ?? "other")
        .trim().toLowerCase().replace(/\s+/g, "_");
      const actionType = ["action", "bonus_action", "reaction"].includes(activation)
        ? activation
        : "other";
      const sourceType = String(featurePage.type ?? "").trim().toLowerCase() === "feat"
        ? "feat" : "feature";

      collectedActions.push({
        ...action,
        action_type: actionType,
        source_type: sourceType,
        source_label: featurePage.name ?? featurePage.file?.name ?? "Feature",
        source_feature_name: featurePage.name ?? featurePage.file?.name ?? "Feature",
        source_feature_path: featurePage.file?.path ?? null,
        file: {
          name: action.name ?? featurePage.name ?? featurePage.file?.name ?? "Unnamed Action",
          path: featurePage.file?.path ?? CHARACTER_PATH
        }
      });
    }
  }

  return uniqueActionObjects(collectedActions);
}

function getAllCharacterSpells() {
  const collectedSpells = [];

  function addSpellRef(spellRef, sourceType="character", sourceLabel="Prepared Spell") {
    const spellPage = resolvePageRef(spellRef, SPELLS_FOLDER);
    if (!spellPage) return;

    const category = String(spellPage.category ?? "").trim().toLowerCase();
    if (category && category !== "spell") return;

    collectedSpells.push({
      ...spellPage,
      source_type: sourceType,
      source_label: sourceLabel,
      file: spellPage.file
    });
  }

  // New builder model:
  // - prepared_spells: leveled spells currently prepared
  // - known_spells: spell access/selection; cantrips are always usable once selected
  const preparedRefs = toPlainArray(c.prepared_spells);
  const knownRefs = toPlainArray(c.known_spells);

  for (const spellRef of preparedRefs) {
    addSpellRef(spellRef, "character", "Prepared");
  }

  for (const spellRef of knownRefs) {
    const spellPage = resolvePageRef(spellRef, SPELLS_FOLDER);
    if (!spellPage || Number(spellPage.level ?? 0) !== 0) continue;
    addSpellRef(spellRef, "character", "Known Cantrip");
  }

  // Legacy compatibility: old characters may still use `spells`.
  const explicitSpellRefs = toPlainArray(c.spells);
  for (const spellRef of explicitSpellRefs) {
    addSpellRef(spellRef, "character", "Character Sheet");
  }

  // Item-granted spells remain available independently of preparation.
  const inventoryEntries = Array.isArray(c.inventory) ? c.inventory : [];
  for (const entry of inventoryEntries) {
    const itemPath = resolvePathRef(entry?.item, ITEMS_FOLDER);
    if (!itemPath) continue;

    const itemPage = dv.page(itemPath);
    if (!itemPage) continue;

    const itemActions = Array.isArray(itemPage.actions) ? itemPage.actions : [];
    for (const action of itemActions) {
      if (!action || typeof action !== "object") continue;
      if (String(action.category ?? "").trim().toLowerCase() !== "spell") continue;

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

  container.createEl("pre", { text: md });
}

if (!c) {
  dv.paragraph("Character-Datei nicht gefunden.");
} else {
  const wrapper = dv.el("div", "");
  wrapper.style.display = "flex";
  wrapper.style.flexDirection = "column";
  wrapper.style.gap = "18px";

  const restBar = wrapper.createEl("div");
  restBar.style.display = "flex";
  restBar.style.justifyContent = "space-between";
  restBar.style.flexWrap = "wrap";
  restBar.style.alignItems = "center";
  restBar.style.gap = "8px";
  restBar.style.padding = "8px 10px";
  restBar.style.border = "1px solid var(--background-modifier-border)";
  restBar.style.borderRadius = "12px";

  const characterGroup = restBar.createEl("div");
  characterGroup.style.display = "flex";
  characterGroup.style.alignItems = "center";
  characterGroup.style.gap = "8px";
  characterGroup.style.flexWrap = "wrap";

  const restGroup = restBar.createEl("div");
  restGroup.style.display = "flex";
  restGroup.style.alignItems = "center";
  restGroup.style.gap = "8px";
  restGroup.style.flexWrap = "wrap";
  restGroup.style.marginLeft = "auto";

  const characterLabel = characterGroup.createEl("span", { text: "Character" });
  characterLabel.style.fontWeight = "700";

  const characterSelect = characterGroup.createEl("select");
  characterSelect.style.minWidth = "220px";
  characterSelect.style.maxWidth = "360px";
  characterSelect.style.padding = "6px 8px";
  characterSelect.style.borderRadius = "8px";

  for (const file of characterFiles) {
    const page = dv.page(file.path);
    const option = characterSelect.createEl("option", {
      text: String(page?.name ?? file.basename)
    });
    option.value = file.path;
  }
  characterSelect.value = CHARACTER_PATH;

  characterSelect.addEventListener("change", async () => {
    const selectedPath = String(characterSelect.value ?? "").trim();
    if (!selectedPath || selectedPath === CHARACTER_PATH) return;

    const sheetFile = app.workspace.getActiveFile();
    if (!sheetFile) {
      new Notice("Sheet-Datei konnte nicht ermittelt werden.");
      return;
    }

    characterSelect.disabled = true;
    try {
      // Persist the selection in this sheet's frontmatter. Dataview watches
      // metadata changes and will re-evaluate this block with the new path.
      await app.fileManager.processFrontMatter(sheetFile, fm => {
        fm.selected_character = selectedPath;
      });

      CHARACTER_PATH = selectedPath;

      // Explicit refresh as an immediate fallback in addition to Dataview's
      // metadata watcher.
      app.workspace.trigger("dataview:refresh-views");
    } finally {
      characterSelect.disabled = false;
    }
  });

  const editCharacterButton = characterGroup.createEl("button", { text: "Edit Character" });
  editCharacterButton.addEventListener("click", async () => {
    const selectedPath = String(characterSelect.value ?? CHARACTER_PATH).trim();
    if (!selectedPath) return;

    // The builder already reads this shared state on startup.
    window.__dndBuilderCharacterPath = selectedPath;

    const builderFile = app.vault.getAbstractFileByPath(BUILDER_PATH);
    if (!builderFile) {
      new Notice(`Builder nicht gefunden: ${BUILDER_PATH}`);
      return;
    }

    await app.workspace.getLeaf(false).openFile(builderFile);
  });

  const restLabel = restGroup.createEl("span", { text: "Rest" });
  restLabel.style.fontWeight = "700";
  restLabel.style.marginRight = "4px";

  const shortRestButton = restGroup.createEl("button", { text: "Short Rest" });
  shortRestButton.addEventListener("click", async () => {
    shortRestButton.disabled = true;
    try {
      await performRest("short_rest");
    } finally {
      shortRestButton.disabled = false;
    }
  });

  const longRestButton = restGroup.createEl("button", { text: "Long Rest" });
  longRestButton.addEventListener("click", async () => {
    longRestButton.disabled = true;
    try {
      await performRest("long_rest");
    } finally {
      longRestButton.disabled = false;
    }
  });

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

  const backgroundPageForHeader = getBackgroundPage();
  const backgroundDisplayName = String(
    backgroundPageForHeader?.name ??
    backgroundPageForHeader?.file?.name ??
    String(c.background ?? "").split("/").pop()?.replace(/\.md$/i, "") ??
    "-"
  );
  addMetaInfoRow(meta, "Background", backgroundDisplayName);
  addMetaInfoRow(meta, "Alignment", c.alignment ?? "-");
  addMetaInfoRow(meta, "Player", c.player ?? "-");

  const portraitWrap = header.createEl("div");
  portraitWrap.style.display = "flex";
  portraitWrap.style.flexDirection = "column";
  portraitWrap.style.justifyContent = "center";
  portraitWrap.style.alignItems = "center";
  portraitWrap.style.gap = "8px";

  const portraitPath = String(c.portrait ?? "").trim();
  const portraitFile = app.vault.getAbstractFileByPath(portraitPath);
  const portraitUrl = portraitPath ? app.vault.adapter.getResourcePath(portraitPath) : "";

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
    ["STR", isSaveProficient("str")],
    ["DEX", isSaveProficient("dex")],
    ["CON", isSaveProficient("con")],
    ["INT", isSaveProficient("int")],
    ["WIS", isSaveProficient("wis")],
    ["CHA", isSaveProficient("cha")],
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
    ["Acrobatics", "dex", isSkillProficient("acrobatics")],
    ["Animal Handling", "wis", isSkillProficient("animal_handling")],
    ["Arcana", "int", isSkillProficient("arcana")],
    ["Athletics", "str", isSkillProficient("athletics")],
    ["Deception", "cha", isSkillProficient("deception")],
    ["History", "int", isSkillProficient("history")],
    ["Insight", "wis", isSkillProficient("insight")],
    ["Intimidation", "cha", isSkillProficient("intimidation")],
    ["Investigation", "int", isSkillProficient("investigation")],
    ["Medicine", "wis", isSkillProficient("medicine")],
    ["Nature", "int", isSkillProficient("nature")],
    ["Perception", "wis", isSkillProficient("perception")],
    ["Performance", "cha", isSkillProficient("performance")],
    ["Persuasion", "cha", isSkillProficient("persuasion")],
    ["Religion", "int", isSkillProficient("religion")],
    ["Sleight of Hand", "dex", isSkillProficient("sleight_of_hand")],
    ["Stealth", "dex", isSkillProficient("stealth")],
    ["Survival", "wis", isSkillProficient("survival")],
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

        const resourceDef = getFeatureResourceDefinition(featurePage);
        if (resourceDef && resourceDef.display === "checkboxes") {
          const resourceWrap = content.createEl("div");
          resourceWrap.style.marginTop = "12px";
          resourceWrap.style.padding = "10px";
          resourceWrap.style.border = "1px solid var(--background-modifier-border)";
          resourceWrap.style.borderRadius = "8px";

          const resourceHeader = resourceWrap.createEl("div");
          resourceHeader.style.display = "flex";
          resourceHeader.style.justifyContent = "space-between";
          resourceHeader.style.alignItems = "center";
          resourceHeader.style.gap = "10px";

          const resourceLabel = resourceHeader.createEl("div", { text: resourceDef.label });
          resourceLabel.style.fontWeight = "600";

          let used = Math.min(resourceDef.max, getFeatureResourceUsed(resourceDef.id));
          const counter = resourceHeader.createEl("div", {
            text: `${used} / ${resourceDef.max} used`
          });
          counter.style.opacity = "0.75";
          counter.style.fontSize = "0.9em";

          const checks = resourceWrap.createEl("div");
          checks.style.display = "flex";
          checks.style.flexWrap = "wrap";
          checks.style.gap = "8px";
          checks.style.marginTop = "8px";

          for (let i = 0; i < resourceDef.max; i++) {
            const label = checks.createEl("label");
            label.style.display = "inline-flex";
            label.style.alignItems = "center";
            label.style.cursor = "pointer";

            const checkbox = label.createEl("input");
            checkbox.type = "checkbox";
            checkbox.checked = i < used;
            checkbox.title = `Use ${i + 1}`;

            checkbox.addEventListener("change", async () => {
              const allChecks = Array.from(checks.querySelectorAll('input[type="checkbox"]'));
              const clickedIndex = allChecks.indexOf(checkbox);
              const nextUsed = checkbox.checked ? clickedIndex + 1 : clickedIndex;

              allChecks.forEach((cb, index) => {
                cb.checked = index < nextUsed;
              });

              used = nextUsed;
              counter.textContent = `${used} / ${resourceDef.max} used`;
              await setFeatureResourceUsed(resourceDef.id, used, resourceDef.max);
            });
          }

          if (resourceDef.recharge) {
            const rechargeText = resourceDef.recharge === "long_rest"
              ? "Long Rest"
              : resourceDef.recharge === "short_rest"
                ? "Short Rest"
                : resourceDef.recharge;
            const rechargeLine = resourceWrap.createEl("div", { text: `Recharges: ${rechargeText}` });
            rechargeLine.style.marginTop = "8px";
            rechargeLine.style.fontSize = "0.85em";
            rechargeLine.style.opacity = "0.7";
          }
        }

        const structuredEffects = Array.isArray(featurePage.effects) ? featurePage.effects : [];
        if (structuredEffects.length > 0) {
          const effectsTitle = content.createEl("div", { text: "Effects" });
          effectsTitle.style.fontWeight = "600";
          effectsTitle.style.marginTop = "12px";
          effectsTitle.style.marginBottom = "6px";

          for (const effect of structuredEffects) {
            if (!effect || typeof effect !== "object") continue;
            const effectCard = content.createEl("div");
            effectCard.style.padding = "8px";
            effectCard.style.marginBottom = "6px";
            effectCard.style.border = "1px solid var(--background-modifier-border)";
            effectCard.style.borderRadius = "8px";

            const headingText = String(effect.name ?? effect.id ?? effect.type ?? "Effect").replace(/_/g, " ");
            const heading = effectCard.createEl("div", { text: headingText });
            heading.style.fontWeight = "600";

            const parts = [];
            if (effect.type) parts.push(`Type: ${String(effect.type).replace(/_/g, " ")}`);
            if (effect.target) parts.push(`Target: ${effect.target}`);
            if (Array.isArray(effect.targets)) parts.push(`Targets: ${effect.targets.join(", ")}`);
            if (effect.value != null) parts.push(`Value: ${effect.value}`);
            if (effect.condition) parts.push(`Condition: ${effect.condition}`);
            if (effect.scope) parts.push(`Scope: ${effect.scope}`);
            if (effect.damage) parts.push(`Damage: ${Array.isArray(effect.damage) ? effect.damage.join(", ") : effect.damage}`);

            if (parts.length) {
              const info = effectCard.createEl("div", { text: parts.join(" • ") });
              info.style.fontSize = "0.9em";
              info.style.opacity = "0.8";
              info.style.marginTop = "3px";
            }
          }
        }

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
      summary.createEl("div", { text: Number(rowWeight).toFixed(2) });

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

      const itemResourceDef = getItemResourceDefinition(item.itemPage);
      if (itemResourceDef && itemResourceDef.display === "checkboxes") {
        for (let quantityIndex = 0; quantityIndex < item.quantity; quantityIndex++) {
          const instanceId = makeInventoryInstanceId(item.itemPath, item.inventoryIndex, quantityIndex);
          const resourceWrap = content.createEl("div");
          resourceWrap.style.marginTop = "12px";
          resourceWrap.style.padding = "10px";
          resourceWrap.style.border = "1px solid var(--background-modifier-border)";
          resourceWrap.style.borderRadius = "8px";

          const resourceHeader = resourceWrap.createEl("div");
          resourceHeader.style.display = "flex";
          resourceHeader.style.justifyContent = "space-between";
          resourceHeader.style.alignItems = "center";
          resourceHeader.style.gap = "10px";

          const instanceLabel = item.quantity > 1 ? `${itemResourceDef.label} #${quantityIndex + 1}` : itemResourceDef.label;
          const resourceLabel = resourceHeader.createEl("div", { text: instanceLabel });
          resourceLabel.style.fontWeight = "600";

          let used = Math.min(itemResourceDef.max, getItemResourceUsed(instanceId, itemResourceDef.id));
          const counter = resourceHeader.createEl("div", { text: `${used} / ${itemResourceDef.max} used` });
          counter.style.fontSize = "0.85em";
          counter.style.opacity = "0.75";

          const checks = resourceWrap.createEl("div");
          checks.style.display = "flex";
          checks.style.flexWrap = "wrap";
          checks.style.gap = "8px";
          checks.style.marginTop = "8px";

          const checkboxes = [];
          for (let i = 0; i < itemResourceDef.max; i++) {
            const checkbox = checks.createEl("input");
            checkbox.type = "checkbox";
            checkbox.checked = i < used;
            checkbox.title = `Use ${i + 1}`;
            checkbox.style.cursor = "pointer";
            checkboxes.push(checkbox);

            checkbox.addEventListener("change", async () => {
              used = checkbox.checked ? i + 1 : i;
              checkboxes.forEach((box, index) => box.checked = index < used);
              counter.textContent = `${used} / ${itemResourceDef.max} used`;
              await setItemResourceUsed(instanceId, itemResourceDef.id, used, itemResourceDef.max);
            });
          }

          if (itemResourceDef.recharge) {
            const rechargeText = itemResourceDef.recharge === "long_rest" ? "Long Rest"
              : itemResourceDef.recharge === "short_rest" ? "Short Rest"
              : itemResourceDef.recharge === "dawn" ? "Dawn"
              : itemResourceDef.recharge;
            const suffix = itemResourceDef.rechargeAmount ? ` (${itemResourceDef.rechargeAmount})` : "";
            const rechargeLine = resourceWrap.createEl("div", { text: `Recharges: ${rechargeText}${suffix}` });
            rechargeLine.style.fontSize = "0.8em";
            rechargeLine.style.opacity = "0.7";
            rechargeLine.style.marginTop = "8px";
          }

          if (itemResourceDef.recharge === "dawn" && itemResourceDef.rechargeAmount) {
            const rechargeBtn = resourceWrap.createEl("button", { text: `Recharge ${itemResourceDef.rechargeAmount}` });
            rechargeBtn.style.marginTop = "8px";
            rechargeBtn.addEventListener("click", async (evt) => {
              evt.preventDefault();
              evt.stopPropagation();
              const recovered = rollSimpleDiceFormula(itemResourceDef.rechargeAmount);
              if (recovered == null) {
                new Notice(`Recharge-Formel nicht unterstützt: ${itemResourceDef.rechargeAmount}`);
                return;
              }
              used = Math.max(0, used - recovered);
              checkboxes.forEach((box, index) => box.checked = index < used);
              counter.textContent = `${used} / ${itemResourceDef.max} used`;
              await setItemResourceUsed(instanceId, itemResourceDef.id, used, itemResourceDef.max);
              new Notice(`${item.name}: ${recovered} Charge(s) recovered.`);
            });
          }
        }
      }

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
              : (action.source_type === "feat" || action.source_type === "feature" || action.source_type === "species")
                ? `From ${action.source_label ?? action.source_feature_name ?? "Feature"}`
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

          const damageEl = summaryEl.createEl("div", { text: damageTextA });
          damageEl.style.opacity = "0.7";
          damageEl.style.fontSize = "0.9em";
          damageEl.style.fontWeight = "400";

          const card = details.createEl("div");
          card.style.padding = "10px";
          card.style.borderTop = "1px solid var(--background-modifier-border)";

          const infoBlock = card.createEl("div");

          addInfoRow(infoBlock, "Type", sectionLabel(type));

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
          if (action.condition != null) addInfoRow(infoBlock, "Condition", action.condition);
          if (action.requires != null) addInfoRow(infoBlock, "Requires", action.requires);
          if (action.resource != null) addInfoRow(infoBlock, "Resource", action.resource);

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

    const spellCharacterFile = app.vault.getAbstractFileByPath(CHARACTER_PATH);

    async function saveSpellSlotsUsed(level, usedCount, maxCount) {
      if (!spellCharacterFile) return;

      const safeUsed = Math.max(0, Math.min(maxCount, Number(usedCount ?? 0)));

      await app.fileManager.processFrontMatter(spellCharacterFile, (fm) => {
        if (!fm.spell_slots_used || typeof fm.spell_slots_used !== "object") {
          fm.spell_slots_used = {};
        }
        fm.spell_slots_used[String(level)] = safeUsed;
      });
    }

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

    function getUsedSpellSlots(level, slotCount) {
      const raw = Number(c.spell_slots_used?.[String(level)] ?? 0);
      if (!Number.isFinite(raw)) return 0;
      return Math.max(0, Math.min(slotCount, raw));
    }

    async function addSpellSlotCheckboxes(parent, level, slotCount) {
      if (level === 0 || slotCount <= 0) return;

      let usedSlots = getUsedSpellSlots(level, slotCount);

      const slotWrapper = parent.createEl("div");
      slotWrapper.style.display = "flex";
      slotWrapper.style.alignItems = "center";
      slotWrapper.style.gap = "8px";
      slotWrapper.style.flexWrap = "wrap";
      slotWrapper.style.marginLeft = "auto";

      const label = slotWrapper.createEl("div", {
        text: `Slots ${usedSlots}/${slotCount}:`
      });
      label.style.fontSize = "0.9em";
      label.style.opacity = "0.75";

      const boxes = [];

      function refreshSlotVisuals() {
        label.setText(`Slots ${usedSlots}/${slotCount}:`);
        for (let i = 0; i < boxes.length; i++) {
          boxes[i].checked = i < usedSlots;
        }
      }

      for (let i = 0; i < slotCount; i++) {
        const boxLabel = slotWrapper.createEl("label");
        boxLabel.style.display = "flex";
        boxLabel.style.alignItems = "center";
        boxLabel.style.gap = "4px";
        boxLabel.style.cursor = "pointer";

        const checkbox = boxLabel.createEl("input");
        checkbox.type = "checkbox";
        checkbox.checked = i < usedSlots;
        boxes.push(checkbox);

        checkbox.addEventListener("change", async () => {
          if (checkbox.checked) {
            usedSlots = Math.max(usedSlots, i + 1);
          } else {
            usedSlots = i;
          }

          usedSlots = Math.max(0, Math.min(slotCount, usedSlots));
          refreshSlotVisuals();
          await saveSpellSlotsUsed(level, usedSlots, slotCount);
        });
      }

      refreshSlotVisuals();
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

        const sectionHeader = section.createEl("div");
        sectionHeader.style.display = "flex";
        sectionHeader.style.alignItems = "center";
        sectionHeader.style.justifyContent = "space-between";
        sectionHeader.style.gap = "12px";
        sectionHeader.style.flexWrap = "wrap";
        sectionHeader.style.marginBottom = "8px";
        sectionHeader.style.paddingBottom = "4px";
        sectionHeader.style.borderBottom = "1px solid var(--background-modifier-border)";

        const sectionTitle = sectionHeader.createEl("div", { text: levelLabel(level) });
        sectionTitle.style.fontWeight = "700";
        sectionTitle.style.fontSize = "1.05em";

        const maxSpellSlots = getMaxSpellSlots();
        const slotCount = Number(maxSpellSlots?.[String(level)] ?? maxSpellSlots?.[level] ?? 0);
        await addSpellSlotCheckboxes(sectionHeader, level, slotCount);

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
          addInfoRow(infoBlock, "Concentration", spell.concentration ? "Yes" : "No");

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

    const attunementSlots = getAttunementSlots();
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

  const passivePerception = 10 + getSkillTotal("Perception", "wis", isSkillProficient("perception"));
  const passiveInvestigation = 10 + getSkillTotal("Investigation", "int", isSkillProficient("investigation"));
  const passiveInsight = 10 + getSkillTotal("Insight", "wis", isSkillProficient("insight"));

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

  const profCard = leftCol.createEl("div");
  profCard.style.padding = "16px";
  profCard.style.border = "1px solid var(--background-modifier-border)";
  profCard.style.borderRadius = "14px";

  const profTitle = profCard.createEl("div", { text: "Proficiencies" });
  profTitle.style.fontWeight = "700";
  profTitle.style.marginBottom = "12px";
  profTitle.style.fontSize = "1.1em";

  const allProfs = getAllProficienciesByCategory();
  const profSections = [
    ["Armor", "armor"],
    ["Weapons", "weapons"],
    ["Tools", "tools"],
    ["Languages", "languages"]
  ];

  for (const [label, key] of profSections) {
    const values = [...allProfs[key]]
      .map(proficiencyDisplayName)
      .filter(Boolean)
      .sort((a,b)=>a.localeCompare(b,"de"));

    if (values.length === 0) continue;

    const section = profCard.createEl("div");
    section.style.marginBottom = "10px";

    const labelEl = section.createEl("div", { text: label });
    labelEl.style.fontWeight = "600";
    labelEl.style.fontSize = ".85em";
    labelEl.style.opacity = ".7";
    labelEl.style.marginBottom = "3px";

    const valueEl = section.createEl("div", { text: values.join(", ") });
    valueEl.style.lineHeight = "1.45";
  }
}
