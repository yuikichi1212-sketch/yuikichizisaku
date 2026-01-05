<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>ゆいきちナビ</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #1a1a1a; font-family: sans-serif; }
        
        /* 地図の回転用コンテナ */
        #map-container { width: 100%; height: 100%; transition: transform 0.1s linear; }
        #map { width: 100%; height: 100%; }

        /* 収納メニュー */
        #menu {
            position: fixed; top: 0; left: -320px; width: 300px; height: 100%;
            background: #fff; z-index: 2000; transition: 0.3s; padding: 20px;
            box-shadow: 2px 0 10px rgba(0,0,0,0.3); display: flex; flex-direction: column;
        }
        #menu.open { left: 0; }
        #toggle { position: fixed; top: 15px; left: 15px; z-index: 2100; padding: 10px 15px; background: #000; color: #fff; border: none; border-radius: 4px; cursor: pointer; }

        /* フォーム入力 */
        .field { margin-bottom: 15px; }
        label { display: block; font-size: 11px; color: #666; font-weight: bold; margin-bottom: 5px; }
        input { width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box; }
        .btn-current { font-size: 11px; background: #eee; border: none; padding: 5px; margin-top: 5px; width: 100%; cursor: pointer; }

        /* モード切替 */
        .modes { display: flex; gap: 5px; margin: 15px 0; }
        .mode-btn { flex: 1; padding: 10px; background: #eee; border: none; border-radius: 4px; font-size: 12px; cursor: pointer; }
        .mode-btn.active { background: #000; color: #fff; }

        /* ナビ情報 */
        #info-panel {
            position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%);
            width: 90%; background: #fff; z-index: 1000; border-radius: 8px;
            padding: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.3); display: none;
        }
        .info-row { display: flex; justify-content: space-between; align-items: center; }
        .main-inst { font-size: 18px; font-weight: bold; }

        .user-icon { width: 0; height: 0; border-left: 10px solid transparent; border-right: 10px solid transparent; border-bottom: 20px solid #007bff; }
    </style>
</head>
<body>

<button id="toggle" onclick="toggleMenu()">MENU</button>

<div id="menu">
    <h2 style="border-bottom: 2px solid #000; padding-bottom: 5px;">YUIKICHI NAVI</h2>
    
    <div class="field">
        <label>START</label>
        <input type="text" id="start-in" placeholder="出発地を入力">
        <button class="btn-current" onclick="setCurrentToStart()">現在地を出発地にする</button>
    </div>
    
    <div class="field">
        <label>DESTINATION</label>
        <input type="text" id="end-in" placeholder="目的地を入力">
    </div>

    <div class="modes">
        <button class="mode-btn active" id="car-btn" onclick="setMode('car')">車</button>
        <button class="mode-btn" id="bike-btn" onclick="setMode('bike')">自転車</button>
        <button class="mode-btn" id="walk-btn" onclick="setMode('walk')">徒歩</button>
    </div>

    <button style="width:100%; padding:15px; background:#000; color:#fff; border:none; font-weight:bold;" onclick="startNavigation()">ナビ開始</button>
    <button style="width:100%; padding:10px; background:#fff; color:#666; border:none; margin-top:10px;" onclick="toggleMenu()">閉じる</button>
</div>

<div id="map-container">
    <div id="map"></div>
</div>

<div id="info-panel">
    <div id="inst" class="main-inst">直進してください</div>
    <div class="info-row">
        <span id="dist-time">-- km / -- min</span>
        <button onclick="stopNavigation()" style="background:#cc0000; color:#fff; border:none; padding:5px 10px; border-radius:4px;">終了</button>
    </div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
    let map, userMarker, routeLine, watchId;
    let currentMode = 'car';
    let userLatLng = null;
    let targetLatLng = null;
    let lastRerouteTime = 0;

    // 初期化
    map = L.map('map', { zoomControl: false }).setView([35.681, 139.767], 14);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

    // メニュー開閉
    function toggleMenu() { document.getElementById('menu').classList.toggle('open'); }

    // 現在地取得
    function setCurrentToStart() {
        if (!userLatLng) {
            map.locate({setView: true});
            alert("位置情報を取得中です。少々お待ちください。");
        } else {
            document.getElementById('start-in').value = "現在地";
        }
    }

    function setMode(mode) {
        currentMode = mode;
        document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
        document.getElementById(mode + '-btn').classList.add('active');
    }

    async function getCoords(query) {
        if (query === "現在地" && userLatLng) return userLatLng;
        try {
            const res = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(query)}`);
            const data = await res.json();
            return data.length > 0 ? { lat: parseFloat(data[0].lat), lng: parseFloat(data[0].lon) } : null;
        } catch (e) { return null; }
    }

    async function startNavigation() {
        const startVal = document.getElementById('start-in').value;
        const endVal = document.getElementById('end-in').value;
        
        const start = await getCoords(startVal);
        const end = await getCoords(endVal);

        if (!start || !end) return alert("場所が見つかりませんでした。");

        targetLatLng = end;
        toggleMenu();
        document.getElementById('info-panel').style.display = 'block';
        
        speak("ナビゲーションを開始します。");
        calculateRoute(start, end);

        // GPS追跡
        if (watchId) navigator.geolocation.clearWatch(watchId);
        watchId = navigator.geolocation.watchPosition(updateNav, null, {enableHighAccuracy: true});
    }

    // 経路計算 (OSRM APIを使用 - 道路沿い)
    async function calculateRoute(s, e) {
        const profile = currentMode === 'walk' ? 'foot' : currentMode === 'bike' ? 'bicycle' : 'car';
        const url = `https://router.project-osrm.org/route/v1/${profile}/${s.lng},${s.lat};${e.lng},${e.lat}?overview=full&geometries=geojson&steps=true`;
        
        try {
            const res = await fetch(url);
            const data = await res.json();
            if (data.code !== 'Ok') return;

            const route = data.routes[0];
            const coords = route.geometry.coordinates.map(c => [c[1], c[0]]);

            if (routeLine) map.removeLayer(routeLine);
            routeLine = L.polyline(coords, {color: '#007bff', weight: 6}).addTo(map);

            const dist = (route.distance / 1000).toFixed(1);
            const time = Math.round(route.duration / 60);
            document.getElementById('dist-time').innerText = `${dist} km / ${time} min`;
            
            const nextStep = route.legs[0].steps[0].maneuver.instruction;
            document.getElementById('inst').innerText = nextStep;
        } catch (err) { console.error("Route Error"); }
    }

    function updateNav(pos) {
        const latlng = { lat: pos.coords.latitude, lng: pos.coords.longitude };
        userLatLng = latlng;

        // 自車位置
        if (!userMarker) {
            userMarker = L.marker(latlng, { icon: L.divIcon({className: 'user-icon'}) }).addTo(map);
        } else {
            userMarker.setLatLng(latlng);
        }

        // 地図回転 (Heading対応)
        if (pos.coords.heading) {
            document.getElementById('map-container').style.transform = `rotate(${-pos.coords.heading}deg)`;
            document.querySelector('.user-icon').style.transform = `rotate(${pos.coords.heading}deg)`;
        }

        map.panTo(latlng);

        // リルート判定 (経路から50m以上離れたら)
        if (routeLine && targetLatLng) {
            const distToPoint = map.distance(latlng, routeLine.getLatLngs()[0]); // 簡易判定
            const now = Date.now();
            if (distToPoint > 50 && (now - lastRerouteTime > 10000)) {
                lastRerouteTime = now;
                speak("ルートを外れました。リルートします。");
                calculateRoute(latlng, targetLatLng);
            }
        }
    }

    function stopNavigation() {
        if (watchId) navigator.geolocation.clearWatch(watchId);
        document.getElementById('info-panel').style.display = 'none';
        if (routeLine) map.removeLayer(routeLine);
        speak("ナビゲーションを終了しました。");
    }

    function speak(text) {
        window.speechSynthesis.cancel();
        const u = new SpeechSynthesisUtterance(text);
        u.lang = 'ja-JP';
        window.speechSynthesis.speak(u);
    }

    map.on('locationfound', e => { userLatLng = e.latlng; });
    map.locate({watch: true});

</script>
</body>
</html>
