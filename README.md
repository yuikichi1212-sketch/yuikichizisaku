<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>ゆいきちChat - スマホオフライン版</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      font-family: "Noto Sans JP", sans-serif;
      background: #f3f6f9;
      margin: 0;
      padding: 0;
      display: flex;
      justify-content: center;
    }
    main {
      width: 100%;
      max-width: 420px;
      background: #fff;
      margin: 1rem;
      padding: 1rem;
      border-radius: 20px;
      box-shadow: 0 4px 18px rgba(0,0,0,0.1);
    }
    h1 {
      text-align: center;
      color: #3b82f6;
      margin: 0.5em 0 0.2em;
    }
    h2 {
      font-size: 1rem;
      margin-top: 1.2em;
      color: #444;
    }
    button {
      background: #3b82f6;
      color: white;
      border: none;
      border-radius: 8px;
      padding: 10px;
      font-size: 1rem;
      width: 100%;
      margin-top: 0.5rem;
    }
    button:hover { background: #2563eb; }
    #chat {
      height: 300px;
      border: 1px solid #ddd;
      border-radius: 10px;
      background: #fafafa;
      overflow-y: auto;
      padding: 8px;
      margin-bottom: 8px;
      font-size: 0.95rem;
    }
    .msg-self { text-align: right; color: #0b7dda; margin: 3px 0; }
    .msg-other { text-align: left; color: #222; margin: 3px 0; }
    .sys { text-align: center; color: #777; font-size: 0.8rem; margin: 5px 0; }
    input[type=text] {
      width: calc(100% - 80px);
      padding: 6px;
      border-radius: 8px;
      border: 1px solid #ccc;
      font-size: 1rem;
    }
    #qr {
      display: flex;
      justify-content: center;
      margin-top: 1rem;
    }
    canvas {
      border-radius: 8px;
      background: #fff;
    }
    footer {
      text-align: center;
      font-size: 0.8rem;
      color: #777;
      margin-top: 1.5rem;
    }
  </style>
</head>
<body>
<main>
  <h1> ゆいきちChat</h1>
  <p style="text-align:center;">インターネット不要・QR接続対応</p>

  <h2>① モード選択</h2>
  <button id="create">ホストとして開始（Offer）</button>
  <button id="answer">参加者として開始（Answer）</button>

  <div id="qr"></div>

  <h2>② チャット</h2>
  <div id="chat"></div>
  <div style="display:flex; gap:4px;">
    <input id="msg" type="text" placeholder="メッセージ..." />
    <button id="send">送信</button>
  </div>

  <footer>オフラインP2P通信 - 作者：ゆいきち</footer>
</main>

<script src="https://cdn.jsdelivr.net/npm/qrcode@1.5.3/build/qrcode.min.js"></script>
<script>
let pc, dc;
const chat = document.getElementById('chat');
const log = (msg, cls="sys") => {
  const p = document.createElement('div');
  p.className = cls;
  p.textContent = msg;
  chat.appendChild(p);
  chat.scrollTop = chat.scrollHeight;
};

async function createPeer() {
  pc = new RTCPeerConnection();
  pc.onicecandidate = e => {
    if (pc.localDescription) {
      showQR(JSON.stringify(pc.localDescription));
    }
  };
  pc.ondatachannel = e => {
    dc = e.channel;
    setupDC();
  };
}

function setupDC() {
  dc.onopen = () => log(" 接続完了！", "sys");
  dc.onmessage = e => log(" 相手: " + e.data, "msg-other");
}

function showQR(text) {
  const qrDiv = document.getElementById('qr');
  qrDiv.innerHTML = "";
  QRCode.toCanvas(text, { width: 200 }, (err, canvas) => {
    if (!err) qrDiv.appendChild(canvas);
  });
}

document.getElementById('create').onclick = async () => {
  await createPeer();
  dc = pc.createDataChannel("chat");
  setupDC();
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);
  log(" Offer QRを相手に読み取ってもらってください。", "sys");
};

document.getElementById('answer').onclick = async () => {
  await createPeer();
  const offerJSON = prompt("ホスト側スマホのQRコードをスキャンし、そのJSONを貼り付けてください。");
  if (!offerJSON) return;
  const offer = JSON.parse(offerJSON);
  await pc.setRemoteDescription(offer);
  const answer = await pc.createAnswer();
  await pc.setLocalDescription(answer);
  showQR(JSON.stringify(pc.localDescription));
  log(" Answer QRをホストに見せてください。", "sys");
};

document.getElementById('send').onclick = () => {
  const msg = document.getElementById('msg').value.trim();
  if (!msg) return;
  if (!dc || dc.readyState !== "open") {
    alert("接続がまだ完了していません。");
    return;
  }
  dc.send(msg);
  log(" あなた: " + msg, "msg-self");
  document.getElementById('msg').value = "";
};
</script>
</body>
</html>
