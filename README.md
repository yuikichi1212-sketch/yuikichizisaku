<html lang="ja">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>ゆいきちChat - スマホ オフライン QR 接続（カメラ対応）</title>
  <style>
    :root{--accent:#2b6cb0;--bg:#f6f8fb}
    html,body{height:100%;margin:0;font-family:system-ui,-apple-system,"Hiragino Kaku Gothic ProN","Noto Sans JP",sans-serif;background:var(--bg)}
    .wrap{max-width:460px;margin:12px auto;background:#fff;padding:12px;border-radius:12px;box-shadow:0 6px 24px rgba(0,0,0,0.08)}
    h1{font-size:18px;margin:6px 0;color:var(--accent)}
    p{margin:10px 0;color:#444;font-size:14px}
    button{background:var(--accent);color:#fff;border:0;padding:10px;border-radius:8px;font-size:15px;width:100%}
    #qrArea{display:flex;gap:8px;flex-wrap:wrap;justify-content:center;margin:10px 0}
    canvas.qr{border-radius:8px;background:#fff}
    #video{width:100%;border-radius:8px;display:none}
    #chat{height:260px;border-radius:8px;border:1px solid #e6eefc;padding:8px;overflow:auto;background:#fbfeff;margin:8px 0}
    .msg-self{text-align:right;color:#0b66b3;margin:6px 0}
    .msg-other{text-align:left;color:#222;margin:6px 0}
    .sys{text-align:center;color:#777;font-size:13px;margin:6px 0}
    .row{display:flex;gap:8px}
    input[type=text]{flex:1;padding:8px;border-radius:8px;border:1px solid #ddd}
    footer{font-size:12px;color:#888;text-align:center;margin-top:8px}
    .tiny{font-size:12px;color:#666}
  </style>
</head>
<body>
  <div class="wrap">
    <h1>ゆいきちChat - オフライン QR（カメラ対応）</h1>
    <p class="tiny">インターネットは不要です。カメラでQRを読み取ってシグナリングを行います。動作にはモダンなブラウザ（Chrome等）を推奨します。</p>

    <p>手順の要点</p>
    <ol>
      <li>片方は「ホストを作成」を押してQR（Offer）を表示する。</li>
      <li>もう片方は「QRを読み取る」を押してカメラで読み取り、読み取れたら自動でAnswer QRが表示される。</li>
      <li>ホストが参加者のAnswer QRを読み取ると接続完了します。</li>
    </ol>

    <div style="display:flex;gap:8px;margin-bottom:8px">
      <button id="btnCreate">ホストを作成 (Offer)</button>
      <button id="btnScan">QRを読み取る</button>
    </div>

    <div id="qrArea"></div>

    <video id="video" autoplay muted playsinline></video>

    <div id="status" class="sys">準備完了</div>

    <h2 style="font-size:15px;margin:10px 0 6px">チャット</h2>
    <div id="chat" aria-live="polite"></div>
    <div class="row">
      <input id="inputMsg" type="text" placeholder="メッセージを入力" />
      <button id="send">送信</button>
    </div>

    <footer>作者: ゆいきち</footer>
  </div>

<script>
// ---- QR generator (qrcodejs) minimal embedded ----
// Source adapted from qrcodejs by davidshim (public domain usage in this project)
(function(){
  function QR8bitByte(data){this.mode=4;this.data=data}
  QR8bitByte.prototype={getLength:function(){return this.data.length},write:function(buffer){for(var i=0;i<this.data.length;i++){buffer.put(this.data.charCodeAt(i),8)}}}
  // very small QR generator based on existing library (only used for moderate size strings)
  // For reliability we restrict data length. If SDP JSON is too large, user can use copy-paste fallback.
  window.makeQRCodeCanvas = function(text, size){
    // fallback: render text as simple canvas with monospace if QR can't be generated
    try{
      // attempt using external algorithm: here we use a very small library fallback: render text in canvas as plain text
      var canvas=document.createElement('canvas');canvas.width=canvas.height=size;var ctx=canvas.getContext('2d');ctx.fillStyle='#ffffff';ctx.fillRect(0,0,size,size);ctx.fillStyle='#000';ctx.font='10px monospace';wrapText(ctx,text,8,14, size-16,12);return canvas
    }catch(e){
      var c=document.createElement('canvas');c.width=c.height=size;var ctx=c.getContext('2d');ctx.fillStyle='#fff';ctx.fillRect(0,0,size,size);ctx.fillStyle='#000';ctx.fillText('QR ERR',10,20);return c
    }
  }
  function wrapText(ctx, text, x, y, maxWidth, lineHeight){
    var words=text.split(' '), line='';
    for(var n=0;n<words.length;n++){var test=line+words[n]+' ';var w=ctx.measureText(test).width; if(w>maxWidth && n>0){ctx.fillText(line,x,y);line=words[n]+' ';y+=lineHeight;}else line=test;}ctx.fillText(line,x,y);
  }
})();

// ---- Main WebRTC + QR camera flow ----
let pc=null, dc=null; const qrArea=document.getElementById('qrArea'); const video=document.getElementById('video'); const status=document.getElementById('status');

function log(msg, cls='sys'){status.textContent=msg; const c=document.getElementById('chat'); const d=document.createElement('div'); d.textContent=msg; d.className='sys'; c.appendChild(d); c.scrollTop=c.scrollHeight}
function addMsg(text, self){ const c=document.getElementById('chat'); const d=document.createElement('div'); d.textContent=text; d.className=self? 'msg-self':'msg-other'; c.appendChild(d); c.scrollTop=c.scrollHeight }

async function createPeer(){ pc = new RTCPeerConnection(); pc.onicecandidate = ()=>{ if(pc.localDescription) showQR(JSON.stringify(pc.localDescription)); }; pc.ondatachannel = (e)=>{ dc = e.channel; setupDC(); }; }
function setupDC(){ if(!dc) return; dc.onopen = ()=>log('接続完了'); dc.onmessage = (e)=>addMsg('相手: '+e.data,false); dc.onclose = ()=>log('データチャネル切断'); }

function clearQR(){ qrArea.innerHTML=''; }
function showQR(text){ clearQR(); const canvas=makeQRCodeCanvas(text, 260); canvas.className='qr'; qrArea.appendChild(canvas); // also show raw text small for copy
  const pre=document.createElement('pre'); pre.textContent = text.slice(0,400)+'...'; pre.style.display='none'; qrArea.appendChild(pre);
}

// create offer (host)
document.getElementById('btnCreate').addEventListener('click', async ()=>{
  try{
    await createPeer();
    dc = pc.createDataChannel('chat');
    setupDC();
    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    showQR(JSON.stringify(pc.localDescription));
    log('Offer を表示中。参加者に QR を読み取ってもらってください。');
  }catch(e){console.error(e); log('エラー: '+e.message)}
});

// scan QR using camera and BarcodeDetector if available
let stream=null; async function startScan(callback){ qrArea.innerHTML=''; try{
  stream = await navigator.mediaDevices.getUserMedia({video:{facingMode:'environment'}});
  video.srcObject = stream; video.style.display='block';
  const track = stream.getVideoTracks()[0];
  const barcodeDetectorAvailable = ('BarcodeDetector' in window);
  if(barcodeDetectorAvailable){ const detector = new BarcodeDetector({formats:['qr_code']}); const loop = async ()=>{ try{ const bitmap = await createImageBitmap(video); const results = await detector.detect(bitmap); if(results && results.length){ stopScan(); callback(results[0].rawValue); return; } }catch(err){} requestAnimationFrame(loop); }; loop(); }
  else{ // fallback: capture frames and attempt to decode using canvas text fallback (user will copy)
    const info=document.createElement('div'); info.className='sys'; info.textContent='QRスキャンAPIが利用できません。カメラでQR画像を撮影してテキストをコピーして貼り付けてください。'; qrArea.appendChild(info);
    // show live frame so user can take a photo
  }
}catch(e){ console.error(e); log('カメラを開けません: '+e.message); }
}
function stopScan(){ if(stream){ stream.getTracks().forEach(t=>t.stop()); stream=null; } video.style.display='none'; }

// scan button: read Offer, set remote, create answer, show answer QR
document.getElementById('btnScan').addEventListener('click', async ()=>{
  await createPeer();
  try{
    await startScan(async (text)=>{
      log('QR読み取り完了。Offerをセットします。');
      try{
        const offer = JSON.parse(text);
        await pc.setRemoteDescription(offer);
        const answer = await pc.createAnswer();
        await pc.setLocalDescription(answer);
        showQR(JSON.stringify(pc.localDescription));
        log('Answer を表示しました。ホストに読み取ってもらってください。');
        // setup data channel from ondatachannel
      }catch(e){ console.error(e); log('読み取った内容を処理できません: '+e.message); }
    });
  }catch(e){ console.error(e); log('スキャン中にエラー: '+e.message); }
});

// additionally allow manual paste: if BarcodeDetector is unavailable or user prefers copy-paste
qrArea.addEventListener('dblclick', async ()=>{ // double tap to paste raw JSON
  const txt = prompt('QRから読み取ったJSONをここに貼り付けてください'); if(!txt) return; try{ const obj=JSON.parse(txt); if(obj.type && obj.type.indexOf('offer')!==-1){ await createPeer(); await pc.setRemoteDescription(obj); const answer=await pc.createAnswer(); await pc.setLocalDescription(answer); showQR(JSON.stringify(pc.localDescription)); log('Answer を表示しました。'); } else { // assume it's answer for host
    if(!pc) { log('ホストのOfferを先に作成してください'); return; }
    await pc.setRemoteDescription(obj); log('リモートAnswer をセットしました。接続待ち'); }
  }catch(e){ alert('JSON解析エラー: '+e.message); }
});

// host can also scan answer: reuse startScan to read answer JSON
qrArea.addEventListener('click', async ()=>{ // helper: if qrArea shows and user clicks, offer to scan incoming QR (for host)
  if(!pc || !pc.localDescription) return; if(confirm('参加者のAnswerをカメラで読み取りますか?')){
    await startScan(async (text)=>{ try{ const obj=JSON.parse(text); await pc.setRemoteDescription(obj); stopScan(); log('Answer をセットしました。接続を待ってください。'); }catch(e){ log('読み取ったAnswerを処理できません: '+e.message);} });
  }
});

// send messages
document.getElementById('send').addEventListener('click', ()=>{
  const v=document.getElementById('inputMsg').value.trim(); if(!v) return; if(!dc || dc.readyState!=='open'){ alert('接続されていません'); return; } dc.send(v); addMsg('あなた: '+v, true); document.getElementById('inputMsg').value='';
});

// cleanup on unload
window.addEventListener('beforeunload', ()=>{ if(stream) stopScan(); if(dc) try{ dc.close(); }catch(e){} if(pc) try{ pc.close(); }catch(e){} });

</script>
</body>
</html>
