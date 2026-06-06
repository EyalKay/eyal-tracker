# TDEE + Meal Templates + Lifetime Stats — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three options to the tracker: a Settings page with TDEE calculator, editable meal templates with one-tap apply, and a lifetime data view on the Stats page.

**Architecture:** All changes live in the single-file vanilla-JS app (`index.html`). New state fields (`mealTemplates`, `tdee`) are added to the existing Firestore `eyal/config` doc — additive, no migration. A single `saveConfig()` is the only writer of the config doc to prevent partial-write data loss. A new Settings page is rendered as a `.page` div, navigated to via a gear icon in the Today header (no bottom-nav change).

**Tech Stack:** Vanilla HTML / CSS / JS. Firebase Firestore (modular SDK already loaded). No build step, no test framework. Deployed via GitHub Pages from `main`. Manual verification on the live site is the test harness.

**Spec:** `docs/superpowers/specs/2026-06-06-tdee-templates-lifetime-design.md`

---

## File map

Only `index.html` is modified. New sections within it:
- **Styles** (in existing `<style>` block): `.gear-btn`, `.tpl-chip-row`, `.tpl-chip`, `.tpl-row`, `.tdee-inputs`, `.tdee-out`, `.range-toggle`, `.all-time-card`
- **Markup** (after existing `#page-stats`, before `</nav>`): new `<div class="page" id="page-settings">`
- **Markup** (inside `#page-today .page-header`): gear button next to sync dot
- **Markup** (inside `#page-stats`): all-time card + range toggle
- **Markup** (inside `#page-today` after `.macros-row`): template chip row
- **JS**: new state (`mealTemplates`, `tdee`), new functions (`renderSettings`, `renderTemplates`, `addTemplate`, `editTemplate`, `deleteTemplate`, `applyTemplate`, `renderTdee`, `computeTdee`, `loadAllRange`, `renderAllTime`, `setStatsRange`)
- **JS modifications**: `saveConfig()` writes full config; `loadToday()` also reads new fields; `goPage()` handles `'settings'`; `renderStats()` accepts a range argument

---

## Order rationale

Task 1 (config safety) must be first — every subsequent task writes to the config doc, and without the merge fix they will silently wipe each other's data. Tasks 2–7 are independent of each other once Task 1 is in place; ordered below to keep navigation testable before features land.

---

### Task 1: Make `saveConfig` the single source of config writes

**Files:**
- Modify: `index.html:553-608` (state declarations + load/save functions)

The current `saveConfig()` writes only `{habits, goal, startWeight}`. As soon as Tasks 4–6 add `mealTemplates` / `tdee` to the doc, the next call to `saveConfig()` will wipe them. Fix: add the new state vars, load them, and have `saveConfig()` write the full set.

- [ ] **Step 1: Add new state declarations**

Locate `index.html:553-557`:
```js
let todayData = {p:0,c:0,f:0,weight:null,triggers:{},habits:{},note:''};
let customHabits = [];
let goalWeight = null;
let startingWeight = 87;
let isSaving = false;
let selectedDate = new Date(); // the date being viewed/edited
```

Add two new state vars immediately after `let startingWeight = 87;`:
```js
let mealTemplates = [];
let tdeeProfile = null; // {height, age, sex, activity} or null when unset
```

- [ ] **Step 2: Read new fields in `loadToday()`**

Locate `index.html:569-576`:
```js
    // load habits config
    const hSnap = await getDoc(doc(db, USER, 'config'));
    if(hSnap.exists()){
      const cfg = hSnap.data();
      if(cfg.habits) customHabits = cfg.habits;
      if(cfg.goal) goalWeight = cfg.goal;
      if(cfg.startWeight) startingWeight = cfg.startWeight;
    }
```

Replace with:
```js
    // load config (habits, goal, startWeight, mealTemplates, tdeeProfile)
    const hSnap = await getDoc(doc(db, USER, 'config'));
    if(hSnap.exists()){
      const cfg = hSnap.data();
      if(cfg.habits) customHabits = cfg.habits;
      if(cfg.goal) goalWeight = cfg.goal;
      if(cfg.startWeight) startingWeight = cfg.startWeight;
      if(Array.isArray(cfg.mealTemplates)) mealTemplates = cfg.mealTemplates;
      if(cfg.tdee && typeof cfg.tdee === 'object') tdeeProfile = cfg.tdee;
    }
```

- [ ] **Step 3: Update `saveConfig()` to write the full set**

Locate `index.html:604-608`:
```js
async function saveConfig(){
  try{
    await setDoc(doc(db, USER, 'config'), {habits: customHabits, goal: goalWeight||null, startWeight: startingWeight});
  }catch(e){}
}
```

Replace with:
```js
async function saveConfig(){
  try{
    await setDoc(doc(db, USER, 'config'), {
      habits: customHabits,
      goal: goalWeight||null,
      startWeight: startingWeight,
      mealTemplates: mealTemplates,
      tdee: tdeeProfile
    });
  }catch(e){}
}
```

- [ ] **Step 4: Reset new state on full reset**

