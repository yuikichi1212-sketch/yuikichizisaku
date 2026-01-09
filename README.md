<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="utf-8">
<title>ゆいきちナビ PRO</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">

<style>
/* ===============================
   基本レイアウト
================================ */
html,body{
  margin:0;
  height:100%;
  overflow:hidden;
}
#map{
  position:absolute;
  inset:0;
  z-index:1;
}

/* ===============================
   Leaflet回転基盤
================================ */
.leaflet-container{
  transform-origin:50% 50% !important;
  will-change:transform;
}

/* ===============================
   Sidebar UI
================================ */
#sidebar{
  position:fixed;
  right:0;
  top:0;
  width:300px;
  height:100%;
  background:#0f172a;
  color:#fff;
  z-index:1000;
  display:flex;
  flex-direction:column;
}
#sidebar h2{
  margin:12px;
}
.sidebar-body{
  flex:1;
  overflow:auto;
  padding:12px;
}
.btn{
  width:100%;
  padding:12px;
  margin-bottom:8px;
  background:#1e90ff;
  border:none;
  color:#fff;
  font-size:15px;
}
.status{
  font-size:12px;
  opacity:.8;
}

/* ===============================
   HUD
================================ */
#hud{
  position:fixed;
  left:12px;
  bottom:12px;
  z-index:1001;
  background:rgba(0,0,0,.6);
  color:#fff;
  padding:8px 12px;
  border-radius:8px;
}
</style>
</head>

<body>

<div id="map"></div>

<div id="sidebar">
  <h2>ゆいきちナビ</h2>
  <div class="sidebar-body">
    <button class="btn" id="btnNav">ナビ開始</button>
    <button class="btn" id="btnRotate">回転 ON/OFF</button>
    <button class="btn" id="btnSave">ルート保存</button>
    <div class="status" id="status">待機中</div>
  </div>
</div>

<div id="hud">-- km/h</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
/* ===============================
   グローバル状態
================================ */
const S={
  nav:false,
  rotate:true,
  follow:true,
  speed:0,
  heading:0,
  filteredHeading:0,
  lastPos:null,
  routes:[],
};

/* ===============================
   Map 初期化
================================ */
const map=L.map('map',{zoomControl:false});
map.setView([35.1709,136.8815],16);

L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{
  attribution:'©OSM'
}).addTo(map);

const marker=L.circleMarker(map.getCenter(),{
  radius:8,
  color:'#1e90ff',
  fillColor:'#1e90ff',
  fillOpacity:1
}).addTo(map);

const mapEl=document.querySelector('.leaflet-container');

/* ===============================
   GPS フィルタ
================================ */
function lowPass(prev,next,a=0.15){
  return prev+(next-prev)*a;
}

/* ===============================
   メインナビ処理
================================ */
navigator.geolocation.watchPosition(pos=>{
  const c=pos.coords;
  if(!c.latitude) return;

  const lat=c.latitude;
  const lon=c.longitude;
  const heading=Number.isFinite(c.heading)?c.heading:S.heading;
  const speed=c.speed||0;

  S.speed=speed;
  S.heading=heading;
  S.filteredHeading=lowPass(S.filteredHeading,heading);

  if(S.follow){
    map.setView([lat,lon],map.getZoom(),{animate:false});
  }

  marker.setLatLng([lat,lon]);

  if(S.nav && S.rotate){
    mapEl.style.transform=`rotate(${-S.filteredHeading}deg)`;
  }else{
    mapEl.style.transform='';
  }

  hud.textContent=(speed*3.6).toFixed(1)+' km/h';
});

/* ===============================
   UIイベント
================================ */
btnNav.onclick=()=>{
  S.nav=!S.nav;
  btnNav.textContent=S.nav?'ナビ停止':'ナビ開始';
  status.textContent=S.nav?'ナビ中':'停止';
};

btnRotate.onclick=()=>{
  S.rotate=!S.rotate;
  status.textContent='回転 '+(S.rotate?'ON':'OFF');
};

btnSave.onclick=()=>{
  const saved=JSON.parse(localStorage.getItem('yk_routes')||'[]');
  saved.push({
    time:Date.now(),
    pos:marker.getLatLng(),
    zoom:map.getZoom()
  });
  localStorage.setItem('yk_routes',JSON.stringify(saved));
  status.textContent='ルート保存完了';
};
</script>
</body>
</html>
