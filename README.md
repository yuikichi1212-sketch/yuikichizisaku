<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>ゆいきちナビ</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">

<style>
/* ===== 基本 ===== */
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow: hidden;
  font-family: system-ui, sans-serif;
}

/* ===== マップ ===== */
#map {
  position: fixed;
  inset: 0;
  z-index: 1;
  transition: transform 0.4s ease;
}

/* ===== サイドバー ===== */
#sidebar {
  position: fixed;
  left: 0;
  top: 0;
  width: 320px;
  height: 100%;
  background: #fff;
  z-index: 5;
  box-shadow: 2px 0 10px rgba(0,0,0,.2);
  padding: 12px;
  overflow-y: auto;
  transition: transform .3s;
}

#sidebar.closed {
  transform: translateX(-100%);
}

#sidebar h1 {
  margin: 0 0 10px;
  font-size: 20px;
}

/* ===== 入力 ===== */
label { font-size: 12px; }
input, select {
  width: 100%;
  padding: 6px;
  margin-bottom: 6px;
}

/* ===== ボタン ===== */
button {
  width: 100%;
  padding: 8px;
  margin-bottom: 6px;
  font-size: 14px;
}

.primary {
  background: #1976d2;
  color: white;
  border: none;
}

/* ===== メニュー ===== */
#menuBtn {
  position: fixed;
  top: 10px;
  left: 10px;
  z-index: 10;
  font-size: 20px;
  padding: 8px 12px;
}

/* ===== ナビUI ===== */
#navUI {
  position: fixed;
  bottom: 0;
  width: 100%;
  background: rgba(255,255,255,.95);
  z-index: 6;
  padding: 10px;
  display: none;
}

#navMain {
  font-size: 18px;
  font-weight: bold;
}

#navSub {
  font-size: 14px;
  opacity: .7;
}

/* ===== 3D ===== */
.map-3d {
  transform: perspective(800px) rotateX(45deg) scale(1.2);
}
</style>
</head>

<body>

<div id="sidebar">
  <h1>ゆいきちナビ</h1>

  <label>出発地</label>
  <input id="startInput" placeholder="現在地">

  <label>目的地</label>
  <input id="goalInput" placeholder="東京駅">

  <label>移動モード</label>
  <select id="mode">
    <option value="driving">車</option>
    <option value="cycling">自転車</option>
    <option value="walking">徒歩</option>
  </select>

  <label>ルート色</label>
  <input type="color" id="routeColor" value="#1976d2">

  <label>マーカー</label>
  <select id="markerType">
    <option value="car">車</option>
    <option value="bike">自転車</option>
    <option value="walk">徒歩</option>
  </select>

  <button onclick="searchRoute()">検索</button>
  <button class="primary" onclick="startNavi()">ナビ開始</button>
  <button onclick="stopNavi()">停止</button>

  <button onclick="setDummy()">擬似現在地</button>
</div>

<button id="menuBtn">≡</button>
<div id="map"></div>

<div id="navUI">
  <div id="navMain">案内待機中</div>
  <div id="navSub">---</div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
/* ===============================
   初期化
=============================== */
const map = L.map("map", { zoomControl:false })
  .setView([35.681236,139.767125], 16);

L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png").addTo(map);

let currentPos = [35.681236,139.767125];
let routeLine, routeCoords = [], stepIndex = 0;
let navigating = false;

/* ===============================
   マーカー
=============================== */
let marker = L.marker(currentPos).addTo(map);

/* ===============================
   サイドバー
=============================== */
menuBtn.onclick = () => sidebar.classList.toggle("closed");

/* ===============================
   現在地
=============================== */
navigator.geolocation.watchPosition(p=>{
  currentPos = [p.coords.latitude, p.coords.longitude];
  marker.setLatLng(currentPos);
  if(navigating) follow();
});

/* ===============================
   ルート取得（OSRM）
=============================== */
async function searchRoute() {
  const mode = document.getElementById("mode").value;
  const color = document.getElementById("routeColor").value;

  const start = `${currentPos[1]},${currentPos[0]}`;
  const goal = "139.767125,35.681236";

  const url = `https://router.project-osrm.org/route/v1/${mode}/${start};${goal}?geometries=geojson&steps=true`;

  const res = await fetch(url);
  const data = await res.json();

  routeCoords = data.routes[0].geometry.coordinates.map(c=>[c[1],c[0]]);

  if(routeLine) map.removeLayer(routeLine);
  routeLine = L.polyline(routeCoords,{color,weight:6}).addTo(map);

  stepIndex = 0;
}

/* ===============================
   ナビ
=============================== */
function startNavi() {
  navigating = true;
  sidebar.classList.add("closed");
  navUI.style.display = "block";
  map.getContainer().classList.add("map-3d");
  speak("ナビを開始します");
}

function stopNavi() {
  navigating = false;
  navUI.style.display = "none";
  map.getContainer().classList.remove("map-3d");
  speak("ナビを終了します");
}

/* ===============================
   追尾 & 回転
=============================== */
function follow() {
  map.setView(currentPos);
  if(routeCoords[stepIndex+1]) {
    const a = currentPos;
    const b = routeCoords[stepIndex+1];
    const angle = Math.atan2(b[1]-a[1],b[0]-a[0])*180/Math.PI;
    map.getContainer().style.transform =
      `rotate(${-angle}deg)`;
  }
}

/* ===============================
   リルート検知
=============================== */
function distance(a,b){
  return Math.hypot(a[0]-b[0],a[1]-b[1]);
}

setInterval(()=>{
  if(!navigating) return;
  if(routeCoords.length===0) return;

  const d = distance(currentPos, routeCoords[stepIndex]);
  if(d > 0.0005) {
    speak("ルートを再探索します");
    searchRoute();
  }
},3000);

/* ===============================
   音声案内
=============================== */
function speak(text){
  const u = new SpeechSynthesisUtterance(text);
  u.lang = "ja-JP";
  speechSynthesis.speak(u);
}

/* ===============================
   擬似移動
=============================== */
function setDummy(){
  let i = 0;
  setInterval(()=>{
    if(routeCoords[i]){
      currentPos = routeCoords[i];
      marker.setLatLng(currentPos);
      i++;
    }
  },800);
}
</script>
</body>
</html>