Locate `index.html:944-946`:
```js
    customHabits = [];
    goalWeight = null;
    startingWeight = 87;
```

Add immediately after `startingWeight = 87;`:
```js
    mealTemplates = [];
    tdeeProfile = null;
```

- [ ] **Step 5: Verify in browser**

Push, hard-refresh live site, sign in. Confirm:
- Today page still renders all macros, weight, habits as before
- DevTools console has no errors
- Open DevTools → Application → IndexedDB → firebase → check the `firestore-clients` if visible (or just open the Firebase Console → Firestore → `eyal/config`) and confirm the doc now also contains `mealTemplates: []` and `tdee: null` after any save action (e.g. tap a `+` to trigger any habit/goal change). No existing data lost.

- [ ] **Step 6: Commit**

```powershell
git add index.html
git commit -m "Centralize config writes through single saveConfig"
git push origin main
```

---

### Task 2: Add Settings page skeleton + gear icon nav

**Files:**
- Modify: `index.html` (CSS block, Today header markup, new `#page-settings` markup, `goPage` JS)

- [ ] **Step 1: Add gear button styles**

Locate the `/* HEADER */` block around `index.html:28-35`. After the existing `.sync-dot` rules, add:
```css
.gear-btn{background:none;border:none;color:var(--muted);font-size:20px;cursor:pointer;padding:4px;margin-top:0;transition:color .2s}
.gear-btn:hover{color:var(--text)}
.back-arrow{background:none;border:none;color:var(--p);font-size:18px;cursor:pointer;padding:4px 8px 4px 0;font-family:inherit}
.settings-section{margin-bottom:24px}
.settings-section-title{font-size:11px;color:var(--muted);text-transform:uppercase;letter-spacing:.1em;margin-bottom:10px;padding-left:4px}
```

- [ ] **Step 2: Add gear icon to Today header**

Locate `index.html:237`:
```html
    <div class="sync-dot" id="syncDot" title="Sync"></div>
```

Replace with:
```html
    <div style="display:flex;align-items:center;gap:8px;margin-top:4px">
      <div class="sync-dot" id="syncDot" title="Sync"></div>
      <button class="gear-btn" onclick="goPage('settings',null)" title="Settings">⚙️</button>
    </div>
```

- [ ] **Step 3: Add Settings page markup**

Locate the closing `</div>` of `#page-stats` at `index.html:503`, immediately followed by `<!-- BOTTOM NAV -->` at line 505. Insert this new page block between them:
```html
<div class="page" id="page-settings">
  <div class="page-header">
    <div>
      <button class="back-arrow" onclick="goPage('today',document.querySelector('.nav-btn'))">◀ Back</button>
      <div class="page-title">Settings ⚙️</div>
    </div>
  </div>

  <!-- TDEE section will be added in Task 5 -->
  <div class="settings-section" id="settingsTdeeSection"></div>

  <!-- Meal templates section will be added in Task 3 -->
  <div class="settings-section" id="settingsTemplatesSection"></div>
</div>
```

- [ ] **Step 4: Update `goPage` to handle `'settings'`**

Locate `index.html:961-968`:
```js
window.goPage = function(name, btn){
  document.querySelectorAll('.page').forEach(p=>p.classList.remove('active'));
  document.getElementById('page-'+name).classList.add('active');
  document.querySelectorAll('.nav-btn').forEach(b=>b.classList.remove('active'));
  if(btn) btn.classList.add('active');
  if(name==='week') renderWeek();
  if(name==='stats') renderStats();
};
```

Add the settings-render call. Replace the function with:
```js
window.goPage = function(name, btn){
  document.querySelectorAll('.page').forEach(p=>p.classList.remove('active'));
  document.getElementById('page-'+name).classList.add('active');
  document.querySelectorAll('.nav-btn').forEach(b=>b.classList.remove('active'));
  if(btn) btn.classList.add('active');
  if(name==='week') renderWeek();
  if(name==='stats') renderStats();
  if(name==='settings') renderSettings();
  window.scrollTo(0,0);
};
```

- [ ] **Step 5: Add empty `renderSettings` shell**

Immediately after `window.goPage` (around line 970 after replacement), add:
```js
function renderSettings(){
  // Populated by Task 3 (templates) and Task 5 (TDEE)
  renderTemplatesUI && renderTemplatesUI();
  renderTdeeUI && renderTdeeUI();
}
```

- [ ] **Step 6: Verify in browser**

Push, hard-refresh, sign in. Confirm:
- Gear icon visible top-right of Today page next to sync dot
- Tapping gear navigates to Settings page with title "Settings ⚙️" and a "◀ Back" button
- Settings page is otherwise empty (two empty section divs)
- Tapping "◀ Back" returns to Today
- Bottom nav (Today/Week/Stats) still works
- No console errors

- [ ] **Step 7: Commit**

```powershell
git add index.html
git commit -m "Add Settings page skeleton + gear icon nav"
git push origin main
```

---

### Task 3: Meal Templates — management UI in Settings

**Files:**
- Modify: `index.html` (CSS, `#settingsTemplatesSection` markup builder, JS functions)

- [ ] **Step 1: Add template management styles**

