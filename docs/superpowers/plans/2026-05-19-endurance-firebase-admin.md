# Endurance Firebase Admin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Admin publishes endurance race data to Firebase RTDB; regular users load from RTDB instead of probing the collector, making the endurance list instant.

**Architecture:** Firebase Realtime Database stores a `/endurance_sessions/{id}` map (metadata) and `/endurance_data/{id}` map (full race data). Regular users read only from RTDB. Admins authenticate via Firebase Auth email/password and use the existing collector flow, with a "Publish" button per race that writes to RTDB. Sets in `pitLapsMap` are serialized as arrays for RTDB storage.

**Tech Stack:** Firebase 10.14.1 compat SDK (app, auth, database); existing vanilla JS single-file app.

---

## File Map

Only `index.html` is modified. All changes are within the single `<head>` block.

| Area | Change |
|---|---|
| `<head>` line ~10 | Add 3 Firebase compat `<script>` tags |
| `<script>` top (line ~12) | Add `_FB_CONFIG`, Firebase globals, `_fbInit()` |
| line ~692 | Call `_fbInit()` |
| line ~790 (before endurance section) | Add all new Firebase functions |
| `renderEndurance()` line ~793 | Branch on `_isAdmin` |
| `_renderEndurancePage()` line ~897 | Add admin UI (login btn, publish btns, published badges) |
| `_loadEnduranceRace()` line ~949 | Use `_fbLoadRaceData()` for non-admins |
| `_loadEnduranceTeamData()` line ~977 | Trigger `_fbSaveCurrentRaceToRTDB` if `_pendingPublish` |
| `<body>` | Add admin login modal HTML |

---

## Task 1: Add Firebase SDK scripts + config globals

**Files:**
- Modify: `index.html` — add script tags in `<head>`, add config + globals in main `<script>`

- [ ] **Step 1: Add Firebase CDN scripts in `<head>` after ExcelJS (line ~10)**

Insert these three lines after `<script src="https://cdn.jsdelivr.net/npm/exceljs@4.4.0/dist/exceljs.min.js"></script>`:

```html
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-database-compat.js"></script>
```

- [ ] **Step 2: Add Firebase config and global variables at the very top of the main `<script>` block (after the opening `<script>` tag, before any other code)**

```javascript
const _FB_CONFIG={apiKey:'AIzaSyA_C7wlAqqEKwKvXjm4aSrxj28-7YRmTcg',authDomain:'nfs-endurance.firebaseapp.com',databaseURL:'https://nfs-endurance-default-rtdb.europe-west1.firebasedatabase.app',projectId:'nfs-endurance',storageBucket:'nfs-endurance.firebasestorage.app',messagingSenderId:'579851470135',appId:'1:579851470135:web:ce6d0cbd076ca581c286b2'};
let _fbApp=null,_fbAuth=null,_fbDb=null;
let _isAdmin=false;
let _fbPublishedIds=new Set();// sessionIds published in RTDB
let _pendingPublish=null;// sessionId to publish after load
```

- [ ] **Step 3: Add `_fbInit()` function immediately after those globals**

```javascript
function _fbInit(){
  _fbApp=firebase.initializeApp(_FB_CONFIG);
  _fbAuth=firebase.auth();
  _fbDb=firebase.database();
  _fbAuth.onAuthStateChanged(user=>{
    _isAdmin=!!user;
    // If endurance list is currently showing, re-render with correct mode
    const ap=document.querySelector('.page.active');
    if(ap&&ap.id==='page-endurance'){
      _enduranceSessions=[];
      _enduranceData=null;
      renderEndurance();
    }
  });
}
```

- [ ] **Step 4: Call `_fbInit()` right after `window._COLLECTOR=COLLECTOR;` line (~line 692)**

```javascript
window._COLLECTOR=COLLECTOR;
window._fmtDateDMY=_fmtDateDMY;
_fbInit();
```

