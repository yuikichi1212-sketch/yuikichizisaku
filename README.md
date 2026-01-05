<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>ゆいきちナビ</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<!-- Leaflet -->
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">

<style>
/* ==============================
   基本レイアウト
============================== */
html,body{
  margin:0;
  padding:0;
  height:100%;
  overflow:hidden;
  font-family:system-ui,-apple-system,BlinkMacSystemFont;
}

/* ==============================
   マップ
============================== */
#map{
  position:fixed;
  inset:0;
  z-index:1;
  transition:transform .5s ease;
}

/* 3Dモード */
.map-3d{
  transform:
    perspective(900px)
    rotateX(45deg)
    scale(1.15);
}

/* ==============================
   サイドバー
============================== */
#sidebar{
  position:fixed;
  left:0;
  top:0;
  width:320px;
  height:100%;
  background:#fff;
  z-index:5;
  box-shadow:2px 0 12px rgba(0,0,0,.25);
  padding:12px;
  overflow:auto;
  transition:transform .3s;
}
#sidebar.closed{transform:translateX(-100%);}

#sidebar h1{
  margin:0 0 10px;
  font-size:20px;
}

/* ==============================
   UI共通
============================== */
label{font-size:12px;}
input,select{
  width:100%;
  padding:6px;
  margin-bottom:6px;
}
button{
  width:100%;
  padding:8px;
  margin-bottom:6px;
}
.primary{
  background:#1976d2;
  color:#fff;
  border:none;
}

/* ==============================
   メニューボタン
============================== */
#menuBtn{
  position:fixed;
  top:10px;
  left:10px;
  z-index:10;
  font-size:22px;
  padding:8px 12px;
}

/* ==============================
   ナビUI
============================== */
#navUI{
  position:fixed;
  bottom:0;
  width:100%;
  background:rgba(255,255,255,.95);
  z-index:6;
  padding:10px;
  display:none;
}
#navMain{font-size:18px;font-weight:bold;}
#navSub{font-size:14px;opacity:.7;}

/* ==============================
   夜間
============================== */
body.night{
  background:#000;
}
body.night #navUI{
  background:rgba(0,0,0,.8);
  color:#fff;
}
</style>
</head>

<body>

<!-- ============================
     サイドバー
============================= -->
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
    <option value="車">車</option>
    <option value="自転車">自転車</option>
    <option value="徒歩">徒歩</option>
  </select>

  <button onclick="searchRoute()">検索</button>
  <button class="primary" onclick="startNavi()">ナビ開始</button>
  <button onclick="stopNavi()">停止</button>

  <button onclick="startDummy()">擬似走行</button>
</div>

<button id="menuBtn">≡</button>
<div id="map"></div>

<div id="navUI">
  <div id="navMain">案内待機中</div>
  <div id="navSub">---</div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
/* ==============================
   設定・状態
============================== */
const CONFIG={
  rerouteDistance:40,
  followZoom:18
};

let STATE={
  current:[35.681236,139.767125],
  goal:null,
  route:[],
  steps:[],
  navigating:false,
  stepIndex:0
};

/* ==============================
   地図初期化
============================== */
const map=L.map("map",{zoomControl:false})
  .setView(STATE.current,16);

const tileDay=L.tileLayer(
  "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
).addTo(map);

const tileNight=L.tileLayer(
  "https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png"
);

/* ==============================
   マーカー
============================== */
let marker=L.marker(STATE.current).addTo(map);
let routeLine=null;

/* ==============================
   UI
============================== */
menuBtn.onclick=()=>sidebar.classList.toggle("closed");

/* ==============================
   現在地取得
============================== */
navigator.geolocation.watchPosition(p=>{
  STATE.current=[p.coords.latitude,p.coords.longitude];
  marker.setLatLng(STATE.current);
  if(STATE.navigating) follow();
});

/* ==============================
   検索（Nominatim）
============================== */
async function geocode(q){
  const url=
    "https://nominatim.openstreetmap.org/search"+
    "?format=json&limit=1&q="+encodeURIComponent(q);
  const r=await fetch(url,{
    headers:{ "User-Agent":"yuikichi-navi" }
  });
  const j=await r.json();
  if(!j[0]) throw "not found";
  return [Number(j[0].lat),Number(j[0].lon)];
}

/* ==============================
   ルート取得（OSRM）
============================== */
async function searchRoute(){
  const mode=modeSel();
  STATE.goal=await geocode(goalInput.value);

  const start=`${STATE.current[1]},${STATE.current[0]}`;
  const goal=`${STATE.goal[1]},${STATE.goal[0]}`;

  const url=
    `https://router.project-osrm.org/route/v1/${mode}/${start};${goal}`+
    `?geometries=geojson&steps=true`;

  const r=await fetch(url);
  const j=await r.json();

  STATE.route=j.routes[0].geometry.coordinates
    .map(c=>[c[1],c[0]]);
  STATE.steps=j.routes[0].legs[0].steps;

  if(routeLine) map.removeLayer(routeLine);
  routeLine=L.polyline(
    STATE.route,
    {color:routeColor.value,weight:6}
  ).addTo(map);

  speak("ルートを検索しました");
}

/* ==============================
   ナビ開始
============================== */
function startNavi(){
  STATE.navigating=true;
  sidebar.classList.add("closed");
  navUI.style.display="block";
  map.getContainer().classList.add("map-3d");
  speak("ナビを開始します");
}

/* ==============================
   ナビ停止
============================== */
function stopNavi(){
  STATE.navigating=false;
  navUI.style.display="none";
  map.getContainer().classList.remove("map-3d");
  speak("ナビを終了します");
}

/* ==============================
   追尾・回転
============================== */
function follow(){
  map.setView(STATE.current,CONFIG.followZoom);
  if(!STATE.route[STATE.stepIndex+1]) return;

  const a=L.latLng(STATE.current);
  const b=L.latLng(STATE.route[STATE.stepIndex+1]);

  const angle=
    Math.atan2(b.lng-a.lng,b.lat-a.lat)*180/Math.PI;
  map.getContainer().style.transform=
    `rotate(${-angle}deg)`;
}

/* ==============================
   リルート
============================== */
function dist(a,b){
  return L.latLng(a).distanceTo(b);
}

setInterval(()=>{
  if(!STATE.navigating||STATE.route.length===0) return;
  const d=dist(STATE.current,STATE.route[STATE.stepIndex]);
  if(d>CONFIG.rerouteDistance){
    speak("ルートを再検索します");
    searchRoute();
  }
},3000);

/* ==============================
   音声
============================== */
function speak(t){
  const u=new SpeechSynthesisUtterance(t);
  u.lang="ja-JP";
  speechSynthesis.speak(u);
}

/* ==============================
   夜間切替
============================== */
setInterval(()=>{
  const h=new Date().getHours();
  if(h>=18||h<=5){
    if(!map.hasLayer(tileNight)){
      map.removeLayer(tileDay);
      tileNight.addTo(map);
      document.body.classList.add("night");
    }
  }else{
    if(!map.hasLayer(tileDay)){
      map.removeLayer(tileNight);
      tileDay.addTo(map);
      document.body.classList.remove("night");
    }
  }
},60000);

/* ==============================
   擬似走行
============================== */
function startDummy(){
  let i=0;
  setInterval(()=>{
    if(!STATE.route[i]) return;
    STATE.current=STATE.route[i];
    marker.setLatLng(STATE.current);
    i++;
  },700);
}

/* ==============================
   補助
============================== */
function modeSel(){
  return document.getElementById("mode").value;
}
</script>
</body>
</html>