Inside the existing `<style>` block (anywhere before `</style>` is fine — group near other card styles around `index.html:38`). Add:
```css
.tpl-list{display:flex;flex-direction:column;gap:8px}
.tpl-row{background:var(--card);border:1px solid var(--border);border-radius:12px;padding:12px;display:flex;align-items:center;gap:10px;flex-wrap:wrap}
.tpl-row input[type=text]{flex:1;min-width:120px;background:var(--card2);border:1px solid var(--border);color:var(--text);padding:8px 10px;border-radius:8px;font-family:inherit;font-size:14px}
.tpl-row .tpl-macro{display:flex;align-items:center;gap:4px;font-size:12px;font-weight:700}
.tpl-row .tpl-macro .lbl{color:var(--muted);font-size:10px;text-transform:uppercase}
.tpl-row .tpl-step{background:var(--card2);border:1px solid var(--border);color:var(--text);width:24px;height:24px;border-radius:6px;font-family:inherit;cursor:pointer;padding:0;line-height:1}
.tpl-row .tpl-val{min-width:24px;text-align:center}
.tpl-row .tpl-del{background:none;border:none;color:var(--red);font-size:18px;cursor:pointer;padding:4px 8px;margin-left:auto}
.tpl-add-btn{background:var(--card2);border:1px dashed var(--border);color:var(--p);padding:12px;border-radius:12px;font-family:inherit;font-size:13px;font-weight:700;cursor:pointer;width:100%}
.tpl-add-btn:hover{border-color:var(--p)}
.tpl-empty{color:var(--muted);font-size:13px;text-align:center;padding:16px}
```

- [ ] **Step 2: Add ID helper and template state mutators**

Place near other top-level functions (after `saveConfig` at `index.html:608` is a good spot):
```js
function newTplId(){ return 'tpl_' + Date.now() + '_' + Math.floor(Math.random()*1000); }

function addTemplate(){
  mealTemplates.push({ id: newTplId(), name: '', p: 0, c: 0, f: 0 });
  saveConfig();
  renderTemplatesUI();
  renderTemplateChips && renderTemplateChips();
}

function deleteTemplate(id){
  mealTemplates = mealTemplates.filter(t => t.id !== id);
  saveConfig();
  renderTemplatesUI();
  renderTemplateChips && renderTemplateChips();
}

function editTemplateName(id, name){
  const t = mealTemplates.find(x => x.id === id);
  if(!t) return;
  t.name = (name||'').slice(0, 30);
  saveConfig();
  renderTemplateChips && renderTemplateChips();
}

function editTemplateMacro(id, macro, delta){
  const t = mealTemplates.find(x => x.id === id);
  if(!t) return;
  const next = Math.max(0, Math.min(10, Math.round(((t[macro]||0) + delta) * 10) / 10));
  t[macro] = next;
  saveConfig();
  renderTemplatesUI();
  renderTemplateChips && renderTemplateChips();
}

window.addTemplate = addTemplate;
window.deleteTemplate = deleteTemplate;
window.editTemplateName = editTemplateName;
window.editTemplateMacro = editTemplateMacro;
```

- [ ] **Step 3: Add `renderTemplatesUI` (Settings page rendering)**

Place immediately after the mutators added in Step 2:
```js
function renderTemplatesUI(){
  const root = document.getElementById('settingsTemplatesSection');
  if(!root) return;
  const rows = mealTemplates.map(t => `
    <div class="tpl-row" data-id="${t.id}">
      <input type="text" maxlength="30" placeholder="Template name" value="${(t.name||'').replace(/"/g,'&quot;')}"
        oninput="editTemplateName('${t.id}', this.value)">
      <div class="tpl-macro" style="color:var(--p)">
        <span class="lbl">P</span>
        <button class="tpl-step" onclick="editTemplateMacro('${t.id}','p',-0.5)">−</button>
        <span class="tpl-val">${formatVal(t.p||0)}</span>
        <button class="tpl-step" onclick="editTemplateMacro('${t.id}','p',0.5)">+</button>
      </div>
      <div class="tpl-macro" style="color:var(--c)">
        <span class="lbl">C</span>
        <button class="tpl-step" onclick="editTemplateMacro('${t.id}','c',-0.5)">−</button>
        <span class="tpl-val">${formatVal(t.c||0)}</span>
        <button class="tpl-step" onclick="editTemplateMacro('${t.id}','c',0.5)">+</button>
      </div>
      <div class="tpl-macro" style="color:var(--f)">
        <span class="lbl">F</span>
        <button class="tpl-step" onclick="editTemplateMacro('${t.id}','f',-0.5)">−</button>
        <span class="tpl-val">${formatVal(t.f||0)}</span>
        <button class="tpl-step" onclick="editTemplateMacro('${t.id}','f',0.5)">+</button>
      </div>
      <button class="tpl-del" onclick="deleteTemplate('${t.id}')" title="Delete">🗑</button>
    </div>`).join('');
  const empty = mealTemplates.length === 0
    ? '<div class="tpl-empty">No templates yet. Add one below.</div>' : '';
  root.innerHTML = `
    <div class="settings-section-title">🍽 Meal Templates</div>
    <div class="tpl-list">${rows}${empty}</div>
    <button class="tpl-add-btn" onclick="addTemplate()" style="margin-top:10px">+ Add template</button>
  `;
}
```