- [ ] **Step 5: Verify in browser console — no errors, `_fbDb` is not null**

Open the page in a browser, open console, type `_fbDb`. Should show a Firebase Database object, not `null` or `undefined`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add Firebase SDK config and init"
```

---

## Task 2: Admin auth functions

**Files:**
- Modify: `index.html` — add auth functions before the `/* ─── ENDURANCE ─── */` comment (~line 789)

- [ ] **Step 1: Add auth functions before `/* ─── ENDURANCE ─── */` comment**

```javascript
function _showAdminModal(){
  const m=document.getElementById('fb-admin-modal');
  m.style.display='flex';
  document.getElementById('fb-admin-err').style.display='none';
  document.getElementById('fb-admin-email').value='';
  document.getElementById('fb-admin-pass').value='';
  setTimeout(()=>document.getElementById('fb-admin-email').focus(),50);
}
function _hideAdminModal(){
  document.getElementById('fb-admin-modal').style.display='none';
}
async function _adminLogin(){
  const email=document.getElementById('fb-admin-email').value.trim();
  const pass=document.getElementById('fb-admin-pass').value;
  const errEl=document.getElementById('fb-admin-err');
  if(!email||!pass){errEl.textContent='Enter email and password';errEl.style.display='';return;}
  try{
    await _fbAuth.signInWithEmailAndPassword(email,pass);
    _hideAdminModal();
  }catch(e){
    errEl.textContent=e.message||'Login failed';
    errEl.style.display='';
  }
}
function _adminLogout(){
  _fbAuth.signOut();
}
async function _fbLoadPublishedIds(){
  try{
    const snap=await _fbDb.ref('endurance_sessions').once('value');
    const val=snap.val();
    if(val)Object.keys(val).forEach(k=>_fbPublishedIds.add(k));
  }catch(e){}
}
```

- [ ] **Step 2: Add admin login modal HTML at the top of `<body>`, right after `<body>` tag and before `<div class="wrap">`**

```html
<div id="fb-admin-modal" style="display:none;position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.75);z-index:9999;align-items:center;justify-content:center">
  <div style="background:#1a1b26;padding:32px;border-radius:8px;width:320px;max-width:90%;box-shadow:0 8px 32px rgba(0,0,0,0.5)">
    <div style="font-size:18px;font-weight:700;color:#fff;margin-bottom:20px">Admin Login</div>
    <input id="fb-admin-email" type="email" placeholder="Email" style="width:100%;padding:10px;background:#0d0e15;border:1px solid #333;color:#fff;border-radius:4px;margin-bottom:12px;box-sizing:border-box">
    <input id="fb-admin-pass" type="password" placeholder="Password" onkeydown="if(event.key==='Enter')_adminLogin()" style="width:100%;padding:10px;background:#0d0e15;border:1px solid #333;color:#fff;border-radius:4px;margin-bottom:16px;box-sizing:border-box">
    <div id="fb-admin-err" style="color:#ef4444;font-size:13px;margin-bottom:12px;display:none"></div>
    <div style="display:flex;gap:8px;justify-content:flex-end">
      <button class="btn" onclick="_hideAdminModal()" style="padding:8px 16px">Cancel</button>
      <button class="btn" onclick="_adminLogin()" style="padding:8px 16px;background:#e8b830;color:#000;font-weight:700">Login</button>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Verify modal works — open browser, navigate to Endurance tab, type `_showAdminModal()` in console. Modal should appear, Cancel should close it.**

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Firebase Auth login modal and auth functions"
```

---

## Task 3: `_fbLoadEnduranceSessions()` — read list from RTDB

**Files:**
- Modify: `index.html` — add function before `/* ─── ENDURANCE ─── */` comment

- [ ] **Step 1: Add `_fbLoadEnduranceSessions()` before the endurance section**

```javascript
async function _fbLoadEnduranceSessions(){
  document.getElementById('endurance-header').innerHTML='<div style="color:#8a8a8a;padding:40px;text-align:center">Loading endurance races...</div>';
  document.getElementById('endurance-table-wrap').innerHTML='';
  try{
    const snap=await _fbDb.ref('endurance_sessions').once('value');
    const val=snap.val();
    if(!val||!Object.keys(val).length){
      document.getElementById('endurance-header').innerHTML='<div style="color:#8a8a8a;padding:40px;text-align:center">No endurance races found</div>';
      return;
    }
    Object.keys(val).forEach(k=>_fbPublishedIds.add(k));
    _enduranceSessions=Object.values(val).sort((a,b)=>b.start_time-a.start_time);
    _renderEndurancePage();
  }catch(e){
    console.error('FB load sessions:',e);
    document.getElementById('endurance-header').innerHTML='<div style="color:#ef4444;padding:40px;text-align:center">Failed to load races</div>';
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add _fbLoadEnduranceSessions from RTDB"
```

---

## Task 4: Modify `renderEndurance()` — branch on admin status

**Files:**
- Modify: `index.html:793` — `renderEndurance()` function

- [ ] **Step 1: Replace the entire `renderEndurance()` function body**

Find the current function (line ~793):
```javascript
function renderEndurance(){
  if(_enduranceSessions.length)return;// already loaded — don't reset
  const LIST_KEY='nfs_endurance_list_v2';
  let cachedSessions=[];
  try{const raw=localStorage.getItem(LIST_KEY);if(raw)cachedSessions=JSON.parse(raw);}catch(e){}
  if(cachedSessions.length){
    _enduranceSessions=cachedSessions;
    if(!_enduranceData){_enduranceIdx=0;_renderEndurancePage();}
  }else{
    document.getElementById('endurance-header').innerHTML='<div style="color:#8a8a8a;padding:40px;text-align:center">Loading endurance races...</div>';
    document.getElementById('endurance-table-wrap').innerHTML='';
    loadEnduranceSessions();
  }
}
```

Replace with:
```javascript
function renderEndurance(){
  if(_enduranceSessions.length){_renderEndurancePage();return;}
  if(!_isAdmin){
    _fbLoadEnduranceSessions();
    return;
  }
  // Admin: use localStorage cache + collector
  _fbLoadPublishedIds();// populate _fbPublishedIds for publish button state
  const LIST_KEY='nfs_endurance_list_v2';
  let cachedSessions=[];
  try{const raw=localStorage.getItem(LIST_KEY);if(raw)cachedSessions=JSON.parse(raw);}catch(e){}
  if(cachedSessions.length){
    _enduranceSessions=cachedSessions;
    _enduranceIdx=0;
    _renderEndurancePage();
  }else{
    document.getElementById('endurance-header').innerHTML='<div style="color:#8a8a8a;padding:40px;text-align:center">Loading endurance races...</div>';
    document.getElementById('endurance-table-wrap').innerHTML='';
    loadEnduranceSessions();
  }
}
```

- [ ] **Step 2: Verify — regular user (not logged in) opens Endurance tab → shows "Loading endurance races..." → then "No endurance races found" (RTDB is empty). No collector requests in Network tab.**

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: regular users load endurance list from RTDB"
```

---

## Task 5: Modify `_renderEndurancePage()` — admin UI

**Files:**
- Modify: `index.html:897` — `_renderEndurancePage()` function

- [ ] **Step 1: Replace the header html line (line ~905) and the race row html (line ~930)**

Find and replace the header line:
```javascript
  let html='<div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px"><div style="font-size:18px;font-weight:700;color:#fff">Endurance Races</div><button class="btn" id="endurance-refresh-btn" onclick="_refreshEnduranceSessions(this)" style="padding:6px 16px">Refresh races</button></div>';
```

Replace with:
```javascript
  const _adminBadge=_isAdmin?`<span style="font-size:11px;color:#e8b830;font-weight:700;margin-right:4px">ADMIN</span><button class="btn" onclick="_adminLogout()" style="padding:4px 10px;font-size:11px;margin-right:6px">Logout</button>`:`<button class="btn" onclick="_showAdminModal()" style="padding:4px 10px;font-size:11px;margin-right:6px">Admin</button>`;
  const _refreshBtn=_isAdmin?`<button class="btn" id="endurance-refresh-btn" onclick="_refreshEnduranceSessions(this)" style="padding:6px 16px">Refresh races</button>`:'';
  let html=`<div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px"><div style="font-size:18px;font-weight:700;color:#fff">Endurance Races</div><div style="display:flex;align-items:center">${_adminBadge}${_refreshBtn}</div></div>`;
```

- [ ] **Step 2: Replace the race row html (the `html+=` block with the row, line ~930)**

Find:
```javascript
      html+=`<div style="display:flex;align-items:center;justify-content:space-between;padding:12px 16px;background:${j%2===0?'#0d0e15':'#0b0c12'};border-bottom:1px solid #111;border-radius:4px;margin-bottom:4px">
        <div>
          <div style="font-size:15px;font-weight:600;color:#fff">${window._fmtDateDMY(s.start_time)} — Endurance Race</div>
          <div style="font-size:12px;color:#8a8a8a;margin-top:2px">${teamCount?teamCount+' teams · ':''}${durH}h ${durM}min</div>
        </div>
        <button class="btn" onclick="_loadEnduranceRace(${i})" style="padding:6px 16px">${cached?'Show':'Load'}</button>
      </div>`;
```

Replace with:
```javascript
      const _published=_fbPublishedIds.has(s.id);
      const _publishBtn=_isAdmin?`<button class="btn" onclick="_fbPublishRace('${s.id}',this)" style="padding:4px 10px;font-size:11px;margin-right:6px;${_published?'opacity:0.6':''}">${_published?'✓ Published':'Publish'}</button>`:'';
      const _loadLabel=cached?'Show':'Load';
      html+=`<div style="display:flex;align-items:center;justify-content:space-between;padding:12px 16px;background:${j%2===0?'#0d0e15':'#0b0c12'};border-bottom:1px solid #111;border-radius:4px;margin-bottom:4px">
        <div>
          <div style="font-size:15px;font-weight:600;color:#fff">${window._fmtDateDMY(s.start_time)} — Endurance Race</div>
          <div style="font-size:12px;color:#8a8a8a;margin-top:2px">${teamCount?teamCount+' teams · ':''}${durH}h ${durM}min${_published?' · <span style="color:#4ade80">published</span>':''}</div>
        </div>
        <div style="display:flex;align-items:center">${_publishBtn}<button class="btn" onclick="_loadEnduranceRace(${i})" style="padding:6px 16px">${_loadLabel}</button></div>
      </div>`;
```

- [ ] **Step 3: Verify in browser — regular user sees "Admin" button in header, no "Refresh races", no "Publish" buttons. Admin badge and buttons appear after login (test with `_adminLogin()` manually).**

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add admin UI to endurance race list"
```

---

## Task 6: `_fbSaveCurrentRaceToRTDB()` and `_fbPublishRace()` — write to RTDB

**Files:**
- Modify: `index.html` — add functions before `/* ─── ENDURANCE ─── */` comment

- [ ] **Step 1: Add `_fbSaveCurrentRaceToRTDB()` function**

```javascript
async function _fbSaveCurrentRaceToRTDB(sessionId){
  const data=_enduranceCache[sessionId];
  if(!data)return;
  // Serialize pitLapsMap: Set → Array
  const pitLapsMapSer={};
  for(const [k,v] of Object.entries(data.pitLapsMap||{})){
    pitLapsMapSer[k]=v instanceof Set?[...v]:Array.isArray(v)?v:Object.values(v);
  }
  const payload={
    sessionId:data.sessionId,
    teams:data.teams,
    meta:data.meta||{},
    laps:data.laps,
    pitLapsMap:pitLapsMapSer,
    pitEntryTsMap:data.pitEntryTsMap||{},
    pitTimes:data.pitTimes||{},
    joins:data.joins||[]
  };
  const s=_enduranceSessions.find(x=>x.id===sessionId);
  const sessionMeta={
    id:sessionId,
    start_time:s?s.start_time:0,
    end_time:s?s.end_time:0,
    teamCount:data.teams?data.teams.length:0,
    publishedAt:Date.now()
  };
  await _fbDb.ref('endurance_data/'+sessionId).set(payload);
  await _fbDb.ref('endurance_sessions/'+sessionId).set(sessionMeta);
  _fbPublishedIds.add(sessionId);
}
```

- [ ] **Step 2: Add `_fbPublishRace()` function**

```javascript
async function _fbPublishRace(sessionId,btn){
  if(!_isAdmin)return;
  if(btn){btn.disabled=true;btn.textContent='Publishing...';}
  try{
    if(_enduranceCache[sessionId]){
      await _fbSaveCurrentRaceToRTDB(sessionId);
      if(btn){btn.textContent='✓ Published';btn.style.opacity='0.6';}
    }else{
      // Load first, publish after load via _pendingPublish flag
      _pendingPublish=sessionId;
      const idx=_enduranceSessions.findIndex(s=>s.id===sessionId);
      if(idx>=0)_loadEnduranceRace(idx);
      else{if(btn){btn.disabled=false;btn.textContent='Publish';}}
    }
  }catch(e){
    console.error('FB publish error:',e);
    if(btn){btn.disabled=false;btn.textContent='Error — retry';}
  }
}
```

- [ ] **Step 3: Modify `_loadEnduranceTeamData()` — trigger publish after load if `_pendingPublish` is set**

Find the line near the end of `_loadEnduranceTeamData` (after `_enduranceCache[sessionId]=_enduranceData;`):
```javascript
    _enduranceData={teams,meta,laps:lapsResp,events:eventsResp,pitTimes:{},joins:[],sessionId,pitLapsMap,pitEntryTsMap};
    _enduranceCache[sessionId]=_enduranceData;
    _renderEnduranceTeams();
```

Replace with:
```javascript
    _enduranceData={teams,meta,laps:lapsResp,events:eventsResp,pitTimes:{},joins:[],sessionId,pitLapsMap,pitEntryTsMap};
    _enduranceCache[sessionId]=_enduranceData;
    _renderEnduranceTeams();
    if(_pendingPublish===sessionId){
      _pendingPublish=null;
      _fbSaveCurrentRaceToRTDB(sessionId).then(()=>{
        const sub=document.getElementById('endurance-subtitle');
        if(sub)sub.textContent+=' · Published to Firebase ✓';
      }).catch(e=>console.error('FB publish after load:',e));
    }
```

- [ ] **Step 4: Verify — log in as admin, click "Publish" on a cached race. Check Firebase console: RTDB → endurance_sessions and endurance_data should have entries. Reload page as regular user — race should appear in list.**

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: admin can publish endurance races to Firebase RTDB"
```

---

## Task 7: `_fbLoadRaceData()` — read race from RTDB + modify `_loadEnduranceRace()`

**Files:**
- Modify: `index.html` — add `_fbLoadRaceData()`, modify `_loadEnduranceRace()`

- [ ] **Step 1: Add `_fbLoadRaceData()` before endurance section**

```javascript
async function _fbLoadRaceData(sessionId){
  document.getElementById('endurance-table-wrap').innerHTML='<div style="color:#8a8a8a;text-align:center;padding:20px">Loading team data...</div>';
  try{
    const snap=await _fbDb.ref('endurance_data/'+sessionId).once('value');
    const data=snap.val();
    if(!data)throw new Error('No data in RTDB for '+sessionId);
    // Deserialize pitLapsMap: Array/Object → Set
    const pitLapsMap={};
    for(const [k,v] of Object.entries(data.pitLapsMap||{})){
      const arr=Array.isArray(v)?v:Object.values(v);
      pitLapsMap[k]=new Set(arr);
    }
    // RTDB may store arrays as objects with numeric keys — normalize
    const laps=Array.isArray(data.laps)?data.laps:Object.values(data.laps||{});
    const joins=Array.isArray(data.joins)?data.joins:Object.values(data.joins||{});
    _enduranceData={
      sessionId:data.sessionId,
      teams:data.teams||[],
      meta:data.meta||{},
      laps,
      events:[],// not stored in RTDB
      pitLapsMap,
      pitEntryTsMap:data.pitEntryTsMap||{},
      pitTimes:data.pitTimes||{},
      joins
    };
    _enduranceCache[sessionId]=_enduranceData;
    _renderEnduranceTeams();
  }catch(e){
    console.error('FB load race data:',e);
    document.getElementById('endurance-table-wrap').innerHTML='<div style="color:#ef4444;text-align:center;padding:20px">Failed to load race data</div>';
  }
}
```

- [ ] **Step 2: Modify `_loadEnduranceRace()` to use RTDB for non-admins**

Find `_loadEnduranceRace` (~line 949). Find the else branch:
```javascript
  } else {
    document.getElementById('endurance-table-wrap').innerHTML='<div style="color:#8a8a8a;text-align:center;padding:20px">Loading team data...</div>';
    _loadEnduranceTeamData(s.id);
  }
```

Replace with:
```javascript
  } else {
    if(!_isAdmin){
      _fbLoadRaceData(s.id);
    }else{
      document.getElementById('endurance-table-wrap').innerHTML='<div style="color:#8a8a8a;text-align:center;padding:20px">Loading team data...</div>';
      _loadEnduranceTeamData(s.id);
    }
  }
