<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>ゆいきちナビ</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@turf/turf@6/turf.min.js"></script>

<style>
/* ============================
   UI / LAYOUT（極限）
============================ */
html,body{margin:0;height:100%;overflow:hidden;font-family:system-ui}
#map{position:fixed;inset:0}
#topbar{
  position:fixed;top:0;left:0;right:0;height:52px;
  background:#111;color:#fff;z-index:2000;
  display:flex;align-items:center;padding:0 12px
}
#title{font-weight:900}
#menuBtn{margin-right:12px;font-size:22px;cursor:pointer}

#sidebar{
  position:fixed;top:52px;left:0;bottom:0;width:360px;
  background:#fff;z-index:1900;
  overflow:auto;padding:10px;
  transform:translateX(-100%);
  transition:.3s
}
#sidebar.open{transform:none}

#navHUD{
  position:fixed;bottom:0;left:0;right:0;
  background:rgba(255,255,255,.95);
  z-index:1800;padding:10px;
  display:none
}
#navMain{font-size:18px;font-weight:900}
#navSub{font-size:14px;opacity:.7}

.map-3d{
  transform:perspective(900px) rotateX(55deg) scale(1.2);
  transform-origin:center center;
}
</style>
</head>

<body>

<div id="topbar">
  <div id="menuBtn">≡</div>
  <div id="title">ゆいきちナビ</div>
</div>

<div id="sidebar">
  <h2>検索</h2>
  <input id="from" placeholder="出発地（現在地）">
  <input id="to" placeholder="目的地">
  <select id="mode">
    <option value="driving">車</option>
    <option value="cycling">自転車</option>
    <option value="walking">徒歩</option>
  </select>
  <button onclick="buildRoute()">ルート検索</button>
  <button onclick="startNav()">ナビ開始</button>
  <button onclick="stopNav()">停止</button>
</div>

<div id="map"></div>

<div id="navHUD">
  <div id="navMain">案内待機中</div>
  <div id="navSub">---</div>
</div>

<script>
/* =========================================================
   CONFIG（大量）
========================================================= */
const CONFIG={
  SNAP_DISTANCE:25,         // m
  REROUTE_DISTANCE:40,      // m
  STEP_TRIGGER:35,          // m
  FOLLOW_ZOOM:18,
  GPS_INTERVAL:1000
};

/* =========================================================
   STATE MACHINE（ナビの心臓）
========================================================= */
const NAV_STATE={
  IDLE:0,
  ROUTE_READY:1,
  NAVIGATING:2,
  REROUTING:3,
  ARRIVED:4
};

let App={
  state:NAV_STATE.IDLE,
  current:[35.681,139.767],
  goal:null,
  routeLine:null,
  routeCoords:[],
  steps:[],
  stepIndex:0,
  snappedIndex:0,
  navigating:false,
  lastVoice:"",
  marker:null
};

/* =========================================================
   MAP INIT
========================================================= */
const map=L.map("map",{zoomControl:false})
  .setView(App.current,16);

L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png")
  .addTo(map);

/* =========================================================
   UI
========================================================= */
menuBtn.onclick=()=>sidebar.classList.toggle("open");

/* =========================================================
   GEOLOCATION
========================================================= */
navigator.geolocation.watchPosition(p=>{
  App.current=[p.coords.latitude,p.coords.longitude];
  updateMarker();
  if(App.state===NAV_STATE.NAVIGATING){
    navTick();
  }
},{},{enableHighAccuracy:true});

/* =========================================================
   MARKER
========================================================= */
function updateMarker(){
  if(!App.marker){
    App.marker=L.marker(App.current).addTo(map);
  }
  App.marker.setLatLng(App.current);
}

/* =========================================================
   GEOCODE
========================================================= */
async function geocode(q){
  if(!q||q==="現在地"){
    return App.current;
  }
  const r=await fetch(
    "https://nominatim.openstreetmap.org/search?format=json&q="+
    encodeURIComponent(q)
  );
  const j=await r.json();
  if(!j[0]) throw "NOT FOUND";
  return [Number(j[0].lat),Number(j[0].lon)];
}