- [ ] **Step 4: Verify in browser**

Push, hard-refresh, open Settings:
- Section "🍽 Meal Templates" appears with "No templates yet. Add one below." and an "+ Add template" button
- Tap "+ Add template" → an empty row appears with name input and three macro steppers (P/C/F)
- Type a name like "Usual breakfast" → reload Settings, name persists
- Tap `+` on P → value goes to 0.5, then 1, etc.
- Tap `−` → value decreases, can't go below 0
- Tap 🗑 → row disappears
- Open Firebase Console → `eyal/config` → confirm `mealTemplates` array reflects what you see (other fields like `habits`, `goal`, `startWeight` unchanged)

- [ ] **Step 5: Commit**

```powershell
git add index.html
git commit -m "Add meal templates management UI in Settings"
git push origin main
```

---

### Task 4: Meal Templates — chip row on Today + apply

**Files:**
- Modify: `index.html` (CSS, Today markup after `.macros-row`, JS apply + render functions)

- [ ] **Step 1: Add chip styles**

Inside the `<style>` block:
```css
.tpl-chip-row{display:flex;gap:8px;overflow-x:auto;padding:4px 0 12px;margin-bottom:8px;scroll-snap-type:x mandatory;-webkit-overflow-scrolling:touch}
.tpl-chip-row::-webkit-scrollbar{display:none}
.tpl-chip{flex:none;background:var(--card2);border:1px solid var(--border);color:var(--text);font-family:inherit;font-size:12px;font-weight:700;padding:8px 14px;border-radius:999px;cursor:pointer;scroll-snap-align:start;white-space:nowrap}
.tpl-chip:active{transform:scale(0.96)}
.tpl-chip .tpl-chip-macro{font-size:10px;color:var(--muted);font-weight:500;margin-left:4px}
.tpl-chip-empty{color:var(--muted);font-size:12px;padding:8px 4px;cursor:pointer}
.tpl-chip-empty:hover{color:var(--text)}
```

- [ ] **Step 2: Add chip-row markup under macros**

Find the closing `</div>` of `.macros-row` on the Today page (search for `<!-- MACROS -->` at `index.html:240`, then locate the matching close — it ends after the third `.macro-tile`). Immediately after that closing `</div>`, before the next markup block, add:
```html
  <!-- MEAL TEMPLATE CHIPS -->
  <div id="tplChips" class="tpl-chip-row"></div>
```

- [ ] **Step 3: Add `renderTemplateChips` and `applyTemplate`**

Place near the existing `window.chg` (around `index.html:810`):
```js
function renderTemplateChips(){
  const root = document.getElementById('tplChips');
  if(!root) return;
  if(!mealTemplates.length){
    root.innerHTML = '<div class="tpl-chip-empty" onclick="goPage(\'settings\',null)">+ Add meal templates in Settings →</div>';
    return;
  }
  root.innerHTML = mealTemplates.map(t => {
    const parts = [];
    if(t.p) parts.push(`<span style="color:var(--p)">${formatVal(t.p)}P</span>`);
    if(t.c) parts.push(`<span style="color:var(--c)">${formatVal(t.c)}C</span>`);
    if(t.f) parts.push(`<span style="color:var(--f)">${formatVal(t.f)}F</span>`);
    const macros = parts.length ? `<span class="tpl-chip-macro">${parts.join(' · ')}</span>` : '';
    const name = (t.name||'(unnamed)').replace(/</g,'&lt;');
    return `<button class="tpl-chip" onclick="applyTemplate('${t.id}')">${name}${macros}</button>`;
  }).join('');
}

window.applyTemplate = function(id){
  const t = mealTemplates.find(x => x.id === id);
  if(!t) return;
  ['p','c','f'].forEach(m => {
    const add = t[m] || 0;
    if(!add) return;
    const cap = TARGETS[m] + 3;
    todayData[m] = Math.max(0, Math.min(Math.round(((todayData[m]||0) + add) * 10) / 10, cap));
  });
  renderToday();
  saveToday();
  ['p','c','f'].forEach(m => {
    if(t[m]){
      const el = document.getElementById('v-'+m);
      el.classList.remove('pop'); void el.offsetWidth; el.classList.add('pop');
    }
  });
};
```

- [ ] **Step 4: Render chips on initial load and date change**

Find `renderToday()` (search for `function renderToday`). At the bottom of `renderToday` add a call to `renderTemplateChips()`. If you cannot quickly locate the end of `renderToday`, alternative: add a call at the end of `loadToday` after `renderHabits();` (line ~588).

Specifically — locate `index.html:586-589`:
```js
  renderToday();
  renderHabits();
}
```

Replace with:
```js
  renderToday();
  renderHabits();
  renderTemplateChips();
}
```

- [ ] **Step 5: Verify in browser**

