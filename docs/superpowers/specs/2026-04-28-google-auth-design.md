# Google Auth + Re-auth-Gated Reset — Design

**Date:** 2026-04-28
**Status:** Approved (pending user spec review)

## Goal

Lock the Eyal Tracker app behind Google sign-in so the public GitHub Pages URL is no longer open, and require a fresh Google re-authentication before destructive "Reset All" runs.

## Background

The app is a single-file `index.html` deployed to GitHub Pages, backed by Firebase Firestore under collection `eyal/`. Currently:

- No authentication — anyone with the URL can read and modify the data.
- The Firebase API key is in page source; without rules, anyone can also write to Firestore directly.
- "Reset All" only requires two browser `confirm()` clicks.

## Architecture

Three layers of protection, all keyed off Firebase Auth:

1. **UI gate** — a sign-in screen replaces the app shell until the user is authenticated and their email matches the allowed account.
2. **Firestore security rules** — the `eyal/**` collection is readable/writable only by a specific UID/email. Even if someone has the API key, they cannot read or write without being signed in as the owner.
3. **Re-auth on Reset All** — destructive delete forces a fresh Google popup before running, so a left-open browser session cannot trivially nuke data.

## Components

### Sign-in screen (new)

Replaces the app shell when signed out.

- Centered card: app name + "Sign in with Google" button.
- On click: `signInWithPopup(auth, googleProvider)`.
- On success: if `user.email === ALLOWED_EMAIL`, show app. Otherwise call `signOut(auth)` and display "Not authorized."

### Auth state listener

Wired into the existing `<script type="module">` block.

- `onAuthStateChanged(auth, user => ...)`:
  - `user` null or wrong email → show sign-in screen, hide app.
  - `user` allowed → hide sign-in screen, show app, run existing init flow (`loadConfig`, `loadToday`, week, etc.).

### Sign-out button

Small button placed in the existing Reset card (above "Reset Today").

- Calls `signOut(auth)` → returns to sign-in screen via the auth state listener.

### Reset All re-auth flow

Replaces the current double-`confirm` flow.

1. One `confirm("Delete all data permanently? This cannot be undone.")`.
2. `reauthenticateWithPopup(user, googleProvider)`.
3. On success: run existing delete logic (`getDocs` + `deleteDoc` over the collection, reset in-memory state, `saveConfig`).
4. On cancel or popup failure: abort, no changes, `setSyncState('')`.

### Firestore security rules (new file)

New file `firestore.rules` at repo root:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /eyal/{doc} {
      allow read, write: if request.auth != null
                         && request.auth.token.email == "eyalkay@gmail.com";
    }
  }
}
```

Plus `firebase.json` minimal config so `firebase deploy --only firestore:rules` works.

## Data flow

- **Signed out** → sign-in screen only; no Firestore reads attempted.
- **Signed in (allowed)** → existing app flow unchanged.
- **Signed in (wrong email)** → immediate `signOut`, error banner on sign-in screen.
- **Reset All** → confirm → re-auth popup → on success, delete proceeds with the same auth context (rules pass).

## Dependencies

Add one Firebase SDK import:

```js
import {
  getAuth, GoogleAuthProvider, signInWithPopup, signOut,
  onAuthStateChanged, reauthenticateWithPopup
} from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";
```

No new npm packages — single-file app stays single-file.

## Error handling

- **Popup blocked / closed** — show inline error on sign-in screen, leave button enabled.
- **Wrong account** — sign out, show "Not authorized: <email>" message, clear after next successful sign-in.
- **Re-auth failed mid-Reset** — abort delete, keep all data, restore sync state.
- **Network / Firestore errors after sign-in** — existing `setSyncState` error handling already covers this.

## Setup steps (outside code)

These are one-time manual steps the user must perform:

1. Firebase Console → Authentication → Sign-in method → enable **Google** provider.
2. Authentication → Settings → Authorized domains → add `eyalkay.github.io` (and keep `localhost` for testing).
3. Install Firebase CLI once: `npm install -g firebase-tools`, then `firebase login`.
4. Deploy rules: `firebase deploy --only firestore:rules` from repo root.

## Out of scope

- Multi-user support (the app remains single-user, keyed to `eyal/` collection).
- Email/password auth, magic links, or other providers.
- Granular per-document permissions.
- Rule-based protection of a future `users/{uid}/` schema (not changing the data layout).

## Open questions

None — all clarified during brainstorming.

## Acceptance criteria

- Visiting the site signed-out shows a sign-in screen, not the app.
- Signing in with the allowed Google account loads the app exactly as before.
- Signing in with any other Google account is rejected and the app does not load.
- Direct Firestore access without auth (e.g. via the SDK in another page using the same API key) is denied by rules.
- Clicking "Reset All" prompts a Google popup; cancelling the popup does not delete data.
- Cancelling the initial confirm does not trigger a popup.
- Sign-out returns the user to the sign-in screen and closes the app view.