/* =========================================================
   ROUTE BUILD（OSRM）
========================================================= */
async function buildRoute(){
  App.goal=await geocode(to.value);
  const start=`${App.current[1]},${App.current[0]}`;
  const goal=`${App.goal[1]},${App.goal[0]}`;
  const profile=mode.value;

  const url=
    `https://router.project-osrm.org/route/v1/${profile}/${start};${goal}`+
    `?overview=full&steps=true&geometries=geojson`;

  const r=await fetch(url);
  const j=await r.json();

  App.routeCoords=j.routes[0].geometry.coordinates.map(c=>[c[1],c[0]]);
  App.steps=j.routes[0].legs[0].steps;
  App.stepIndex=0;

  if(App.routeLine) map.removeLayer(App.routeLine);
  App.routeLine=L.polyline(App.routeCoords,{color:"#1e90ff",weight:8})
    .addTo(map);

  App.state=NAV_STATE.ROUTE_READY;
  speak("ルートを作成しました");
}

/* =========================================================
   NAV START / STOP
========================================================= */
function startNav(){
  if(App.state!==NAV_STATE.ROUTE_READY) return;
  App.state=NAV_STATE.NAVIGATING;
  navHUD.style.display="block";
  map.getContainer().classList.add("map-3d");
  sidebar.classList.remove("open");
  speak("ナビを開始します");
}

function stopNav(){
  App.state=NAV_STATE.IDLE;
  navHUD.style.display="none";
  map.getContainer().classList.remove("map-3d");
  speak("ナビを終了します");
}

/* =========================================================
   NAV CORE LOOP（本体）
========================================================= */
function navTick(){
  snapToRoute();
  updateStep();
  checkReroute();
  updateHUD();
  followAndRotate();
}

/* =========================================================
   SNAP TO ROUTE（最近接点）
========================================================= */
function snapToRoute(){
  let min=Infinity,idx=0;
  App.routeCoords.forEach((p,i)=>{
    const d=L.latLng(App.current).distanceTo(p);
    if(d<min){min=d;idx=i}
  });
  App.snappedIndex=idx;
}

/* =========================================================
   STEP PROGRESSION
========================================================= */
function updateStep(){
  if(!App.steps[App.stepIndex]) return;
  const step=App.steps[App.stepIndex];
  const target=step.maneuver.location;
  const d=L.latLng(App.current)
    .distanceTo([target[1],target[0]]);

  if(d<CONFIG.STEP_TRIGGER){
    App.stepIndex++;
    voiceStep();
  }
}

/* =========================================================
   VOICE STEP（交差点音声）
========================================================= */
function voiceStep(){
  const step=App.steps[App.stepIndex];
  if(!step) return;
  const text=`${step.maneuver.type} ${step.name||""}`;
  if(text!==App.lastVoice){
    speak(text);
    App.lastVoice=text;
  }
}

/* =========================================================
   REROUTE
========================================================= */
function checkReroute(){
  const d=L.latLng(App.current)
    .distanceTo(App.routeCoords[App.snappedIndex]);
  if(d>CONFIG.REROUTE_DISTANCE){
    App.state=NAV_STATE.REROUTING;
    speak("ルートを再検索します");
    buildRoute();
    App.state=NAV_STATE.NAVIGATING;
  }
}

/* =========================================================
   HUD UPDATE
========================================================= */
function updateHUD(){
  const step=App.steps[App.stepIndex];
  if(step){
    navMain.textContent=step.maneuver.type;
    navSub.textContent=step.name||"";
  }
}

/* =========================================================
   FOLLOW & ROTATE
========================================================= */
function followAndRotate(){
  map.setView(App.current,CONFIG.FOLLOW_ZOOM,{animate:false});
  const next=App.routeCoords[App.snappedIndex+1];
  if(!next) return;
  const a=L.latLng(App.current);
  const b=L.latLng(next);
  const angle=Math.atan2(b.lng-a.lng,b.lat-a.lat)*180/Math.PI;
  map.getContainer().style.transform=
    `rotate(${-angle}deg)`;
}

/* =========================================================
   VOICE
========================================================= */
function speak(t){
  const u=new SpeechSynthesisUtterance(t);
  u.lang="ja-JP";
  speechSynthesis.speak(u);
}
</script>
</body>
</html>