Push, hard-refresh, sign in.
- With **no** templates: under the macro tiles you see "+ Add meal templates in Settings →" (clickable, goes to Settings).
- Go to Settings, add two templates: "Coffee + oats" (P:0 C:1 F:0) and "Steak dinner" (P:2 C:0 F:1). Back to Today.
- Chip row shows two chips with name + colored macro hints.
- Note current macro values. Tap "Coffee + oats" → C goes up by 1, the C tile pops, sync dot blinks. P and F unchanged.
- Tap "Steak dinner" → P +2, F +1.
- Refresh page — chips persist; today's values persist.
- Edit a template's name in Settings → return to Today → chip label updated.
- Delete a template in Settings → return to Today → chip gone.
- Sign out / sign in — templates still load.

- [ ] **Step 6: Commit**

```powershell
git add index.html
git commit -m "Add meal-template chips on Today with one-tap apply"
git push origin main
```

---

### Task 5: TDEE — inputs + readout in Settings

**Files:**
- Modify: `index.html` (CSS, `#settingsTdeeSection` builder, save/compute functions)

- [ ] **Step 1: Add TDEE styles**

Inside the `<style>` block:
```css
.tdee-card{background:var(--card);border:1px solid var(--border);border-radius:12px;padding:14px}
.tdee-inputs{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:14px}
.tdee-field{display:flex;flex-direction:column;gap:4px}
.tdee-field label{font-size:10px;color:var(--muted);text-transform:uppercase;letter-spacing:.08em}
.tdee-field input,.tdee-field select{background:var(--card2);border:1px solid var(--border);color:var(--text);padding:8px 10px;border-radius:8px;font-family:inherit;font-size:14px}
.tdee-sex{display:flex;gap:6px}
.tdee-sex button{flex:1;background:var(--card2);border:1px solid var(--border);color:var(--muted);padding:8px;border-radius:8px;font-family:inherit;font-size:13px;font-weight:700;cursor:pointer}
.tdee-sex button.active{background:var(--p);border-color:var(--p);color:#000}
.tdee-out{background:var(--card2);border:1px solid var(--border);border-radius:10px;padding:12px;font-size:13px;line-height:1.6}
.tdee-out .row{display:flex;justify-content:space-between}
.tdee-out .row.big{font-size:15px;font-weight:700;margin-top:6px}
.tdee-out .warn{margin-top:8px;color:var(--f);font-size:11px}
.tdee-out .err{color:var(--muted);font-style:italic}
</style>
```

(Place the `</style>` correctly — the closing tag is somewhere around `index.html:205-220`; just add these rules before whatever is the existing last rule.)

- [ ] **Step 2: Add TDEE state mutators and compute**

Near the other config mutators added in Task 3:
```js
function ensureTdee(){
  if(!tdeeProfile) tdeeProfile = { height:175, age:35, sex:'M', activity:'moderate' };
}

function setTdeeField(field, value){
  ensureTdee();
  if(field === 'height') tdeeProfile.height = Math.max(100, Math.min(250, parseInt(value)||0));
  else if(field === 'age') tdeeProfile.age = Math.max(10, Math.min(100, parseInt(value)||0));
  else if(field === 'sex') tdeeProfile.sex = (value === 'F') ? 'F' : 'M';
  else if(field === 'activity') tdeeProfile.activity = value;
  saveConfig();
  renderTdeeUI();
}
window.setTdeeField = setTdeeField;

const ACTIVITY_MULT = {
  sedentary: 1.2,
  light:     1.375,
  moderate:  1.55,
  active:    1.725,
  very:      1.9
};

async function mostRecentWeight(){
  // Walk back up to 60 days from today looking for the latest logged weight
  for(let i=0; i<60; i++){
    const d = new Date(); d.setDate(d.getDate()-i);
    const key = dateKey(d);
    try{
      const snap = await getDoc(doc(db, USER, key));
      if(snap.exists()){
        const data = snap.data();
        if(data && typeof data.weight === 'number' && data.weight > 0) return data.weight;
      }
    }catch(e){ /* ignore single-day fetch errors */ }
  }
  return null;
}

function computeTdee(weight){
  if(!tdeeProfile || !weight) return null;
  const { height, age, sex, activity } = tdeeProfile;
  if(!height || !age) return null;
  const bmr = (sex === 'F')
    ? 10*weight + 6.25*height - 5*age - 161
    : 10*weight + 6.25*height - 5*age + 5;
  const mult = ACTIVITY_MULT[activity] || ACTIVITY_MULT.moderate;
  const tdee = bmr * mult;
  return { bmr: Math.round(bmr), tdee: Math.round(tdee) };
}
```

- [ ] **Step 3: Add `renderTdeeUI`**

