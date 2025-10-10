<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>リアルタイムチャット（Firebase）</title>

<style>
  :root{--bg:#f7f7f8;--card:#ffffff;--muted:#666;--accent:#2b7cff}
  html,body{height:100%;margin:0;font-family:system-ui,-apple-system,'Segoe UI',Roboto,'Hiragino Kaku Gothic ProN','Noto Sans JP',sans-serif;background:var(--bg)}
  .wrap{max-width:980px;margin:24px auto;padding:16px;display:grid;grid-template-columns:320px 1fr;gap:16px}
  .panel{background:var(--card);border-radius:8px;box-shadow:0 2px 12px rgba(0,0,0,0.06);padding:12px}
  .left .row{margin-bottom:8px}
  label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
  input[type=text], input[type=search]{width:100%;padding:10px;border:1px solid #ddd;border-radius:6px;font-size:14px}
  button{width:100%;padding:10px;border-radius:6px;border:1px solid #ccc;background:#fafafa;cursor:pointer}
  .chat{display:flex;flex-direction:column;height:70vh}
  .messages{flex:1;overflow:auto;padding:12px;border-radius:6px;border:1px solid #eee;background:#fff}
  .msg{margin-bottom:10px}
  .meta{font-size:12px;color:var(--muted);margin-bottom:4px}
  .text{font-size:15px;line-height:1.4;white-space:pre-wrap}
  .inputRow{display:flex;gap:8px;margin-top:8px}
  .inputRow input{flex:1}
  .small{font-size:12px;color:var(--muted);margin-top:8px}
  @media(max-width:880px){.wrap{grid-template-columns:1fr;max-width:680px}}
</style>
</head>
<body>

<div class="wrap">
  <div class="panel left">
    <div class="row">
      <label>表示名（任意）</label>
      <input id="name" type="text" placeholder="例: ゆいきち" />
    </div>
    <div class="row">
      <label>ルーム名</label>
      <input id="room" type="text" placeholder="例: friends" />
    </div>
    <div class="row">
      <button id="join">ルームに参加</button>
      <button id="leave" style="margin-top:8px" disabled>退出</button>
    </div>
    <div class="row small" id="status">未参加</div>
    <hr />
    <div class="small">注意: Firebase の設定を下に貼ってください（安全上、rules を適切に設定してください）。</div>
  </div>

  <div class="panel chat">
    <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:8px">
      <div style="font-weight:600">チャット</div>
      <div style="font-size:13px;color:var(--muted)"><span id="currentRoom">-</span></div>
    </div>

    <div class="messages" id="messages"></div>

    <div class="inputRow">
      <input id="msgInput" type="text" placeholder="メッセージを入力して Enter で送信" disabled />
      <button id="send" disabled>送信</button>
    </div>
    <div class="small">最新200件のみ表示。Enterで送信。ページを閉じると自分の投稿は消えません。</div>
  </div>
</div>

<!-- Firebase compat SDK -->
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database-compat.js"></script>

<script>
/*
  使い方:
  1) 下の firebaseConfig にあなたの設定を貼る
  2) Firebase Realtime Database を有効化して、rules を適切に設定してください
*/

const firebaseConfig = {
  // ここにあなたの firebaseConfig を貼ってください
  // 例:
  // apiKey: "...",
  // authDomain: "...",
  // databaseURL: "https://yourproject-default-rtdb.firebaseio.com",
  // projectId: "...",
  // storageBucket: "...",
  // messagingSenderId: "...",
  // appId: "..."
};

// 初期化
let app, db;
try {
  app = firebase.initializeApp(firebaseConfig);
  db = firebase.database();
} catch (e) {
  console.warn('Firebase 初期化エラー', e);
}

// UI
const joinBtn = document.getElementById('join');
const leaveBtn = document.getElementById('leave');
const nameInput = document.getElementById('name');
const roomInput = document.getElementById('room');
const statusEl = document.getElementById('status');
const messagesEl = document.getElementById('messages');
const sendBtn = document.getElementById('send');
const msgInput = document.getElementById('msgInput');
const currentRoomEl = document.getElementById('currentRoom');

let clientId = 'u_' + Math.random().toString(36).slice(2,9);
let joinedRoom = null;
let messagesRef = null;
let childAddedListener = null;

function setStatus(t){ statusEl.textContent = t; }

function escapeHtml(str){
  return str.replace(/[&<>"']/g, s => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[s]));
}

function joinRoom(){
  const room = (roomInput.value||'').trim();
  if(!room) return alert('ルーム名を入力してください');
  if(!db) return alert('Firebase が初期化されていません。firebaseConfig を貼ってください');

  joinedRoom = room;
  currentRoomEl.textContent = room;
  setStatus('参加中: ' + room);
  joinBtn.disabled = true;
  leaveBtn.disabled = false;
  msgInput.disabled = false;
  sendBtn.disabled = false;

  // 最新200件を監視
  messagesRef = db.ref(`rooms/${room}/messages`);
  const query = messagesRef.limitToLast(200);
  // 既存を消してから再登録
  messagesEl.innerHTML = '';

  childAddedListener = query.on('child_added', snapshot => {
    const obj = snapshot.val();
    if(!obj) return;
    appendMessage(obj);
  });

  // Enterで送信
  msgInput.focus();
}

function leaveRoom(){
  if(!joinedRoom) return;
  if(messagesRef && childAddedListener){
    // 解除: on から off にする
    messagesRef.off('child_added', childAddedListener);
  } else if(messagesRef){
    messagesRef.off();
  }
  joinedRoom = null;
  currentRoomEl.textContent = '-';
  setStatus('未参加');
  joinBtn.disabled = false;
  leaveBtn.disabled = true;
  msgInput.disabled = true;
  sendBtn.disabled = true;
  messagesEl.innerHTML = '';
}

function appendMessage(obj){
  const wrap = document.createElement('div');
  wrap.className = 'msg';
  const time = new Date(obj.ts || Date.now());
  const hh = ('0' + time.getHours()).slice(-2);
  const mm = ('0' + time.getMinutes()).slice(-2);
  const ss = ('0' + time.getSeconds()).slice(-2);
  const meta = document.createElement('div');
  meta.className = 'meta';
  meta.textContent = `${escapeHtml(obj.name || '名無し')} ・ ${hh}:${mm}:${ss}`;
  const text = document.createElement('div');
  text.className = 'text';
  text.textContent = obj.text || '';
  wrap.appendChild(meta);
  wrap.appendChild(text);
  messagesEl.appendChild(wrap);
  // 上限を保つ: 最新200件のみDOMに残す
  while(messagesEl.children.length > 220){
    messagesEl.removeChild(messagesEl.firstChild);
  }
  messagesEl.scrollTop = messagesEl.scrollHeight;
}

function sendMessage(){
  const txt = (msgInput.value || '').trim();
  if(!txt) return;
  if(!joinedRoom) return alert('先にルームに参加してください');
  const name = (nameInput.value || '').trim() || '名無し';
  const payload = { name, text: txt, ts: Date.now(), from: clientId };
  db.ref(`rooms/${joinedRoom}/messages`).push(payload).catch(err => {
    console.warn('送信エラー', err);
    alert('メッセージ送信に失敗しました');
  });
  msgInput.value = '';
  msgInput.focus();
}

joinBtn.addEventListener('click', joinRoom);
leaveBtn.addEventListener('click', leaveRoom);
sendBtn.addEventListener('click', sendMessage);
msgInput.addEventListener('keydown', (e)=>{
  if(e.key === 'Enter') sendMessage();
});

// Ctrl/Cmd+K で入力にフォーカス（ショートカット）
window.addEventListener('keydown', (e)=>{
  if((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'k'){
    e.preventDefault();
    msgInput.focus();
  }
});

// 最低限の確認メッセージ
if(!firebaseConfig || !firebaseConfig.apiKey){
  setStatus('firebaseConfig を貼ってください');
}
</script>
</body>
</html>
