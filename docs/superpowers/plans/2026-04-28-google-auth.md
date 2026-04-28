# Google Auth + Re-auth-Gated Reset Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lock the Eyal Tracker app behind Google sign-in (allowed email only), enforce the same restriction at the Firestore rules layer, and require a fresh Google re-auth before "Reset All" deletes data.

**Architecture:** Single-file `index.html` deployed to GitHub Pages, backed by Firebase Firestore (`eyal/` collection). We add Firebase Auth (Google provider) on top of the existing Firebase SDK imports. Auth state drives a sign-in screen vs. app-shell toggle. A new `firestore.rules` file enforces UID-based access at the database layer. The destructive "Reset All" path calls `reauthenticateWithPopup` before running its existing delete logic.

**Tech Stack:** Firebase Auth v10.7.1 (Google provider), Firebase Firestore v10.7.1, vanilla JS in a single HTML file, Firebase CLI for rules deployment.

**Spec:** `docs/superpowers/specs/2026-04-28-google-auth-design.md`

**Testing note:** This codebase has no automated test framework — it's a single HTML file the user iterates on by editing → pushing → reloading the live site. Verification in this plan is **manual browser testing** at each task boundary. Each task lists exact steps to perform in a browser and the expected outcome. Do not skip these — they replace unit tests.

---

## File Structure

- **Modify:** `index.html` — add auth imports, sign-in screen markup, auth-state listener, sign-out button, re-auth gate on Reset All.
- **Create:** `firestore.rules` — security rules restricting `eyal/{doc}` to the allowed email.
- **Create:** `firebase.json` — minimal config so `firebase deploy --only firestore:rules` finds the rules file.
- **Create:** `.firebaserc` — pins the Firebase project ID so deploy works without `--project` flag.

The single-file structure is preserved — no JS modules added. New code lives inside the existing `<script type="module">` block.

---

## Task 1: Add Firebase config files for rules deployment

**Files:**
- Create: `F:/Code/Eyal-Tracker/firestore.rules`
- Create: `F:/Code/Eyal-Tracker/firebase.json`
- Create: `F:/Code/Eyal-Tracker/.firebaserc`

- [ ] **Step 1: Create `firestore.rules` with deny-by-default rules locked to the allowed email**

Write file `F:/Code/Eyal-Tracker/firestore.rules`:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /eyal/{doc} {
      allow read, write: if request.auth != null
                         && request.auth.token.email == "eyalkay@gmail.com"
                         && request.auth.token.email_verified == true;
    }
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

- [ ] **Step 2: Create `firebase.json` pointing at the rules file**

Write file `F:/Code/Eyal-Tracker/firebase.json`:

```json
{
  "firestore": {
    "rules": "firestore.rules"
  }
}
```

- [ ] **Step 3: Create `.firebaserc` pinning the project ID**

Write file `F:/Code/Eyal-Tracker/.firebaserc`:

```json
{
  "projects": {
    "default": "eyal-tracker"
  }
}
```

- [ ] **Step 4: Commit**

```bash
git add firestore.rules firebase.json .firebaserc
git commit -m "Add Firestore security rules locking eyal/ to owner email"
```

---

## Task 2: Add the Firebase Auth SDK import

**Files:**
- Modify: `F:/Code/Eyal-Tracker/index.html` (lines ~491-493, the existing Firebase import block)

- [ ] **Step 1: Read the current import block**

Read `index.html` lines 489-506 to confirm the existing Firebase imports are unchanged.

- [ ] **Step 2: Add the auth import right after the Firestore import**

In `index.html`, change:

```js
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getFirestore, doc, setDoc, getDoc, deleteDoc, collection, getDocs, query, orderBy, limit }
  from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
```

to:

```js
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getFirestore, doc, setDoc, getDoc, deleteDoc, collection, getDocs, query, orderBy, limit }
  from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
import { getAuth, GoogleAuthProvider, signInWithPopup, signOut,
         onAuthStateChanged, reauthenticateWithPopup }
  from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";
```

- [ ] **Step 3: Initialize the auth instance after `getFirestore`**

Find:

```js
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
```

Change to:

```js
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);
const googleProvider = new GoogleAuthProvider();
const ALLOWED_EMAIL = "eyalkay@gmail.com";
```

- [ ] **Step 4: Verify the page still loads (no JS errors)**

Open `index.html` in a browser. Open DevTools console.