```js
async function renderTdeeUI(){
  const root = document.getElementById('settingsTdeeSection');
  if(!root) return;
  const p = tdeeProfile || { height:'', age:'', sex:'M', activity:'moderate' };
  root.innerHTML = `
    <div class="settings-section-title">📊 TDEE & Goal Check</div>
    <div class="tdee-card">
      <div class="tdee-inputs">
        <div class="tdee-field">
          <label>Height (cm)</label>
          <input type="number" min="100" max="250" value="${p.height||''}" oninput="setTdeeField('height', this.value)">
        </div>
        <div class="tdee-field">
          <label>Age</label>
          <input type="number" min="10" max="100" value="${p.age||''}" oninput="setTdeeField('age', this.value)">
        </div>
        <div class="tdee-field" style="grid-column:1 / span 2">
          <label>Sex</label>
          <div class="tdee-sex">
            <button class="${p.sex==='M'?'active':''}" onclick="setTdeeField('sex','M')">Male</button>
            <button class="${p.sex==='F'?'active':''}" onclick="setTdeeField('sex','F')">Female</button>
          </div>
        </div>
        <div class="tdee-field" style="grid-column:1 / span 2">
          <label>Activity level</label>
          <select onchange="setTdeeField('activity', this.value)">
            <option value="sedentary" ${p.activity==='sedentary'?'selected':''}>Sedentary (desk job)</option>
            <option value="light"     ${p.activity==='light'?'selected':''}>Light (1–3 days/wk)</option>
            <option value="moderate"  ${p.activity==='moderate'?'selected':''}>Moderate (3–5 days/wk)</option>
            <option value="active"    ${p.activity==='active'?'selected':''}>Active (6–7 days/wk)</option>
            <option value="very"      ${p.activity==='very'?'selected':''}>Very active (2x/day)</option>
          </select>
        </div>
      </div>
      <div class="tdee-out" id="tdeeOut"><div class="err">Loading…</div></div>
    </div>
  `;
  // Async fill of readout
  const weight = await mostRecentWeight();
  const out = document.getElementById('tdeeOut');
  if(!out) return; // user navigated away
  if(!tdeeProfile || !tdeeProfile.height || !tdeeProfile.age){
    out.innerHTML = '<div class="err">Enter height and age to see your TDEE.</div>';
    return;
  }
  if(!weight){
    out.innerHTML = '<div class="err">Log a weight on Today first to see your TDEE.</div>';
    return;
  }
  const r = computeTdee(weight);
  if(!r){ out.innerHTML = '<div class="err">Invalid inputs.</div>'; return; }
  const deficit = r.tdee - TARGET_CAL; // positive = deficit (target below TDEE)
  const isDeficit = deficit >= 0;
  const kgPerWeek = (Math.abs(deficit) * 7) / 7700;
  const sign = isDeficit ? '−' : '+';
  const label = isDeficit ? 'Deficit' : 'Surplus';
  const projColor = isDeficit ? 'var(--p)' : 'var(--red)';
  const warn = isDeficit && deficit > 1000
    ? `<div class="warn">⚠ Deficit > 1000 kcal/day is aggressive — consider reviewing with your dietitian.</div>` : '';
  out.innerHTML = `
    <div class="row"><span>BMR</span><span>${r.bmr} kcal</span></div>
    <div class="row"><span>TDEE</span><span>${r.tdee} kcal</span></div>
    <div class="row"><span>Target (1960)</span><span>${TARGET_CAL} kcal</span></div>
    <div class="row big"><span>${label}</span><span style="color:${projColor}">${sign}${Math.abs(deficit)} kcal/day</span></div>
    <div class="row" style="margin-top:4px"><span>Projected</span><span style="color:${projColor}">${sign}${kgPerWeek.toFixed(2)} kg/week</span></div>
    ${warn}
  `;
}
```

- [ ] **Step 4: Verify in browser**

Push, hard-refresh, open Settings:
- "📊 TDEE & Goal Check" section appears
- Initially: empty Height/Age fields, Male selected, Moderate selected, readout says "Enter height and age…"
- Enter Height: `175`, Age: `38` → readout populates: BMR/TDEE/Target/Deficit/Projected
- Tap Female → values recompute (BMR drops by 166)
- Change activity to "Sedentary" → TDEE drops
- Change activity to "Very active" → TDEE rises, deficit may turn into surplus (label flips, color flips)
- Set activity back to "Sedentary" and reduce height to 100 to force a tiny TDEE — if TDEE − 1960 < −1000 (i.e. target > TDEE+1000), no warning fires (warning is only for deficit). Then push age way up and pick Female + Sedentary to produce a deficit > 1000 → amber warning appears.
- Sign out / sign in → values persist
- Firebase Console → `eyal/config` → `tdee` field present with the values you set; `mealTemplates`, `habits`, `goal`, `startWeight` all still present

- [ ] **Step 5: Commit**

```powershell
git add index.html
git commit -m "Add TDEE calculator with target deficit readout in Settings"
git push origin main
```

---

### Task 6: Lifetime range — fetch + toggle UI

**Files:**
- Modify: `index.html` (CSS, Stats page markup, `loadAllDays` / `loadAllRange`, `renderStats` signature)

- [ ] **Step 1: Add range-toggle styles**

In the `<style>` block:
```css
.range-toggle{display:flex;gap:6px;margin:0 0 12px}
.range-toggle button{flex:1;background:var(--card2);border:1px solid var(--border);color:var(--muted);padding:8px;border-radius:8px;font-family:inherit;font-size:12px;font-weight:700;cursor:pointer}
.range-toggle button.active{background:var(--p);border-color:var(--p);color:#000}
```