```

- [ ] **Step 3: Verify end-to-end — as regular user, open endurance page → see published races → click on one → race loads from RTDB without hitting collector (check Network tab: no requests to `ekarting.duckdns.org`).**

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: regular users load endurance race data from RTDB"
```

---

## Task 8: Apply RTDB security rules

- [ ] **Step 1: Open Firebase Console → Realtime Database → Rules tab**

- [ ] **Step 2: Replace the default rules with:**

```json
{
  "rules": {
    "endurance_sessions": {
      ".read": true,
      ".write": "auth != null"
    },
    "endurance_data": {
      ".read": true,
      ".write": "auth != null"
    }
  }
}
```

- [ ] **Step 3: Click "Publish"**

- [ ] **Step 4: Verify — open browser in incognito (not logged in), open console, type:**

```javascript
firebase.database().ref('endurance_sessions').once('value').then(s=>console.log('read ok',s.val()))
```
Should log `"read ok"` with data (or null if empty). Then try writing:
```javascript
firebase.database().ref('endurance_sessions/test').set({x:1}).catch(e=>console.log('write blocked:',e.message))
```
Should log `"write blocked: PERMISSION_DENIED"`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "docs: RTDB security rules applied (manual step documented)"
```

---

## Task 9: End-to-end smoke test

- [ ] **Step 1: Admin flow** — Open app → Endurance tab → click "Admin" button → log in with Firebase credentials → see "ADMIN" badge + "Refresh races" button. Click "Refresh races" → collector probes sessions (existing behavior). Find a race → click "Publish" → see "✓ Published" on button. Check Firebase Console: data in `/endurance_sessions` and `/endurance_data`.

- [ ] **Step 2: Regular user flow** — Open incognito window → Endurance tab → should show loading → then published races appear (no collector requests in Network tab). Click a published race → data loads instantly from RTDB. Excel export still works.

- [ ] **Step 3: Auth persistence** — Refresh page as admin → "ADMIN" badge reappears automatically (Firebase Auth persists session).

- [ ] **Step 4: Final commit**

```bash
git add index.html
git commit -m "feat: complete Firebase admin publish flow for endurance races"
git push
```