Expected: page renders as before, no red errors. The Firestore sync dot still works. (Auth is loaded but not yet wired to anything.)

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Import firebase-auth SDK and initialize Google provider"
```

---

## Task 3: Add the sign-in screen markup and styles

**Files:**
- Modify: `F:/Code/Eyal-Tracker/index.html`

- [ ] **Step 1: Add sign-in screen styles inside the existing `<style>` block**

Inside `index.html`'s `<style>` block, just before the closing `</style>`, append:

```css
/* AUTH */
.auth-screen{display:none;position:fixed;inset:0;background:var(--bg);z-index:200;align-items:center;justify-content:center;padding:24px}
.auth-screen.active{display:flex}
.auth-card{background:var(--card);border:1px solid var(--border);border-radius:16px;padding:32px 24px;max-width:340px;width:100%;text-align:center}
.auth-title{font-size:22px;font-weight:900;margin-bottom:6px}
.auth-sub{font-size:13px;color:var(--muted);margin-bottom:24px}
.auth-btn{width:100%;padding:14px;border-radius:12px;border:1px solid var(--border);background:var(--card2);color:var(--text);font-family:'Inter',sans-serif;font-size:14px;font-weight:700;cursor:pointer;transition:background .15s}
.auth-btn:hover{background:#26262a}
.auth-btn:active{transform:scale(.98)}
.auth-error{margin-top:14px;font-size:12px;color:var(--red);min-height:16px}
body.auth-locked{overflow:hidden}
body.auth-locked .bottom-nav,body.auth-locked .page{display:none !important}
```

- [ ] **Step 2: Add the sign-in screen markup as the first child of `<body>`**

In `index.html`, find the opening `<body>` tag and immediately after it, insert:

```html
<div class="auth-screen" id="authScreen">
  <div class="auth-card">
    <div class="auth-title">Eyal | Tracker</div>
    <div class="auth-sub">Sign in to continue</div>
    <button class="auth-btn" id="signInBtn" onclick="doSignIn()">Sign in with Google</button>
    <div class="auth-error" id="authError"></div>
  </div>
</div>
```

- [ ] **Step 3: Verify the sign-in screen exists but is hidden by default**

Open `index.html` in a browser.

Expected: app renders normally (sign-in screen is hidden because `.auth-screen` is `display:none` without the `active` class). Open DevTools and run `document.getElementById('authScreen').classList.add('active')` — the sign-in card should appear over the app. Run `...remove('active')` to hide it again.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add sign-in screen markup and styles"
```

---

## Task 4: Wire auth state listener to gate the app

**Files:**
- Modify: `F:/Code/Eyal-Tracker/index.html` (the init line at the bottom and surrounding script)

- [ ] **Step 1: Replace the bare `loadToday()` init call with an auth-gated init**

Find the end of the `<script type="module">` block:

```js
// ── INIT ──
loadToday();
</script>
```

Change it to:

```js
// ── AUTH ──
const authScreen = document.getElementById('authScreen');
const authError = document.getElementById('authError');

function showAuthScreen(message){
  document.body.classList.add('auth-locked');
  authScreen.classList.add('active');
  authError.textContent = message || '';
}
function hideAuthScreen(){
  document.body.classList.remove('auth-locked');
  authScreen.classList.remove('active');
  authError.textContent = '';
}

window.doSignIn = async function(){
  authError.textContent = '';
  try{
    await signInWithPopup(auth, googleProvider);
  }catch(e){
    if(e.code === 'auth/popup-closed-by-user' || e.code === 'auth/cancelled-popup-request') return;
    authError.textContent = 'Sign-in failed. Try again.';
    console.error(e);
  }
};

window.doSignOut = async function(){
  try{ await signOut(auth); }catch(e){ console.error(e); }
};

let appInitialized = false;
onAuthStateChanged(auth, async (user) => {
  if(!user){
    showAuthScreen();
    return;
  }
  if(user.email !== ALLOWED_EMAIL){
    await signOut(auth);
    showAuthScreen('Not authorized: ' + user.email);
    return;
  }
  hideAuthScreen();
  if(!appInitialized){
    appInitialized = true;
    loadToday();
  }
});
</script>
```

- [ ] **Step 2: Verify signed-out state shows sign-in screen**

Open `index.html` (locally, with no Firebase Auth session). Or in production after deploy.

Expected: sign-in screen is visible covering the whole viewport. App content and bottom nav are hidden. No Firestore reads happen (check Network tab — no `firestore.googleapis.com` requests on load).

- [ ] **Step 3: Verify sign-in with allowed account loads the app**

Click "Sign in with Google", choose `eyalkay@gmail.com`.

Expected: popup closes, sign-in screen disappears, app renders, today's data loads from Firestore.

Note: this requires Google provider enabled in Firebase Console and your domain added to Authorized domains. If those aren't done yet, defer this verification to Task 7 and proceed — the listener code is still correct.

- [ ] **Step 4: Verify wrong-account rejection**

If you have a second Google account, sign in with it.

Expected: popup completes, but the app immediately signs you out and shows "Not authorized: <email>" on the sign-in screen.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Gate app behind Google sign-in for allowed email only"
```

---

## Task 5: Add the sign-out button to the Reset card

**Files:**
- Modify: `F:/Code/Eyal-Tracker/index.html` (lines ~366-371, the Reset card)

- [ ] **Step 1: Add a sign-out button at the top of the Reset card**

Find:

```html
<!-- RESET -->
<div class="card danger-card">
  <div class="card-title">🗑️ Reset</div>
  <button class="reset-btn" onclick="resetToday()">Reset Today</button>
  <button class="reset-btn reset-all" onclick="resetAll()">Reset All — All Data & Goals</button>
</div>
```

Change to:

```html
<!-- RESET -->
<div class="card danger-card">
  <div class="card-title">🗑️ Reset</div>
  <button class="reset-btn" onclick="doSignOut()">Sign Out</button>
  <button class="reset-btn" onclick="resetToday()">Reset Today</button>
  <button class="reset-btn reset-all" onclick="resetAll()">Reset All — All Data & Goals</button>
</div>
```

- [ ] **Step 2: Verify sign-out returns to sign-in screen**

While signed in, click "Sign Out".

Expected: sign-in screen reappears, app content hides. Reload the page — sign-in screen still shows (session is gone).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add Sign Out button to Reset card"
```

---

## Task 6: Gate Reset All behind Google re-authentication

**Files:**
- Modify: `F:/Code/Eyal-Tracker/index.html` (lines ~897-920, the `resetAll` function)

- [ ] **Step 1: Replace the double `confirm` with a single confirm + re-auth popup**

Find:

```js
window.resetAll = async function(){
  if(!confirm('Delete all data, habits, and goals?\nThis cannot be undone!')) return;
  if(!confirm('Are you sure? All history will be permanently deleted.')) return;
  setSyncState('syncing');
  try{
    const snap = await getDocs(collection(db, USER));
    const deletes = [];
    snap.forEach(d => { deletes.push(deleteDoc(doc(db, USER, d.id))); });
    await Promise.all(deletes);
    customHabits = [];
    goalWeight = null;
    startingWeight = 87;
    await saveConfig();
    todayData = {p:0,c:0,f:0,weight:null,triggers:{},habits:{},note:''};
    document.getElementById('weightIn').value='';
    document.getElementById('noteInput').value='';
    renderToday();
    renderHabits();
    setSyncState('synced');
  }catch(e){
    console.error(e);
    setSyncState('');
  }
};
```

Change to:

```js
window.resetAll = async function(){
  if(!confirm('Delete all data, habits, and goals?\nThis cannot be undone!')) return;
  const user = auth.currentUser;
  if(!user){ alert('Not signed in.'); return; }
  try{
    await reauthenticateWithPopup(user, googleProvider);
  }catch(e){
    if(e.code === 'auth/popup-closed-by-user' || e.code === 'auth/cancelled-popup-request') return;
    console.error(e);
    alert('Re-authentication failed. Reset cancelled.');
    return;
  }
  setSyncState('syncing');
  try{
    const snap = await getDocs(collection(db, USER));
    const deletes = [];
    snap.forEach(d => { deletes.push(deleteDoc(doc(db, USER, d.id))); });
    await Promise.all(deletes);
    customHabits = [];
    goalWeight = null;
    startingWeight = 87;
    await saveConfig();
    todayData = {p:0,c:0,f:0,weight:null,triggers:{},habits:{},note:''};
    document.getElementById('weightIn').value='';
    document.getElementById('noteInput').value='';
    renderToday();
    renderHabits();
    setSyncState('synced');
  }catch(e){
    console.error(e);
    setSyncState('');
  }
};
```

- [ ] **Step 2: Verify cancelling the confirm aborts the popup**

While signed in with test data, click "Reset All — All Data & Goals" and click Cancel on the browser confirm.

Expected: nothing happens, no popup, no data deleted.

- [ ] **Step 3: Verify cancelling the re-auth popup aborts the delete**

Click "Reset All", accept the confirm, then close the Google popup without choosing an account.

Expected: no error alert (popup-closed errors are swallowed), no data deleted, sync dot stays as it was.

- [ ] **Step 4: Verify successful re-auth performs the delete (USE TEST DATA ONLY)**

⚠️ This step actually deletes data. Do this on a test day or after backing up, not on real tracked data.

Click "Reset All", accept the confirm, complete the Google popup with the allowed account.

Expected: today resets to zero, week view shows no past entries, Firestore `eyal/` collection in the Firebase Console is empty.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Require Google re-authentication before Reset All"
```

---

## Task 7: Configure Firebase project (one-time manual setup)

These are manual steps in Firebase Console and CLI, not code changes. They must be completed for the app to actually work in production.

- [ ] **Step 1: Enable Google sign-in provider**

Go to https://console.firebase.google.com/project/eyal-tracker/authentication/providers
Click "Google" → toggle Enable → set support email → Save.

Expected: Google appears in the enabled providers list.

- [ ] **Step 2: Add authorized domains**

Go to https://console.firebase.google.com/project/eyal-tracker/authentication/settings
Under "Authorized domains", confirm `localhost` is present and click "Add domain" → enter `eyalkay.github.io` → Add.

Expected: both `localhost` and `eyalkay.github.io` are listed.

- [ ] **Step 3: Install and log in to Firebase CLI (skip if already done)**

```bash
npm install -g firebase-tools
firebase login
```

Expected: `firebase login` opens a browser, you authenticate, terminal prints "Success! Logged in as <email>".

- [ ] **Step 4: Deploy security rules**

From `F:/Code/Eyal-Tracker`:

```bash
firebase deploy --only firestore:rules
```

Expected: output ends with "✔ Deploy complete!" and shows the rules were released.

- [ ] **Step 5: Verify rules in console**

Go to https://console.firebase.google.com/project/eyal-tracker/firestore/rules

Expected: the published rules match `firestore.rules` (deny-by-default + allowed email check).

- [ ] **Step 6: Verify denied access without auth**

Open `index.html` directly in a browser without signing in. Open DevTools → Network → reload.

Expected: no Firestore document fetches succeed. (The auth gate prevents requests entirely; no `permission-denied` errors should appear since reads don't fire pre-auth.)

Optional sanity check: in DevTools console while signed out, run:
```js
import('https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js').then(m => m.getDoc(m.doc(db, 'eyal', '2026-04-28'))).then(s => console.log('READ', s.exists())).catch(e => console.log('DENIED', e.code));
```
Expected: `DENIED permission-denied`.

---

## Task 8: Deploy and verify on live site

**Files:** none (deployment).

- [ ] **Step 1: Push all commits to `main`**

```bash
git push origin main
```

Expected: push succeeds.

- [ ] **Step 2: Wait for GitHub Pages to redeploy**

Check https://github.com/EyalKay/eyal-tracker/actions for the Pages workflow.

Expected: latest workflow run is green.

- [ ] **Step 3: Verify live sign-in flow**

Open https://eyalkay.github.io/eyal-tracker/ in an incognito window.

Expected: sign-in screen shows. Sign in with `eyalkay@gmail.com` → app loads with existing data intact.

- [ ] **Step 4: Verify live re-auth-gated reset**

In the live app, navigate to Today → Reset card → click "Reset All" → cancel both at confirm and at popup stages.

Expected: data is unchanged after each cancellation. Do NOT actually run the delete on live unless that's intended.

- [ ] **Step 5: Verify sign-out**

Click "Sign Out" → sign-in screen appears → reload → still signed out.

Expected: must sign in again to access data.

---

## Self-Review Notes

- **Spec coverage:** UI gate (Task 3+4), Firestore rules (Task 1+7), re-auth on Reset (Task 6), sign-out (Task 5), wrong-account rejection (Task 4 step 4), all setup steps (Task 7) — all sections present.
- **Placeholders:** none.
- **Type/name consistency:** `auth`, `googleProvider`, `ALLOWED_EMAIL`, `authScreen`, `authError`, `appInitialized`, `doSignIn`, `doSignOut` are defined in Task 2/4 and referenced consistently in Tasks 4–6.
- **Testing:** No automated tests exist in this repo; verification is manual browser checks listed in each task.

---

Plan complete and saved to `docs/superpowers/plans/2026-04-28-google-auth.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