- [ ] **Step 2: Add the toggle markup on Stats page**

Locate `index.html:451-457` (top of `#page-stats`):
```html
<div class="page" id="page-stats">
  <div class="page-header">
    <div>
      <div class="page-title">Statistics 🧠</div>
      <div class="page-sub">Last 30 days</div>
    </div>
  </div>
```

Change the `page-sub` text to be ID-driven and add the toggle. Replace those lines with:
```html
<div class="page" id="page-stats">
  <div class="page-header">
    <div>
      <div class="page-title">Statistics 🧠</div>
      <div class="page-sub" id="statsRangeLabel">Last 30 days</div>
    </div>
  </div>

  <div class="range-toggle" id="statsRangeToggle">
    <button class="active" data-range="30" onclick="setStatsRange(30)">30d</button>
    <button data-range="90" onclick="setStatsRange(90)">90d</button>
    <button data-range="all" onclick="setStatsRange('all')">All</button>
  </div>
```

- [ ] **Step 3: Add `loadAllRange` (uses collection query for "all")**

After the existing `loadAllDays` function (around `index.html:971-981`), add:
```js
async function loadAllRange(range){
  // range: number of days (e.g. 30, 90) OR 'all'
  if(range === 'all'){
    const result = {};
    try{
      const snap = await getDocs(collection(db, USER));
      snap.forEach(docSnap => {
        if(docSnap.id === 'config') return;
        // Doc IDs are YYYY-MM-DD strings; sanity-check
        if(/^\d{4}-\d{2}-\d{2}$/.test(docSnap.id)){
          result[docSnap.id] = docSnap.data();
        }
      });
    }catch(e){ console.error('loadAllRange all failed', e); }
    return result;
  }
  return loadAllDays(range);
}
```

- [ ] **Step 4: Refactor `renderStats` to take a range and add `setStatsRange`**

Locate `renderStats` at `index.html:1082-1083`:
```js
async function renderStats(){
  const data=await loadAllDays(30);
```

Add a module-level state var near the other stats-related state (just before `async function renderStats`) and refactor the signature:
```js
let statsRange = 30; // 30 | 90 | 'all'

async function renderStats(){
  const range = statsRange;
  const data = await loadAllRange(range);
```

And update the chart card title to reflect range. Locate `index.html:494`:
```html
    <div class="card-title">📈 Weight Trend — 30 Days</div>
```

Change to:
```html
    <div class="card-title" id="statsChartTitle">📈 Weight Trend — 30 Days</div>
```

Inside `renderStats` add this near the top after `const entries=...sort(...)`:
```js
  const rangeLabel = range === 'all' ? 'All time' : `Last ${range} days`;
  const chartLabel = range === 'all' ? 'All time' : `${range} Days`;
  document.getElementById('statsRangeLabel').textContent = rangeLabel;
  document.getElementById('statsChartTitle').textContent = `📈 Weight Trend — ${chartLabel}`;
  // The "Success Days / 30" tile label needs to reflect the range:
  const sLabel = document.querySelector('#s30days')?.parentElement?.querySelector('.stat-lbl');
  if(sLabel) sLabel.textContent = `Success Days / ${range === 'all' ? entries.filter(([_,e])=>!!e).length : range}`;
  const wLabel = document.querySelector('#sWeightDelta')?.parentElement?.querySelector('.stat-lbl');
  if(wLabel) wLabel.textContent = range === 'all' ? 'Lifetime Weight Change' : (range === 30 ? 'Monthly Weight Change' : `${range}d Weight Change`);
```

Add the setter (place at top level near other window functions, e.g. after `window.goPage`):
```js
window.setStatsRange = function(r){
  statsRange = (r === 'all') ? 'all' : parseInt(r);
  document.querySelectorAll('#statsRangeToggle button').forEach(b=>{
    const v = b.dataset.range;
    b.classList.toggle('active', (v === 'all' && statsRange === 'all') || (parseInt(v) === statsRange));
  });
  renderStats();
};
```

- [ ] **Step 5: Verify in browser**

Push, hard-refresh, sign in, go to Stats:
- Sub-title shows "Last 30 days"; "30d" button is active
- Tap "90d" → sub-title becomes "Last 90 days"; chart title becomes "📈 Weight Trend — 90 Days"; "Success Days / 90" label updates; chart re-renders with up to 90 days of points
- Tap "All" → sub-title "All time"; chart title "📈 Weight Trend — All time"; "Success Days / N" where N is total days logged
- DevTools Network: confirm the "All" tap makes one query against the collection (not 365 individual gets)
- Verify weakest-trigger, weight-delta, best-streak all recompute (manually cross-check at least one — e.g. weight delta should match earliest minus latest over the chosen range)

- [ ] **Step 6: Commit**

```powershell
git add index.html
git commit -m "Add 30d/90d/All range toggle on Stats page"
git push origin main
```

---

### Task 7: All-time summary card on Stats page

**Files:**
- Modify: `index.html` (CSS, Stats markup, `renderAllTime` JS, `renderStats` hook)

