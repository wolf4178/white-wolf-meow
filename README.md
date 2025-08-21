
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>SnapLite — single file</title>
  <style>
    :root{--bg:#0b1020;--card:#0f1724;--muted:#98a0b3;--accent:#ff5aa5}
    html,body{height:100%;margin:0;font-family:Inter,ui-sans-serif,system-ui,Segoe UI,Roboto,'Helvetica Neue',Arial}
    body{background:linear-gradient(180deg,#05060a,#071128);color:#e6eef8;display:flex;align-items:center;justify-content:center;padding:18px}
    .app{width:980px;max-width:100%;display:grid;grid-template-columns:340px 1fr;gap:18px}
    .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), transparent);border:1px solid rgba(255,255,255,0.04);padding:12px;border-radius:12px}
    header{display:flex;align-items:center;gap:10px}
    img.avatar{width:40px;height:40px;border-radius:50%;object-fit:cover}
    button{background:var(--accent);color:#071128;border:none;padding:8px 12px;border-radius:10px;cursor:pointer}
    input,textarea{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:8px;border-radius:8px;color:inherit;width:100%;outline:none}
    .list{max-height:420px;overflow:auto;margin-top:8px}
    .friend{display:flex;gap:10px;align-items:center;padding:8px;border-radius:8px}
    .msg{max-width:70%;padding:8px;border-radius:10px;margin:6px 0}
    .msg.me{margin-left:auto;background:linear-gradient(90deg,#ff8ac1,#ff5aa5);color:#071128}
    .msg.you{background:#0b1220;border:1px solid rgba(255,255,255,0.03)}
    .small{font-size:12px;color:var(--muted)}
    .stories{display:flex;gap:8px;overflow:auto}
    .story{width:64px;height:64px;border-radius:999px;overflow:hidden;border:3px solid rgba(255,255,255,0.06)}
    video{width:100%;border-radius:12px}
    .camera-actions{display:flex;gap:8px;margin-top:8px}
    .topbar{display:flex;justify-content:space-between;align-items:center}
    .hidden{display:none}
  </style>
</head>
<body>
  <div class="app">
    <div class="card">
      <div class="topbar">
        <div style="display:flex;gap:10px;align-items:center">
          <div style="font-weight:700">SnapLite</div>
          <div class="small">single-file demo</div>
        </div>
        <div id="auth-area"></div>
      </div>

      <div id="profile" class="small" style="margin-top:10px"></div>

      <section style="margin-top:12px">
        <div style="font-weight:600">Stories</div>
        <div id="stories" class="stories" style="margin-top:8px"></div>
        <div style="margin-top:8px">
          <input id="story-file" type="file" accept="image/*" />
          <button id="share-story">Share story</button>
        </div>
      </section>

      <section style="margin-top:12px">
        <div style="font-weight:600">Friends</div>
        <div style="display:flex;gap:8px;margin-top:8px">
          <input id="friend-uid" placeholder="Friend UID" />
          <button id="add-friend">Add</button>
        </div>
        <div class="list" id="friends-list"></div>
      </section>
    </div>

    <div class="card" style="display:flex;flex-direction:column;height:720px">
      <div style="display:flex;gap:8px;align-items:center;justify-content:space-between">
        <div style="font-weight:700">Chat</div>
        <div class="small">Open a chat with a friend by clicking them</div>
      </div>

      <div id="chat-area" style="display:flex;gap:12px;margin-top:12px;flex:1">
        <div style="flex:1;display:flex;flex-direction:column;height:100%">
          <div id="messages" style="flex:1;overflow:auto;padding:8px;border-radius:10px;background:linear-gradient(180deg,rgba(0,0,0,0.2),transparent)"></div>
          <div style="margin-top:8px;display:flex;gap:8px;align-items:center">
            <input id="msg-input" placeholder="Type a message" />
            <button id="send-btn">Send</button>
            <button id="ghost-btn">Send (Disappearing)</button>
            <button id="attach-img">📷</button>
            <input id="img-file" type="file" accept="image/*" class="hidden" />
          </div>
        </div>

        <div style="width:320px">
          <div style="font-weight:600">Camera</div>
          <video id="video" autoplay playsinline muted></video>
          <div class="camera-actions">
            <button id="capture">Capture</button>
            <button id="flip">Flip</button>
            <button id="stop-cam">Stop</button>
          </div>
          <canvas id="canvas" class="hidden"></canvas>
          <div style="margin-top:12px">
            <div style="font-weight:600">Presence / Debug</div>
            <div id="debug" class="small"></div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Firebase (compat) CDN for quick demo. For production use modular SDK. -->
  <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-auth-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-firestore-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-storage-compat.js"></script>

  <script>
    // === REPLACE THIS WITH YOUR FIREBASE KEYS ===
    const FIREBASE_CONFIG = {
      apiKey: "REPLACE_ME",
      authDomain: "REPLACE_ME.firebaseapp.com",
      projectId: "REPLACE_ME",
      storageBucket: "REPLACE_ME.appspot.com",
      messagingSenderId: "REPLACE_ME",
      appId: "REPLACE_ME"
    }

    firebase.initializeApp(FIREBASE_CONFIG)
    const auth = firebase.auth()
    const db = firebase.firestore()
    const storage = firebase.storage()

    // Elements
    const authArea = document.getElementById('auth-area')
    const profileDiv = document.getElementById('profile')
    const friendsList = document.getElementById('friends-list')
    const friendUidInput = document.getElementById('friend-uid')
    const addFriendBtn = document.getElementById('add-friend')
    const sendBtn = document.getElementById('send-btn')
    const ghostBtn = document.getElementById('ghost-btn')
    const msgInput = document.getElementById('msg-input')
    const messagesDiv = document.getElementById('messages')
    const videoEl = document.getElementById('video')
    const captureBtn = document.getElementById('capture')
    const flipBtn = document.getElementById('flip')
    const stopCamBtn = document.getElementById('stop-cam')
    const canvas = document.getElementById('canvas')
    const debug = document.getElementById('debug')
    const storyFile = document.getElementById('story-file')
    const shareStoryBtn = document.getElementById('share-story')
    const storiesDiv = document.getElementById('stories')
    const attachImg = document.getElementById('attach-img')
    const imgFile = document.getElementById('img-file')

    let currentPeerUid = null
    let me = null

    // Simple UI: sign in/out
    function renderAuth(u){
      authArea.innerHTML = ''
      if(!u){
        const btn = document.createElement('button')
        btn.textContent = 'Sign in with Google'
        btn.onclick = ()=>{
          const p = new firebase.auth.GoogleAuthProvider()
          auth.signInWithPopup(p)
        }
        authArea.appendChild(btn)
      } else {
        const img = document.createElement('img')
        img.className = 'avatar'
        img.src = u.photoURL || `https://ui-avatars.com/api/?name=${encodeURIComponent(u.displayName||'User')}`
        const span = document.createElement('span')
        span.textContent = u.displayName||u.email
        const btn = document.createElement('button')
        btn.textContent = 'Sign out'
        btn.onclick = ()=>auth.signOut()
        authArea.appendChild(img); authArea.appendChild(span); authArea.appendChild(btn)
      }
    }

    // Auth listener
    auth.onAuthStateChanged(async user=>{
      me = user
      renderAuth(user)
      profileDiv.textContent = user ? `${user.displayName || user.email} • UID: ${user.uid}` : 'Not signed in'
      if(user) initAfterAuth()
      else cleanupAfterSignout()
    })

    // ====== After sign in: load friends, stories, presence ======
    let friendsUnsub = null, storiesUnsub = null

    function initAfterAuth(){
      const uid = me.uid

      // ensure user doc exists
      db.collection('users').doc(uid).set({ displayName: me.displayName || null, photoURL: me.photoURL || null, createdAt: firebase.firestore.FieldValue.serverTimestamp() }, { merge:true })

      // friends
      const friendsRef = db.collection('users').doc(uid).collection('friends')
      friendsUnsub = friendsRef.onSnapshot(snap=>{
        friendsList.innerHTML = ''
        snap.forEach(d=>{
          const f = d.data()
          const el = document.createElement('div')
          el.className = 'friend'
          el.innerHTML = `<img class="avatar" src="${f.photoURL||'https://ui-avatars.com/api/?name=User'}"/><div style="flex:1"><div style="font-weight:600">${f.displayName||f.uid}</div><div class="small">UID: ${f.uid}</div></div>`
          el.onclick = ()=>openChatWith(f.uid)
          friendsList.appendChild(el)
        })
      })

      // stories: show all stories from friends + me (simple 24h filter done client-side)
      storiesUnsub = db.collection('stories').orderBy('createdAt','desc').onSnapshot(snap=>{
        storiesDiv.innerHTML = ''
        const now = Date.now()
        snap.forEach(d=>{
          const s = d.data()
          const created = s.createdAt ? s.createdAt.toDate().getTime() : now
          if(now - created > 24*60*60*1000) return // expired
          const b = document.createElement('button')
          b.className = 'story'
          b.onclick = ()=> window.open(s.photoUrl,'_blank')
          b.innerHTML = `<img src="${s.photoUrl}" style="width:100%;height:100%;object-fit:cover" />`
          storiesDiv.appendChild(b)
        })
      })

      // presence (very basic) - set a field on my user doc
      const userRef = db.collection('users').doc(uid)
      userRef.set({ lastSeen: firebase.firestore.FieldValue.serverTimestamp(), online:true }, { merge:true })
      window.addEventListener('beforeunload', ()=> userRef.set({ online:false }, { merge:true }))

      debug.innerText = 'Connected'

      // listen for incoming friend requests? (skipped)
    }

    function cleanupAfterSignout(){
      profileDiv.textContent = 'Not signed in'
      friendsList.innerHTML = ''
      storiesDiv.innerHTML = ''
      messagesDiv.innerHTML = ''
      if(friendsUnsub) friendsUnsub(); friendsUnsub=null
      if(storiesUnsub) storiesUnsub(); storiesUnsub=null
      debug.innerText = ''
    }

    // Add friend by UID — writes reciprocal simple friend records
    addFriendBtn.onclick = async ()=>{
      const uid = friendUidInput.value.trim(); if(!uid) return alert('Enter UID')
      if(!me) return alert('Sign in first')
      try{
        // read user doc to get their displayName/photo
        const snap = await db.collection('users').doc(uid).get()
        const data = snap.exists ? snap.data() : { uid }
        await db.collection('users').doc(me.uid).collection('friends').doc(uid).set({ uid, displayName: data.displayName || null, photoURL: data.photoURL || null })
        await db.collection('users').doc(uid).collection('friends').doc(me.uid).set({ uid: me.uid, displayName: me.displayName || null, photoURL: me.photoURL || null })
        friendUidInput.value = ''
      }catch(e){ alert(e.message) }
    }

    // Share story
    shareStoryBtn.onclick = async ()=>{
      const f = storyFile.files?.[0]; if(!f) return alert('Pick an image')
      const key = `stories/${me.uid}/${Date.now()}_${f.name}`
      const r = storage.ref(key)
      const snap = await r.put(f)
      const url = await r.getDownloadURL()
      await db.collection('stories').add({ uid: me.uid, photoUrl: url, createdAt: firebase.firestore.FieldValue.serverTimestamp() })
      storyFile.value = ''
      alert('Story shared')
    }

    // ===== Chat logic =====
    let msgsUnsub = null
    function makeRoomId(a,b){ return [a,b].sort().join('_') }

    function openChatWith(peerUid){
      currentPeerUid = peerUid
      messagesDiv.innerHTML = ''
      msgInput.focus()

      // show existing messages + realtime
      if(msgsUnsub) msgsUnsub();
      const room = makeRoomId(me.uid, peerUid)
      const msgsRef = db.collection('rooms').doc(room).collection('messages').orderBy('createdAt','asc')
      msgsUnsub = msgsRef.onSnapshot(async snap=>{
        messagesDiv.innerHTML = ''
        for(const d of snap.docs){
          const m = d.data();
          const el = document.createElement('div')
          el.className = 'msg '+(m.from===me.uid? 'me':'you')
          if(m.image) el.innerHTML = `<div><img src="${m.image}" style="max-width:240px;border-radius:8px;display:block;margin-bottom:6px"/></div>`
          if(m.body) el.innerHTML += `<div>${escapeHtml(m.body)}</div>`
          if(m.ephemeral) el.innerHTML += `<div class="small">disappearing</div>`
          messagesDiv.appendChild(el)
          // mark seen
          if(m.to===me.uid && !m.seen){
            await d.ref.update({ seen:true })
            // if ephemeral, delete immediately
            if(m.ephemeral) await d.ref.delete()
          }
        }
        messagesDiv.scrollTop = messagesDiv.scrollHeight
      })
    }

    sendBtn.onclick = ()=> sendMessage(false)
    ghostBtn.onclick = ()=> sendMessage(true)

    async function sendMessage(ephemeral=false){
      if(!me) return alert('Sign in')
      if(!currentPeerUid) return alert('Select a friend to chat with')
      const body = msgInput.value.trim();
      if(!body) return
      const room = makeRoomId(me.uid, currentPeerUid)
      await db.collection('rooms').doc(room).collection('messages').add({ from: me.uid, to: currentPeerUid, body, ephemeral, seen:false, createdAt: firebase.firestore.FieldValue.serverTimestamp() })
      msgInput.value = ''
    }

    // Attach image via file picker
    attachImg.onclick = ()=> imgFile.click()
    imgFile.onchange = async ()=>{
      const f = imgFile.files?.[0]; if(!f) return
      if(!me || !currentPeerUid) return alert('Sign in and open chat')
      const key = `chats/${makeRoomId(me.uid,currentPeerUid)}/${Date.now()}_${f.name}`
      const r = storage.ref(key)
      await r.put(f)
      const url = await r.getDownloadURL()
      await db.collection('rooms').doc(makeRoomId(me.uid,currentPeerUid)).collection('messages').add({ from:me.uid,to:currentPeerUid,image:url,ephemeral:false,seen:false,createdAt: firebase.firestore.FieldValue.serverTimestamp() })
      imgFile.value = ''
    }

    // ===== Camera capture =====
    let stream = null
    let facingMode = 'user'
    async function startCam(){
      stopCam()
      try{
        stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode }, audio:false })
        videoEl.srcObject = stream
        await videoEl.play()
      }catch(e){ console.error(e); alert('Camera blocked or not available') }
    }
    function stopCam(){ if(stream){ stream.getTracks().forEach(t=>t.stop()); stream=null; videoEl.srcObject = null } }
    captureBtn.onclick = ()=>{
      if(!stream) return alert('Start camera first')
      canvas.width = videoEl.videoWidth; canvas.height = videoEl.videoHeight
      const ctx = canvas.getContext('2d'); ctx.drawImage(videoEl,0,0)
      canvas.toBlob(async blob =>{
        // upload to storage and send as chat image
        if(!me || !currentPeerUid) return alert('Open a chat to send image')
        const key = `chats/${makeRoomId(me.uid,currentPeerUid)}/${Date.now()}.jpg`
        const r = storage.ref(key)
        await r.put(blob)
        const url = await r.getDownloadURL()
        await db.collection('rooms').doc(makeRoomId(me.uid,currentPeerUid)).collection('messages').add({ from:me.uid,to:currentPeerUid,image:url,ephemeral:false,seen:false,createdAt: firebase.firestore.FieldValue.serverTimestamp() })
      }, 'image/jpeg', 0.9)
    }
    flipBtn.onclick = ()=>{ facingMode = (facingMode==='user')? 'environment':'user'; startCam() }
    stopCamBtn.onclick = ()=> stopCam()

    // Start camera automatically (ask permission)
    startCam()

    // Helper
    function escapeHtml(s){ return s.replaceAll('&','&amp;').replaceAll('<','&lt;').replaceAll('>','&gt;') }

    // Small UX: clicking a friend by UID opens chat (handled in friends listener)

    // Notes: This single-file demo intentionally simplifies many things:
    // - No strict security checks (use Firestore rules in production)
    // - Uses compat SDK for ease of embedding
    // - Ephemeral messages are removed client-side when seen; production should use security rules + server cleanup
    // - Stories do not auto-expire on server (you can enable Firestore TTL)

  </script>
</body>
</html>


