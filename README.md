<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>ゆいきちナビ 完全版</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<!-- Leaflet -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">

<style>
/* ===== 全体 ===== */
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
}

/* ===== サイドバー ===== */
#sidebar {
  position: fixed;
  left: 0;
  top: 0;
  width: 300px;
  height: 100%;
  background: #ffffff;
  z-index: 5;
  box-shadow: 2px 0 10px rgba(0,0,0,0.15);
  padding: 12px;
  overflow-y: auto;
  transition: transform 0.3s;
}

#sidebar.closed {
  transform: translateX(-100%);
}

#sidebar h2 {
  margin: 4px 0 8px;
}

.sidebar-group {
  margin-bottom: 12px;
}

.sidebar-group label {
  font-size: 12px;
  display: block;
}

.sidebar-group input {
  width: 100%;
  padding: 6px;
}

/* ===== ボタン ===== */
button {
  padding: 8px;
  margin: 4px 0;
  width: 100%;
  font-size: 14px;
}

button.primary {
  background: #1976d2;
  color: #fff;
  border: none;
}

button.warn {
  background: #d32f2f;
  color: #fff;
  border: none;
}

/* ===== メニューボタン ===== */
#menuBtn {
  position: fixed;
  top: 12px;
  right: 12px;
  z-index: 10;
  font-size: 22px;
  padding: 8px 12px;
}

/* ===== ナビパネル ===== */
#navPanel {
  position: fixed;
  bottom: 0;
  width: 100%;
  background: rgba(255,255,255,0.95);
  z-index: 6;
  padding: 10px;
  display: none;
}

#nextGuide {
  font-size: 18px;
  font-weight: bold;
}

#remain {
  font-size: 14px;
  opacity: 0.7;
}
</style>
</head>
<body>

<!-- ===== サイドバー ===== -->
<div id="sidebar" class="open">
  <h2>ゆいきちナビ</h2>

  <div class="sidebar-group">
    <label>出発地</label>
    <input id="startInput" placeholder="現在地 / 名古屋駅 / 緯度,経度">
  </div>

  <div class="sidebar-group">
    <label>目的地</label>
    <input id="goalInput" placeholder="東京駅 / 緯度,経度">
  </div>

  <div class="sidebar-group">
    <button onclick="searchRoute()">検索</button>
    <button class="primary" onclick="startNavi()">ナビ開始</button>
    <button class="warn" onclick="stopNavi()">停止</button>
  </div>

  <div class="sidebar-group">
    <label><input type="checkbox" checked> 追尾（中央固定）</label>
    <label><input type="checkbox" checked> コンパス回転</label>
  </div>

  <div class="sidebar-group">
    <button onclick="setDummyLocation()">擬似現在地</button>
  </div>
</div>

<!-- ===== マップ ===== -->
<div id="map"></div>

<!-- ===== UI ===== -->
<button id="menuBtn">≡</button>

<div id="navPanel">
  <div id="nextGuide">案内待機中</div>
  <div id="remain">---</div>
</div>

<!-- Leaflet -->
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
/* ================================
   初期化
================================ */
const map = L.map("map").setView([35.681236, 139.767125], 16);

L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
  attribution: "© OpenStreetMap"
}).addTo(map);

/* ================================
   マーカー
================================ */
let currentMarker = L.marker([35.681236, 139.767125]).addTo(map);
let routeLine = null;

/* ================================
   サイドバー制御
================================ */
const sidebar = document.getElementById("sidebar");
document.getElementById("menuBtn").onclick = () => {
  sidebar.classList.toggle("closed");
};

/* ================================
   現在地
================================ */
function updateCurrent(lat, lng) {
  currentMarker.setLatLng([lat, lng]);
  map.setView([lat, lng]);
}

navigator.geolocation.watchPosition(pos => {
  updateCurrent(pos.coords.latitude, pos.coords.longitude);
});

/* 擬似 */
function setDummyLocation() {
  updateCurrent(35.6895, 139.6917);
}

/* ================================
   検索（ダミー）
================================ */
function searchRoute() {
  if (routeLine) map.removeLayer(routeLine);

  routeLine = L.polyline([
    currentMarker.getLatLng(),
    [35.685, 139.76],
    [35.681236, 139.767125]
  ], {color:"blue"}).addTo(map);

  document.getElementById("remain").textContent = "距離: 1.2km / 5分";
}

/* ================================
   ナビ
================================ */
function startNavi() {
  sidebar.classList.add("closed");
  document.getElementById("navPanel").style.display = "block";
  speak("ナビを開始します");
  updateGuide("次は右方向です");
}

function stopNavi() {
  sidebar.classList.remove("closed");
  document.getElementById("navPanel").style.display = "none";
  speak("ナビを終了します");
}

function updateGuide(text) {
  document.getElementById("nextGuide").textContent = text;
}

/* ================================
   音声
================================ */
function speak(text) {
  const u = new SpeechSynthesisUtterance(text);
  u.lang = "ja-JP";
  speechSynthesis.speak(u);
}

/* ================================
   地図回転（仮）
================================ */
function rotateMap(angle) {
  map.getContainer().style.transform = `rotate(${-angle}deg)`;
}
</script>

</body>
</html>