- [ ] **Step 1: Add all-time card styles**

In the `<style>` block:
```css
.all-time-card{background:linear-gradient(135deg,var(--card) 0%,var(--card2) 100%);border:1px solid var(--border);border-radius:14px;padding:14px;margin-bottom:12px}
.all-time-title{font-size:10px;color:var(--muted);text-transform:uppercase;letter-spacing:.1em;margin-bottom:10px}
.all-time-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;text-align:center}
.all-time-grid .at-val{font-size:20px;font-weight:900;line-height:1}
.all-time-grid .at-lbl{font-size:10px;color:var(--muted);margin-top:4px}
```

- [ ] **Step 2: Add the all-time card markup**

Locate the markup just before the range toggle added in Task 6 (immediately after the closing `</div>` of `.page-header` on the Stats page). Insert:
```html
  <div class="all-time-card">
    <div class="all-time-title">🏁 All time</div>
    <div class="all-time-grid">
      <div><div class="at-val" id="atDays" style="color:var(--p)">—</div><div class="at-lbl">Days logged</div></div>
      <div><div class="at-val" id="atDelta" style="color:var(--c)">—</div><div class="at-lbl">Weight Δ (kg)</div></div>
      <div><div class="at-val" id="atStreak" style="color:var(--f)">—</div><div class="at-lbl">Best streak</div></div>
    </div>
  </div>
```

- [ ] **Step 3: Add `renderAllTime` and hook into `renderStats`**

Place near `renderStats`:
```js
let allTimeCache = null; // {days, delta, bestStreak}

async function renderAllTime(){
  // Fetch all and compute once per session unless explicitly invalidated
  if(!allTimeCache){
    const data = await loadAllRange('all');
    const entries = Object.entries(data).sort((a,b)=> a[0]>b[0]?1:-1);
    let days = 0, firstW = null, lastW = null, bestStreak = 0, curStreak = 0;
    entries.forEach(([_, e])=>{
      if(!e) return;
      days++;
      if(typeof e.weight === 'number' && e.weight > 0){
        if(firstW === null) firstW = e.weight;
        lastW = e.weight;
      }
      if(isSuccess(e)){ curStreak++; bestStreak = Math.max(bestStreak, curStreak); }
      else curStreak = 0;
    });
    const delta = (firstW != null && lastW != null) ? (lastW - firstW) : null;
    allTimeCache = { days, delta, bestStreak };
  }
  const { days, delta, bestStreak } = allTimeCache;
  document.getElementById('atDays').textContent = days;
  if(delta == null){
    document.getElementById('atDelta').textContent = '—';
    document.getElementById('atDelta').style.color = 'var(--c)';
  } else {
    document.getElementById('atDelta').textContent = (delta > 0 ? '+' : '') + delta.toFixed(1);
    document.getElementById('atDelta').style.color = delta <= 0 ? 'var(--p)' : 'var(--red)';
  }
  document.getElementById('atStreak').textContent = bestStreak;
}
```

At the top of `renderStats`, **before** `const data = await loadAllRange(range);`, add:
```js
  renderAllTime(); // fire-and-forget; updates DOM when ready
```

To make sure deleting / editing data invalidates the cache, also add cache invalidation in two places:
- At the end of `saveToday()` (after `setSyncState('synced')`):
```js
    allTimeCache = null;
```
- In the full-reset block at `index.html:944-946` area (where the new state was added in Task 1, Step 4):
```js
    allTimeCache = null;
```

- [ ] **Step 4: Verify in browser**

Push, hard-refresh, sign in, go to Stats:
- "🏁 All time" card appears above the range toggle
- "Days logged" matches total docs in `eyal/` minus `config` (cross-check against Firebase Console)
- "Weight Δ" matches `latest_logged_weight - earliest_logged_weight` (sign correct: negative & green if you lost, positive & red if you gained)
- "Best streak" is at least as large as the 30d "Best Streak 🏆" tile below
- Switch ranges — all-time card values do NOT change
- Edit any day's weight on Today → back to Stats → all-time card refreshes (cache was invalidated by `saveToday`)
- Edge case: if no weights ever logged, Weight Δ shows "—"

- [ ] **Step 5: Commit**

```powershell
git add index.html
git commit -m "Add all-time summary card on Stats page"
git push origin main
```

---

## Final verification (after all tasks)

End-to-end smoke test on the live site:
- [ ] Sign-in still works
- [ ] Today page: macros, weight, triggers, habits, daily note all function as before
- [ ] Meal templates: 3 templates, tap each, macros add correctly, edit name, delete one
- [ ] TDEE: enter your real height/age/sex/activity, verify the projection looks plausible (e.g. moderate activity at ~85 kg, 175 cm, male, 38 yrs → TDEE ~2800, deficit ~840, projected loss ~0.76 kg/week)
- [ ] Stats: All-time card shows correct totals; toggle 30/90/All; chart redraws each time
- [ ] Reset All still works and clears templates + TDEE alongside existing data
- [ ] Firebase Console → `eyal/config` doc has: `habits`, `goal`, `startWeight`, `mealTemplates`, `tdee` — none missing after any single mutation
