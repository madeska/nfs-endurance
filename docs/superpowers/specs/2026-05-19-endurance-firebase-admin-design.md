# Endurance Firebase Admin — Design Spec

## Overview

Add an admin role to the endurance tracker. Admin loads race data from the collector (existing flow) and publishes it to Firebase Realtime Database. Regular users read only from Firebase — no collector access, instant loading.

## Problem

Loading the endurance race list requires probing the collector for every candidate session (>1h), fetching all events per session to detect team structure. First load from 2026-01-01 can take 30+ seconds.

## Firebase Structure

**Realtime Database:**

```
/endurance_sessions  ← array of published session metadata
  [
    { id, start_time, end_time, teamCount, publishedAt },
    ...
  ]

/endurance_data/{sessionId}  ← full race data per session
  {
    sessionId,
    teams,          // processed team summaries
    meta,           // race-wide bests
    laps,           // raw lap array from collector
    pitLapsMap,     // {teamNum: [lapNums]} — Set serialized as array
    pitEntryTsMap,  // {teamNum: {lapNum: ts}}
    pitTimes,       // {}
    joins           // []
    // events NOT stored — too large (5–50MB), already processed into above
  }
```

**Security rules:**
```json
{
  "rules": {
    "endurance_sessions": { ".read": true, ".write": "auth != null" },
    "endurance_data":     { ".read": true, ".write": "auth != null" }
  }
}
```

## User Flows

### Regular user
1. Opens Endurance page → `_fbLoadEnduranceSessions()` reads `/endurance_sessions` from RTDB → instant list
2. Clicks a race → reads `/endurance_data/{id}` from RTDB → instant race view
3. Never contacts collector API

### Admin
1. Clicks "Admin" button in endurance page header → email/password modal
2. Firebase Auth → admin badge appears in header
3. Race list shows all races (published from RTDB + unpublished from collector)
4. "Refresh races" button still works (probes collector for new sessions)
5. Each race in the list has a "Publish" button
   - If `_enduranceCache[id]` exists: publish immediately
   - If not: load from collector first (existing `_loadEnduranceTeamData`), then publish
6. Published races show a "✓" badge in the list

## New Functions

| Function | Purpose |
|---|---|
| `_fbInit()` | Initialize Firebase app + Auth + Database on page load |
| `_fbLoadEnduranceSessions()` | Read `/endurance_sessions` from RTDB, populate `_enduranceSessions` |
| `_fbLoadRaceData(sessionId)` | Read `/endurance_data/{id}` from RTDB, populate `_enduranceData` |
| `_fbPublishRace(sessionId)` | Serialize `_enduranceCache[sessionId]` → write to RTDB |
| `_adminLogin(email, pass)` | Firebase Auth `signInWithEmailAndPassword` |
| `_adminLogout()` | Firebase Auth `signOut`, hide admin UI |

## Changes to Existing Functions

- `loadEnduranceSessions()`: if not admin → call `_fbLoadEnduranceSessions()` instead of collector flow
- `_openEnduranceRace(s, i)`: if not admin and race is published → call `_fbLoadRaceData(s.id)` instead of `_loadEnduranceTeamData`
- `_renderEndurancePage()`: if admin → show "Publish" button per race + "Refresh races" button; admin sees merged list (RTDB published + unpublished from collector, deduplicated by session ID)

## Serialization Notes

`pitLapsMap` contains `Set` values. Before writing to RTDB:
```js
const serialized = {};
for (const [k, v] of Object.entries(pitLapsMap)) {
  serialized[k] = [...v];  // Set → Array
}
```
On read: convert arrays back to Sets.

## Firebase SDK

Use Firebase 10 compat CDN (simplest for single-file app):
```html
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-database-compat.js"></script>
```

Firebase config embedded as a JS constant (safe to expose — security enforced by Auth rules):
```js
const _FB_CONFIG = {
  apiKey: "...",
  authDomain: "...",
  databaseURL: "...",
  projectId: "..."
};
```

## Prerequisites Before Implementation

- Firebase config object (apiKey, authDomain, databaseURL, projectId)
- Admin email registered in Firebase Authentication console
- RTDB security rules applied (see above)
