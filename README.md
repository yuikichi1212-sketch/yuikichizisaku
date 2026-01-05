<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>YUIKICHI NAVI PROFESSIONAL</title>
    
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

    <style>
        /* --- 全体レイアウト --- */
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #111; font-family: "Helvetica Neue", Arial, sans-serif; }
        
        #map-wrapper { position: relative; width: 100%; height: 100%; }
        #map { width: 100%; height: 100%; z-index: 1; }

        /* --- 収納メニュー (ドロワー) --- */
        #side-drawer {
            position: fixed; top: 0; left: -320px; width: 300px; height: 100%;
            background: #ffffff; z-index: 2000; transition: transform 0.4s cubic-bezier(0.16, 1, 0.3, 1);
            box-shadow: 5px 0 25px rgba(0,0,0,0.4); display: flex; flex-direction: column;
            border-right: 1px solid #ddd;
        }
        #side-drawer.is-open { transform: translateX(320px); }

        /* メニュー内要素 */
        .drawer-header { background: #000; color: #fff; padding: 20px; font-size: 18px; font-weight: bold; letter-spacing: 2px; }
        .drawer-content { padding: 20px; overflow-y: auto; flex-grow: 1; }

        /* 入力フォーム */
        .form-group { margin-bottom: 20px; }
        .form-group label { display: block; font-size: 10px; color: #999; text-transform: uppercase; margin-bottom: 5px; font-weight: bold; }
        .form-group input { width: 100%; padding: 12px; border: 1px solid #ccc; border-radius: 2px; box-sizing: border-box; font-size: 14px; background: #fdfdfd; }
        .current-loc-link { display: inline-block; margin-top: 5px; font-size: 11px; color: #0066cc; cursor: pointer; text-decoration: underline; }

        /* モード選択（車・自転車・歩） */
        .transport-modes { display: flex; border: 1px solid #000; border-radius: 2px; overflow: hidden; margin-bottom: 25px; }
        .mode-option { flex: 1; padding: 10px; text-align: center; font-size: 12px; cursor: pointer; background: #fff; transition: 0.2s; }
        .mode-option:not(:last-child) { border-right: 1px solid #000; }
        .mode-option.is-active { background: #000; color: #fff; }

        /* ナビ開始ボタン */
        .btn-primary { width: 100%; padding: 16px; background: #000; color: #fff; border: none; font-size: 15px; font-weight: bold; cursor: pointer; letter-spacing: 1px; }

        /* --- 地図上の操作系 --- */
        #menu-trigger { position: fixed; top: 20px; left: 20px; z-index: 2100; padding: 12px 20px; background: #000; color: #fff; border: none; font-size: 12px; font-weight: bold; cursor: pointer; box-shadow: 0 4px 10px rgba(0,0,0,0.3); }

        /* ナビゲーション表示パネル (下部) */
        #nav-guide-panel {
            position: fixed; bottom: 0; left: 0; right: 0; background: #fff; z-index: 1500;
            padding: 20px; transform: translateY(100%); transition: transform 0.4s;
            box-shadow: 0 -5px 20px rgba(0,0,0,0.2); display: flex; flex-direction: column; gap: 10px;
        }
        #nav-guide-panel.is-visible { transform: translateY(0); }
        .instruction-box { font-size: 20px; font-weight: bold; color: #000; line-height: 1.2; }
        .metrics-row { display: flex; justify-content: space-between; font-size: 14px; color: #666; border-top: 1px solid #eee; pt: 10px; }
        .btn-abort { background: #d00; color: #fff; border: none; padding: 8px 15px; font-size: 12px; border-radius: 2px; font-weight: bold; }

        /* 自車マーカー */
        .user-loc-arrow {
            width: 0; height: 0; border-left: 10px solid transparent; border-right: 10px solid transparent; border-bottom: 25px solid #0055ff;
            filter: drop-shadow(0 0 4px rgba(0,0,0,0.5));
        }
    </style>
</head>
<body>

    <button id="menu-trigger" onclick="toggleDrawer()">MENU</button>

    <div id="side-drawer">
        <div class="drawer-header">YUIKICHI NAVI</div>
        <div class="drawer-content">
            <div class="form-group">
                <label>Origin</label>
                <input type="text" id="origin-input" placeholder="Enter starting point">
                <span class="current-loc-link" onclick="setOriginToCurrent()">Use current location</span>
            </div>
            <div class="form-group">
                <label>Destination</label>
                <input type="text" id="destination-input" placeholder="Enter destination">
            </div>

            <label style="font-size: 10px; color: #999; font-weight: bold;">Transport Mode</label>
            <div class="transport-modes">
                <div class="mode-option is-active" id="m-car" onclick="updateMode('car')">CAR</div>
                <div class="mode-option" id="m-bike" onclick="updateMode('bicycle')">BIKE</div>
                <div class="mode-option" id="m-walk" onclick="updateMode('foot')">WALK</div>
            </div>

            <button class="btn-primary" onclick="initiateNavigation()">START NAVIGATION</button>
        </div>
    </div>

    <div id="map-wrapper">
        <div id="map"></div>
    </div>

    <div id="nav-guide-panel">
        <div id="current-instruction" class="instruction-box">Calculating...</div>
        <div class="metrics-row">
            <div>
                <span id="label-dist">0.0 km</span> | <span id="label-time">0 min</span>
            </div>
            <button class="btn-abort" onclick="terminateNavigation()">EXIT</button>
        </div>
    </div>

    <script>
        /** * GLOBAL VARIABLES
         */
        let map, userMarker, routeLine, navWatchId;
        let currentMode = 'car';
        let currentPos = null;
        let destinationPos = null;
        let routeData = null;

        /**
         * INITIALIZATION
         */
        function initMap() {
            map = L.map('map', { zoomControl: false }).setView([35.6812, 139.7671], 15);
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                attribution: '&copy; OpenStreetMap'
            }).addTo(map);

            // 初期位置取得
            map.locate({setView: true, maxZoom: 15});
            map.on('locationfound', (e) => { currentPos = e.latlng; updatePositionMarker(e); });
        }

        /**
         * UI CONTROLS
         */
        function toggleDrawer() {
            document.getElementById('side-drawer').classList.toggle('is-open');
        }

        function updateMode(mode) {
            currentMode = mode;
            document.querySelectorAll('.mode-option').forEach(el => el.classList.remove('is-active'));
            document.getElementById('m-' + (mode === 'bicycle' ? 'bike' : mode === 'foot' ? 'walk' : 'car')).classList.add('is-active');
        }

        function setOriginToCurrent() {
            if (currentPos) {
                document.getElementById('origin-input').value = "CURRENT_LOCATION";
            } else {
                alert("GPS not ready.");
            }
        }

        /**
         * CORE NAVIGATION LOGIC
         */
        async function initiateNavigation() {
            const startVal = document.getElementById('origin-input').value;
            const endVal = document.getElementById('destination-input').value;

            if (!startVal || !endVal) return alert("Please fill both fields.");

            // 1. Geocoding (地点の座標化)
            const startNode = (startVal === "CURRENT_LOCATION") ? currentPos : await fetchCoords(startVal);
            const endNode = await fetchCoords(endVal);

            if (!startNode || !endNode) return alert("Location not found.");

            destinationPos = endNode;
            toggleDrawer();

            // 2. 経路計算
            await requestRoute(startNode, endNode);

            // 3. ナビゲーション開始
            document.getElementById('nav-guide-panel').classList.add('is-visible');
            speakVoice("Starting navigation.");

            // 4. GPS追跡
            if (navWatchId) navigator.geolocation.clearWatch(navWatchId);
            navWatchId = navigator.geolocation.watchPosition(handleNavUpdate, handleNavError, {
                enableHighAccuracy: true
            });
        }

        async function fetchCoords(query) {
            try {
                const response = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(query)}`);
                const data = await response.json();
                return data.length > 0 ? { lat: parseFloat(data[0].lat), lng: parseFloat(data[0].lon) } : null;
            } catch (e) { return null; }
        }

        async function requestRoute(start, end) {
            const profile = currentMode; // car, bicycle, foot
            const url = `https://router.project-osrm.org/route/v1/${profile}/${start.lng},${start.lat};${end.lng},${end.lat}?overview=full&geometries=geojson&steps=true`;

            const res = await fetch(url);
            const data = await res.json();
            
            if (data.code !== 'Ok') return;

            routeData = data.routes[0];
            const coordinates = routeData.geometry.coordinates.map(c => [c[1], c[0]]);

            if (routeLine) map.removeLayer(routeLine);
            routeLine = L.polyline(coordinates, {color: '#0055ff', weight: 8, opacity: 0.7}).addTo(map);
            
            map.fitBounds(routeLine.getBounds());
            updateDashboard(routeData);
        }

        function handleNavUpdate(pos) {
            const latlng = L.latLng(pos.coords.latitude, pos.coords.longitude);
            currentPos = latlng;

            // マーカー更新
            updatePositionMarker({latlng: latlng, heading: pos.coords.heading});
            
            // 地図回転（ヘディングアップ）
            if (pos.coords.heading) {
                document.getElementById('map').style.transform = `rotate(${-pos.coords.heading}deg)`;
            }
            map.panTo(latlng);

            // リルート判定（経路から離れすぎた場合）
            checkReroute(latlng);
        }

        function updatePositionMarker(e) {
            if (!userMarker) {
                const arrow = L.divIcon({ className: 'user-loc-arrow', iconSize: [20, 25], iconAnchor: [10, 20] });
                userMarker = L.marker(e.latlng, {icon: arrow}).addTo(map);
            } else {
                userMarker.setLatLng(e.latlng);
            }
            if (e.heading) {
                const el = document.querySelector('.user-loc-arrow');
                if (el) el.style.transform = `rotate(${e.heading}deg)`;
            }
        }

        function updateDashboard(route) {
            const dist = (route.distance / 1000).toFixed(1);
            const mins = Math.round(route.duration / 60);
            document.getElementById('label-dist').innerText = `${dist} km`;
            document.getElementById('label-time').innerText = `${mins} min`;
            
            if (route.legs[0].steps.length > 0) {
                const nextStep = route.legs[0].steps[0].maneuver.instruction;
                document.getElementById('current-instruction').innerText = nextStep;
            }
        }

        function checkReroute(latlng) {
            if (!routeLine) return;
            const points = routeLine.getLatLngs();
            const distToPath = latlng.distanceTo(points[0]); // 簡易的に始点からの距離
            
            if (distToPath > 60) { // 60メートル以上離れた
                speakVoice("Rerouting.");
                requestRoute(latlng, destinationPos);
            }
        }

        function terminateNavigation() {
            if (navWatchId) navigator.geolocation.clearWatch(navWatchId);
            document.getElementById('nav-guide-panel').classList.remove('is-visible');
            if (routeLine) map.removeLayer(routeLine);
            speakVoice("Navigation terminated.");
        }

        function speakVoice(text) {
            window.speechSynthesis.cancel();
            const msg = new SpeechSynthesisUtterance(text);
            msg.lang = 'en-US'; // 日本語にする場合は 'ja-JP'
            window.speechSynthesis.speak(msg);
        }

        function handleNavError(err) { console.error("GPS Error: ", err); }

        window.onload = initMap;
    </script>
</body>
</html>
