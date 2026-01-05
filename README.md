<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>ゆいきちナビ</title>
    
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine@3.2.12/dist/leaflet-routing-machine.css" />

    <style>
        /* 全体デザイン：モノトーン */
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000; }
        
        /* 地図（全画面） */
        #map { position: absolute; top: 0; bottom: 0; width: 100%; z-index: 1; }

        /* 収納式メニュー（左からスライド） */
        #side-menu {
            position: fixed; top: 0; left: -300px; width: 280px; height: 100%;
            background: rgba(255, 255, 255, 0.98); z-index: 2000;
            box-shadow: 2px 0 10px rgba(0,0,0,0.5);
            transition: 0.3s; padding: 20px;
            display: flex; flex-direction: column; gap: 15px;
        }
        #side-menu.open { left: 0; }

        /* メニュー開閉ボタン */
        #menu-toggle {
            position: fixed; top: 15px; left: 15px; width: 45px; height: 45px;
            background: #fff; border: 1px solid #ccc; border-radius: 4px;
            z-index: 2100; cursor: pointer; display: flex; align-items: center; justify-content: center;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2); font-weight: bold;
        }

        /* フォーム要素 */
        .search-group { display: flex; flex-direction: column; gap: 10px; }
        label { font-size: 12px; color: #666; font-weight: bold; }
        input { padding: 12px; border: 1px solid #ccc; border-radius: 4px; font-size: 14px; background: #f9f9f9; }
        button { padding: 12px; background: #000; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; }
        
        /* ナビ情報（メニュー内蔵） */
        #nav-status {
            margin-top: 20px; padding-top: 20px; border-top: 1px solid #eee;
            display: none; font-size: 14px; line-height: 1.6;
        }
        .info-label { color: #888; margin-right: 10px; }
        
        /* 現在地ボタン（地図上に配置） */
        #locate-btn {
            position: fixed; bottom: 25px; right: 15px; width: 50px; height: 50px;
            background: #fff; border-radius: 50%; z-index: 1000;
            display: flex; align-items: center; justify-content: center;
            box-shadow: 0 2px 8px rgba(0,0,0,0.3); cursor: pointer; font-weight: bold;
        }

        /* 自車アイコン */
        .user-marker {
            width: 16px; height: 16px; background: #000; border: 2px solid #fff;
            border-radius: 50%; box-shadow: 0 0 5px rgba(0,0,0,0.5);
        }

        /* 道路ルートの調整 */
        .leaflet-routing-container { display: none !important; }
    </style>
</head>
<body>

    <div id="menu-toggle" onclick="toggleMenu()">MENU</div>

    <div id="side-menu">
        <div style="font-size: 18px; font-weight: bold; border-bottom: 2px solid #000; padding-bottom: 10px;">
            NAVIGATION
        </div>
        
        <div class="search-group">
            <label>START</label>
            <input type="text" id="start-input" placeholder="出発地を入力" />
            <label>DESTINATION</label>
            <input type="text" id="end-input" placeholder="目的地を入力" />
            <button onclick="searchAndRoute()">経路を検索する</button>
        </div>

        <div id="nav-status">
            <div><span class="info-label">DISTANCE:</span><span id="dist">--</span></div>
            <div><span class="info-label">TIME:</span><span id="time">--</span></div>
            <div style="margin-top: 10px; font-weight: bold;" id="guide">案内を開始してください</div>
        </div>
        
        <button style="margin-top: auto; background: #666;" onclick="toggleMenu()">メニューを閉じる</button>
    </div>

    <div id="map"></div>

    <div id="locate-btn" onclick="locateUser()">GPS</div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script src="https://unpkg.com/leaflet-routing-machine@3.2.12/dist/leaflet-routing-machine.js"></script>

    <script>
        let map, routingControl, userMarker;
        let isNavigating = false;

        // 地図初期化
        map = L.map('map', { zoomControl: false }).setView([35.6812, 139.7671], 15);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

        // メニュー開閉
        function toggleMenu() {
            document.getElementById('side-menu').classList.toggle('open');
        }

        // 住所検索とルート表示
        async function searchAndRoute() {
            const startStr = document.getElementById('start-input').value;
            const endStr = document.getElementById('end-input').value;

            if(!startStr || !endStr) return alert("入力が必要です");

            try {
                const start = await getPos(startStr);
                const end = await getPos(endStr);

                if(!start || !end) return alert("場所を特定できませんでした");

                if(routingControl) map.removeControl(routingControl);

                routingControl = L.Routing.control({
                    waypoints: [L.latLng(start.lat, start.lon), L.latLng(end.lat, end.lon)],
                    show: false,
                    createMarker: () => null,
                    lineOptions: { styles: [{color: '#000', weight: 5, opacity: 0.7}] },
                    language: 'ja'
                }).addTo(map);

                routingControl.on('routesfound', (e) => {
                    const r = e.routes[0];
                    document.getElementById('nav-status').style.display = 'block';
                    document.getElementById('dist').innerText = (r.summary.totalDistance / 1000).toFixed(1) + " km";
                    document.getElementById('time').innerText = Math.round(r.summary.totalTime / 60) + " min";
                    document.getElementById('guide').innerText = r.instructions[0].text;
                    
                    // 目的地へフィット
                    map.fitBounds(L.latLngBounds([start.lat, start.lon], [end.lat, end.lon]));
                    toggleMenu(); // 検索後は地図を見せるため閉じる
                });

            } catch(e) { alert("検索エラーが発生しました"); }
        }

        async function getPos(q) {
            const r = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(q)}`);
            const d = await r.json();
            return d[0];
        }

        // 現在地取得機能
        function locateUser() {
            map.locate({setView: true, maxZoom: 16});
        }

        map.on('locationfound', (e) => {
            if(!userMarker) {
                userMarker = L.marker(e.latlng, {
                    icon: L.divIcon({ className: 'user-marker', iconSize: [16,16] })
                }).addTo(map);
            } else {
                userMarker.setLatLng(e.latlng);
            }
        });

        map.on('locationerror', () => alert("GPS情報を取得できませんでした。設定を確認してください。"));

        // 自動回転（ヘディングアップ）の実装
        window.addEventListener('deviceorientationabsolute', (e) => {
            if(e.alpha !== null) {
                const compass = e.alpha; // 北に対する角度
                document.getElementById('map').style.transform = `rotate(${-compass}deg)`;
            }
        }, true);

    </script>
</body>
</html>
